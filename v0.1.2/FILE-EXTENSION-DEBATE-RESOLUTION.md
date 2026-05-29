# Hardware Script v0.1.2 - File Extension Debate Resolution

**Document Type**: Architectural Debate and Final Decision  
**Status**: External Research Integration  
**Last Updated**: March 2026

---

## Purpose of This Document

This document captures a critical architectural debate about Hardware Script's file extension system and provides the definitive resolution based on the actual Rust implementation.

**The debate**: Should Hardware Script add more file extensions for different abstraction levels (logic, FPGA, SPICE, system architecture), or keep the current 10-file ecosystem and use compiler targets instead?

**The resolution**: Lock the file extensions. Use compiler targets and explicit abstraction blocks.

---

## The Debate Context

### The Question

"Is there still a gap of people not represented in the file extension ecosystem?"

**Specifically**:
- Logic designers (HDL/Verilog level)
- FPGA engineers
- SPICE simulation users
- System architects
- Different abstraction levels

### The Two Positions

**Position A: Add more file extensions**
```
.hwlogic  - For behavioral logic
.hwfpga   - For FPGA-specific code
.hwspice  - For analog simulation
.hwsys    - For system architecture
```

**Position B: Keep 10 files, use compiler targets**
```
Same .hw language
Different compiler interpretations
Multiple targets (--target pcb, --target fpga, etc.)
```

---

## The Core Architectural Principle

### The LLVM Parallel

**The key insight from the debate**:

> Use one core language and expand capability via libraries, targets, and profiles — not by multiplying file types.

**This philosophy is exactly how successful ecosystems stayed coherent**:
- **Rust**: One language, multiple targets (x86, ARM, WASM)
- **TypeScript**: One language, multiple outputs (ES5, ES6, CommonJS, ESM)
- **Go**: One language, multiple platforms

**Hardware Script should follow the same pattern.**

---

## The Risk: The C++ Trap

### What Could Go Wrong

If `.hw` tries to be everything all at once without clear boundaries, it becomes:

**C++**: Powerful but impossible to reason about

Instead of:

**Rust**: Strict, readable, predictable

### The Historical Evidence

**In EDA tools, different abstraction levels have required different semantics**:

| Level | What you describe | Typical representation |
|-------|-------------------|------------------------|
| System | Data flow | Block diagrams |
| Logic | Gates | HDL (Verilog/VHDL) |
| Circuit | Transistors | SPICE |
| Layout | Geometry | GDSII/Gerber |

**Companies like Cadence and Synopsys split these because the mathematical models are completely different.**

**Example**:

**Logic**:
```
Sum = A + B
```

**Circuit**:
```
MOSFET transistor physics
Voltage curves
Capacitance
```

**Layout**:
```
Polygons at (x, y, z)
Copper traces
Via positions
```

**If `.hw` becomes responsible for all three without clear boundaries, the compiler must internally manage multiple semantic modes.**

**That is possible — but very hard.**

---

## The Solution: Explicit Abstraction Blocks

### The Semantic Boundary Strategy

**Just like**:
- HTML has `<style>` for CSS and `<script>` for JS
- Rust has `unsafe {}` blocks
- C has `extern "C"` blocks

**`.hw` must use strict block scoping.**

**You don't mix logic and physical geometry on the same line.**

### Example: Component with Multiple Abstractions

```hw
define Component "Custom_ALU":
    # 1. INTERFACE (The Pins/Ports)
    # Universal across all targets
    pins:
        A[8]
        B[8]
        Sum[8]
        Carry
        VCC
        GND
    
    # 2. BEHAVIOR (The Logic / HDL level)
    # Compiler synthesizes this into gates for FPGA/ASIC targets
    # Ignored for PCB target (treated as black box)
    behavior:
        Sum = A + B
        Carry = (A[7] AND B[7]) OR ((A[7] XOR B[7]) AND CarryIn)
    
    # 3. PHYSICAL (The Geometry / Layout level)
    # Used for PCB placement and mechanical constraints
    # Used for ASIC layout generation
    layout:
        shape: Rectangle(10mm, 10mm, 2mm)
        pin_spacing: 0.5mm
        pin_positions:
            A[0..7] at left_edge
            B[0..7] at right_edge
            Sum[0..7] at top_edge
            VCC at [0, 0]
            GND at [10, 10]
    
    # 4. ELECTRICAL (The Circuit / SPICE level)
    # Used for analog simulation
    # Optional for digital components
    electrical:
        max_voltage: 5V
        max_current: 100mA
        input_capacitance: 10pF
        output_resistance: 50Ω
```

### How Compiler Targets Interpret This

**Target: PCB**
```
Reads: pins, layout
Ignores: behavior, electrical (treats as black box)
Output: Gerber files with component footprint
```

**Target: FPGA**
```
Reads: pins, behavior
Ignores: layout (FPGA tools handle placement)
Output: Verilog/VHDL for synthesis
```

**Target: ASIC**
```
Reads: pins, behavior, layout, electrical
Uses all information for full chip design
Output: GDSII with synthesized gates
```

**Target: SPICE**
```
Reads: pins, electrical
Ignores: behavior (or converts to SPICE model)
Output: SPICE netlist for analog simulation
```

**Target: Simulation**
```
Reads: pins, behavior
Uses behavioral model for fast simulation
Output: .hwx simulation executable
```

### The Key Insight

**By forcing the user to separate**:
- `pins:` (interface)
- `behavior:` (logic)
- `layout:` (geometry)
- `electrical:` (circuit)

**The language stays universally readable.**

**The compiler chooses which blocks to process based on the target.**

---

## The Current Rust Implementation

### What You Already Have

Looking at your actual code:

```rust
// Cargo.toml workspace
[workspace]
members = [
    "crates/hwc-cli",
    "crates/hwc-parser",
    "crates/hwc-compiler",
    "crates/hwc-engine",
    "crates/hwc-physics",
    "crates/hwc-export",
    "crates/hwc-materials",
    "crates/hwc-stdlib",
]
```

**You have the perfect 8-crate architecture.**

### The Material State Enum

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[repr(u8)]
pub enum MaterialState {
    Air = 0,
    FR4 = 1,
    Copper = 2,
    Pad = 3,
    Body = 4,
    Silicon = 5,
    Gold = 6,
    Aluminum = 7,
}
```

**This is clean, efficient, and extensible.**

### The Sparse Voxel Engine

```rust
pub struct HardwareSpace {
    pub name: String,
    pub dimensions: Dimensions,
    pub grid: GridCells,
    pub voxel_size: VoxelSize,
    pub voxels: FxHashMap<(usize, usize, usize), u8>,
    pub substrate: MaterialState,
}
```

**Using `FxHashMap` for sparse storage is brilliant.**

**Memory stats method shows the efficiency**:
```rust
pub fn memory_stats(&self) -> String {
    format!("Voxels: {} / {} ({:.2}% sparse)",
        self.voxels.len(),
        self.grid.x_cols * self.grid.y_rows * self.grid.z_layers,
        (self.voxels.len() as f64 / 
         (self.grid.x_cols * self.grid.y_rows * self.grid.z_layers) as f64) * 100.0
    )
}
```

---

## The Missing Piece: Hardware IR Layer

### Current Pipeline (Needs Improvement)

```rust
// In hwc-compiler/src/compiler.rs
pub fn compile_file(&mut self, path: &Path) -> Result<CompiledOutput, Box<dyn std::error::Error>> {
    // 1. Parse to AST
    let source = std::fs::read_to_string(path)?;
    let parser = Parser::new();
    let ast = parser.parse(&source)?;
    
    // 2. Initialize space
    let mut space = HardwareSpace::new(...);
    
    // 3. Direct voxel manipulation (PROBLEM: No IR layer)
    let placer = ComponentPlacer::new();
    for component in &ast.components {
        placer.place_component(&mut space, &component.component_type, component.position)?;
    }
    
    let router = Router::new();
    for route in &ast.routes {
        router.route_trace(&mut space, &route.from, &route.waypoints, 1)?;
    }
    
    Ok(CompiledOutput { space })
}
```

**The problem**: Going straight from AST to voxel manipulation.

**Missing**: Hardware IR for constraint resolution and abstraction handling.

### The Correct Pipeline

```
hwc-parser: Source → AST
    ↓
hwc-compiler: AST → Hardware IR
    ↓
hwc-engine: Hardware IR → Voxel Grid
    ↓
hwc-physics: Voxel Grid → Validation
    ↓
hwc-export: Validated Grid → Files
```

### The Hardware IR Structure

```rust
// In hwc-compiler/src/ir.rs (NEW FILE NEEDED)

/// Hardware Intermediate Representation
/// Pure data structure representing resolved hardware design
pub struct HardwareIR {
    /// Metadata
    pub space_name: String,
    pub dimensions: (f64, f64, f64),  // mm
    pub grid: (usize, usize, usize),
    pub voxel_size: (f64, f64, f64),
    
    /// Target-specific information
    pub target: CompilationTarget,
    
    /// Resolved components with global coordinates
    pub components: Vec<PlacedComponent>,
    
    /// Resolved nets with waypoints and constraints
    pub nets: Vec<NetRoute>,
    
    /// Behavioral blocks (for FPGA/ASIC targets)
    pub behaviors: Vec<BehaviorBlock>,
    
    /// Material assignments
    pub materials: MaterialDatabase,
    
    /// Constraints from .hwp, .hwsig, .hwtc
    pub constraints: ConstraintSet,
}

/// Compilation target determines which blocks to process
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum CompilationTarget {
    PCB,        // Physical board layout
    FPGA,       // FPGA bitstream
    ASIC,       // Silicon chip
    SPICE,      // Analog simulation
    Simulation, // Behavioral simulation
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
    
    // Optional blocks (presence depends on component definition)
    pub behavior: Option<BehaviorBlock>,
    pub layout: Option<LayoutBlock>,
    pub electrical: Option<ElectricalBlock>,
}

/// Behavioral logic block (for FPGA/ASIC synthesis)
pub struct BehaviorBlock {
    pub expressions: Vec<LogicExpression>,
    pub state_machines: Vec<StateMachine>,
}

/// Layout geometry block (for PCB/ASIC placement)
pub struct LayoutBlock {
    pub shape: Shape,
    pub pin_positions: HashMap<String, (f64, f64, f64)>,  // mm
    pub keepout_zones: Vec<KeepoutZone>,
}

/// Electrical characteristics block (for SPICE simulation)
pub struct ElectricalBlock {
    pub max_voltage: f64,
    pub max_current: f64,
    pub input_capacitance: f64,
    pub output_resistance: f64,
    pub spice_model: Option<SpiceModel>,
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

### The Improved Compilation Process

```rust
// In hwc-compiler/src/compiler.rs (IMPROVED)

pub fn compile_file(
    &mut self, 
    path: &Path,
    target: CompilationTarget
) -> Result<CompiledOutput, Box<dyn std::error::Error>> {
    if self.verbose {
        println!("📖 Reading: {}", path.display());
        println!("🎯 Target: {:?}", target);
    }
    
    // 1. Parse to AST
    let source = std::fs::read_to_string(path)?;
    let parser = Parser::new();
    let ast = parser.parse(&source)?;
    
    if self.verbose {
        println!("✅ Parsed successfully");
    }
    
    // 2. Compile to Hardware IR
    let ir = self.compile_to_ir(ast, target)?;
    
    if self.verbose {
        println!("✅ Built Hardware IR");
        println!("   - Components: {}", ir.components.len());
        println!("   - Nets: {}", ir.nets.len());
        println!("   - Behaviors: {}", ir.behaviors.len());
    }
    
    // 3. Target-specific processing
    match target {
        CompilationTarget::PCB => {
            // Render to voxel grid
            let space = HardwareSpace::from_ir(&ir)?;
            
            // Validate physics
            let validation = validate_physics(&space, &ir)?;
            if !validation.is_valid() {
                return Err(Box::new(validation.into_error()));
            }
            
            if self.verbose {
                println!("   {}", space.memory_stats());
            }
            
            Ok(CompiledOutput::PCB { space, ir })
        }
        
        CompilationTarget::FPGA => {
            // Synthesize behavioral blocks to HDL
            let hdl = synthesize_to_hdl(&ir)?;
            
            if self.verbose {
                println!("   Generated {} lines of Verilog", hdl.line_count());
            }
            
            Ok(CompiledOutput::FPGA { hdl, ir })
        }
        
        CompilationTarget::ASIC => {
            // Full chip design flow
            let space = HardwareSpace::from_ir(&ir)?;
            let hdl = synthesize_to_hdl(&ir)?;
            let validation = validate_physics(&space, &ir)?;
            
            Ok(CompiledOutput::ASIC { space, hdl, ir })
        }
        
        CompilationTarget::SPICE => {
            // Generate SPICE netlist
            let netlist = generate_spice_netlist(&ir)?;
            
            Ok(CompiledOutput::SPICE { netlist, ir })
        }
        
        CompilationTarget::Simulation => {
            // Generate simulation executable
            let sim = generate_simulation(&ir)?;
            
            Ok(CompiledOutput::Simulation { sim, ir })
        }
    }
}

fn compile_to_ir(
    &self,
    ast: AST,
    target: CompilationTarget
) -> Result<HardwareIR, CompileError> {
    let mut ir = HardwareIR::new(target);
    
    // Extract space definition
    let space_def = ast.space.ok_or("No space definition")?;
    ir.space_name = space_def.name;
    ir.dimensions = space_def.dimensions;
    ir.grid = space_def.grid;
    
    // Load materials
    ir.materials = MaterialDatabase::load_standard()?;
    
    // Resolve components
    for component_ast in ast.components {
        let component_def = load_component_definition(&component_ast.component_type)?;
        
        let placed = PlacedComponent {
            id: ComponentId::new(),
            name: component_ast.instance_name,
            component_type: component_ast.component_type,
            position: component_ast.position,
            rotation: component_ast.rotation,
            pins: transform_pins_to_global(
                &component_def.pins,
                component_ast.position,
                component_ast.rotation
            ),
            bounding_box: calculate_bounding_box(&component_def, component_ast.position),
            
            // Include blocks based on target
            behavior: if target.needs_behavior() {
                component_def.behavior.clone()
            } else {
                None
            },
            layout: if target.needs_layout() {
                component_def.layout.clone()
            } else {
                None
            },
            electrical: if target.needs_electrical() {
                component_def.electrical.clone()
            } else {
                None
            },
        };
        
        ir.components.push(placed);
    }
    
    // Resolve nets
    for route_ast in ast.routes {
        let from_pin = resolve_pin_reference(&route_ast.from, &ir.components)?;
        let to_pin = resolve_pin_reference(&route_ast.to, &ir.components)?;
        
        let net = NetRoute {
            id: NetId::new(),
            name: format!("{} -> {}", route_ast.from, route_ast.to),
            from_pin,
            to_pin,
            waypoints: route_ast.waypoints,
            width_voxels: 1,  // TODO: Calculate from constraints
            material: MaterialId::Copper,
            constraints: RouteConstraints::default(),
        };
        
        ir.nets.push(net);
    }
    
    Ok(ir)
}
```

---

## The CLI Interface with Targets

### Command Structure

```bash
# PCB target (default)
hws build board.hw
hws build board.hw --target pcb

# FPGA target
hws build design.hw --target fpga

# ASIC target
hws build chip.hw --target asic

# SPICE simulation
hws build circuit.hw --target spice

# Behavioral simulation
hws build system.hw --target sim
```

### Implementation

```rust
// In hwc-cli/src/commands/build.rs

use clap::Parser;
use hwc_compiler::{Compiler, CompilationTarget};

#[derive(Parser)]
pub struct BuildCommand {
    /// Input .hw file
    pub input: PathBuf,
    
    /// Compilation target
    #[arg(long, default_value = "pcb")]
    pub target: TargetArg,
    
    /// Verbose output
    #[arg(short, long)]
    pub verbose: bool,
}

#[derive(Clone, Copy, Debug, clap::ValueEnum)]
pub enum TargetArg {
    Pcb,
    Fpga,
    Asic,
    Spice,
    Sim,
}

impl From<TargetArg> for CompilationTarget {
    fn from(arg: TargetArg) -> Self {
        match arg {
            TargetArg::Pcb => CompilationTarget::PCB,
            TargetArg::Fpga => CompilationTarget::FPGA,
            TargetArg::Asic => CompilationTarget::ASIC,
            TargetArg::Spice => CompilationTarget::SPICE,
            TargetArg::Sim => CompilationTarget::Simulation,
        }
    }
}

pub fn execute(cmd: BuildCommand) -> Result<(), Box<dyn std::error::Error>> {
    let mut compiler = Compiler::new();
    compiler.set_verbose(cmd.verbose);
    
    let target = cmd.target.into();
    let output = compiler.compile_file(&cmd.input, target)?;
    
    // Export based on target
    match output {
        CompiledOutput::PCB { space, ir } => {
            export_gerber(&space, &ir)?;
            export_3d_model(&space, &ir)?;
            export_bom(&ir)?;
        }
        CompiledOutput::FPGA { hdl, ir } => {
            export_verilog(&hdl)?;
            export_constraints(&ir)?;
        }
        CompiledOutput::ASIC { space, hdl, ir } => {
            export_gdsii(&space, &ir)?;
            export_verilog(&hdl)?;
        }
        CompiledOutput::SPICE { netlist, ir } => {
            export_spice_netlist(&netlist)?;
        }
        CompiledOutput::Simulation { sim, ir } => {
            export_hwc_binary(&sim)?;
        }
    }
    
    println!("✅ Build complete!");
    Ok(())
}
```

---

## The Final Verdict

### Lock the File Extensions

**The 10-file ecosystem is complete**:

```
1. hw.toml       - Project manifest
2. .hw           - Hardware source (with abstraction blocks)
3. .hwx          - Component definitions
4. .hwm          - Mechanical constraints
5. .hwmat        - Materials database
6. .hwf          - Firmware interface
7. .hwsig        - Signal integrity
8. .hwtc         - Timing constraints
9. .hwt          - Test benches
10. .hwp         - Fabrication profiles
11. .hwa         - Assembly instructions
```

**Do NOT add**:
- ❌ `.hwlogic`
- ❌ `.hwfpga`
- ❌ `.hwspice`
- ❌ `.hwsys`

### Use Explicit Abstraction Blocks

**In `.hw` and `.hwx` files**:

```hw
define Component "MyChip":
    pins: ...
    behavior: ...    # For FPGA/ASIC targets
    layout: ...      # For PCB/ASIC targets
    electrical: ...  # For SPICE target
```

**The compiler processes blocks based on target.**

### Use Compiler Targets

```bash
hws build design.hw --target pcb
hws build design.hw --target fpga
hws build design.hw --target asic
hws build design.hw --target spice
hws build design.hw --target sim
```

**Same language, different interpretations.**

---

## Who Is Represented Now?

### Complete Stakeholder Coverage

| Stakeholder | How They Interact |
|-------------|-------------------|
| Hobbyist | `.hw` with libraries |
| PCB Engineer | `.hw` + `.hwp` + `--target pcb` |
| RF Engineer | `.hwsig` + `.hw` |
| Mechanical Engineer | `.hwm` + `.hw` |
| Firmware Developer | `.hwf` + auto-generated headers |
| FPGA Engineer | `.hw` behavior blocks + `--target fpga` |
| ASIC Designer | `.hw` all blocks + `--target asic` |
| Analog Engineer | `.hw` electrical blocks + `--target spice` |
| System Architect | `.hw` with high-level components |
| Test Engineer | `.hwt` + simulation |
| Manufacturer | `.hwp` + `.hwa` |

**No major group is excluded.**

---

## Key Takeaways

1. **File extensions are locked at 10** - No more additions

2. **Use abstraction blocks** - `pins:`, `behavior:`, `layout:`, `electrical:`

3. **Use compiler targets** - `--target pcb|fpga|asic|spice|sim`

4. **Implement Hardware IR layer** - Between AST and voxel grid

5. **Target determines block processing** - FPGA reads behavior, PCB reads layout

6. **All stakeholders covered** - From hobbyists to ASIC designers

7. **Follow LLVM model** - One language, multiple targets

8. **Avoid C++ trap** - Clear semantic boundaries via blocks

9. **Rust implementation is excellent** - 8 crates, sparse voxels, clean separation

10. **Next step is IR layer** - Add `HardwareIR` struct and target-specific compilation

---

## Summary

The file extension debate is resolved: **Lock the 10-file ecosystem and use compiler targets.**

**The architecture**:
- One `.hw` language with explicit abstraction blocks
- Multiple compilation targets (`--target pcb|fpga|asic|spice|sim`)
- Hardware IR layer between AST and backend
- Target-specific block processing

**All stakeholders are represented**:
- PCB engineers use layout blocks
- FPGA engineers use behavior blocks
- ASIC designers use all blocks
- Analog engineers use electrical blocks

**The implementation strategy**:
1. Add `HardwareIR` struct to `hwc-compiler`
2. Implement target-specific compilation
3. Add abstraction block parsing to `hwc-parser`
4. Update `hwc-engine` to consume IR instead of AST
5. Add target selection to CLI

**You are building the LLVM for Physical Reality. Stay the course.**

---

**Document Status**: Architectural Debate and Final Decision  
**Last Updated**: March 2026  
**Part of**: Hardware Script v0.1.2 Documentation Suite
