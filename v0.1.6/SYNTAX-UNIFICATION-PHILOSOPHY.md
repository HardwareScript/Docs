# Syntax Unification Philosophy (v0.1.6)

**Base Documentation**: [v0.1.5 LANGUAGE-SPEC.md](../v0.1.5/LANGUAGE-SPEC.md)  
**Status**: Major syntax overhaul - Unification of four mini-languages  
**Version**: 0.1.6

---

## The Problem: Four Languages in a Trench Coat

Hardware Script v0.1.5 achieved a massive architectural win by consolidating the 10-file ecosystem (`.hwmat`, `.hwp`, `.hwx`, etc.) into a single unified `.hw` extension. This eliminated ecosystem fragmentation.

However, bringing everything under one roof exposed a critical flaw: **Hardware Script v0.1.5 is not one language. It is secretly four different mini-languages wearing a trench coat.**

Because the language evolved organically, the parser was written in "silos." The syntax rules for materials, profiles, components, and modules were all designed independently. As a result, the language violates the **Principle of Least Surprise**.

If an AI or human learns how to do something in one block, they naturally assume it works the same way in another block—but the v0.1.5 compiler punishes them for it.

---

## The Seven Deadly Syntax Inconsistencies

### 1. The Identifier Stringification Trap (The Quote Problem)

**The Flaw**: The language forces users to use string literals (`" "`) for defining top-level names, but bare identifiers everywhere else.

```hw
# ❌ v0.1.5 - Inconsistent
define component "Resistor":     # Requires quotes
    pins: A, B

add Resistor named R1            # No quotes
```

**Why it fails**: In modern programming (Rust, Python, Ruby), quotes denote data (strings), not types. If `Resistor` is a type you can instantiate, it should be a bare identifier.

**The AI naturally wrote**:
```hw
define component BusConnector:   # Compiler crashed
```

### 2. The Collection Chaos (Arrays vs. Lists)

**The Flaw**: Hardware Script has completely disjointed rules for defining lists depending on what block you're in.

```hw
# ❌ v0.1.5 - Three different list syntaxes
pins:
    A, B, C                      # Comma-separated or newlines

allowed_dielectrics: [FR4, Air]  # YAML-style brackets

path:
    - [1, 5, 5]                  # YAML-style dashes
```

**Why it fails**: The AI intuitively wrote `pins: [D0, D1]` because brackets are the universal computer science standard for "a list of things." The parser rigidly rejected it.

### 3. The Assignment Schism (: vs =)

**The Flaw**: The language draws an arbitrary, confusing line between "Declarative Hardware" and "Procedural Logic."

```hw
# ❌ v0.1.5 - Two assignment operators
electrical:
    tolerance: 5%                # YAML-style colon

logic:
    Count = count + 1            # Rust-style equals
```

**Why it fails**: As the language grows more advanced (like comptime math `x: 50% + 5mm`), the line between a "static property" and an "expression" blurs. The transition feels disjointed.

### 4. The Parameter Ambiguity (Positional vs. Keyword)

**The Flaw**: Instantiating components has multiple, conflicting syntaxes.

```hw
# ❌ v0.1.5 - Magic positional arguments
add Battery (5V) named Power                    # Compiler magically knows 5V is voltage

add Resistor_0805 (val: 10kΩ, tol: 1%) named R1 # Explicit keyword arguments
```

**Why it fails**: This violates the "Explicitness Over Brevity" rule. Allowing `(5V)` is a holdover from early C-style thinking. If a component takes parameters, it should always use keyword arguments for self-documenting code.

### 5. Rigid Structs vs. Flexible Dictionaries

**The Flaw**: Blocks like `metadata:` and `profile:` act like strict Rust structs when users expect them to act like flexible JSON/YAML dictionaries.

```hw
# ❌ v0.1.5 - Compiler crashes on unknown keys
metadata:
    manufacturer: "TDK"
    process: "GenericPCB"        # Error: Unknown profile field
```

**Why it fails**: If the goal of `metadata` is to allow factories and humans to attach arbitrary string data, the parser shouldn't crash on unrecognized keys. It should accept any `key: "value"` pair.

### 6. The "Soft Keyword" Band-Aid

**The Flaw**: v0.1.5 required a complex `expect_identifier_or_keyword()` helper just so users could type `tolerance: 5%`.

**Why it fails**: This means the lexer treats `tolerance`, `trace`, and `via` as reserved language keywords. But they aren't commands (like `define` or `route`)—they're just property keys! By making them reserved keywords, we bloat the parser.

**Property keys should just be parsed as standard `IDENTIFIER` tokens.**

### 7. The `origin: tl by t` Abstraction Leak

**The Flaw**: The unified coordinate system was a great idea to solve the Top-Left vs Bottom-Left holy war, but the syntax leaks implementation details.

```hw
# ❌ v0.1.5 - Reads poorly
origin: tl by t                  # "Top-left by top" doesn't sound natural
```

**Why it fails**: If we're optimizing for human readability, it should be elegant. However, the `by` keyword creates perfect visual rhythm with `dimensions` and `grid`, so `tl by t` is actually superior and stays in v0.1.6.

---

## The Diagnosis: Why the AI Failed

The AI didn't fail because it lacked context. **It failed because Hardware Script v0.1.5 requires rote memorization of exceptions.**

- The AI knows that `pins:` represents a list. So it wrote `[D0, D1]`. The parser rejected it.
- The AI knows that `BusConnector` is a component name. So it wrote `define component BusConnector`. The parser demanded quotes.

**When a language's syntax is perfectly unified, an AI (or a human) can guess how to write a feature they've never used before, and they will be right 99% of the time.**

---

## The v0.1.6 Mission: The Grand Syntax Unification

Now that the God-Tier Engine (routing, physics, SDF) is complete and working flawlessly, we're in the perfect position to clean house.

**For v0.1.6, we will not add any new features. We will rewrite the Parser to obey Universal Grammar Rules.**

### Universal Grammar Rules

1. **Universal Identifiers**: `component Resistor:` (No more quotes for types)
2. **Universal Lists**: `[` and `]` are explicitly allowed for any list in the language
3. **Universal Assignment**: Rock-solid rule for when to use `:` (Hierarchy/Structure) and when to use `=` (Values/Math)
4. **Universal Parameters**: All component instantiations must use `(key: value)` syntax. No more magic positional arguments.
5. **Flexible Metadata**: `metadata:` blocks natively accept any key-value pair as a string

This will dramatically simplify the `hwc-parser` crate, delete hundreds of lines of "exception" code, and make Hardware Script the cleanest, most beautiful hardware language in existence.

---

## The "Define" Trap: Why It Feels Like a Configuration File

The culprit is the `define` keyword and the rigid structural straitjacket it forces the user into.

### How We Fell Into the Trap

When building the parser, we needed a way to tell the compiler: "Hey, I'm about to declare something new."

So we naturally reached for the word `define`, followed by what we were defining, followed by a string name:

```hw
define component "Resistor":
define space "Motherboard":
define material "Copper":
```

**Why this is a failure of language design**: Look at how actual programming languages do this:

- In C or Rust, you don't write `define struct User`. You just write `struct User`.
- In Python, you don't write `define class Node`. You just write `class Node`.
- In SQL, you don't write `DEFINE QUERY SELECT`. You just write `SELECT`.

By using `define component "Name":`, we made `define` a command, `component` an argument, and `"Name"` a string literal. It inherently feels restrictive and bureaucratic.

---

## Option 1: The "Implicit" Paradigm (Rejected)

**The Idea**: Why use keywords at all? Just write the name, provide the data, and let the compiler figure out what it is based on the contents.

```hw
# The compiler sees 'pins' and 'electrical', so it knows this is a Component
Resistor_0805:
    pins: [A, B]
    electrical:
        resistance: 10k

# The compiler sees 'dimensions' and 'grid', so it knows this is a Space
Motherboard:
    dimensions: 100mm by 100mm by 2mm
    grid: 100 by 100 by 2
    add Resistor_0805 named R1 at [x:10, y:10]
```

### The Pros
- Ultimate freedom
- Removes all boilerplate
- Looks incredibly clean

### The Fatal Flaw: The Error Message Nightmare

If we do this, we lose **Contextual Error Recovery**.

Imagine a beginner makes a typo and writes `dimentions:` instead of `dimensions:`. Because there's no keyword telling the compiler "This is supposed to be a Space," the compiler looks at the block, gets confused, and throws a massive, unhelpful error:

```
Error: Unknown block structure
```

**Explicit keywords act as Anchors.** If the compiler sees the word `space`, it knows exactly what rules apply. If you misspell `dimensions`, it can say:

```
Error: Spaces require a 'dimensions' property. Did you mean 'dimensions'?
```

---

## Option 2: The "Type-as-Keyword" Paradigm (The Sweet Spot) ✅

If implicit typing ruins error messages, but `define component` is too restrictive, we look to C, Rust, and F# for the answer.

**We promote the hardware types (`component`, `space`, `material`, `profile`, `module`) to be first-class language keywords, exactly like `class`, `struct`, or `interface` in software.**

**We drop the `define` and we drop the quotes.**

### The Old Way (Restrictive & Clunky)

```hw
define material "Copper":
    category: conductor

define component "Resistor_0805":
    pins:
        A, B

define space "Motherboard":
    dimensions: 100mm by 100mm by 2mm
    add Resistor_0805 named R1 at [10, 10, 1]
```

### The New Way (Fluent, Free, and Software-Native)

```hw
material Copper:
    category: conductor

component Resistor_0805:
    pins: [A, B]

space Motherboard:
    dimensions: 100mm by 100mm by 2mm
    add Resistor_0805 named R1 at [10, 10, 1]
```

### Why Option 2 Solves the Problem

1. **It acts like a real programming language**: `component` is just a struct. `space` is just a function execution environment. It instantly maps to a software engineer's mental model.

2. **It frees the parser**: We no longer have to parse a complex `define` command. If a line starts with `component`, the lexer immediately jumps into component-parsing mode.

3. **It separates Nouns and Verbs**:
   - **Nouns (Declarations)**: `material`, `component`, `space`, `module`, `profile` (Used to invent new things)
   - **Verbs (Actions)**: `add`, `route`, `import`, `expose` (Used inside a space to do things)

---

## Pushing the Freedom Further: Unified Properties

To truly free the language, we must also fix the punctuation traps. Once we're inside a component or space, the language should act like a clean, unified scripting environment.

### The Rules

- **Variables/Math use `=`**: `Count = count + 1`
- **Properties/Metadata use `:`**: `dimensions: 100mm by 100mm`
- **Lists ALWAYS use `[]`**: `pins: [A, B, C]`

### A Completely Unified Example

```hw
# A completely unified, free-flowing language example:

component CustomALU (bits: Number):
    pins: [InA, InB, OutSum]
    
    metadata:
        author: "John Doe"
        version: "1.0"
    
    logic:
        # Standard software math
        OutSum = InA + InB

space MainBoard:
    dimensions: 50mm by 50mm by 2mm
    
    # Verbs do the work
    add CustomALU (bits: 8) named ALU1 at [x:10, y:10]
    add CustomALU (bits: 16) named ALU2 at [x:30, y:10]
    
    route ALU1.OutSum to ALU2.InA
```

---

## The Logic Layer: What to STRICTLY AVOID

Because this is a hardware DSL, the `logic:` layer is fundamentally different from a software language. **We must violently reject software features that have no physical equivalent in silicon:**

### NO Dynamic Memory

**No `new`, `malloc`, or `Array.push()`**: In hardware, every wire and register must physically exist before the power is turned on. You cannot "create" a new wire at runtime. Arrays must be fixed size (`Data[64]`).

### NO `while` Loops in `logic:`

Software uses `while` loops to wait for things. In hardware, a `while` loop would create an infinite physical short-circuit (a combinational loop). We only use `for` loops, and they only run at **Compile-Time (Comptime)** to stamp out physical gates.

**Time in hardware is handled exclusively by Clocks and Registers, not loops.**

### NO Standard I/O

**No `print()`, `console.log()`**: Silicon chips don't have consoles. Debugging is done via the `test` block (`assert Voltage = 5V`), not by printing strings to a terminal.

**By keeping these out, the language remains a pure, mathematically provable representation of physics and gates.**

---

## The "Verilog Killers": What We Must Nail

To do "literally everything Verilog can do, but easier," we only need to perfect three concepts in our Option 2 syntax.

### A. The "Blocking vs Non-Blocking" Solution

**Verilog's biggest flaw**: The difference between `=` (blocking) and `<=` (non-blocking). Millions of dollars of silicon have failed because an engineer used the wrong equals sign, causing race conditions.

**Our "Better" Solution**: We separate the concept of a "Wire" (instant math) from a "Register" (time-delayed memory).

- **Wires use `=`**: They are instant. `let Sum = A + B`
- **Registers use `.next`**: They update on the clock tick. `let Count = reg(clock: clk); Count.next = Count + 1`

**This completely eliminates Verilog's race conditions with zero confusing syntax.**

### B. First-Class State Machines (FSMs)

In Verilog, writing a State Machine requires three separate, ugly `always` blocks. It takes 40 lines of code just to blink an LED in a sequence.

Because our logic layer borrows from Rust, we use `enum` and `match` as expression-based primitives.

**The Option 2 Hardware Script Way**:

```hw
enum TrafficLight: [Red, Green, Yellow]

module Controller:
    pins: [input Clk, output LightColor[2]]
    
    logic:
        # Define the state register (lowercase reg primitive)
        let state = reg(clock: Clk, init: TrafficLight.Red)
        
        # Calculate the next state instantly (single = for comparison)
        state.next = match state:
            TrafficLight.Red:    TrafficLight.Green
            TrafficLight.Green:  TrafficLight.Yellow
            TrafficLight.Yellow: TrafficLight.Red
        
        # Drive the output pins
        LightColor = state
```

**This is computationally identical to Verilog, but reads like a beautiful software script.**

### C. Comptime Generation (Killing Verilog `generate` blocks)

In Verilog, if you want to instantiate 64 Adders to make a 64-bit CPU, you have to use a `genvar` and a `generate` block, which has awful syntax.

Because Hardware Script is a DSL, we blend structural logic and instantiation seamlessly using compile-time `for` loops.

```hw
module ALU_64:
    pins: [input A[64], input B[64], output Sum[64]]
    
    # This loop runs during compilation, stamping out 64 physical adders
    for i in 0..63:
        add FullAdder named Bit[i]
        
        route A[i] to Bit[i].InA
        route B[i] to Bit[i].InB
        route Bit[i].Out to Sum[i]
```

---

## Is There Something Even Better?

Yes. The ultimate holy grail of hardware design is **Dataflow / Pipeline semantics** (which modern languages like Chisel try to do).

Imagine you're building a GPU pipeline. Data needs to flow from Stage 1 to Stage 2, but only if Stage 2 is "Ready" to receive it, and Stage 1 is "Valid" (has real data).

**Should we add Ready/Valid keywords to the syntax? NO.**

If we add that to the syntax, we violate the DSL rule and bloat the language. The "Better" approach is to use the **Standard Library and Interfaces** (which we already planned).

**We keep the language syntax small (Option 2), but we allow users to import powerful interfaces:**

```hw
import ReadyValid from @std/interfaces

module GPU_Stage:
    # We use the standard library interface instead of raw pins
    pins: [
        input DataIn implements ReadyValid(bits: 32),
        output DataOut implements ReadyValid(bits: 32)
    ]
    
    logic:
        # Logic only processes when upstream is valid and downstream is ready
        let fire = DataIn.valid & DataOut.ready
        
        let buffer = reg(clock: clk)
        if fire:
            buffer.next = DataIn.data * 2
        
        DataOut.data = buffer
```

---

## The Final Verdict

We have successfully navigated the language design minefield.

### We STOP adding syntax here

The language does not need classes, `while` loops, or dynamic memory.

### We implement Option 2

- Drop `define`
- Drop quotes on identifiers
- Use `[]` for all lists
- Use `:` for structure and `=` for logic

### We rely on the Standard Library

Any advanced hardware concepts (like AXI buses, Ready/Valid handshakes, or differential pairs) will be built using our simple primitives and imported via `@std`, keeping the core parser lightning fast and microscopic.

---

## Summary: The v0.1.6 Boundary

If you agree with this boundary, we have effectively finalized the ultimate specification for Hardware Script v0.1.6: **The Syntax Unification**.

This is the exact mindset that separates a bloated, failed language from a generational leap in engineering.
