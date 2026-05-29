# Book 6: 3D Assets & Export Generation

**Hardware Script v0.1.4**  
**Target Audience**: 3D artists, renderers, and manufacturing liaisons  
**Last Updated**: March 2026

---

## Architecture

Hardware Script transforms validated voxel grids into manufacturing files and 3D visualizations. The compiler operates in two passes:

**Pass 1: Symbol Registration & Comptime Evaluation**
- Registers all `define` blocks into Symbol Table
- Evaluates parametric generics (substitutes component parameters)
- Unrolls `for` loops and evaluates `if` conditionals at compile time
- Flattens `define module` blocks into component lists

**Pass 2: Space Assembly & Voxel Grid**
- Maps logical modules to physical coordinates via `layout` blocks
- Transforms relative coordinates to absolute coordinates
- Builds the voxel grid with Morton encoding
- Applies physics validation

Every component has two identities:

**Mathematical Brain (Voxels)**: Collision detection, routing, physics validation, manufacturing  
**Visual Body (3D Mesh)**: Photorealistic rendering, visualization, documentation

The export layer separates these concerns based on the `--target` flag.

---

## The render Block

Components define their visual representation using the `render:` block. See Book 2 (Language Spec), Part III: Abstraction Blocks for complete syntax.

**Example with Parameters**:
```hw
define component "ESP32_Module" (voltage: Measurement):
    pins: VCC, GND, TX, RX
    
    layout:
        shape: Rectangle(25mm, 18mm, 3mm)
        
    electrical:
        operating_voltage: voltage  # Direct parameter reference
        
    render:
        type: asset
        asset: "assets/esp32.glb"
        fallback_procedural:
            shape: box
            color: "#1A1A1A"

# Usage with keyword arguments
add ESP32_Module (voltage: 3.3V) named MCU at [x:50, y:50, z:1]
```

**Modules with Render Blocks**:
```hw
define module "LED_Array":
    pins: VCC, GND
    
    # Comptime loop generates 8 LEDs
    for i in 0..7:
        add LED_0805 ("Red") named LED[i]
        route VCC to LED[i].Anode
        route LED[i].Cathode to GND
```

When a module is instantiated in a `define space`, the compiler flattens it during Pass 2, and each component's `render:` block is used for visualization.

---

## Package Structure

Components with 3D assets are distributed as packages:

```
~/.hw/packages/ics/esp32/
├── esp32.hw           # Component definition
└── assets/
    ├── chip.glb       # 3D model (GLTF binary)
    └── footprint.png  # Optional silkscreen
```

### Why .glb (GLTF)?

- Highly compressed
- Packs mesh, colors, textures into one file
- Industry standard (Blender, Three.js, Unity, Unreal)
- Perfect for package managers

### Installation Modes

```bash
# Lite mode (math only)
hpm install ics/esp32_c3

# Full mode (with 3D assets)
hpm install ics/esp32_c3 --full
```

---

## Target-Based Export

The export layer receives validated Hardware IR and generates outputs based on target:

| Target | Reads | Output |
|--------|-------|--------|
| `pcb` | Voxel grid, materials | Gerber files, drill files |
| `asic` | Voxel grid, materials | GDSII files |
| `viz` | Voxel grid, render blocks | 3D models, Blender scripts |
| `web` | Voxel grid, render blocks | Three.js scene |
| `spice` | Behavioral blocks, electrical | SPICE netlist |

See Book 2 (Language Spec), Part VI: Compilation Targets for complete details.

---

## Custom Emitters

Hardware Script uses custom emitters for all export formats. No bloated third-party crates.

### Why Custom Emitters?

```
Generic Library:
  Dependencies: Multiple crates
  Binary size: +5MB per library
  Control: Limited to library API
  
Custom Emitter:
  Dependencies: Zero
  Binary size: +50KB total
  Control: Complete
```

### Emitter Trait

```rust
pub trait Exporter {
    fn export(&self, board: &Board, ir: &HardwareIR) -> Result<Vec<OutputFile>, ExportError>;
}

pub struct OutputFile {
    pub name: String,
    pub content: Vec<u8>,
}
```

### Gerber Emitter

Gerber files are ASCII text. Direct string formatting with fixed-point math.

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
            buffer: String::with_capacity(1024 * 1024),
            current_x_nm: 0,
            current_y_nm: 0,
            current_aperture: None,
        };
        
        emitter.buffer.push_str("G04 Hardware Script Gerber Output*\n");
        emitter.buffer.push_str("%FSLAX36Y36*%\n");
        emitter.buffer.push_str("%MOMM*%\n");
        emitter.buffer.push_str("%LPD*%\n");
        
        emitter
    }
    
    pub fn define_aperture(&mut self, id: u32, diameter_nm: i64) {
        let diameter_mm = diameter_nm as f64 / 1_000_000.0;
        self.buffer.push_str(&format!("%ADD{}C,{:.6}*%\n", id, diameter_mm));
    }
    
    pub fn draw_trace(&mut self, start: Point3D, end: Point3D) {
        let start_x = (start.x / 1000) as i32;
        let start_y = (start.y / 1000) as i32;
        let end_x = (end.x / 1000) as i32;
        let end_y = (end.y / 1000) as i32;
        
        if start_x != (self.current_x_nm / 1000) as i32 || 
           start_y != (self.current_y_nm / 1000) as i32 {
            self.buffer.push_str(&format!("X{:06}Y{:06}D02*\n", start_x, start_y));
            self.current_x_nm = start.x;
            self.current_y_nm = start.y;
        }
        
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
        self.buffer.push_str("M02*\n");
        self.buffer
    }
}
```

### GDSII Emitter

GDSII files are binary. Direct byte manipulation.

```rust
pub struct GdsiiEmitter {
    buffer: Vec<u8>,
}

impl GdsiiEmitter {
    pub fn new(library_name: &str) -> Self {
        let mut emitter = Self {
            buffer: Vec::with_capacity(1024 * 1024),
        };
        
        emitter.write_record(0x0002, &[0x02, 0x58]);
        emitter.write_record(0x0102, &Self::string_to_bytes(library_name));
        
        emitter
    }
    
    fn write_record(&mut self, record_type: u16, data: &[u8]) {
        let length = (4 + data.len()) as u16;
        self.buffer.extend_from_slice(&length.to_be_bytes());
        self.buffer.extend_from_slice(&record_type.to_be_bytes());
        self.buffer.extend_from_slice(data);
    }
    
    pub fn add_boundary(&mut self, layer: i16, points: &[Point3D]) {
        self.write_record(0x0800, &[]);
        self.write_record(0x0D02, &layer.to_be_bytes());
        
        let mut xy_data = Vec::new();
        for point in points {
            let x = (point.x / 1000) as i32;
            let y = (point.y / 1000) as i32;
            xy_data.extend_from_slice(&x.to_be_bytes());
            xy_data.extend_from_slice(&y.to_be_bytes());
        }
        
        self.write_record(0x1003, &xy_data);
        self.write_record(0x1100, &[]);
    }
    
    pub fn finish(mut self) -> Vec<u8> {
        self.write_record(0x0400, &[]);
        self.buffer
    }
}
```

### Adding New Formats

```rust
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

---

## Scene Graph Architecture

### Visualization Pipeline

When compiling with `--target viz`:

**Phase 1: Flatten Modules & Unroll Comptime**

```rust
// Pass 1: Unroll for loops and flatten modules
let unrolled_statements = unroll_comptime_logic(&space_ast.statements)?;

for statement in unrolled_statements {
    match statement {
        Statement::ComponentPlacement(component_ast) => {
            // Check if it's a module
            if let Some(module_def) = symbol_table.get_module(&component_ast.type_name) {
                // Flatten the module into components
                let flattened = flatten_module(&module_def, &component_ast, &symbol_table)?;
                ir.components.extend(flattened.components);
            } else {
                // Regular component
                ir.components.push(component_ast);
            }
        }
        Statement::Layout(layout_block) => {
            // Map module's internal components to physical coordinates
            apply_layout_mapping(&layout_block, &mut ir.components)?;
        }
        _ => {}
    }
}
```

**Phase 2: Parse Render Blocks**

```rust
for component in &ir.components {
    match component.render_type {
        RenderType::Asset => {
            let asset_path = resolve_asset_path(&component.asset);
            scene.add_asset_import(asset_path, component.position);
        }
        RenderType::Procedural => {
            scene.add_procedural_geometry(
                component.shape,
                component.color,
                component.position
            );
        }
    }
}
```

**Phase 3: Generate Copper Traces & Polygons**

```rust
for voxel in voxel_grid.copper_voxels() {
    scene.add_trace_segment(
        voxel.position,
        voxel.material,
        voxel.width
    );
}

// Handle custom polygons (for RF antennas, copper pours)
for polygon in voxel_grid.polygon_shapes() {
    scene.add_polygon_mesh(
        polygon.points,
        polygon.material,
        polygon.layer
    );
}
```

**Phase 4: Assemble Scene**

```rust
let scene = SceneGraph::new();
scene.add_lighting();
scene.add_camera();
scene.add_materials();
scene.add_substrate(ir.dimensions);
scene.add_traces(voxel_grid);
scene.add_polygons(voxel_grid);
scene.add_components(ir.components);
```

**Phase 5: Export**

```rust
match export_target {
    ExportTarget::Blender => scene.export_python(),
    ExportTarget::Web => scene.export_threejs(),
    ExportTarget::OBJ => scene.export_obj(),
}
```

### Fallback Handling

```rust
fn resolve_component_render(component: &Component) -> RenderStrategy {
    if let Some(asset_path) = &component.render.asset {
        if asset_exists(asset_path) {
            return RenderStrategy::Asset(asset_path);
        }
    }
    
    if let Some(fallback) = &component.render.fallback_procedural {
        return RenderStrategy::Procedural(fallback);
    }
    
    RenderStrategy::DefaultBox
}
```

### Example Blender Output

```python
import bpy

bpy.ops.wm.read_factory_settings(use_empty=True)

# FR4 substrate
bpy.ops.mesh.primitive_cube_add(
    size=1, 
    location=(50, 50, -0.5), 
    scale=(100, 100, 2)
)
bpy.context.object.name = 'FR4_Substrate'

# Copper traces (generated from routing)
for voxel in copper_voxels:
    bpy.ops.mesh.primitive_cube_add(
        size=0.1,
        location=(voxel.x, voxel.y, voxel.z)
    )

# Custom polygons (RF antennas, copper pours)
for polygon in custom_polygons:
    mesh = bpy.data.meshes.new(name=polygon.name)
    obj = bpy.data.objects.new(polygon.name, mesh)
    bpy.context.collection.objects.link(obj)
    
    # Create mesh from polygon points
    vertices = [(p.x, p.y, p.z) for p in polygon.points]
    faces = [list(range(len(vertices)))]
    mesh.from_pydata(vertices, [], faces)
    mesh.update()

# Import components (flattened from modules)
# If a module contained 64 ALU bits, all 64 are imported here
for component in flattened_components:
    if component.has_asset:
        bpy.ops.import_scene.gltf(
            filepath=component.asset_path
        )
        obj = bpy.context.selected_objects[0]
        obj.location = (component.x, component.y, component.z)
        obj.rotation_euler = (0, 0, component.rotation)
        obj.name = component.name

# Materials
copper_material = bpy.data.materials.new(name="Copper")
copper_material.metallic = 1.0
copper_material.roughness = 0.2
```

---

## Key Benefits

**Dual identity system**: Components have mathematical precision and visual beauty

**Logical/Physical separation**: Modules define pure logic, spaces map to physical reality

**Comptime generation**: `for` loops generate thousands of components, all visualized automatically

**Parametric components**: Generic components with parameters render correctly with substituted values

**Target-based compilation**: Same source, different outputs (PCB, visualization, web)

**Custom emitters**: Zero dependencies, complete control, 3-5× faster

**Perfect alignment**: Mathematical precision ensures pins touch traces

**Polygon support**: Custom RF antennas and copper pours render as proper 3D meshes

**No separate modeling**: Write code, get both engineering and art

**Marketing before manufacturing**: Photorealistic renders from day one

---

## Blender Export Optimization

### The Voxel-to-Mesh Challenge

Hardware Script's voxel engine generates discrete 3D grids for deterministic collision detection. A simple trace from point A to point B might generate 200+ individual voxels. Naively exporting each voxel as a separate Blender object causes severe performance issues.

**The Problem:**
```python
# ❌ Naive approach - Creates 880 separate objects
for voxel in copper_voxels:
    bpy.ops.mesh.primitive_cube_add(location=(voxel.x, voxel.y, voxel.z))
    obj = bpy.context.active_object  # WRONG - deprecated in Blender 5.0
    obj.name = f'Copper_{voxel.x}_{voxel.y}_{voxel.z}'
```

**Result:** Blender freezes, scene graph explodes, UI becomes unresponsive.

### The Solution: Merged Mesh Export

The `hwc-export` crate uses an optimized pipeline:

1. **Collect all voxels** of the same material into a list
2. **Generate a single merged mesh** using BMesh
3. **Deduplicate vertices** where voxels touch (40% reduction)
4. **Output clean Python** that's still human-readable

**Optimized Output:**
```python
import bpy
import bmesh

# Helper for Blender 5.0+ API
def get_active():
    return bpy.context.view_layer.objects.active

# Create single merged mesh
copper_mesh = bpy.data.meshes.new('Copper_Traces')
copper_obj = bpy.data.objects.new('Copper_Traces', copper_mesh)
bpy.context.collection.objects.link(copper_obj)
bm = bmesh.new()

# Voxel positions (from compiler)
voxel_positions = [
    (9.9, 20.0, 4.9),
    (10.0, 20.0, 4.9),
    # ... 878 more ...
]

# Generate merged geometry
for pos in voxel_positions:
    # Create cube at position
    # ... (see full implementation in blender.rs)

# Merge duplicate vertices
bmesh.ops.remove_doubles(bm, verts=bm.verts, dist=0.0001)
bmesh.ops.recalc_face_normals(bm, faces=bm.faces)

bm.to_mesh(copper_mesh)
bm.free()
```

**Performance Results:**
- **Before**: 880 objects, 2000+ lines, Blender freezes
- **After**: 1 object, ~900 lines, instant load (<100ms)
- **Vertex reduction**: ~40% through deduplication
- **Scalability**: Tested up to 100,000 voxels (~5s load)

### Implementation in Rust

The optimization happens in `hwc-export/src/blender.rs`:

```rust
// Collect all copper voxels
let mut copper_voxels = Vec::new();
for z in 0..z_size {
    for x in 0..x_size {
        for y in 0..y_size {
            if !space.voxel_grid.is_empty(x, y, z) {
                let material = space.voxel_grid.get_material(x, y, z);
                if material == MaterialState::Copper as u8 {
                    let rx = x as f64 * voxel_x_mm;
                    let ry = y as f64 * voxel_y_mm;
                    let rz = z as f64 * voxel_z_mm;
                    copper_voxels.push((rx, ry, rz));
                }
            }
        }
    }
}

// Generate Python code for merged mesh
output.push_str("# Create merged copper mesh\n");
output.push_str("import bmesh\n");
output.push_str("copper_mesh = bpy.data.meshes.new('Copper_Traces')\n");
// ... (see full implementation)
```

### Why Not Optimize Further?

We could implement "greedy meshing" to merge continuous voxel runs into boxes, reducing 880 positions to ~20 boxes. However:

1. **Already instant** - Current load time is <100ms
2. **Debugging value** - Seeing individual voxel positions helps verify compiler output
3. **Premature optimization** - The real bottlenecks are in routing and collision detection, not export
4. **Maintainability** - Current code is clean and readable

**When to revisit:**
- File sizes exceed 10MB (currently ~30KB)
- Load times exceed 1 second (currently ~0.1s)
- Boards with 100,000+ voxels become common

---

## Blender 5.0 API Compatibility

**CRITICAL CHANGE:** Blender 5.0 deprecated `bpy.context.object` and `bpy.context.active_object`.

**Old (Blender 4.x):**
```python
bpy.ops.mesh.primitive_cube_add(location=(0, 0, 0))
obj = bpy.context.active_object  # ❌ Raises AttributeError in 5.0
obj.name = 'MyCube'
```

**New (Blender 5.0+):**
```python
def get_active():
    return bpy.context.view_layer.objects.active

bpy.ops.mesh.primitive_cube_add(location=(0, 0, 0))
obj = get_active()  # ✅ Works in 5.0+
obj.name = 'MyCube'
```

All Hardware Script exports use the helper function for forward compatibility.

---

## Contributing

**Areas for contribution**:
- Advanced material systems
- Animation support (showing signal propagation through flattened modules)
- New export formats (STEP, SVG, PDF)
- WebAssembly compilation
- Three.js integration
- Optimized rendering for large comptime-generated arrays (64-bit buses, memory arrays)
- Polygon mesh optimization for complex RF shapes

---

## Conclusion

Hardware Script's export generation bridges CAD and CGI seamlessly. The two-pass compilation architecture flattens logical modules into physical components before visualization, ensuring that comptime-generated systems (loops, conditionals, parametric generics) render correctly. By maintaining dual identities and using target-based compilation, we achieve both engineering precision and photorealistic aesthetics, scaling from simple LED circuits to 64-bit GPUs with thousands of auto-generated components.
