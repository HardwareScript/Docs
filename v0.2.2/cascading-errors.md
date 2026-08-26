The cascading errors are caused by an issue in the parser called a **Token Stream Desynchronization** (or **Error Recovery Failure**). 

The issue is not just that the compiler collects multiple errors, but that when line 112 failed, the parser did not cleanly **synchronize to the next line or boundary**. Instead, it left unconsumed tokens in the stream or exited the indented block state, causing every subsequent line to be parsed from the wrong grammar rule (e.g., parsing a property as a top-level statement).

In an indentation-sensitive language like HardwareScript, this issue can be fixed natively without losing the ability to report multiple independent, genuine errors.

---

### The Anatomy of the Bug: What Happened Internally

```
Line 111: add pour(Polysilicon) named Shared_Gate_Poly on layer: poly:
Line 112:     device: M_NMOS.G, M_PMOS.G    <-- Fails with S99 on 'M_PMOS'
Line 113:     net: In
...
Line 164:     device: M_NMOS.S              <-- Fails with S14 (Unexpected ':')
```

1. The parser hit `M_PMOS` in `parse_device_binding()`, saw that it did not match `M_NMOS`, and returned `Err(ParseError::...)` using the `?` operator.
2. The `?` operator bubbled the error straight out of `parse_pour_properties()`.
3. Because it bailed immediately, the remaining tokens on line 112 (and line 113 `net: In`) and the closing `Dedent` token **were never consumed**.
4. The parser then tried to parse line 119 (`Gate_Input_Head`) and line 164 (`NMOS_Source_LI`) while thinking it was in an invalid indentation depth or expecting an expression instead of a property list.
5. Result: **1 real error turned into 7 ghost syntax errors.**

---

### How to Fix Cascading Natively (4 Architectural Steps)

To make error reporting clean (like `rustc` or `clang`), the parser needs **four native mechanisms**:

---

#### 1. Panic-Mode Line Resynchronization (Skip to `Newline`)

When a property parser (like `device:`, `align:`, `dimensions:`) fails, **do not bail out of the whole block with `?`**. Instead, catch the error, record it in a diagnostic accumulator, and consume tokens until the next `\n` or `Dedent`.

```rust
// Inside hwc-parser/src/parser/statements/pour.rs

impl<'a> Parser<'a> {
    pub fn parse_pour_body(&mut self) -> Result<PourProperties, ParseError> {
        let mut props = PourProperties::default();
        self.expect(&Token::Newline)?;
        self.expect(&Token::Indent)?;

        while !self.check(&Token::Dedent) && !self.is_at_end() {
            // Try to parse the next property line
            match self.parse_single_pour_property(&mut props) {
                Ok(()) => {
                    // Property parsed cleanly, consume trailing newline if present
                    let _ = self.match_token(&Token::Newline);
                }
                Err(err) => {
                    // 1. Record the actual error
                    self.diagnostics.push(err);

                    // 2. NATIVELY SYNCHRONIZE: Skip the rest of the broken line
                    self.synchronize_to_next_property();
                }
            }
        }

        self.expect(&Token::Dedent)?;
        Ok(props)
    }

    /// Resynchronize token stream to the start of the next property
    fn synchronize_to_next_property(&mut self) {
        while !self.is_at_end() {
            match self.peek() {
                // If we reach a newline, consume it and resume parsing next line
                Some(Token::Newline) => {
                    self.advance();
                    break;
                }
                // If the block ended, stop synchronizing so Dedent can be consumed
                Some(Token::Dedent) => break,
                // Skip broken tokens on the current line (e.g. M_PMOS.G)
                _ => { self.advance(); }
            }
        }
    }
}
```

---

#### 2. Top-Level Statement Synchronization Barrier

If an entire statement (e.g., an `add pour` or `route`) is fundamentally corrupted and cannot find its indentation block, the parser must jump to the next **Statement Keyword Barrier** (`add`, `route`, `let`, `for`, `space`, `module`):

```rust
impl<'a> Parser<'a> {
    pub fn synchronize_statement(&mut self) {
        while !self.is_at_end() {
            // If the next token is a known statement starter, stop discarding
            if let Some(token) = self.peek() {
                match token {
                    Token::Add
                    | Token::Route
                    | Token::Let
                    | Token::For
                    | Token::If
                    | Token::Space
                    | Token::Module
                    | Token::Device => return,
                    _ => { self.advance(); }
                }
            }
        }
    }
}
```

---

#### 3. Strict Pass Gates (Fail-Fast Between Phases)

A common cause of cascading errors in compilers is running later compiler phases when earlier phases failed. 

HardwareScript should enforce a strict phase gate: **If Pass 0 (Parsing) produces ANY errors, never run Semantic Lowering, Relational Resolving, or DRC.**

```rust
// In hwc-compiler/src/pipeline.rs

pub fn compile_source(source: &str) -> Result<HardwareSpace, Vec<Diagnostic>> {
    let mut parser = Parser::new(source);
    let ast = parser.parse_program();

    // ⛔ PHASE GATE 1: Lexer & Parser
    if parser.has_errors() {
        // Return ONLY the parser errors. Stop immediately.
        // Do NOT attempt symbol resolution or spatial lowering on a malformed AST.
        return Err(parser.take_diagnostics());
    }

    let mut compiler = SemanticCompiler::new(ast);
    compiler.resolve_symbols()?;

    // ⛔ PHASE GATE 2: Semantic Analysis
    if compiler.has_errors() {
        return Err(compiler.take_diagnostics());
    }

    compiler.synthesize_space()
}
```

---

#### 4. Error Throttling & `--fail-fast` Flag

To prevent terminal spam during development, add both a maximum error ceiling and an explicit `--fail-fast` mode to the CLI:

```rust
// CLI flag: hwc build --fail-fast
if config.fail_fast && !diagnostics.is_empty() {
    return Err(vec![diagnostics.remove(0)]); // Return ONLY the first error
}

// Default mode: Max 5 errors to prevent wall of text
if diagnostics.len() > 5 {
    diagnostics.truncate(5);
    eprintln!("⚠️ Showing first 5 errors (further errors truncated). Fix these and recompile.");
}
```

---

### What the Output Looks Like After Implementing This Fix

When you run `hwc build tests\CMOS\cmos_inverter.hw`, the compiler will catch line 112, skip the rest of that broken property, preserve the indentation balance, and exit after the parse phase:

```text
🔥 hwc COMPILER v0.2.1
[ 1.13ms] Source file read successfully (17524 bytes)
[ 4.78ms] Lexer complete (2383 tokens)

error[S99]: Multi-terminal binding must use same device name. Expected 'M_NMOS', found 'M_PMOS'
  --> tests\CMOS\cmos_inverter.hw:112:33
   │
111│ add pour(Polysilicon) named Shared_Gate_Poly on layer: poly:
112│     device: M_NMOS.G, M_PMOS.G
   │                       ^^^^^^^^ Multi-terminal binding must use same device name.
113│     net: In
   │
   = help: Split shared gate poly into separate device pours or use pure interconnect routing.

1 error found. Compilation aborted.
```

**Summary of the fix:**
1. Use `synchronize_to_next_property()` in `hwc-parser` instead of bubbling up errors with `?`.
2. Don't discard `Indent` / `Dedent` tokens when skipping malformed lines.
3. Stop compilation immediately after parsing if any syntax errors exist (Phase Gate).