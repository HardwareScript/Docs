# Book 4: The Compiler Internals

**Hardware Script v0.1.3**  
**Target Audience**: Rust systems programmers contributing to the compiler  
**Last Updated**: March 2026

---

## Foundation

This document consolidates the following source materials:

**Source Files**:
- `Docs/v0.1.1/ARCHITECTURE-COMPLETE.md` — The North Star overview of the complete architecture
- `Docs/v0.1.2/COMPILER-ARCHITECTURE-PHILOSOPHY.md` — Why it's a synthesizer, not a traditional compiler
- `Docs/v0.1.2/MULTI-LEVEL-IR-ARCHITECTURE.md` — The 5-layer pipeline (Intent → Logical → Physical → Physics → Export)
- `Docs/v0.1.2/ARCHITECTURAL-VALIDATION-AND-IR-PIPELINE.md` — The specific Rust implementations of HardwareIR
- `Docs/v0.1.2/RUST-IMPLEMENTATION-STRATEGY.md` — The breakdown of the 8 crates
- `Docs/v0.1.2/ERROR-HANDLING-PHILOSOPHY.md` — The miette + thiserror diagnostic strategy
- `Docs/v0.1.2/HYPER-LEAN-ARCHITECTURE.md` — The 5-dependency stack and custom implementations
- `Docs/v0.1.2/FIRST-PRINCIPLES-IMPLEMENTATION.md` — Data-Oriented Design, Morton encoding, and NetlistArena

---

## Introduction

This document is the blueprint for Hardware Script's compiler architecture. If you're a Rust systems programmer who wants to understand or contribute to the compiler, this is your guide.

Hardware Script is not a traditional compiler that generates machine code. It's a hardware synthesis engine that transforms text-based hardware descriptions into physical manufacturing files. Think of it as the intersection of OpenSCAD (3D modeling), Verilog (hardware description), and Terraform (infrastructure as code) — but for electronics and silicon.

---

## What We're Actually Building

### The Fundamental Architecture

Hardware Script is a **domain-specific language (DSL) transpiler and synthesis engine**, not a traditional compiler.

**Traditional compiler**:
```
Source Code → AST → LLVM IR → Machine Code → Executable
```

**Hardware Script**:
```
.hw Source → AST → Hardware IR → Voxel Grid → Multiple Output Formats
```

We're generating files (Gerber, GDSII, 3D models), not CPU instructions. This means we don't need LLVM, and our architecture follows proven patterns from Terraform, OpenSCAD, and Verilog synthesis tools.

### Why This Matters

**We're building the LLVM for Physical Reality** — a universal infrastructure platform that:
- Compiles text-based hardware descriptions
- Validates against physics
- Generates manufacturing files
- Enables AI-native workflows
- Maintains deterministic, reproducible builds

---

## The Multi-Level IR Architecture

Hardware Script uses a **Multi-Level Intermediate Representation (MLIR)** pipeline with five distinct layers. This architecture breaks the historical trap of EDA fragmentation where logic, layout, and simulation lived in separate proprietary tools.

### The Five Layers

```
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
```

Each layer has a specific purpose and clean boundaries. The 3-Phase Routing Sub-Pipeline (detailed in Book 5) executes within Layers 3 and 4. Let's examine each layer.

---

## Layer 1: Intent / Behavioral Layer

### What Happens Here

This is what the user writes — their intent expressed in `.hw` files. Users don't want to manually place billions of voxels; they write parametric descriptions and let the compiler handle the details.

### Example Code

```hw
import CMOS_FullAdder from standard.silicon.logic

for i in 0..64:
    add CMOS_FullAdder named Bit[i]
    route Bit[i-1].CarryOut to Bit[i].CarryIn
```

### Key Observation

The user writes logical behavior using physical building blocks from the standard library. The language stays clean and expressive while maintaining a direct connection to physical reality.

### Who Interacts Here

- Hobbyists writing simple LED circuits
- Engineers designing complex systems
- LLMs generating parametric designs
- Scripts automating hardware generation

---

## Layer 2: Logical IR (hwc-compiler)

### What Happens Here

The parser reads `.hw` files and builds an Abstract Syntax Tree (AST). The `hwc-compiler` then:
1. Unrolls loops and resolves imports
2. Transforms local coordinates to global space
3. Creates a netlist (graph of pin connections)
4. Loads constraints from materials and fabrication profiles

At this layer, the compiler understands the logic and connectivity but hasn't drawn any voxels yet.

### Data Structure

```rust
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
    
    /// Material assignments
    pub materials: MaterialDatabase,
    
    /// Constraints from .hwp profiles
    pub constraints: ConstraintSet,
}

/// A component placed in global space
pub struct PlacedComponent {
    pub id: ComponentId,
    pub name: String,
    pub component_type: String,
    pub position: (usize, usize, usize),  // Global voxel coordinates
    pub rotation: Rotation,
    pub pins: HashMap<String, (usize, usize, usize)>,  // Global pin positions
    pub bounding_box: BoundingBox,
}

/// A net with routing waypoints and constraints
pub struct NetRoute {
    pub id: NetId,
    pub name: String,
    pub from_pin: PinRef,
    pub to_pin: PinRef,
    pub waypoints: Vec<(usize, usize, usize)>,  // Global coordinates
    pub width_voxels: usize,
    pub material: MaterialId,
    pub constraints: RouteConstraints,
}
```

### The Compilation Process

```rust
pub fn compile_to_ir(ast: AST) -> Result<HardwareIR, CompileError> {
    let mut ir = HardwareIR::new();
    
    // 1. Extract space definition
    ir.space_name = ast.space.name;
    ir.dimensions_nm = convert_mm_to_nm(ast.space.dimensions);
    ir.grid = ast.space.grid;
    ir.voxel_size_nm = calculate_voxel_size_nm(ir.dimensions_nm, ir.grid);
    
    // 2. Load materials database
    ir.materials = MaterialDatabase::load_standard()?;
    
    // 3. Resolve component placements
    for component_ast in ast.components {
        let component_def = load_component_definition(&component_ast.type_name)?;
        
        // Transform from local to global coordinates
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
    
    // 4. Resolve net routes
    for route_ast in ast.routes {
        let from_pin = resolve_pin_reference(&route_ast.from, &ir.components)?;
        let to_pin = resolve_pin_reference(&route_ast.to, &ir.components)?;
        
        let net = NetRoute {
            id: NetId::new(),
            name: route_ast.name,
            from_pin,
            to_pin,
            waypoints: route_ast.waypoints,
            width_voxels: calculate_width_voxels(route_ast.width, ir.voxel_size_nm),
            material: resolve_material(&route_ast.material, &ir.materials)?,
            constraints: resolve_constraints(&route_ast.constraints)?,
        };
        
        ir.nets.push(net);
    }
    
    // 5. Load fabrication constraints
    if let Some(profile_path) = ast.fabrication_profile {
        ir.constraints = load_fabrication_profile(profile_path)?;
    }
    
    Ok(ir)
}
```

### Why This Layer Is Critical

This separation between parsing and spatial computation enables:
- Clean interface between compiler and engine
- Optimization passes on the IR
- Validation before voxel generation
- Multiple backend targets from one IR
- Testability and debugging

---

## Layer 3: Physical IR (hwc-engine)

### What Happens Here

The `hwc-engine` takes the Logical IR and maps it to physical reality. We do not use standard HashMaps or 3D Game Engine math. To guarantee cache locality, sub-millisecond execution, and 100% determinism across all CPU architectures, we use **Data-Oriented Design (Struct of Arrays)**, **Fixed-Point Math (i64)**, and **Morton Z-Curve Encoding**.

**Routing Sub-Pipeline**: Within this layer, the routing engine executes Phases 1 and 2 of the 3-Phase Routing Sub-Pipeline (Constraint Manager and Geometry Router). See Book 5 (Routing & Physics) for detailed routing algorithms and constraint generation.

### The ECS Arena (Netlist)

Generic graph libraries (like `petgraph`) require runtime borrow checking (`Rc<RefCell<T>>`), which kills performance on 100,000-net motherboards. Instead, we use a custom Arena:

```rust
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

struct ComponentData {
    name: String,
    position_nm: (i64, i64, i64),  // Fixed-point nanometers
    first_pin: PinId,
    pin_count: u32,
}

struct PinData {
    parent_component: ComponentId,
    connected_net: Option<NetId>,
    local_offset_nm: (i64, i64, i64),  // Fixed-point nanometers! No f32/f64!
}

struct NetData {
    name: String,
    width_nm: i64,
    pins: Vec<PinId>,
}

impl NetlistArena {
    pub fn get_pin_position(&self, pin: PinId) -> (i64, i64, i64) {
        let pin_data = &self.pins[pin.0 as usize];
        let component = &self.components[pin_data.parent_component.0 as usize];
        
        // Add component position + pin offset (pure integer math)
        (
            component.position_nm.0 + pin_data.local_offset_nm.0,
            component.position_nm.1 + pin_data.local_offset_nm.1,
            component.position_nm.2 + pin_data.local_offset_nm.2,
        )
    }
    
    pub fn get_connected_net(&self, pin: PinId) -> Option<NetId> {
        self.pins[pin.0 as usize].connected_net
    }
}
```

**Why this is superior**: When the routing algorithm asks "What net is this pin on?", it doesn't traverse a complex graph of pointers. It does a single, instantaneous array lookup: `arena.pins[pin_id.0 as usize].connected_net`. You bypass the Rust borrow checker entirely because you are just passing around `u32` integers, not memory references.

### The Spatial Voxel Engine (Morton Encoding)

Instead of `FxHashMap<(usize, usize, usize), MaterialState>`, which causes CPU cache misses, we map 3D space into a 1D array using **Morton Codes (Z-Order Curve)**. Voxels close in physical space are close in computer RAM.

```rust
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
    
    size_nm: (i64, i64, i64),
}

impl VoxelGrid {
    fn is_empty(&self, x: usize, y: usize, z: usize) -> bool {
        let index = morton_encode(x as u32, y as u32, z as u32) as usize;
        let chunk_index = index / 64;
        let bit_index = index % 64;
        
        // Single bitwise AND operation - insanely fast
        (self.collision_mask[chunk_index] & (1u64 << bit_index)) == 0
    }
    
    fn set_occupied(&mut self, x: usize, y: usize, z: usize, material: u8, net: u32) {
        let index = morton_encode(x as u32, y as u32, z as u32) as usize;
        
        // Set material
        self.materials[index] = material;
        self.net_ids[index] = net;
        
        // Set collision bit
        let chunk_index = index / 64;
        let bit_index = index % 64;
        self.collision_mask[chunk_index] |= 1u64 << bit_index;
    }
    
    fn get_neighbors(&self, x: usize, y: usize, z: usize) -> [u8; 6] {
        // These 6 neighbors are guaranteed to be close in memory due to Z-curve
        [
            self.materials[morton_encode((x + 1) as u32, y as u32, z as u32) as usize],
            self.materials[morton_encode((x - 1) as u32, y as u32, z as u32) as usize],
            self.materials[morton_encode(x as u32, (y + 1) as u32, z as u32) as usize],
            self.materials[morton_encode(x as u32, (y - 1) as u32, z as u32) as usize],
            self.materials[morton_encode(x as u32, y as u32, (z + 1) as u32) as usize],
            self.materials[morton_encode(x as u32, y as u32, (z - 1) as u32) as usize],
        ]
    }
}
```

**Performance impact**:
```
HashMap approach:
  Routing 10,000 nets = 60,000 neighbor queries
  60,000 cache misses = ~600ms of RAM waiting

Morton Z-curve approach:
  Routing 10,000 nets = 60,000 neighbor queries
  Most queries hit L1 cache = ~6ms total

100× performance improvement for routing
```

This architecture scales from simple LED circuits to complex motherboards without modification.

### Fixed-Point Geometry Data Structures

The routing engine operates on integer-based geometric primitives to guarantee determinism. All coordinates use **i64 nanometers** for perfect reproducibility across all CPU architectures.

**Fixed-Point 3D Coordinates**:

```rust
/// Fixed-point 3D coordinate in nanometers
#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub struct Point3D {
    pub z: i64,  // Layer (nanometers)
    pub x: i64,  // Horizontal (nanometers)
    pub y: i64,  // Vertical (nanometers)
}

impl Point3D {
    pub fn new(z: i64, x: i64, y: i64) -> Self {
        Self { z, x, y }
    }
    
    pub fn from_mm(z: f64, x: f64, y: f64) -> Self {
        Self {
            z: (z * 1_000_000.0) as i64,
            x: (x * 1_000_000.0) as i64,
            y: (y * 1_000_000.0) as i64,
        }
    }
    
    pub fn to_mm(&self) -> (f64, f64, f64) {
        (
            self.z as f64 / 1_000_000.0,
            self.x as f64 / 1_000_000.0,
            self.y as f64 / 1_000_000.0,
        )
    }
    
    pub fn manhattan_distance(&self, other: &Point3D) -> i64 {
        (self.z - other.z).abs() +
        (self.x - other.x).abs() +
        (self.y - other.y).abs()
    }
}
```

**Manhattan Directions**:

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum Direction {
    North, South, East, West, Up, Down
}

impl Point3D {
    pub fn move_direction(&self, dir: Direction, distance_nm: i64) -> Point3D {
        match dir {
            Direction::North => Point3D::new(self.z, self.x, self.y + distance_nm),
            Direction::South => Point3D::new(self.z, self.x, self.y - distance_nm),
            Direction::East  => Point3D::new(self.z, self.x + distance_nm, self.y),
            Direction::West  => Point3D::new(self.z, self.x - distance_nm, self.y),
            Direction::Up    => Point3D::new(self.z + distance_nm, self.x, self.y),
            Direction::Down  => Point3D::new(self.z - distance_nm, self.x, self.y),
        }
    }
}
```

**Axis-Aligned Bounding Boxes**:

```rust
/// Axis-Aligned Bounding Box (integer coordinates)
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct BoundingBox {
    pub min: Point3D,
    pub max: Point3D,
}

impl BoundingBox {
    pub fn new(min: Point3D, max: Point3D) -> Self {
        Self { min, max }
    }
    
    pub fn from_point(point: Point3D, size_nm: i64) -> Self {
        Self {
            min: point,
            max: Point3D::new(
                point.z + size_nm,
                point.x + size_nm,
                point.y + size_nm,
            ),
        }
    }
    
    pub fn intersects(&self, other: &BoundingBox) -> bool {
        self.min.z <= other.max.z && self.max.z >= other.min.z &&
        self.min.x <= other.max.x && self.max.x >= other.min.x &&
        self.min.y <= other.max.y && self.max.y >= other.min.y
    }
    
    pub fn contains(&self, point: Point3D) -> bool {
        point.z >= self.min.z && point.z <= self.max.z &&
        point.x >= self.min.x && point.x <= self.max.x &&
        point.y >= self.min.y && point.y <= self.max.y
    }
    
    pub fn expand(&self, margin_nm: i64) -> BoundingBox {
        BoundingBox {
            min: Point3D::new(
                self.min.z - margin_nm,
                self.min.x - margin_nm,
                self.min.y - margin_nm,
            ),
            max: Point3D::new(
                self.max.z + margin_nm,
                self.max.x + margin_nm,
                self.max.y + margin_nm,
            ),
        }
    }
}
```

**Trace Segments**:

```rust
/// Manhattan-routed trace segment (integer coordinates)
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct TraceSegment {
    pub start: Point3D,
    pub end: Point3D,
    pub width_nm: i64,
}

impl TraceSegment {
    pub fn new(start: Point3D, end: Point3D, width_nm: i64) -> Self {
        Self { start, end, width_nm }
    }
    
    pub fn bounding_box(&self) -> BoundingBox {
        let half_width = self.width_nm / 2;
        
        BoundingBox {
            min: Point3D::new(
                self.start.z.min(self.end.z) - half_width,
                self.start.x.min(self.end.x) - half_width,
                self.start.y.min(self.end.y) - half_width,
            ),
            max: Point3D::new(
                self.start.z.max(self.end.z) + half_width,
                self.start.x.max(self.end.x) + half_width,
                self.start.y.max(self.end.y) + half_width,
            ),
        }
    }
    
    pub fn length(&self) -> i64 {
        self.start.manhattan_distance(&self.end)
    }
    
    pub fn is_horizontal(&self) -> bool {
        self.start.z == self.end.z && self.start.y == self.end.y
    }
    
    pub fn is_vertical(&self) -> bool {
        self.start.z == self.end.z && self.start.x == self.end.x
    }
    
    pub fn is_via(&self) -> bool {
        self.start.x == self.end.x && self.start.y == self.end.y
    }
}
```

**Why Fixed-Point Math Matters**:

```rust
// Floating-point (non-deterministic)
let clearance = 5.2 * 1.5;  // Different results on x86 vs ARM
// x86: 7.799999952316284
// ARM: 7.800000047683716

// Fixed-point (deterministic)
let clearance_nm = 5_200_000 * 15 / 10;  // Always 7_800_000
// x86: 7_800_000
// ARM: 7_800_000
// Identical on all platforms
```

These data structures are used throughout the routing engine (Book 5) to ensure perfect reproducibility.

---

## Layer 4: Physics IR (hwc-physics)

### What Happens Here

Before generating output files, the `hwc-physics` crate validates the physical design against the laws of physics. It sweeps the voxel grid and checks:

- Electrical properties (resistance, voltage drop, current capacity)
- Thermal properties (temperature rise, heat dissipation)
- Electromagnetic properties (impedance, signal integrity)
- Clearance requirements (dielectric breakdown prevention)

**Routing Sub-Pipeline**: Within this layer, Phase 3 of the 3-Phase Routing Sub-Pipeline (Design Rule Check) executes to validate the routed design. See Book 5 (Routing & Physics) for detailed validation algorithms and DRC rules.

### The Physics Engine

```rust
pub struct ElectricalAnalyzer {
    materials: MaterialDatabase,
}

impl ElectricalAnalyzer {
    pub fn analyze_resistance(
        &self,
        space: &HardwareSpace,
        net: &NetRoute
    ) -> Result<PhysicsReport, PhysicsError> {
        // Count copper voxels in trace
        let trace_voxels = self.count_trace_voxels(space, net);
        
        // Get material properties
        let copper = self.materials.get_conductor("copper")?;
        let resistivity = copper.resistivity_ohm_m;
        
        // Calculate geometry (using fixed-point math)
        let length_nm = trace_voxels as i64 * space.voxel_size_nm;
        let width_nm = net.width_voxels as i64 * space.voxel_size_nm;
        let thickness_nm = 35_000;  // 1oz copper = 35 micrometers
        
        // Convert to meters for physics calculation
        let length_m = length_nm as f64 / 1_000_000_000.0;
        let width_m = width_nm as f64 / 1_000_000_000.0;
        let thickness_m = thickness_nm as f64 / 1_000_000_000.0;
        let area_m2 = width_m * thickness_m;
        
        // Calculate resistance: R = ρ × (L / A)
        let resistance = resistivity * (length_m / area_m2);
        
        // Validate against constraints
        if resistance > net.constraints.max_resistance {
            return Err(PhysicsError::ResistanceTooHigh {
                net: net.name.clone(),
                actual: resistance,
                max: net.constraints.max_resistance,
            });
        }
        
        Ok(PhysicsReport {
            resistance,
            voltage_drop: net.current * resistance,
            power_dissipation: net.current.powi(2) * resistance,
        })
    }
}
```

### The Validation Process

```rust
pub fn validate_physics(
    space: &HardwareSpace,
    ir: &HardwareIR
) -> Result<ValidationReport, PhysicsError> {
    let mut report = ValidationReport::new();
    
    let electrical = ElectricalAnalyzer::new(&ir.materials);
    let thermal = ThermalAnalyzer::new(&ir.materials);
    let electromagnetic = ElectromagneticAnalyzer::new(&ir.materials);
    
    // Validate each net
    for net in &ir.nets {
        let elec_report = electrical.analyze_resistance(space, net)?;
        report.add_electrical(net.id, elec_report);
        
        let thermal_report = thermal.analyze_temperature(space, net)?;
        report.add_thermal(net.id, thermal_report);
        
        if let Some(impedance) = net.constraints.impedance {
            let si_report = electromagnetic.analyze_impedance(space, net, impedance)?;
            report.add_signal_integrity(net.id, si_report);
        }
    }
    
    // Validate clearances
    for (net_a, net_b) in ir.nets.iter().tuple_combinations() {
        let clearance = calculate_clearance(space, net_a, net_b);
        let required = calculate_required_clearance(net_a, net_b, &ir.materials);
        
        if clearance < required {
            report.add_error(PhysicsError::ClearanceViolation {
                net_a: net_a.name.clone(),
                net_b: net_b.name.clone(),
                actual: clearance,
                required,
            });
        }
    }
    
    Ok(report)
}
```

This layer validates the design without modifying it, ensuring physics compliance before manufacturing.

---

## Layer 5: Manufacturing Layer (hwc-export)

### What Happens Here

The validated voxel grid is converted into industry-standard manufacturing formats. This layer is pure translation — no validation, no modification, just format conversion.

### The Export Engine

This layer utilizes Custom Emitters to generate output files without bloated third-party dependencies. For the complete implementation of the GerberEmitter, GdsiiEmitter, and QasmExporter, see Book 6 (Exports & Assets).

### Supported Output Formats

- **Gerber files** (`.gtl`, `.gbl`) — PCB manufacturing
- **GDSII files** (`.gds`) — Silicon chip manufacturing
- **3D models** (`.obj`, `.glb`, `.step`) — Visualization and mechanical CAD
- **Blender scripts** (`.py`) — Photorealistic rendering
- **BOM files** (`.csv`) — Bill of materials
- **Drill files** (`.drl`) — Via specifications

---

## The 8 Rust Crates

Hardware Script's compiler is organized into 8 focused crates, each with a single responsibility.

### Crate Architecture

```
hwc-cli        - Command-line interface and pipeline orchestration
hwc-parser     - Lexer, parser, and AST construction
hwc-compiler   - AST to IR compilation and constraint resolution
hwc-engine     - Voxel grid management and spatial operations
hwc-physics    - Physics validation (electrical, thermal, EM)
hwc-export     - Output file generation (Gerber, GDSII, 3D)
hwc-materials  - Material database and property lookup
hwc-stdlib     - Standard component library and templates
```

### Crate Responsibilities

**hwc-parser**:
- Tokenization and lexing
- Grammar-based parsing (using Pest)
- AST construction
- Syntax validation

**hwc-compiler**:
- Import resolution
- Loop unrolling
- Coordinate transformation (local → global, with configurable XY and Z origin points)
- Constraint loading from profiles
- Hardware IR construction

For details on the unified `origin: XY by Z` syntax and coordinate transformation math, see `Docs/v0.1.2/COORDINATE-SYSTEM-ABSTRACTION.md`.

**hwc-materials**:
- Material property database (YAML-based)
- Conductor, dielectric, and substrate properties
- Custom material loading
- Physics parameter lookup

**hwc-stdlib**:
- Standard component definitions
- Footprint library
- Template generators
- Community-contributed components

**hwc-engine**:
- Sparse voxel grid (`FxHashMap`)
- Component placement
- Route interpolation
- Collision detection
- Bounding box calculations

**hwc-physics**:
- Electrical analysis (resistance, voltage drop, ampacity)
- Thermal analysis (temperature rise, heat dissipation)
- Electromagnetic analysis (impedance, signal integrity)
- Clearance validation (dielectric breakdown)

**hwc-export**:
- Gerber generation (PCB)
- GDSII generation (silicon)
- 3D model export (OBJ, GLB, STEP)
- Blender script generation
- BOM generation

**hwc-cli**:
- Command-line argument parsing
- Pipeline orchestration
- User interaction
- Progress reporting
- Error display

### The Complete Pipeline

```rust
// In hwc-cli/src/main.rs

fn main() -> Result<()> {
    let args = parse_args();
    
    // 1. Parse to AST
    let source = std::fs::read_to_string(&args.input)?;
    let ast = hwc_parser::parse(&source)?;
    
    // 2. Compile to IR
    let ir = hwc_compiler::compile_to_ir(ast)?;
    
    // 3. Render to voxel grid
    let space = hwc_engine::HardwareSpace::from_ir(&ir)?;
    
    // 4. Validate physics
    let validation = hwc_physics::validate_physics(&space, &ir)?;
    if !validation.is_valid() {
        return Err(Error::PhysicsViolation(validation));
    }
    
    // 5. Generate outputs
    match args.target {
        Target::PCB => {
            let gerber = hwc_export::GerberExporter::new();
            gerber.export(&space, &ir)?;
        }
        Target::Viz => {
            let blender = hwc_export::BlenderExporter::new();
            blender.export(&space, &ir)?;
        }
        Target::Sim => {
            let sim = hwc_export::SimulationExporter::new();
            sim.export(&space, &ir)?;
        }
    }
    
    Ok(())
}
```

---

## Error Handling Philosophy

Hardware Script treats physics violations with the same developer experience as Rust's borrow checker errors. We use `miette` and `thiserror` to generate world-class diagnostics with short, memorable error codes.

**Note**: This section focuses on the Rust implementation of error handling. For complete error handling philosophy and the DSTV insight, see `Docs/v0.1.2/ERROR-HANDLING-PHILOSOPHY.md`. For user-facing error examples, see Book 2 (Language Spec), Part VIII: Error Handling and Validation.

### The 3-Character Error Code System

**Format**: `[Letter][Digit][Digit]` (e.g., P16, R12, S22)

**Subsystems**:
- **S** - Syntax errors (parser)
- **C** - Compiler errors (IR & logic)
- **R** - Routing & engine errors (physical placement)
- **P** - Physics errors (design rule check)
- **M** - Manufacturing errors (export & fabrication)

**Why 3 characters?**
- ✅ Speakable: "I'm getting P-sixteen"
- ✅ Memorable: Short enough to remember and share
- ✅ Searchable: "Hardware Script P16" finds documentation instantly
- ✅ Universal: Becomes the vocabulary of the community

**Inspired by**: DSTV satellite TV error codes (E16, E48) that became part of everyday language across Africa.

### The Approach

The compiler's job is to produce beautiful, actionable error messages with short codes. The developer decides whether to fix manually or use an LLM. We never embed LLM calls in the compiler — it must remain fast, offline, and deterministic.

### Rust Implementation

This Rust struct generates the famous P16 (Dielectric Breakdown) error. Here's how to implement it using `miette` and `thiserror`:

```rust
use miette::{Diagnostic, SourceSpan};
use thiserror::Error;

#[derive(Error, Diagnostic, Debug)]
#[error("Dielectric Breakdown Risk")]
#[diagnostic(
    code(P16),  // Short, memorable 3-character code
    url("https://docs.hw-script.org/errors/P16"),
    help("The voltage difference is {voltage_diff}V. Through {material}, this requires a minimum clearance of {required_gap}mm to prevent arcing.")
)]
pub struct ClearanceViolation {
    #[source_code]
    pub src: String,
    
    #[label("High voltage net ({voltage_a}V)")]
    pub net_a_span: SourceSpan,
    
    #[label("Approaches within {actual_gap}mm of {net_b_name} net here")]
    pub collision_span: SourceSpan,
    
    pub voltage_diff: f32,
    pub material: String,
    pub required_gap: f32,
    pub actual_gap: f32,
    pub net_b_name: String,
    pub voltage_a: f32,
}
```

**Terminal output**:
```
❌ Error[P16]: Dielectric Breakdown Risk
   ╭─[board.hw:42:1]
 42 │     route Power_120V.out to Relay.in:
   ·           ──────┬───────
   ·                 ╰── High voltage net (120V)
 45 │             - [1, 50, 60]
   ·               ─────┬─────
   ·                    ╰── Approaches within 0.02mm of MCU_5V net here
   ╰────
  help: The voltage difference is 115V. Through Air, this requires a 
        minimum clearance of 0.08mm to prevent arcing.
```

**User conversation**:
- "I'm getting P16 when I compile."
- "P16 is clearance. What's your voltage difference?"
- "120V to 5V."
- "You need at least 0.08mm clearance through air."

This approach:
- Teaches users hardware engineering through errors
- Provides full context for LLM assistance
- Maintains compiler purity (no external API calls)
- Follows the UNIX philosophy (do one thing well)
- Creates a shared vocabulary (P16 becomes legendary like DSTV's E16)

---

## Why Rust Is Perfect for This

### Performance

Rust provides C/C++ level performance with zero-cost abstractions. Compilation times are measured in milliseconds, not minutes.

### Safety

The ownership system prevents memory leaks and data races. No garbage collector overhead. Thread safety guaranteed at compile time.

### Ecosystem

#### The Hyper-Lean Rust Stack

Hardware Script rejects "Library Maximalism". Every generic library imported brings edge-cases and memory overhead. We aggressively own our core abstractions.

Our `Cargo.toml` contains only **5 external dependencies**:

- **logos** — Insanely fast lexing (feeds our hand-written recursive descent parser)
- **miette & thiserror** — World-class, rustc-style terminal diagnostics
- **rayon** — Multi-threaded data parallelism
- **serde & serde_yaml** — Parsing configuration and material databases

**Everything else is pure, custom, Hardware Script IP**:

- **No parser combinators** (Pest/Chumsky/Tree-sitter) → We use a hand-written parser for 100% control over error recovery
- **No generic 3D math** (Nalgebra/Glam) → We use custom integer-based Manhattan Math
- **No graph libraries** (Petgraph) → We use our custom NetlistArena

**Why this matters**:
```
Library Maximalism:
  Dependencies: 15+ libraries
  Compile time: 2+ minutes
  Binary size: 50MB
  Control: Low (at mercy of library design)

Hyper-Lean Approach:
  Dependencies: 5 libraries
  Compile time: 10 seconds
  Binary size: 5MB
  Control: Complete (you own the code)
```

### Distribution

Rust compiles to single, standalone executables with no runtime dependencies. Users download one binary and run it immediately — no Python installation, no dependency hell, no virtual environments.

---

## The Semantic Boundary

Hardware Script maintains a clean semantic boundary: `.hw` files describe geometry and connections, not behavioral logic.

### What Hardware Script Is NOT

We are not building:
- A better Verilog (no behavioral blocks like `Sum = A + B`)
- An open-source Cadence (no black-box place & route)
- A simplified Altium (no GUI-first design)

### What Hardware Script IS

We are building:
- Software-Defined Bare-Metal
- The LLVM for Physical Reality
- Git-friendly hardware design
- Deterministic, reproducible compilation
- LLM-native workflows

### The Distinction

**Traditional EDA**:
```
Behavioral description (Verilog)
    ↓
Logic synthesis (black box)
    ↓
Place & Route (black box)
    ↓
Physical layout
```

**Hardware Script**:
```
Physical topology (.hw)
    ↓
Constraint resolution (IR)
    ↓
Voxel grid (deterministic)
    ↓
Physics validation (transparent)
    ↓
Manufacturing files
```

Every step is transparent, deterministic, and version-controllable.

---

## Contributing to the Compiler

### Getting Started

1. Clone the repository
2. Install Rust (rustup.rs)
3. Run `cargo build` in the `hwc/` directory
4. Run tests with `cargo test`

### Architecture Guidelines

**Respect layer boundaries**:
- Parser only builds AST
- Compiler only builds IR
- Engine only manipulates voxels
- Physics only validates
- Export only generates files

**Keep crates focused**:
- Each crate has one responsibility
- No circular dependencies
- Clean, documented interfaces

**Write excellent errors**:
- Use miette for diagnostics
- Provide actionable hints
- Include physics explanations
- Link to documentation

### Areas for Contribution

**Parser enhancements**:
- Better error recovery
- Incremental parsing
- Tree-sitter integration

**Compiler optimizations**:
- IR optimization passes
- Parallel compilation
- Incremental builds

**Engine improvements**:
- Advanced routing algorithms
- GPU acceleration
- Multi-layer support

**Physics validation**:
- More material properties
- Advanced EM simulation
- Thermal modeling

**Export formats**:
- Additional CAD formats
- Simulation outputs
- Documentation generation

---

## Conclusion

Hardware Script's compiler architecture is revolutionary because it unifies what has historically been fragmented across proprietary tools. By using a multi-level IR pipeline, sparse voxel storage, and Rust's performance and safety guarantees, we've built a system that scales from simple LED circuits to complex silicon chips.

The architecture is proven, the implementation is clean, and the ecosystem is ready for growth. If you're a Rust systems programmer who wants to help build the future of hardware design, we'd love your contributions.

Welcome to the compiler internals. Let's build something amazing.
