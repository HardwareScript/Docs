# Hardware Script v0.1.5 - Error Handling Philosophy

**Base Documentation**: [v0.1.4 ERROR-HANDLING-PHILOSOPHY.md](../v0.1.4/ERROR-HANDLING-PHILOSOPHY.md)  
**Status**: Incremental update - Logic error refinements  
**Version**: 0.1.5

---

## What's New in v0.1.5

This document covers ONLY the updates to error handling in v0.1.5. For the complete error handling philosophy (error code system, subsystems S/L/C/R/P/M), see [v0.1.4 ERROR-HANDLING-PHILOSOPHY.md](../v0.1.4/ERROR-HANDLING-PHILOSOPHY.md).

**New Features**:
1. L10 Warning (Implicit Truncation)
2. Semantic Pass Ordering (prevents error masking)
3. Enhanced miette diagnostics with Span extraction
4. Inline Comment Robustness (lexer-level skipping)
5. Scientific Notation Support (unitless floats)
6. Soft Keyword Support (keywords as identifiers)
7. Positive Number Prefix Support (explicit + signs)
8. Keywords in Metadata Blocks
9. Doc Comments in Inline Pin Lists
10. Array Pins in pin_positions

---

## Compiler Robustness (v0.1.5)

Six compiler gaps were discovered during stdlib stress testing and fixed in v0.1.5. These fixes ensure the compiler handles edge cases correctly without breaking the lean architecture.

### Gap #1: Scientific Notation Without Units

**Fixed**: Unitless scientific notation now works correctly (Float token priority increased from 2 to 12).

**Example**:
```hw
electrical:
    q_factor: 1.23456789e1  # ✅ Works (12.3456789)
    gain: 2.5e3             # ✅ Works (2500)
```

### Gap #2: Soft Keywords

**Fixed**: Keywords can now be used as parameter names, property names, and variable references.

**Example**:
```hw
define component "Resistor" (tolerance: Measurement):
    electrical:
        tolerance: tolerance  # ✅ Works
```

### Gap #3: Positive Number Prefixes

**Fixed**: Explicit positive prefixes (`+`) now work on all number types.

**Example**:
```hw
electrical:
    tolerance_plus: +80%      # ✅ Works
    voltage_rating: +10V      # ✅ Works
```

### Gap #4: Keywords in Metadata Blocks

**Fixed**: Keywords can now be used as field names in `metadata:` blocks.

**Example**:
```hw
metadata:
    trace: "0.1mm"      # ✅ Works
    via: "0.2mm"        # ✅ Works
    clearance: "0.3mm"  # ✅ Works
```

### Gap #5: Doc Comments in Inline Pin Lists

**Fixed**: Doc comments after the last pin in inline lists now work correctly.

**Example**:
```hw
pins:
    Data0, Data1,      ## Data bus pins
    Ground0, Ground1   ## Ground pins  # ✅ Works
```

### Gap #6: Array Pins in pin_positions

**Fixed**: Array syntax can now be used in `pin_positions:` and `pad_shapes:` blocks.

**Example**:
```hw
pins:
    Data[32]

layout:
    pin_positions:
        Data[0] at [1mm, 1mm]   # ✅ Works
        Data[31] at [9mm, 9mm]  # ✅ Works
    pad_shapes:
        Data[0]: Circle(0.2mm)  # ✅ Works
```

**See**: [COMPILER-INTERNALS.md](./COMPILER-INTERNALS.md) for technical details.

---

## L10 Warning: Implicit Truncation (NEW in v0.1.5)

**What it is**: A warning (not an error) when assigning a wider value to a narrower wire

**Why it matters**: Modern HDL developers expect truncation to be allowed with a warning, not a hard error

### The Problem

**v0.1.4 Behavior**: Implicit truncation was a hard error (L02: Width Mismatch)

```hw
logic:
    let Sum[9] = A[8] + B[8]     # OK: 9-bit result
    let Result[8] = Sum          # ERROR L02: Cannot assign 9-bit to 8-bit
```

**User Feedback**: "This is too strict. Verilog and VHDL allow truncation with a warning."

### The Solution

**v0.1.5 Behavior**: Implicit truncation generates a warning (L10), not an error

```hw
logic:
    let Sum[9] = A[8] + B[8]     # OK: 9-bit result
    let Result[8] = Sum          # WARNING L10: Implicit truncation (9 bits → 8 bits)
```

**Output**:
```
⚠️  Warning[L10]: Implicit truncation: Assigning 9-bit value to 8-bit wire 'Result'
   ╭─[alu.hw:22:1]
 21 │     logic:
 22 │         let Result[8] = Sum
   ·             ───┬───      ─┬─
   ·                │          ╰── 9-bit value (from A[8] + B[8])
   ·                ╰── 8-bit wire declared here
 23 │ 
   ╰────
  help: The source expression is 9 bits wide, but the destination wire 
        is only 8 bits wide. The most significant bit will be discarded.
        
  hint: To silence this warning, use explicit slicing: Sum[7..0]
        Or extend the destination: let Result[9] = Sum
```

### When L10 is Issued

**Allowed Truncations** (generates L10 warning):
- `let Result[8] = Sum[9]` - Explicit width mismatch
- `let Result[4] = A[8] + B[8]` - Expression result wider than destination
- `let Byte[8] = Word[16]` - Assigning wider wire to narrower wire

**Still Errors** (generates L02 error):
- `let Result = Sum` where widths are incompatible and cannot be inferred
- `let Result[9] = Sum[8]` - Trying to extend without explicit padding

### Compiler Behavior

**Default**: Warnings are printed but compilation continues

**Strict Mode** (future): `hwc build --deny-warnings` treats warnings as errors

---

## Semantic Pass Ordering (Error Masking Prevention)

**What it is**: Strict ordering of semantic validation passes to prevent cascading errors

**Why it matters**: Prevents "error masking" where a secondary error hides the root cause

### The Problem

**Before v0.1.5**: All semantic checks ran simultaneously

```hw
logic:
    let Sum = A + B     # A is undefined
    let Result[4] = Sum # Sum is 9 bits (if A and B were 8 bits)
```

**Old Behavior**: Compiler throws BOTH errors:
- L01: Unbound wire 'A'
- L02: Width mismatch (9 bits → 4 bits)

**Problem**: The width error is meaningless because A doesn't exist. This confuses users.

### The Solution

**v0.1.5**: Four-pass semantic validation in strict order

**Pass 1: Dependency Analysis**
- Build dependency graph of all wires and expressions
- Detect combinational loops (L03 error)
- Verify registers break cycles
- **Stop if errors found**

**Pass 2: Name Resolution**
- Resolve all variable references
- Check for undefined wires (L01 error)
- Validate enum variants and struct fields (L07, L08 errors)
- **Stop if errors found**

**Pass 3: Width Inference**
- Infer bit-widths bottom-up from operands
- Detect width mismatches (L02 error)
- Generate warnings for implicit truncation (L10 warning)
- **Stop if errors found**

**Pass 4: Hardware Generation**
- Map expressions to Standard Library components
- Generate component placements
- Generate nets between components

### Example: Error Masking Prevented

**Code**:
```hw
logic:
    let feedback = adder_out + 1
    let adder_out = feedback + input
    let result[4] = feedback
```

**Old Behavior** (all errors at once):
- L03: Combinational loop (feedback → adder_out → feedback)
- L01: Unbound wire 'input'
- L02: Width mismatch (unknown width → 4 bits)

**New Behavior** (Pass 1 catches loop first):
```
❌ Error[L03]: Combinational loop detected: feedback → adder_out → feedback
   ╭─[fsm.hw:18:1]
 17 │     logic:
 18 │         let feedback = adder_out + 1
   ·             ────┬────
   ·                 ╰── Wire 'feedback' depends on itself
 19 │         let adder_out = feedback + input
   ·             ─────┬─────
   ·                  ╰── Creates a loop back to 'feedback'
 20 │ 
   ╰────
  help: Insert a register to break the loop:
        let feedback = Reg(clock: Clk, reset: Rst, init: 0)
        feedback.next = adder_out + 1
```

**Result**: User fixes the loop first, then sees the other errors. No confusion.

---

## Enhanced miette Diagnostics with Span Extraction

**What it is**: Error messages now point to the exact character where the error occurred

**Why it matters**: Makes errors instantly debuggable (like Rust's borrow checker)

### The Implementation

**Before v0.1.5**: Errors pointed to entire lines

```
❌ Error[L01]: Unbound wire 'result'
   Line 22: let output = result + 1
```

**After v0.1.5**: Errors extract Span from AST and draw arrows

```
❌ Error[L01]: Unbound wire 'result' in logic block
   ╭─[alu.hw:22:9]
 21 │     logic:
 22 │         let output = result + 1
   ·                      ───┬───
   ·                         ╰── Wire 'result' not found
 23 │ 
   ╰────
  Available wires: A, B, CarryIn, Sum, CarryOut
  
  help: Declare 'result' as a pin in the module definition or as an 
        internal wire with 'let result = ...'
```

### Span Extraction

**AST Nodes**: Every AST node now carries a `Span` (start byte, end byte)

```rust
pub struct LogicExpression {
    pub kind: ExpressionKind,
    pub span: Span,  // <-- NEW in v0.1.5
}
```

**Error Construction**: Errors extract the span and pass it to miette

```rust
use miette::Diagnostic;

#[derive(Error, Debug, Diagnostic)]
#[error("Unbound wire '{name}' in logic block")]
#[diagnostic(
    code(L01),
    url("https://docs.hw-script.org/errors/L01"),
    help("Declare '{name}' as a pin or use 'let {name} = ...'")
)]
pub struct UnboundWireError {
    #[source_code]
    src: String,
    
    #[label("Wire '{name}' not found")]
    span: SourceSpan,  // <-- Extracted from AST
    
    name: String,
}
```

**Result**: Beautiful, terminal-friendly error messages with exact character positions

---

## Inline Comment Robustness (v0.1.5)

**What it is**: Comments are now completely invisible to the parser, using the "Trash Can" pattern from Rust and Elixir

**Why it matters**: Inline comments can now appear anywhere without causing parser errors

### The Problem

**Before v0.1.5**: The lexer created `Token::Comment` tokens that the parser had to explicitly skip in every parsing function.

```hw
metadata:
    manufacturer: "Test Corp"  # This inline comment broke the parser
    package: "0805"            # Parser never reached this line
```

**Issue**: Hardware Script uses significant indentation (like Python), so `Token::Newline` is critical. Creating `Token::Comment` tokens meant the parser had to manually skip them everywhere, violating DRY principles and causing bugs.

### The Solution

**v0.1.5**: Comments are completely skipped at the lexer level using `logos::skip`

**Lexer Implementation**:
```rust
// BEFORE (BROKEN) - created Token::Comment that parser had to handle:
#[regex(r"#[^#\[\n][^\n]*", priority = 0, callback = |lex| Some(lex.slice()[1..].trim().to_string()))]
Comment(String),

// AFTER (FIXED) - completely skipped at lexer level:
#[regex(r"#[^#\[\n][^\n]*", logos::skip)]
```

**Key Insight**: Match `#` followed by anything that isn't `#`, `[`, or newline, then SKIP IT. The newline is left untouched for `Token::Newline` to catch.

### Benefits

1. **Parser Simplification**: Removed all `Token::Comment` references from 6 parser files
2. **Bulletproof Comments**: Comments work everywhere without special handling
3. **Rust/Elixir Pattern**: Comments are truly invisible, just like in mature languages
4. **No Performance Cost**: Skipping at lexer level is faster than creating and discarding tokens

### Files Modified

- `hwc/crates/hwc-parser/src/lexer/token.rs` - Changed comment regex to use `logos::skip`
- `hwc/crates/hwc-parser/src/parser/definitions/component.rs` - Removed `Token::Comment` checks
- `hwc/crates/hwc-parser/src/parser/helpers.rs` - Removed `Token::Comment` checks
- `hwc/crates/hwc-parser/src/parser/mod.rs` - Removed `Token::Comment` checks
- `hwc/crates/hwc-parser/src/parser/definitions/module.rs` - Removed `Token::Comment` checks
- `hwc/crates/hwc-parser/src/parser/definitions/space.rs` - Removed `Token::Comment` checks

---

## Scientific Notation Support (v0.1.5)

**What it is**: Unitless scientific notation now works correctly for Q-factors, ratios, and other dimensionless values

**Why it matters**: Hardware components often have unitless properties that require scientific notation for precision

### The Problem

**Before v0.1.5**: Unitless scientific notation caused lexer errors

```hw
electrical:
    q_factor: 1.23456789e1  # ❌ Error S11: Invalid character
```

**Root Cause**: Float token had `priority = 2`, lower than Measurement token's `priority = 10`. The lexer tried to parse as a measurement first, failed (no unit), and rejected the token.

### The Solution

**v0.1.5**: Increased Float token priority to 12 (higher than Measurement)

```rust
// hwc/crates/hwc-parser/src/lexer/token.rs
#[regex(r"\d+\.\d+([eE][+-]?\d+)?", priority = 12, callback = |lex| lex.slice().parse::<f64>().ok())]
#[regex(r"\d+[eE][+-]?\d+", priority = 12, callback = |lex| lex.slice().parse::<f64>().ok())]
Float(f64),
```

**Now Works**:
```hw
electrical:
    q_factor: 1.23456789e1        # ✅ 12.3456789 (unitless)
    gain: 2.5e3                   # ✅ 2500 (voltage gain)
    ratio: 1.5e-2                 # ✅ 0.015 (percentage as decimal)
    test_positive: 1.5e10         # ✅ 15,000,000,000
    test_negative: 2.5e-6         # ✅ 0.0000025
```

### Impact

**Stress Test Results**: 21/21 passive components compile successfully, including components with:
- Extreme scientific notation (1.23456789e-15, 1.23456789e1, 1e10)
- Unitless Q-factors and ratios
- Mixed number formats in the same file

---

## Soft Keyword Support (v0.1.5)

**What it is**: Keywords can now be used as parameter names, property names, and variable references

**Why it matters**: Natural property names like `tolerance`, `trace`, `via` are also language keywords

### The Problem

**Before v0.1.5**: Keywords couldn't be used as identifiers

```hw
# ❌ Error S14: Unexpected the 'tolerance' keyword
define component "Resistor" (resistance: Measurement, tolerance: Measurement):
    electrical:
        tolerance: tolerance  # Error: keyword as value
```

**Root Cause**: The lexer correctly tokenizes `tolerance` as `Token::Tolerance`, but the parser only accepted `Token::Identifier` in parameter and value contexts.

### The Solution

**v0.1.5**: Parser now accepts keywords in identifier contexts using `expect_identifier_or_keyword()`

**Implementation** (hwc/crates/hwc-parser/src/parser/helpers.rs):
```rust
pub(super) fn expect_identifier_or_keyword(&mut self) -> Result<String, ParseError> {
    if let Some(current) = self.current() {
        match &current.token {
            Token::Identifier(name) => {
                let result = name.clone();
                self.advance();
                Ok(result)
            }
            Token::Tolerance => { self.advance(); Ok("tolerance".to_string()) }
            Token::Category => { self.advance(); Ok("category".to_string()) }
            // ... (all other keywords)
            _ => Err(ParseError::UnexpectedToken { ... })
        }
    } else {
        Err(ParseError::UnexpectedEof { ... })
    }
}
```

**Now Works**:
```hw
# ✅ Keywords as parameter names
define component "Resistor" (resistance: Measurement, tolerance: Measurement):
    electrical:
        resistance: resistance
        tolerance: tolerance  # Works!

# ✅ Keywords as property names
electrical:
    tolerance: 5%
    trace: 0.2mm
    via: 0.3mm
    clearance: 0.5mm
```

### Architecture Insight

This is the classic "soft keyword" problem solved the same way as Rust, C#, and TypeScript:
- **Lexer stays dumb**: Always outputs keywords, never guesses context
- **Parser stays smart**: Accepts keywords in specific contexts where identifiers are expected

### Impact

**Stress Test Results**: All 21 passive components compile successfully, including components that use keywords as:
- Parameter names (`tolerance: Measurement`)
- Property names (`tolerance: 5%`)
- Variable references (`tolerance: tolerance`)

**Files Modified**:
- `hwc/crates/hwc-parser/src/parser/helpers.rs` - Added `expect_identifier_or_keyword()`
- `hwc/crates/hwc-parser/src/parser/definitions/component.rs` - Use new helper for parameters and values

---

## Updated Logic Error Registry (v0.1.5)

### L - Logic Errors (L01-L39)

| Code | Description | Type |
|------|-------------|------|
| L01  | **Unbound wire** - Variable not declared | Error |
| L02  | **Width mismatch** - Bit width incompatibility | Error |
| L03  | **Combinational loop** - Feedback without register | Error |
| L04  | **Clock domain crossing** - Missing synchronizer | Error |
| L05  | **Multiple drivers** - Wire assigned multiple times | Error |
| L06  | **Uninitialized register** - Register without init | Error |
| L07  | **Invalid enum variant** - Unknown enum value | Error |
| L08  | **Struct field mismatch** - Wrong field name | Error |
| L09  | **Type mismatch** - Incompatible types | Error |
| L10  | **Implicit truncation** - Assigning wider value to narrower wire | Warning |
| L11  | Invalid operand type | Error |
| L12  | Division by zero | Error |
| L20  | Unreachable code | Warning |
| L21  | Missing match arm | Error |
| L22  | Duplicate match arm | Error |
| L30  | **Missing clock signal** - Register needs clock | Error |
| L31  | **Missing reset signal** - Register needs reset | Error |
| L32  | Invalid clock edge | Error |

**Key Change**: L10 is now a Warning (not an Error)

---

## Implementation in Rust (v0.1.5)

### L10 Warning Implementation

```rust
use miette::Diagnostic;
use thiserror::Error;

#[derive(Error, Debug, Diagnostic)]
#[error("Implicit truncation: Assigning {src_width}-bit value to {dst_width}-bit wire '{name}'")]
#[diagnostic(
    code(L10),
    severity(Warning),  // <-- This makes it a warning
    url("https://docs.hw-script.org/errors/L10"),
    help("Use explicit slicing to silence this warning: {name}[{dst_width_minus_1}..0]")
)]
pub struct ImplicitTruncationWarning {
    #[source_code]
    src: String,
    
    #[label("{src_width}-bit value")]
    src_span: SourceSpan,
    
    #[label("{dst_width}-bit wire declared here")]
    dst_span: SourceSpan,
    
    name: String,
    src_width: usize,
    dst_width: usize,
}
```

### Pass Ordering Implementation

```rust
// hwc/crates/hwc-compiler/src/logic_synthesizer/mod.rs

pub fn synthesize_logic_block(
    &mut self,
    logic_block: &LogicBlock,
    module_pins: &[(String, Option<usize>)],
) -> Result<(Vec<PlacedComponent>, Vec<NetRoute>, Vec<Warning>), SynthesisError> {
    // Pass 1: Dependency Analysis (detect combinational loops)
    self.analyze_dependencies(logic_block)?;
    
    // Pass 2: Name Resolution (detect unbound wires)
    self.resolve_names(logic_block, module_pins)?;
    
    // Pass 3: Width Inference (detect width mismatches, generate L10 warnings)
    let warnings = self.infer_widths(logic_block)?;
    
    // Pass 4: Hardware Generation (map to components)
    let (components, nets) = self.generate_hardware(logic_block)?;
    
    Ok((components, nets, warnings))
}
```

---

## Key Takeaways (v0.1.5)

1. **L10 Warning is developer-friendly** - Truncation is allowed with a warning, matching modern HDL expectations

2. **Pass ordering prevents confusion** - Users see root causes first, not cascading errors

3. **miette diagnostics are beautiful** - Exact character positions with arrows and context

4. **Warnings don't block compilation** - Users can iterate quickly without fixing every warning

5. **Strict mode available** - `--deny-warnings` flag for CI/CD pipelines

6. **Same quality everywhere** - Logic errors = Physics errors = Rust borrow checker quality

---

## References

**Implementation Files**:
- `hwc/crates/hwc-compiler/src/logic_synthesizer/error.rs` - Logic error definitions
- `hwc/crates/hwc-compiler/src/logic_synthesizer/dependency_graph.rs` - Pass 1 (loops)
- `hwc/crates/hwc-compiler/src/logic_synthesizer/statement.rs` - Pass 2 (names)
- `hwc/crates/hwc-compiler/src/logic_synthesizer/expression.rs` - Pass 3 (widths)
- `hwc/crates/hwc-compiler/src/logic_synthesizer/mod.rs` - Pass 4 (hardware)

**Related Documentation**:
- [v0.1.4 ERROR-HANDLING-PHILOSOPHY.md](../v0.1.4/ERROR-HANDLING-PHILOSOPHY.md) - Base error handling
- [v0.1.5 COMPILER-INTERNALS.md](./COMPILER-INTERNALS.md) - Logic synthesis architecture
- [v0.1.5 LANGUAGE-SPEC.md](./LANGUAGE-SPEC.md) - Logic block syntax

---

**Document Status**: Developer Experience Guide  
**Last Updated**: April 1, 2026  
**Part of**: Hardware Script v0.1.5 Documentation Suite
