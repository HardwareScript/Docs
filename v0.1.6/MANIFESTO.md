# Hardware Script v0.1.6: The Syntax Unification Manifesto

**Version**: 0.1.6  
**Status**: Language Design Authoritative Reference  
**Focus**: Syntax Unification & Universal Grammar  
**Supersedes**: All language specifications in v0.1.1 through v0.1.5

---

## 1. The Journey: From Fragmentation to Unity

The development of Hardware Script has been a journey of moving from **Physical Complexity** to **Logical Simplicity**.

### The Age of Fragmentation (Pre-v0.1.1)

Initially, Hardware Script followed the "Traditional EDA" mindset. We planned 10 different file extensions (`.hwmat` for materials, `.hwp` for profiles, `.hwx` for components, etc.). 

While this kept data separated, it created a **"Context Switching Nightmare."** Developers had to manage dozens of files just to blink an LED.

### The Great Consolidation (v0.1.2 - v0.1.4)

We made the bold decision to bring the entire ecosystem under one roof: the `.hw` extension. This was an immediate win for project management. 

However, because each block was implemented in a "silo," the language developed a **split personality**:
- `material` definitions felt like YAML
- `logic` blocks felt like C
- `space` assembly felt like a configuration script

### The God-Tier Engine (v0.1.5)

In v0.1.5, we ignored the language and focused entirely on the Engine. We achieved God-Tier performance:
- Magic Morton encoding
- O(1) bit-parallel DRC
- SDF Leap-Frog routing

**We built a Ferrari engine, but we left it with a steering wheel that was difficult for even an AI to turn.**

### The Syntax Crisis (The Catalyst for v0.1.6)

During v0.1.5 testing, we discovered that even highly capable AIs struggled to write valid Hardware Script. The language required **rote memorization of exceptions** rather than an understanding of **universal rules**.

**v0.1.6 is the response to that crisis.**

---

## 2. The Failures of v0.1.5: Why "Simple" was "Hard"

We identified seven structural flaws in the previous version that made the language feel like a "straitjacket" rather than a tool:

### 1. The "Define" Bureaucracy

Forcing `define component "Name":` made the language feel like a static configuration file. It added unnecessary boilerplate that hindered the flow of intent.

### 2. The Identifier Quote Trap

Requiring quotes around names (`"Copper"`, `"Resistor"`) was a major regression. In real programming languages, types and identifiers are bare; only data is quoted.

### 3. The Collection Schism

We had three different ways to write lists (commas, newlines, or dashes). The AI intuitively wanted to use brackets `[]`, but the parser rigidly refused them.

### 4. The Assignment Schism

Using `:` for hardware properties and `=` for logic created a mental wall. As we added math to layout blocks, this distinction became arbitrary and confusing.

### 5. Rigid Struct Panics

Blocks like `metadata` and `profile` acted like fixed code structs. If a user added a custom field, the compiler crashed. They should have been flexible dictionaries.

### 6. Soft Keyword Band-Aids

We had to write complex code to allow words like `tolerance` to be used as names because the lexer treated them as reserved commands.

### 7. The `origin: tl by t` Syntax

The coordinate system syntax uses the elegant `by` keyword for visual alignment with `dimensions` and `grid`. This creates perfect visual rhythm and is terse, professional, industry-standard shorthand.

**This stays unchanged in v0.1.6** - it's one of the best parts of Hardware Script's design.

---

## 3. The v0.1.6 Solution: Universal Grammar Rules

To solve these issues, v0.1.6 introduces the **"Type-as-Keyword" paradigm** and **three Universal Laws of Punctuation**.

### Rule 1: Drop the "Define" (Nouns as Keywords)

We are promoting hardware types to first-class language keywords. No more `define`. No more quotes.

```hw
# ❌ Old (v0.1.5)
define component "Resistor":

# ✅ New (v0.1.6)
component Resistor:
```

**Hardware types become first-class keywords**: `component`, `space`, `material`, `profile`, `module`, `enum`, `struct`

### Rule 2: The Universal Assignment Logic

We have unified when to use Colons and when to use Equals:

**The Colon (`:`) is for Hierarchy**  
Use it to open blocks or define structural relationships.

```hw
pins:
layout:
origin:
electrical:
```

**The Equals (`=`) is for Values**  
Use it to assign data, math, or state, whether you're in a material property or a logic wire.

```hw
# In properties
tolerance = 5%
resistance = 10kΩ

# In logic
let Sum = A + B
Count.next = Count + 1
```

### Rule 3: The Universal List (`[]`)

Any collection of items in Hardware Script now natively supports brackets.

```hw
# ✅ New (v0.1.6) - Unified list syntax
pins: [VCC, GND, Out]
match ports: [PA0, PA1]
allowed_dielectrics: [FR4, Rogers4350B]
```

---

## 4. Influences and Avoided Failures

### What Inspired the HWS v0.1.6 Syntax?

**Ruby**: The inspiration for prose-like readability and the use of prepositions (`at`, `to`, `by`) to make code read like English.

**F#**: The inspiration for Units of Measure. Physics is not a string; it is a first-class mathematical type.

**Rust**: The inspiration for the logic layer—`match` expressions, `enum` safety, and the "Immutable-by-Default" wire philosophy.

**SQL**: The inspiration for a declarative DSL that focuses on "Intent" (what I want) rather than "Procedure" (how to draw it).

### Failures We Are Actively Avoiding (The "Verilog Trap")

**The Blocking/Non-Blocking Nightmare**: We avoid Verilog's `=` vs `<=` mess by separating Wires (math) from Registers (memory). In HWS, a register update is always explicit via `.next`.

**The "Always" Block Chaos**: In Verilog, logic is scattered across `always @(posedge clk)` blocks. In HWS, all logic is unified in a single `logic:` block, which the compiler synthesizes into a clean dependency graph.

**The VHDL Verbosity**: We avoid VHDL's "Entity vs Architecture" boilerplate. In HWS, a module is a single, encapsulated unit of logic and structure.

---

## 5. The v0.1.6 Shift Summary

| Feature | v0.1.5 (Old) | v0.1.6 (New/Unified) |
|---------|--------------|----------------------|
| **Declaration** | `define component "CPU":` | `component CPU:` |
| **Punctuation** | Mixed (`:`, `,`, `-`) | Unified (`:` for blocks, `[]` for lists) |
| **Assignment** | Mixed (`:`, `=`) | Unified (`:` declarative, `=` behavioral) |
| **Comparison** | `==` | `=` (single equals) |
| **Identifiers** | String Literals (`" "`) | Bare Identifiers |
| **Flexibility** | Rigid Structs | Open Dictionaries (Metadata/Profiles) |
| **Keywords** | Soft keyword hacks | Property keys are identifiers |
| **Coordinate Origin** | `origin: tl by t` | `origin: tl by t` (unchanged) |
| **Struct Syntax** | `fields:` keyword | Bare bit-width table |
| **Register Primitive** | `Reg(...)` | `reg(...)` (lowercase) |
| **XOR Operator** | `^` | `xor` (word-only) |
| **Keywords** | Soft keyword hacks | Property keys are identifiers |
| **Coordinate Origin** | `origin: tl by t` | `origin: tl by t` (unchanged) |

---

## 6. Conclusion: Why This Route?

We chose this route because **Hardware Script must be a Language, not a File Format.**

By adopting the **Option 2 (Type-as-Keyword) paradigm**, we free the developer (and the AI) from the "YAML Straitjacket." We allow the logic layer and the physical layer to share the same grammar, making the transition between "writing code" and "placing atoms" seamless.

**Hardware Script v0.1.6 is the final step in the unification.**

We have a God-Tier Engine from v0.1.5; we now have a God-Tier Grammar to control it. We have created a system that is:
- As strict as a physics simulator
- As fast as a C compiler
- As readable as a Ruby script

---

## The Path Forward

**v0.1.6 is ready for implementation. The straitjacket is off. The language is free.**

### Implementation Priorities

1. **Lexer Simplification**: Remove soft keyword hacks, treat property keys as identifiers, add logic operators
2. **Parser Unification**: Implement universal list syntax `[]` across all blocks
3. **Assignment Operator Consolidation**: Use `:` for declarative, `=` for behavioral (including comparison)
4. **Type-as-Keyword**: Drop `define`, promote hardware types to keywords
5. **Flexible Dictionaries**: Allow arbitrary key-value pairs in `metadata` and `profile` blocks
6. **Struct Simplification**: Remove `fields:` keyword, parse as bare bit-width tables
7. **Register Primitive**: Change `Reg` to lowercase `reg`
8. **Logic Operators**: Implement `and` (`&`), `or` (`|`), `not` (`!`), `xor` (word-only)

### Migration Strategy

All v0.1.5 code will be automatically migrated using a syntax transformer. No manual intervention required.

### Documentation Updates

- Complete language specification with unified grammar
- Migration guide with before/after examples
- Updated standard library examples
- Compiler internals documentation

---

**This is the definitive architectural manifesto for Hardware Script v0.1.6.**

It documents the transition from an experimental, fragmented syntax into a professional, unified programming language.
