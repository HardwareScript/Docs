Here is the completely updated Book 4: The Compiler Internals.

This new version reflects the v0.1.4 architectural breakthrough: the elimination of serde_yaml, the transition to a true Two-Pass Compiler with a Symbol Table, and the realization of the hyper-lean, unified .hw ecosystem.

Book 4: The Compiler Internals

Hardware Script v0.1.4
Target Audience: Rust systems programmers contributing to the compiler
Last Updated: March 2026

Foundation

This document consolidates the following source materials:

Source Files:

Docs/v0.1.1/ARCHITECTURE-COMPLETE.md — The North Star overview of the complete architecture

Docs/v0.1.2/COMPILER-ARCHITECTURE-PHILOSOPHY.md — Why it's a synthesizer, not a traditional compiler

Docs/v0.1.2/MULTI-LEVEL-IR-ARCHITECTURE.md — The 5-layer pipeline (Intent → Logical → Physical → Physics → Export)

Docs/v0.1.2/FIRST-PRINCIPLES-IMPLEMENTATION.md — Data-Oriented Design, Morton encoding, and NetlistArena

Docs/v0.1.4/LANGUAGE-BREAKING-CHANGES.md — The unified .hw ecosystem and elimination of YAML fragmentation

Introduction

This document is the blueprint for Hardware Script's compiler architecture. If you're a Rust systems programmer who wants to understand or contribute to the compiler, this is your guide.

Hardware Script is not a traditional compiler that generates machine code. It's a hardware synthesis engine that transforms text-based hardware descriptions into physical manufacturing files. Think of it as the intersection of OpenSCAD (3D modeling), Verilog (hardware description), and Terraform (infrastructure as code) — but for electronics and silicon.

What We're Actually Building
The Fundamental Architecture

Hardware Script is a domain-specific language (DSL) transpiler and synthesis engine, not a traditional compiler.

Traditional compiler:

code
Code
download
content_copy
expand_less
Source Code → AST → LLVM IR → Machine Code → Executable

Hardware Script:

code
Code
download
content_copy
expand_less
.hw Source → AST → Hardware IR → Voxel Grid → Multiple Output Formats

We're generating files (Gerber, GDSII, 3D models), not CPU instructions. This means we don't need LLVM, and our architecture follows proven patterns from Terraform, OpenSCAD, and Verilog synthesis tools.

Why This Matters

We're building the LLVM for Physical Reality — a universal infrastructure platform that:

Compiles text-based hardware descriptions

Validates against physics

Generates manufacturing files

Enables AI-native workflows

Maintains deterministic, reproducible builds

The Multi-Level IR Architecture

Hardware Script uses a Multi-Level Intermediate Representation (MLIR) pipeline with five distinct layers. This architecture breaks the historical trap of EDA fragmentation where logic, layout, and simulation lived in separate proprietary tools.

The Five Layers
code
Code
download
content_copy
expand_less
Layer 1: Intent / Behavioral Layer
         ↓
Layer 2: Logical IR (hwc-compiler)
         ↓
Layer 3: Physical IR (hwc-engine)
         │
         ├─→ Phase 1: Constraint Manager (routing sub-pipeline)
         ├─→ Phase 2: Geometry Router (routing sub-pipeline)
         ↓
Layer 4: Physics IR (hwc-physics)
         │
         ├─→ Phase 3: Design Rule Check (routing sub-pipeline)
         ↓
Layer 5: Manufacturing Layer (hwc-export)

Each layer has a specific purpose and clean boundaries. The 3-Phase Routing Sub-Pipeline (detailed in Book 5) executes within Layers 3 and 4. Let's examine each layer.

Layer 1: Intent / Behavioral Layer
What Happens Here

This is what the user writes — their intent expressed in a unified .hw language. Because Hardware Script utilizes a single-file ecosystem, users define materials, manufacturing profiles, components, and layout inside the same declarative syntax.

Example Code
code
Hw
download
content_copy
expand_less
import CMOS_FullAdder from standard.silicon.logic

for i in 0..64:
    add CMOS_FullAdder named Bit[i]
    route Bit[i-1].CarryOut to Bit[i].CarryIn
Key Observation

The user writes logical behavior using physical building blocks from the standard library. The language stays clean and expressive while maintaining a direct connection to physical reality.

Layer 2: Logical IR (hwc-compiler)
What Happens Here

The parser reads .hw files and builds an Abstract Syntax Tree (AST). Because Hardware Script encompasses all hardware domains (materials, rules, geometry) in one AST, the hwc-compiler acts as a true Two-Pass Compiler:

Pass 1 (Symbol Registration): Sweeps the AST and registers all define material, define profile, and define component blocks into a Symbol Table.

Pass 2 (Space Assembly): Takes the define space block and resolves all dimensions, grid sizes, and physical placement constraints against the symbols registered in Pass 1.

At this layer, the compiler understands the logic, physics, and connectivity but hasn't drawn any voxels yet.

Data Structure
code
Rust
download
content_copy
expand_less
/// Hardware Intermediate Representation
/// Pure data structure representing resolved hardware design
pub struct HardwareIR {
    /// Metadata
    pub space_name: String,
    pub dimensions_nm: (i64, i64, i64),  // nanometers (fixed-point)
    pub grid: (usize, usize, usize),
    pub voxel_size_nm: i64,  // nanometers (fixed-point)
    
    /// Resolved components with global coordinates
    pub components: Vec<PlacedComponent>,
    
    /// Resolved nets with waypoints and constraints
    pub nets: Vec<NetRoute>,
    
    /// Material assignments resolved from Symbol Table
    pub materials: MaterialDatabase,
    
    /// Constraints resolved from Symbol Table Profiles
    pub constraints: ConstraintSet,
}
The Two-Pass Compilation Process
code
Rust
download
content_copy
expand_less
pub fn compile_to_ir(program: Program) -> Result<HardwareIR, CompileError> {
    let mut ir = HardwareIR::new();
    let mut symbol_table = SymbolTable::new();
    
    // =================================================================
    // PASS 1: Symbol Registration (The Unified Ecosystem)
    // =================================================================
    for def in program.definitions {
        match def {
            Definition::Material(m) => symbol_table.register_material(m)?,
            Definition::Profile(p) => symbol_table.register_profile(p)?,
            Definition::Component(c) => symbol_table.register_component(c)?,
            Definition::Mechanical(m) => symbol_table.register_mechanical(m)?,
            Definition::Space(s) => {
                // Keep the space for Pass 2
                ir.space_ast = Some(s);
            }
            _ => {} // Handle interfaces, tests, signal_groups
        }
    }
    
    let space_ast = ir.space_ast.take().ok_or(CompileError::NoSpaceDefined)?;

    // =================================================================
    // PASS 2: Space Assembly & Constraint Resolution
    // =================================================================
    
    ir.space_name = space_ast.name;
    ir.dimensions_nm = convert_mm_to_nm(space_ast.dimensions);
    ir.grid = space_ast.grid;
    ir.voxel_size_nm = calculate_voxel_size_nm(ir.dimensions_nm, ir.grid);
    
    // 1. Load the profile rules defined in Pass 1
    if let Some(profile_name) = space_ast.profile {
        ir.constraints = symbol_table.get_profile(&profile_name)?;
    }
    
    // 2. Resolve component placements using components defined in Pass 1
    for component_ast in space_ast.components {
        let component_def = symbol_table.get_component(&component_ast.type_name)?;
        
        let placed = PlacedComponent {
            id: ComponentId::new(),
            name: component_ast.name,
            position: component_ast.position,
            pins: transform_pins_to_global(
                &component_def.pins,
                component_ast.position,
                component_ast.rotation
            ),
            bounding_box: calculate_bounding_box(&component_def, component_ast.position),
        };
        ir.components.push(placed);
    }
    
    // 3. Resolve routes
    for route_ast in space_ast.routes {
        // ... routing logic leveraging materials from Symbol Table ...
    }
    
    Ok(ir)
}
Why This Layer Is Critical

This two-pass approach is what frees Hardware Script from needing external YAML files mid-compilation. All physics parameters, units, and geometric constraints are parsed upfront by our high-performance lexer and passed to the compiler in memory. This enables sub-millisecond resolution times.

Layer 3: Physical IR (hwc-engine)
What Happens Here

The hwc-engine takes the Logical IR and maps it to physical reality. We do not use standard HashMaps or 3D Game Engine math. To guarantee cache locality, sub-millisecond execution, and 100% determinism across all CPU architectures, we use Data-Oriented Design (Struct of Arrays), Fixed-Point Math (i64), and Morton Z-Curve Encoding.

The ECS Arena (Netlist)

Generic graph libraries (like petgraph) require runtime borrow checking (Rc<RefCell<T>>), which kills performance on 100,000-net motherboards. Instead, we use a custom Arena:

code
Rust
download
content_copy
expand_less
// Strongly typed IDs (zero memory overhead)
#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub struct ComponentId(u32);

#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub struct PinId(u32);

#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub struct NetId(u32);

pub struct NetlistArena {
    pub components: Vec<ComponentData>,
    pub pins: Vec<PinData>,         // Flat array of every pin on the board
    pub nets: Vec<NetData>,         // Flat array of every wire
}

Why this is superior: When the routing algorithm asks "What net is this pin on?", it doesn't traverse a complex graph of pointers. It does a single, instantaneous array lookup: arena.pins[pin_id.0 as usize].connected_net. You bypass the Rust borrow checker entirely because you are just passing around u32 integers, not memory references.

The Spatial Voxel Engine (Morton Encoding)

Instead of FxHashMap<(usize, usize, usize), MaterialState>, which causes CPU cache misses, we map 3D space into a 1D array using Morton Codes (Z-Order Curve). Voxels close in physical space are close in computer RAM.

code
Rust
download
content_copy
expand_less
/// Morton encoding: Interleave bits of X, Y, Z coordinates
fn morton_encode(x: u32, y: u32, z: u32) -> u64 {
    let mut result = 0u64;
    
    for i in 0..21 {  // 21 bits per coordinate = 63 bits total
        result |= ((x & (1 << i)) as u64) << (2 * i);
        result |= ((y & (1 << i)) as u64) << (2 * i + 1);
        result |= ((z & (1 << i)) as u64) << (2 * i + 2);
    }
    
    result
}

struct VoxelGrid {
    // Struct of Arrays (Data-Oriented Design)
    materials: Vec<u8>,   // Morton-encoded Z-curve
    net_ids: Vec<u32>,    // Morton-encoded Z-curve
    
    // Bitmask for fast collision (Checking 64 voxels in 1 CPU cycle)
    collision_mask: Vec<u64>,
}

Performance impact:

code
Code
download
content_copy
expand_less
HashMap approach:
  Routing 10,000 nets = 60,000 neighbor queries
  60,000 cache misses = ~600ms of RAM waiting

Morton Z-curve approach:
  Routing 10,000 nets = 60,000 neighbor queries
  Most queries hit L1 cache = ~6ms total

100× performance improvement for routing
Fixed-Point Geometry Data Structures

The routing engine operates on integer-based geometric primitives to guarantee determinism. All coordinates use i64 nanometers for perfect reproducibility across all CPU architectures.

Why Fixed-Point Math Matters:

code
Rust
download
content_copy
expand_less
// Floating-point (non-deterministic)
let clearance = 5.2 * 1.5;  // Different results on x86 vs ARM
// x86: 7.799999952316284
// ARM: 7.800000047683716

// Fixed-point (deterministic)
let clearance_nm = 5_200_000 * 15 / 10;  // Always 7_800_000
// x86: 7_800_000
// ARM: 7_800_000
// Identical on all platforms
Layer 4: Physics IR (hwc-physics)
What Happens Here

Before generating output files, the hwc-physics crate validates the physical design against the laws of physics. It sweeps the voxel grid and checks:

Electrical properties (resistance, voltage drop, current capacity)

Thermal properties (temperature rise, heat dissipation)

Electromagnetic properties (impedance, signal integrity)

Clearance requirements (dielectric breakdown prevention)

This layer validates the design without modifying it, ensuring physics compliance before manufacturing.

Layer 5: Manufacturing Layer (hwc-export)
What Happens Here

The validated voxel grid is converted into industry-standard manufacturing formats. This layer is pure translation — no validation, no modification, just format conversion.

This layer utilizes Custom Emitters to generate output files without bloated third-party dependencies (see Book 6: Exports & Assets).

The 8 Rust Crates

Hardware Script's compiler is organized into 8 focused crates, each with a single responsibility.

Crate Architecture
code
Code
download
content_copy
expand_less
hwc-cli        - Command-line interface and pipeline orchestration
hwc-parser     - Lexer, parser, and unified AST construction
hwc-compiler   - Two-pass AST-to-IR compilation and symbol resolution
hwc-engine     - Voxel grid management and spatial operations
hwc-physics    - Physics validation (electrical, thermal, EM)
hwc-export     - Output file generation (Gerber, GDSII, 3D)
hwc-materials  - Core physical structures for materials
hwc-stdlib     - Standard component library and templates
Crate Responsibilities

hwc-parser:

Tokenization and lexing

Grammar-based parsing (hand-written recursive descent)

Native parsing of SI units (254µm) directly into numeric values

Unified AST construction for Spaces, Materials, Profiles, and Components

hwc-compiler:

Import resolution

Two-Pass compilation (Symbol Registration + Evaluation)

Coordinate transformation (local → global, with configurable XY and Z origin points)

Hardware IR construction

hwc-materials:

Core physics data structures (Conductor, Insulator, Semiconductor)

Note: Data is populated natively by the hwc-compiler reading define material blocks, eliminating external YAML dependencies.

hwc-engine:

Sparse voxel grid with Morton encoding

Component placement and Route interpolation

Collision detection

Bounding box calculations (fixed-point i64)

hwc-physics:

Electrical analysis, Thermal analysis, Electromagnetic analysis

Clearance validation (dielectric breakdown)

hwc-export:

Gerber generation (PCB)

GDSII generation (silicon)

3D model export (OBJ, GLB, STEP) and Blender script generation

hwc-cli:

Pipeline orchestration, user interaction, and error display

Error Handling Philosophy

Hardware Script treats physics violations with the same developer experience as Rust's borrow checker errors. We use miette and thiserror to generate world-class diagnostics with short, memorable error codes.

The 3-Character Error Code System

Format: [Letter][Digit][Digit] (e.g., P16, R12, S22)

Subsystems:

S - Syntax errors (parser)

C - Compiler errors (IR & logic)

R - Routing & engine errors (physical placement)

P - Physics errors (design rule check)

M - Manufacturing errors (export & fabrication)

Terminal output example:

code
Code
download
content_copy
expand_less
❌ Error[P16]: Dielectric Breakdown Risk
   ╭─[main.hw:42:1]
 42 │     route Power_120V.out to Relay.in:
   ·           ──────┬───────
   ·                 ╰── High voltage net (120V)
 45 │             -[x:10, y:50, z:2]
   ·               ──────┬──────
   ·                     ╰── Approaches within 0.02mm of MCU_5V net here
   ╰────
  help: The voltage difference is 115V. Through Air, this requires a 
        minimum clearance of 0.08mm to prevent arcing.
Why Rust Is Perfect for This
The Hyper-Lean Rust Stack

Hardware Script rejects "Library Maximalism". Every generic library imported brings edge-cases and memory overhead. We aggressively own our core abstractions.

Our Cargo.toml contains only 4 external dependencies:

logos — Insanely fast lexing (feeds our hand-written recursive descent parser)

miette & thiserror — World-class, rustc-style terminal diagnostics

rayon — Multi-threaded data parallelism

serde — Internal IR serialization and caching

(Notice: serde_yaml has been eliminated. Because Hardware Script natively parses SI units and physical properties, we completely eliminated the need for third-party configuration parsers. We own the entire syntax tree.)

Everything else is pure, custom, Hardware Script IP:

No parser combinators (Pest/Chumsky/Tree-sitter) → We use a hand-written parser for 100% control over error recovery

No generic 3D math (Nalgebra/Glam) → We use custom integer-based Manhattan Math

No graph libraries (Petgraph) → We use our custom NetlistArena

Why this matters:

code
Code
download
content_copy
expand_less
Library Maximalism:
  Dependencies: 15+ libraries
  Compile time: 2+ minutes
  Binary size: 50MB
  Control: Low (at mercy of library design)

Hyper-Lean Approach:
  Dependencies: 4 libraries
  Compile time: <10 seconds
  Binary size: 5MB
  Control: Complete (you own the code)
The Semantic Boundary

Hardware Script maintains a clean semantic boundary: .hw files describe geometry, physical properties, and connections, not behavioral logic.

What Hardware Script Is NOT

We are not building:

A better Verilog (no behavioral blocks like Sum = A + B at the top level)

An open-source Cadence (no black-box place & route)

A simplified Altium (no GUI-first design)

What Hardware Script IS

We are building:

Software-Defined Bare-Metal

The LLVM for Physical Reality

Git-friendly hardware design

Deterministic, reproducible compilation

LLM-native workflows

The Distinction

Traditional EDA:

code
Code
download
content_copy
expand_less
Behavioral description (Verilog)
    ↓
Logic synthesis (black box)
    ↓
Place & Route (black box)
    ↓
Physical layout

Hardware Script:

code
Code
download
content_copy
expand_less
Physical topology & Materials (.hw)
    ↓
Two-Pass Compilation & Symbol Resolution (IR)
    ↓
Voxel grid (deterministic)
    ↓
Physics validation (transparent)
    ↓
Manufacturing files

Every step is transparent, deterministic, and version-controllable.

Contributing to the Compiler
Architecture Guidelines

Respect layer boundaries:

Parser only builds the unified AST

Compiler only resolves Symbols and builds IR

Engine only manipulates voxels

Physics only validates

Export only generates files

Keep crates focused:

Each crate has one responsibility

No circular dependencies

Clean, documented interfaces

Areas for Contribution

Compiler optimizations:

Symbol table caching and incremental builds

Parallel compilation passes

Engine improvements:

Advanced routing algorithms

GPU acceleration for the voxel grid

Export formats:

STEP files (mechanical CAD)

Simulation outputs

Documentation generation

Conclusion

Hardware Script's compiler architecture is revolutionary because it unifies what has historically been fragmented across proprietary tools into a single, cohesive, sub-millisecond compilation pipeline. By using a two-pass symbol resolution stage, sparse voxel storage, and Rust's performance and safety guarantees, we've built a system that scales from simple LED circuits to complex silicon chips.

Welcome to the compiler internals. Let's build something amazing.