# Book 6: 3D Assets & Export Generation

**Hardware Script v0.1.3**  
**Target Audience**: 3D artists, renderers, and manufacturing liaisons  
**Last Updated**: March 2026

---

## Foundation

This document consolidates the following source materials:

**Source Files**:
- `Docs/v0.1.1/3D-ASSET-MAPPING.md` — The dual identity system (mathematical brain vs. visual body), `.glb` asset importing, and render block architecture
- `Docs/v0.1.2/EXPORT-OPTIMIZATION-AND-POLISH.md` — FR4 substrate rendering, Gerber D01 Draw vs D03 Flash optimization
- `Docs/v0.1.2/HYPER-LEAN-ARCHITECTURE.md` — Custom emitters (GerberEmitter, GdsiiEmitter, QasmExporter)

---

## Introduction

This document is the blueprint for Hardware Script's export generation system. If you're a 3D artist, renderer, or manufacturing liaison who wants to understand how mathematical voxels transform into beautiful visualizations and factory-ready files, this is your guide.

Hardware Script completely rethinks the relationship between engineering and aesthetics. Traditional EDA tools treat 3D rendering as an afterthought, producing blocky visualizations that look nothing like real hardware. Hardware Script treats components like prefabs in a game engine, maintaining perfect mathematical precision while enabling photorealistic rendering.

---

## The Dual Identity System

### The Core Concept

Every component in Hardware Script has two identities that serve different purposes:

**1. The Mathematical Brain (Voxels)**
- Used for collision detection
- Used for routing calculations
- Used for Gerber manufacturing
- Used for physics validation
- Discrete, precise, computational

**2. The Visual Body (3D Mesh)**
- Used for photorealistic rendering
- Used for marketing materials
- Used for documentation
- Used for interactive visualization
- Continuous, beautiful, cinematic

**Key insight**: The compiler separates these concerns based on the `--target` flag.

### Why This Separation Matters

**The physics engine is happy**: The router sees a transistor as a 3×3 block of voxels. It won't let you route a wire through the middle of the component. Collision detection remains O(1) array lookup.

**The manufacturer is happy**: The Gerber exporter only prints the copper voxels. It ignores the 3D graphics entirely. Manufacturing files are pure mathematics, no graphics.

**The designer is amazed**: When they run `blender --python sim.py`, Blender doesn't draw blocky Minecraft-style components. It seamlessly connects beautifully calculated copper traces to photorealistic 3D component meshes.

---

## The render Block

**Note**: The `render:` block is one of the five official abstraction blocks in Hardware Script. For complete syntax specification and grammar rules, see Book 2 (Language Spec), Part III: Abstraction Blocks. This section focuses on how the compiler uses the render block for export generation.

### How It Works

When a creator defines a component in a `.hwx` file, they map the mathematical grid to a real 3D asset using the `render:` block. The syntax for mapping 3D assets via the render: block is strictly defined in Book 2 (Language Spec), Part III: Abstraction Blocks. This book explains how the compiler takes that parsed syntax and generates the Scene Graph.

---

## Package Structure

### Beyond Single Files

If a component includes 3D models and textures, it can no longer be just a single `.hwx` text file. When a user runs `hpm install ics/esp32`, the package manager downloads a lightweight directory:

```
~/.hw/packages/ics/esp32/
├── esp32.hwx          # The math, pins, and physics (The Brain)
└── assets/
    ├── chip.glb       # The 3D model and textures (The Body)
    └── footprint.png  # Optional silkscreen graphic
```

### Why .glb (GLTF) Format?

**GLTF is the "JPEG of 3D."**

**Benefits**:
- Highly compressed (small downloads)
- Packs mesh, colors, and textures into one file
- Industry standard for web and real-time 3D
- Native support in Blender, Three.js, Unity, Unreal
- Perfect for package managers

**Comparison**:
- `.obj` — Simple but no textures, no compression
- `.fbx` — Proprietary, bloated
- `.glb` — Open standard, compressed, includes everything ✅

### Installation Modes

**Lite Mode** (default):
```bash
hpm install ics/esp32_c3

# Downloads:
# - esp32_c3.hwx (math and physics only)
# - No 3D assets
# - Uses procedural fallback for visualization
```

**Full Mode** (with assets):
```bash
hpm install ics/esp32_c3 --full

# Downloads:
# - esp32_c3.hwx (math and physics)
# - assets/chip.glb (photorealistic 3D model)
# - assets/footprint.png (silkscreen)
```

---

## Target-Based Export Generation

### How Targets Work

When the compiler reaches Layer 5 (Manufacturing Layer) of the MLIR pipeline, the export generation is configured based on the target specified by the user. See Book 2 (Language Spec), Part VI: Compilation Targets for CLI usage and which language blocks are read for each target.

The export layer receives the validated Hardware IR and generates output files using Custom Emitters:

**For Manufacturing Targets** (pcb, asic):
- Reads validated voxel grid from Layer 3
- Reads material assignments from Layer 2
- Generates manufacturing files (Gerber, GDSII)
- Ignores render blocks

**For Visualization Targets** (viz, web):
- Reads validated voxel grid from Layer 3
- Reads render blocks from components
- Generates scene graphs and 3D assets
- Calculates exact physical copper traces using procedural geometry
- Imports high-fidelity 3D assets (`.glb`)
- Snaps components perfectly onto copper traces at exact mathematical coordinates

**For Simulation Targets** (sim, spice):
- Reads behavioral blocks from components
- Reads electrical properties from Layer 4
- Generates simulation netlists
- Ignores visual rendering

---

## Custom Emitters (Owning the Output)

### The Philosophy: No Bloated Third-Party Crates

We do not use bloated third-party crates (like `gerber-rs` or `gdsii-writer`) to generate outputs. Standard formats are simply ASCII text or sequential binary.

**Why this matters**:
- **Complete control** over buffering and performance
- **Exact format specifications** without library limitations
- **Zero dependencies** for export generation
- **No waiting** for upstream maintainers to merge features
- **Minimal binary size** - no generic overhead

By owning our Emitters, we control every aspect of the output generation.

### The Custom Emitter Architecture

All exporters implement a common trait:

```rust
pub trait Exporter {
    fn export(&self, board: &Board, ir: &HardwareIR) -> Result<Vec<OutputFile>, ExportError>;
}

pub struct OutputFile {
    pub name: String,
    pub content: Vec<u8>,
}
```

### Custom Gerber Emitter

Gerber files are ASCII text. We generate them directly using string formatting with fixed-point integer math.

```rust
pub struct GerberEmitter {
    buffer: String,
    current_x_nm: i64,
    current_y_nm: i64,
    current_aperture: Option<u32>,
}

impl GerberEmitter {
    pub fn new() -> Self {
        let mut emitter = Self {
            buffer: String::with_capacity(1024 * 1024),  // Pre-allocate 1MB
            current_x_nm: 0,
            current_y_nm: 0,
            current_aperture: None,
        };
        
        // Gerber header
        emitter.buffer.push_str("G04 Hardware Script Gerber Output*\n");
        emitter.buffer.push_str("%FSLAX36Y36*%\n");  // Format: 3.6 (microns)
        emitter.buffer.push_str("%MOMM*%\n");        // Units: millimeters
        emitter.buffer.push_str("%LPD*%\n");         // Layer polarity: dark
        
        emitter
    }
    
    pub fn define_aperture(&mut self, id: u32, diameter_nm: i64) {
        let diameter_mm = diameter_nm as f64 / 1_000_000.0;
        self.buffer.push_str(&format!("%ADD{}C,{:.6}*%\n", id, diameter_mm));
    }
    
    pub fn select_aperture(&mut self, id: u32) {
        if self.current_aperture != Some(id) {
            self.buffer.push_str(&format!("D{}*\n", id));
            self.current_aperture = Some(id);
        }
    }
    
    pub fn draw_trace(&mut self, start: Point3D, end: Point3D) {
        // Point3D uses i64 nanometers. We divide to get standard Gerber formatting.
        let start_x = (start.x / 1000) as i32;  // Convert nm to microns
        let start_y = (start.y / 1000) as i32;
        let end_x = (end.x / 1000) as i32;
        let end_y = (end.y / 1000) as i32;
        
        // Move to start (pen up)
        if start_x != (self.current_x_nm / 1000) as i32 || 
           start_y != (self.current_y_nm / 1000) as i32 {
            self.buffer.push_str(&format!("X{:06}Y{:06}D02*\n", start_x, start_y));
            self.current_x_nm = start.x;
            self.current_y_nm = start.y;
        }
        
        // Draw to end (pen down)
        self.buffer.push_str(&format!("X{:06}Y{:06}D01*\n", end_x, end_y));
        self.current_x_nm = end.x;
        self.current_y_nm = end.y;
    }
    
    pub fn flash_pad(&mut self, position: Point3D) {
        let x = (position.x / 1000) as i32;
        let y = (position.y / 1000) as i32;
        
        self.buffer.push_str(&format!("X{:06}Y{:06}D03*\n", x, y));
    }
    
    pub fn finish(mut self) -> String {
        self.buffer.push_str("M02*\n");  // End of file
        self.buffer
    }
}

impl Exporter for GerberEmitter {
    fn export(&self, board: &Board, _ir: &HardwareIR) -> Result<Vec<OutputFile>, ExportError> {
        let mut emitter = GerberEmitter::new();
        
        // Define apertures
        emitter.define_aperture(10, 100_000);  // 0.1mm trace
        emitter.define_aperture(11, 500_000);  // 0.5mm pad
        
        // Draw traces
        emitter.select_aperture(10);
        for trace in &board.traces {
            for segment in &trace.segments {
                emitter.draw_trace(segment.start, segment.end);
            }
        }
        
        // Flash pads
        emitter.select_aperture(11);
        for component in &board.components {
            for pin in &component.pins {
                emitter.flash_pad(pin.position);
            }
        }
        
        let content = emitter.finish();
        
        Ok(vec![OutputFile {
            name: "board_top.gtl".to_string(),
            content: content.into_bytes(),
        }])
    }
}
```

### Custom GDSII Emitter

GDSII files are binary. We write them directly using byte manipulation.

```rust
pub struct GdsiiEmitter {
    buffer: Vec<u8>,
}

impl GdsiiEmitter {
    pub fn new(library_name: &str) -> Self {
        let mut emitter = Self {
            buffer: Vec::with_capacity(1024 * 1024),  // Pre-allocate 1MB
        };
        
        // GDSII header
        emitter.write_record(0x0002, &[0x02, 0x58]);  // HEADER (version 600)
        emitter.write_record(0x0102, &Self::string_to_bytes(library_name));
        
        emitter
    }
    
    fn write_record(&mut self, record_type: u16, data: &[u8]) {
        let length = (4 + data.len()) as u16;
        self.buffer.extend_from_slice(&length.to_be_bytes());
        self.buffer.extend_from_slice(&record_type.to_be_bytes());
        self.buffer.extend_from_slice(data);
    }
    
    fn string_to_bytes(s: &str) -> Vec<u8> {
        let mut bytes = s.as_bytes().to_vec();
        if bytes.len() % 2 == 1 {
            bytes.push(0);  // Pad to even length
        }
        bytes
    }
    
    pub fn begin_structure(&mut self, name: &str) {
        self.write_record(0x0502, &Self::string_to_bytes(name));
    }
    
    pub fn add_boundary(&mut self, layer: i16, points: &[Point3D]) {
        self.write_record(0x0800, &[]);  // BOUNDARY
        self.write_record(0x0D02, &layer.to_be_bytes());  // LAYER
        
        // Convert points to GDSII format (nanometers to database units)
        let mut xy_data = Vec::new();
        for point in points {
            let x = (point.x / 1000) as i32;  // Convert to microns
            let y = (point.y / 1000) as i32;
            xy_data.extend_from_slice(&x.to_be_bytes());
            xy_data.extend_from_slice(&y.to_be_bytes());
        }
        
        self.write_record(0x1003, &xy_data);  // XY coordinates
        self.write_record(0x1100, &[]);  // ENDEL
    }
    
    pub fn end_structure(&mut self) {
        self.write_record(0x0700, &[]);  // ENDSTR
    }
    
    pub fn finish(mut self) -> Vec<u8> {
        self.write_record(0x0400, &[]);  // ENDLIB
        self.buffer
    }
}

impl Exporter for GdsiiEmitter {
    fn export(&self, board: &Board, ir: &HardwareIR) -> Result<Vec<OutputFile>, ExportError> {
        let mut emitter = GdsiiEmitter::new(&ir.space_name);
        
        emitter.begin_structure("TOP_LEVEL");
        
        // Export each trace as a boundary
        for (layer_idx, layer) in board.layers.iter().enumerate() {
            for trace in &layer.traces {
                let points: Vec<Point3D> = trace.segments
                    .iter()
                    .flat_map(|seg| vec![seg.start, seg.end])
                    .collect();
                
                emitter.add_boundary(layer_idx as i16, &points);
            }
        }
        
        emitter.end_structure();
        
        let content = emitter.finish();
        
        Ok(vec![OutputFile {
            name: format!("{}.gds", ir.space_name),
            content,
        }])
    }
}
```

### Custom QASM Exporter (Quantum Computing)

For quantum computing, we export to QASM (Quantum Assembly Language) format.

```rust
pub struct QasmExporter;

impl Exporter for QasmExporter {
    fn export(&self, board: &Board, ir: &HardwareIR) -> Result<Vec<OutputFile>, ExportError> {
        let mut qasm = String::new();
        
        // QASM header
        qasm.push_str("OPENQASM 2.0;\n");
        qasm.push_str("include \"qelib1.inc\";\n\n");
        
        // Count qubits
        let num_qubits = board.components.iter()
            .filter(|c| c.component_type == "Qubit")
            .count();
        
        qasm.push_str(&format!("qreg q[{}];\n", num_qubits));
        qasm.push_str(&format!("creg c[{}];\n\n", num_qubits));
        
        // Translate quantum gates from behavioral layer
        for gate in &ir.quantum_gates {
            match gate.gate_type {
                GateType::Hadamard => {
                    qasm.push_str(&format!("h q[{}];\n", gate.target));
                }
                GateType::CNOT => {
                    qasm.push_str(&format!("cx q[{}],q[{}];\n", 
                        gate.control, gate.target));
                }
                GateType::Measure => {
                    qasm.push_str(&format!("measure q[{}] -> c[{}];\n", 
                        gate.target, gate.target));
                }
            }
        }
        
        Ok(vec![OutputFile {
            name: format!("{}.qasm", ir.space_name),
            content: qasm.into_bytes(),
        }])
    }
}
```

### Why Custom Emitters Are Superior

**Compared to generic libraries**:

```
Generic Library Approach:
  Dependencies: gerber-rs, gdsii, etc.
  Binary size: +5MB per library
  Compile time: +30 seconds per library
  Control: Limited to library API
  Bugs: Wait for upstream fixes
  
Custom Emitter Approach:
  Dependencies: Zero
  Binary size: +50KB total
  Compile time: +2 seconds
  Control: Complete
  Bugs: Fix immediately
```

### Performance Comparison

**Generic library** (with intermediate data structures):
```rust
// Must convert to library's format
let gerber_lib_data = convert_to_gerber_lib_format(board);
let output = gerber_lib.export(gerber_lib_data);
// Two conversions, extra allocations
```

**Custom emitter** (direct generation):
```rust
// Direct string formatting
let mut emitter = GerberEmitter::new();
emitter.draw_trace(start, end);
let output = emitter.finish();
// Zero conversions, minimal allocations
```

**Result**: 3-5× faster export generation.

### The Plugin Architecture

New export formats are trivial to add:

```rust
// Register exporters
pub fn get_exporter(target: &str) -> Box<dyn Exporter> {
    match target {
        "gerber" => Box::new(GerberEmitter::new()),
        "gdsii" => Box::new(GdsiiEmitter::new("design")),
        "qasm" => Box::new(QasmExporter),
        "obj" => Box::new(ObjExporter),
        "step" => Box::new(StepExporter),
        _ => panic!("Unknown target: {}", target),
    }
}
```

Adding a new format requires:
1. Implement the `Exporter` trait
2. Add to the match statement
3. Done

**No external dependencies. No library updates. Complete control.**

---

## The Scene Graph Architecture

### How the Compiler Handles 3D

When the user types `hws build main.hw --target viz`, the compiler follows this process:

**Step A: The Trace Generator** (voxel engine)
- Loop through 3D voxel grid
- For every copper voxel, generate 3D mesh for trace
- For every FR4 voxel, generate green board substrate
- Traces perfectly calculated using sparse voxel engine

**Step B: The Scene Graph** (assembly)
- Create scene graph (JSON or Python script)
- "Draw the copper traces here"
- "At coordinates X:50, Y:100, Z:2, import `chip.glb` and rotate 90°"
- "At coordinates X:10, Y:20, Z:2, procedurally draw resistor"

**Step C: The Blender Execution**
- Output is intelligent Python script (`sim.py`)
- When user opens Blender, script executes:
  1. Builds copper traces (procedural geometry from voxels)
  2. Pulls in high-fidelity `.glb` chips from `hpm` cache
  3. Applies metallic shader to copper
  4. Sets up camera and lighting automatically
  5. Renders final scene

### Example Output Script

```python
import bpy

# Clear scene
bpy.ops.wm.read_factory_settings(use_empty=True)

# 1. Draw FR4 substrate
bpy.ops.mesh.primitive_cube_add(
    size=1, 
    location=(50, 50, -0.5), 
    scale=(100, 100, 2)
)
bpy.context.object.name = 'FR4_Substrate'

# 2. Draw copper traces (from voxel engine)
for voxel in copper_voxels:
    bpy.ops.mesh.primitive_cube_add(
        size=0.1,
        location=(voxel.x, voxel.y, voxel.z)
    )

# 3. Import high-fidelity component models
bpy.ops.import_scene.gltf(
    filepath='/home/user/.hw/packages/ics/esp32/assets/chip.glb'
)
obj = bpy.context.selected_objects[0]
obj.location = (50, 100, 2)
obj.rotation_euler = (0, 0, 1.5708)  # 90 degrees

# 4. Apply materials
copper_material = bpy.data.materials.new(name="Copper")
copper_material.metallic = 1.0
copper_material.roughness = 0.2
```

---

## Why This Rethinks Conventional Systems

### The Digital Twin

The moment a user writes `add Component at [X, Y, Z]`, they are simultaneously:
- Designing the mathematical circuit
- Programming a cinematic 3D rendering

**No separate 3D modeling step required.**

### Marketing & Documentation

A hardware startup can:
1. Write `.hw` code
2. Type `hws build --target viz`
3. Instantly have photorealistic marketing renders

**Before they even manufacture it.**

### No More Misalignment

Because the 3D model is explicitly anchored to exact voxel coordinates (`offset: [0, 0, 0]`), pins will perfectly touch copper traces every single time.

**Traditional tools**: Manual alignment, often wrong

**Hardware Script**: Mathematically perfect alignment, always

---

## Scene Graph Processing

### How the Scene Graph Handles Render Blocks

The Scene Graph architecture processes the `render:` blocks defined in component definitions (see Book 2, Part III: Abstraction Blocks for render block syntax). When the compiler reaches the visualization export stage, it follows this pipeline:

**Phase 1: Parse Render Blocks**

The Scene Graph reads each component's render block from the Hardware IR:

```rust
for component in &ir.components {
    let render_config = component.render_block;
    
    match render_config.render_type {
        RenderType::Asset => {
            // Load .glb file from package cache
            let asset_path = resolve_asset_path(&render_config.asset);
            scene.add_asset_import(asset_path, component.position);
        }
        RenderType::Procedural => {
            // Generate geometry from shape parameters
            scene.add_procedural_geometry(
                render_config.shape,
                render_config.color,
                component.position
            );
        }
    }
}
```

**Phase 2: Generate Copper Traces**

The voxel engine's copper data is converted to procedural geometry:

```rust
for voxel in voxel_grid.copper_voxels() {
    scene.add_trace_segment(
        voxel.position,
        voxel.material,
        voxel.width
    );
}
```

**Phase 3: Assemble Scene**

The Scene Graph combines all elements with proper materials, lighting, and camera:

```rust
let scene = SceneGraph::new();
scene.add_lighting();
scene.add_camera();
scene.add_materials();
scene.add_substrate(ir.dimensions);
scene.add_traces(voxel_grid);
scene.add_components(ir.components);
```

**Phase 4: Export to Target Format**

```rust
match export_target {
    ExportTarget::Blender => scene.export_python(),
    ExportTarget::Web => scene.export_threejs(),
    ExportTarget::OBJ => scene.export_obj(),
}
```

### Fallback Handling

When an asset file is unavailable, the Scene Graph automatically uses the `fallback_procedural` configuration:

```rust
fn resolve_component_render(component: &Component) -> RenderStrategy {
    if let Some(asset_path) = &component.render.asset {
        if asset_exists(asset_path) {
            return RenderStrategy::Asset(asset_path);
        }
    }
    
    // Fallback to procedural
    if let Some(fallback) = &component.render.fallback_procedural {
        return RenderStrategy::Procedural(fallback);
    }
    
    // Ultimate fallback: simple box
    RenderStrategy::DefaultBox
}
```

This ensures that visualization always works, even when high-fidelity assets aren't available.

---

## Key Takeaways

1. **Dual identity system** — Components have mathematical brain (voxels) and visual body (3D mesh)

2. **Target-based compilation** — Same source, different outputs (PCB, visualization, web)

3. **FR4 substrate is critical** — Makes 3D views look professional and realistic

4. **Procedural for simple components** — Avoid bloat, generate on-the-fly

5. **`.glb` for complex components** — Industry standard, compressed, web-ready

6. **Flash vs Draw in Gerber** — Flash for pads, Draw for traces, hybrid is best

7. **Scene graph architecture** — Separate concerns: physics vs aesthetics

8. **Perfect alignment guaranteed** — Mathematical precision ensures pins touch traces

9. **No separate modeling step** — Write code, get both engineering and art

10. **Marketing before manufacturing** — Photorealistic renders from day one

---

## Contributing to Export Generation

### Getting Started

1. Clone the repository
2. Navigate to `hwc/crates/hwc-export/`
3. Run tests with `cargo test`
4. Read the export module documentation

### Areas for Contribution

**3D export enhancements**:
- Better material systems
- Advanced lighting setups
- Camera positioning algorithms
- Animation support

**Gerber optimization**:
- Aperture optimization
- Multi-layer support
- Silkscreen generation
- Solder mask generation

**New export formats**:
- STEP files (mechanical CAD)
- GDSII (silicon manufacturing)
- SVG (documentation)
- PDF (assembly instructions)

**Web viewer**:
- WebAssembly compilation
- Three.js integration
- Interactive controls
- Real-time editing

---

## Conclusion

Hardware Script's export generation system bridges the gap between CAD (Computer-Aided Design) and CGI (Computer-Generated Imagery) seamlessly. By maintaining dual identities for components and using target-based compilation, we achieve both perfect engineering precision and photorealistic aesthetics.

The FR4 substrate rendering and Gerber Draw optimization transform Hardware Script from a working prototype into a professional tool that produces both beautiful visualizations and factory-ready manufacturing files.

If you're a 3D artist, renderer, or manufacturing liaison who wants to help build the future of hardware visualization, we'd love your contributions.

Welcome to the export generation system. Let's make hardware design cinematic.
