# Hardware Script v0.1.2 - Compiler Architecture Philosophy

**Document Type**: Fundamental System Architecture  
**Status**: External Research Integration  
**Last Updated**: March 2026

---

## What Are We Actually Building?

You're not necessarily building a traditional compiler — you're building something closer to a **domain-specific language (DSL) transpiler / synthesizer**.

This is a critical distinction that affects every architectural decision.

---

## The Three Main Types of "Language Engines"

### Type 1: Compiler

**What it does**: Converts language → machine code

**Examples**:
- Rust Compiler (rustc)
- GCC (C/C++)
- Clang

**Output**: Binary executables that run on CPU

**Process**:
```
Source Code → AST → LLVM IR → Machine Code → Executable
```

### Type 2: Interpreter

**What it does**: Executes code directly

**Examples**:
- Python
- JavaScript (V8)
- Ruby

**Output**: Direct execution, no intermediate files

**Process**:
```
Source Code → Parse → Execute immediately
```

### Type 3: Transpiler / Synthesizer

**What it does**: Converts language → another language or format

**Examples**:
- TypeScript (TypeScript → JavaScript)
- Sass (Sass → CSS)
- Babel (ES6+ → ES5)

**Output**: Files in different formats

**Process**:
```
Source Code → AST → IR → Multiple Output Formats
```

---

## Hardware Script Is Type #3: A Synthesis Engine


### The Hardware Script Pipeline

```
HardwareScript (.hw files)
         ↓
    Parser
         ↓
    AST (Abstract Syntax Tree)
         ↓
    Hardware IR (Intermediate Representation)
         ↓
    Synthesis Engine
         ↓
    Generators
    ├─ Gerber generator (.gtl) - PCB manufacturing
    ├─ Blender generator (.py) - 3D visualization
    ├─ OBJ generator (.obj) - 3D models
    ├─ Simulation generator - Physics simulation
    └─ CAD generator - Mechanical design
```

**This is closer to a hardware synthesis engine than a traditional compiler.**

---

## What You Are Actually Building

### The Core Architecture

```
HardwareScript (your language)
         ↓
    Tokenizer/Lexer
         ↓
    Parser
         ↓
    AST
         ↓
    Execution Table / IR
         ↓
    Generators
    ├─ Gerber generator (.gtl)
    ├─ Blender generator (.py)
    ├─ Simulation generator
    └─ CAD generator
```

### The Two-Phase Process

**Phase 1**: Language → Hardware representation
```
.hw file → Parser → Hardware IR
```

**Phase 2**: Hardware representation → Output formats
```
Hardware IR → Generators → Multiple file formats
```

**This is very powerful architecture.**

---

## Why Rust Is a Very Good Choice

Rust is actually one of the **best languages** for writing compilers and DSL engines.

### Reasons

#### 1. Performance

Rust is close to C/C++ speed.
- No garbage collector overhead
- Zero-cost abstractions
- Compile-time optimizations

#### 2. Safety

No garbage collector and no memory bugs.
- Ownership system prevents memory leaks
- No null pointer exceptions
- Thread safety guaranteed at compile time

#### 3. Ecosystem

Rust already has excellent parsing libraries.

**Example libraries**:

| Library | Purpose |
|---------|---------|
| `nom` | Parser combinator framework |
| `pest` | Grammar-based parser (PEG) |
| `lalrpop` | Compiler-style parser (LALR) |
| `tree-sitter` | Incremental parsing |

**Most modern DSLs use one of these.**

### Example with Pest

```rust
// grammar.pest
space = { "space" ~ string ~ "{" ~ space_body ~ "}" }
trace = { "trace" ~ "{" ~ trace_body ~ "}" }
coordinate = { "[" ~ number ~ "," ~ number ~ "," ~ number ~ "]" }

// parser.rs
use pest::Parser;

#[derive(Parser)]
#[grammar = "grammar.pest"]
struct HWParser;

fn parse_hw_file(input: &str) -> Result<AST, Error> {
    let pairs = HWParser::parse(Rule::space, input)?;
    // Build AST...
}
```

---

## Why You DON'T Need LLVM

### What LLVM Is For

LLVM is used when you want to produce:
- Machine code
- Assembly
- CPU executables

**Example languages using LLVM**:
- Rust
- Swift
- Clang (C/C++)

### What Hardware Script Produces

Your outputs are:
- Gerber files (text format)
- Python scripts (text format)
- CAD instructions (text/binary format)

**LLVM would actually be unnecessary complexity.**

You're doing **format synthesis**, not **machine code compilation**.

### The Right Tool for the Job

```
Traditional Compiler:
  Source → LLVM IR → Machine Code → Executable

Hardware Script:
  Source → Hardware IR → Format Generators → Files
```

**No CPU execution = No need for LLVM**

---

## The Most Important Layer: Intermediate Representation (IR)

### This Is the Heart of the System

The IR is your internal representation of the hardware design.

**Everything flows through the IR**:
```
Parser → IR → Generators
```

### Example: Trace Definition

**HardwareScript input**:
```hw
trace {
    from [1, 2, 5]
    to [1, 7, 5]
    width 0.25mm
    layer top
}
```

**Your IR might become**:
```rust
Trace {
    start: Coordinate { z: 1, x: 2, y: 5 },
    end: Coordinate { z: 1, x: 7, y: 5 },
    width: 0.25,
    layer: Layer::Top,
}
```

**Then each generator converts it**:

**GTL generator**:
```gerber
G01 X020000 Y050000
G01 X070000 Y050000
```

**Blender generator**:
```python
create_trace(start=(2,5), end=(7,5), width=0.25)
```

**Simulation generator**:
```rust
CopperTrace {
    start: (2, 5),
    end: (7, 5),
    resistance: 0.0042,
}
```

**This is exactly how good compilers work.**

---

## Your System Is Similar To These

### Hardware Languages

**Verilog / VHDL**:
```
Code → Circuit synthesis → FPGA bitstream
```

They do:
- Parse hardware description language
- Synthesize to logic gates
- Generate configuration files

**Hardware Script does**:
- Parse hardware description language
- Synthesize to physical layout
- Generate manufacturing files

### Build Systems

**Bazel / CMake**:
```
Script → Build graph → Multiple outputs
```

They do:
- Parse build configuration
- Generate build instructions
- Output multiple file types

**Hardware Script does**:
- Parse hardware configuration
- Generate hardware instructions
- Output multiple file types

### Modern DSL Engines

**OpenSCAD**:
```scad
cube(10);
translate([5,5,0])
    sphere(5);
```

Then output:
- STL 3D models
- PNG renders
- DXF 2D exports

**Hardware Script does**:
```hw
space "Board" {
    Battery @ [1, 10, 10]
    LED @ [1, 90, 90]
    Battery.Plus -> LED.Anode
}
```

Then output:
- Gerber PCB files
- Blender 3D models
- Simulation data

### The Pattern

**Your system is basically**:
```
OpenSCAD + Verilog + Blender pipeline
```

For hardware design instead of 3D modeling or logic synthesis.

---

## The Hardest Part (Most People Get Wrong)

### It's Not Parsing

Parsing is well-understood. Libraries like `pest` make it easy.

### The Hardest Part Is: The Hardware Abstraction Model

You need a core schema that accurately represents physical hardware.

**Example core types**:
```rust
struct Board {
    dimensions: (f32, f32, f32),
    grid: (usize, usize, usize),
    layers: Vec<Layer>,
    components: Vec<Component>,
    traces: Vec<Trace>,
    vias: Vec<Via>,
    pads: Vec<Pad>,
}

struct Component {
    name: String,
    position: Coordinate,
    rotation: Rotation,
    pins: Vec<Pin>,
    footprint: Footprint,
}

struct Trace {
    net: String,
    waypoints: Vec<Coordinate>,
    width: f32,
    layer: Layer,
}

struct Via {
    position: Coordinate,
    from_layer: Layer,
    to_layer: Layer,
    diameter: f32,
}
```

### Your IR Should Describe Hardware, Not Files

**Bad IR** (file-focused):
```rust
struct GerberLine {
    x: i32,
    y: i32,
    command: String,
}
```

**Good IR** (hardware-focused):
```rust
struct Trace {
    start: Coordinate,
    end: Coordinate,
    width: f32,
    net: String,
}
```

**Why**: The good IR can generate ANY output format. The bad IR is locked to Gerber.

### Example Hierarchy

```
Board
  ├─ Layers
  │   ├─ Layer 1 (Top)
  │   ├─ Layer 2 (Ground Plane)
  │   └─ Layer 3 (Bottom)
  ├─ Traces
  │   ├─ Trace 1 (Power)
  │   └─ Trace 2 (Data)
  ├─ Components
  │   ├─ Battery
  │   └─ LED
  ├─ Pads
  │   ├─ Battery.Plus
  │   └─ LED.Anode
  └─ Vias
      └─ Via 1 (Layer 1 → Layer 3)
```

**Then generators translate it to different formats.**


---

## Ideal Architecture For HardwareScript

### The Complete Pipeline

```
HardwareScript (.hw files)
      ↓
Tokenizer (Lexer)
      ↓
Parser
      ↓
AST (Abstract Syntax Tree)
      ↓
Hardware IR (Intermediate Representation)
      ↓
Synthesis Engine (Physics validation, routing, optimization)
      ↓
Generators
    ├─ Gerber Generator
    ├─ Blender Generator
    ├─ Simulation Generator
    └─ CAD Generator
```

### Stage-by-Stage Breakdown

#### Stage 1: Tokenizer

**Input**: Raw text
**Output**: Token stream

```rust
"space \"Board\" { }" 
    ↓
[
    Token::Keyword("space"),
    Token::String("Board"),
    Token::LBrace,
    Token::RBrace,
]
```

#### Stage 2: Parser

**Input**: Token stream
**Output**: AST

```rust
[Token::Keyword("space"), Token::String("Board"), ...]
    ↓
SpaceNode {
    name: "Board",
    body: [...],
}
```

#### Stage 3: AST

**Input**: Parse tree
**Output**: Structured tree

```rust
Program {
    spaces: [
        Space {
            name: "Board",
            dimensions: (100, 100, 2),
            components: [...],
            traces: [...],
        }
    ]
}
```

#### Stage 4: Hardware IR

**Input**: AST
**Output**: Hardware representation

```rust
Board {
    dimensions: (100.0, 100.0, 2.0),
    grid: (100, 100, 2),
    voxel_size: 1.0,
    tensor: Array3D<CellState>,
    components: Vec<Component>,
    nets: Vec<Net>,
}
```

#### Stage 5: Synthesis Engine

**Input**: Hardware IR
**Output**: Validated, routed hardware

**Operations**:
- Physics validation
- Routing algorithm
- Collision detection
- Optimization

#### Stage 6: Generators

**Input**: Validated hardware IR
**Output**: Multiple file formats

```rust
trait Generator {
    fn generate(&self, board: &Board) -> Result<String, Error>;
}

struct GerberGenerator;
struct BlenderGenerator;
struct SimulationGenerator;
```

---

## The Real Power: One Script, Multiple Outputs

### The Vision

```hw
space "Board1" {
    dimensions: 100mm × 100mm × 2mm
    grid: 100 × 100 × 2
    
    Battery @ [1, 10, 10]
    LED @ [1, 90, 90]
    
    Battery.Plus -> LED.Anode
}
```

**Outputs simultaneously**:
1. ✅ PCB layout (Gerber files)
2. ✅ 3D model (Blender/OBJ)
3. ✅ Simulation model (Physics engine)
4. ✅ Manufacturing files (BOM, drill files)
5. ✅ Documentation (Auto-generated)

**That is huge if you succeed.**

### Why This Matters

**Traditional workflow**:
```
Design in KiCad → Export Gerber
Design in Blender → Export 3D model
Design in SPICE → Export simulation
Manually create BOM
Manually write documentation
```

**Hardware Script workflow**:
```
Write .hw file → Get everything automatically
```

**Result**: 10× faster iteration, perfect consistency across all outputs.

---

## Comparison to Similar Systems

### Terraform → Infrastructure

```hcl
resource "aws_instance" "web" {
  ami           = "ami-123456"
  instance_type = "t2.micro"
}
```

**Outputs**:
- AWS infrastructure
- State files
- Documentation

### OpenSCAD → CAD

```scad
cube([10, 10, 10]);
translate([5, 5, 0])
    sphere(5);
```

**Outputs**:
- STL files
- PNG renders
- DXF exports

### Verilog → Hardware

```verilog
module adder(a, b, sum);
    input a, b;
    output sum;
    assign sum = a + b;
endmodule
```

**Outputs**:
- FPGA bitstream
- Simulation models
- Synthesis reports

### HardwareScript → Electronics + Mechanical + Simulation

```hw
space "Board" {
    Battery @ [1, 10, 10]
    LED @ [1, 90, 90]
    Battery.Plus -> LED.Anode
}
```

**Outputs**:
- PCB manufacturing files
- 3D mechanical models
- Physics simulation
- Documentation

**HardwareScript unifies all three domains.**

---

## Key Architectural Decisions

### Decision 1: IR-Centric Design

**Good**: Everything goes through IR
```
Parser → IR → Generators
```

**Bad**: Direct translation
```
Parser → Gerber (no IR)
```

**Why IR is better**:
- Add new output formats easily
- Optimize once, benefit everywhere
- Validate once, trust everywhere

### Decision 2: Immutable IR

**Good**: IR is immutable after creation
```rust
let board = parse_and_build_ir(source)?;
let gerber = generate_gerber(&board);
let blender = generate_blender(&board);
```

**Bad**: Mutable IR that generators modify
```rust
let mut board = parse(source)?;
generate_gerber(&mut board);  // Modifies board!
generate_blender(&mut board);  // Different board now!
```

**Why immutable is better**:
- Generators can run in parallel
- No hidden state changes
- Easier to debug

### Decision 3: Separate Validation from Generation

**Good**: Validate first, generate second
```rust
let board = parse(source)?;
validate(&board)?;  // Fails early
generate_all(&board)?;
```

**Bad**: Validate during generation
```rust
let board = parse(source)?;
generate_gerber(&board)?;  // Might fail here
generate_blender(&board)?;  // Or here
```

**Why separate is better**:
- Fail fast
- Clear error messages
- Don't waste time generating invalid designs

---

## Implementation Roadmap

### Phase 1: Core IR (Current)

- [x] Define core types (Board, Component, Trace, etc.)
- [x] Implement basic parser
- [x] Build simple IR
- [ ] Add validation layer

### Phase 2: Generators (Next Sprint)

- [ ] Gerber generator (complete)
- [ ] Blender generator (complete)
- [ ] OBJ generator (complete)
- [ ] Drill file generator

### Phase 3: Advanced Features (Next Month)

- [ ] Physics simulation generator
- [ ] BOM generator
- [ ] Documentation generator
- [ ] CAD generator

### Phase 4: Optimization (Future)

- [ ] IR optimization passes
- [ ] Parallel generation
- [ ] Incremental compilation
- [ ] Caching

---

## Key Takeaways

1. **Hardware Script is a transpiler/synthesizer** - Not a traditional compiler

2. **No need for LLVM** - We're generating files, not machine code

3. **IR is the heart** - Everything flows through the intermediate representation

4. **Rust is perfect** - Performance, safety, and great parsing libraries

5. **One input, many outputs** - The real power of the architecture

6. **Hardware abstraction is hard** - The IR design is more important than parsing

7. **Similar to Terraform/OpenSCAD/Verilog** - Proven architecture pattern

8. **Immutable IR** - Generators don't modify the hardware representation

9. **Validate early** - Catch errors before generation

10. **Extensible design** - Easy to add new output formats

---

## Summary

Hardware Script is a **hardware synthesis engine** that:
- Parses a domain-specific language
- Builds an intermediate representation of physical hardware
- Generates multiple output formats simultaneously

This architecture is:
- ✅ Proven (similar to Terraform, OpenSCAD, Verilog)
- ✅ Powerful (one source, many outputs)
- ✅ Extensible (easy to add new generators)
- ✅ Maintainable (clear separation of concerns)

**You're building the right thing, the right way.**

---

**Document Status**: Fundamental System Architecture  
**Last Updated**: March 2026  
**Part of**: Hardware Script v0.1.2 Documentation Suite

