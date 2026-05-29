# 3D Asset Mapping: The Cinematic EDA

**Bridging the Gap Between Hardware Engineering and Game Development**

---

## The Paradigm Shift

Traditional hardware design tools (EDA) treat 3D rendering as an afterthought. Components look like blocky extrusions, and exporting to a high-fidelity 3D software like Blender requires hours of manual alignment.

Hardware Script completely rethinks this by treating components like **Prefabs in a Game Engine**.

---

## The Dual Identity: Brain and Body

A component in Hardware Script has two identities:

### 1. The Mathematical Brain (Voxels)
Used by the compiler for:
- Collision detection
- Routing calculations
- Gerber manufacturing
- Physics validation

### 2. The Visual Body (3D Mesh)
Used by the exporter for:
- Photorealistic rendering
- Marketing materials
- Documentation
- Interactive visualization

**Key Insight**: The compiler separates these concerns based on the `--target` flag.

---

## The render Block: Mapping Math to Graphics

When a creator defines a component in a `.hwx` file, they map the mathematical grid to a real 3D asset (like a `.glb` or `.obj` file).

### Basic Syntax

```hw
// Inside an esp32.hwx file

component ESP32_Chip {
    // 1. The Mathematical Brain
    dimensions: (18mm, 25mm, 3mm)
    
    pins {
        GND @ [1, 0, 5]
        VCC @ [1, 0, 10]
    }
    
    // 2. The Visual Body (New Feature)
    render {
        asset: "assets/esp32_photoreal.glb"
        offset: [0, 0, 0]   // Aligns the 3D model origin to the voxel grid
        scale: 1.0
    }
}
```


---

## The Package Structure: Beyond Single Files

If a component includes 3D models and textures, it can no longer be just a single `.hwx` text file. When a user runs `hpm install ics/esp32`, the package manager should download a lightweight directory:

```
~/.hw/packages/ics/esp32/
├── esp32.hwx          # The math, pins, and physics (The Brain)
└── assets/
    ├── chip.glb       # The 3D model and textures (The Body)
    └── footprint.png  # Optional silkscreen graphic
```

### Why .glb (GLTF) Format?

**GLTF is the "JPEG of 3D."**

- Highly compressed
- Packs mesh, colors, and metallic/roughness textures into one file
- Industry standard for web and real-time 3D
- Perfect for package managers (small download size)
- Native support in Blender, Three.js, Unity, Unreal

**Comparison**:
- `.obj` - Simple but no textures, no compression
- `.fbx` - Proprietary, bloated
- `.glb` - Open standard, compressed, includes everything ✅

---

## The render Block: Complete Specification

### Full Syntax with Asset

```hw
define Component "ESP32_WROOM_32":
    dimensions: 18mm by 25mm by 3mm
    grid: 180 by 250 by 30
    
    pins:
        GND at [1, 0, 15]
        3V3 at [1, 0, 20]
        GPIO0 at [1, 50, 15]
        # ... more pins
    
    # === THE 3D MAPPING ===
    render:
        # Link to the 3D asset
        asset: "assets/chip.glb"
        
        # Where is the "origin" (0,0,0) of the 3D model relative to our voxel grid?
        # This ensures the 3D pins perfectly touch the copper traces!
        origin_offset: [0, 0, 0]
        
        # Match the 3D model scale to the voxel scale
        scale: 1.0
        
        # If the user doesn't want to download the 3D asset (Lite mode),
        # what should the compiler procedurally draw as a fallback?
        fallback_procedural:
            shape: box
            color: "#222222"
            material: "matte_plastic"
            label: "ESP32"
```

### Procedural Generation (For Simple Components)

We don't want people downloading a 3D `.glb` file for a basic 10-cent resistor. That would bloat the system. For 90% of passive components (resistors, capacitors, LEDs), the system should just procedurally generate the 3D graphics.

```hw
define Component "Resistor_0805":
    dimensions: 2.0mm by 1.2mm by 0.5mm
    grid: 20 by 12 by 5
    
    pins:
        A at [1, 0, 6]
        B at [1, 20, 6]
    
    # === PROCEDURAL RENDERING ===
    render:
        type: procedural
        shape: smd_passive
        body_color: "#111111"
        endcap_color: "#C0C0C0"  # Silver
        label: "{value}"  # Automatically prints "10k" on top in 3D!
```

**Procedural shapes available**:
- `smd_passive` - Standard SMD resistor/capacitor
- `box` - Simple rectangular box
- `cylinder` - Cylindrical (for through-hole)
- `chip` - Generic IC package
- `connector` - Pin headers

---


## The Magic of the Compiler

When the user types `hws build main.hw --target blender`, here is the genius of how your engine will handle it:

### Step A: The Trace Generator (The Voxel Engine)

The Rust compiler loops through the 3D voxel grid:
- For every voxel that is `Copper`, it generates a 3D mesh for the trace
- For every voxel that is `FR4`, it generates the green board substrate
- Traces are perfectly calculated using the sparse voxel engine

### Step B: The Scene Graph (The Assembly)

Instead of trying to convert the ESP32 into voxels, the compiler creates a **Scene Graph** (a JSON or Python script). It says:

1. "Draw the copper traces here."
2. "At coordinates X: 50, Y: 100, Z: 2, import `~/.hw/packages/ics/esp32/assets/chip.glb` and rotate it 90 degrees."
3. "At coordinates X: 10, Y: 20, Z: 2, procedurally draw a black box with silver ends (the resistor)."

### Step C: The Blender Execution

The output of your compiler isn't just an `.obj` file. It's an intelligent Python script (`sim.py`). When the user opens Blender, the script executes:

1. It builds the copper traces (procedural geometry from voxels)
2. It pulls in the high-fidelity `.glb` chips from the `hpm` cache
3. It applies a metallic shader to the copper
4. It sets up a camera and lighting automatically
5. It renders the final scene

**Example output script**:
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
copper_material.use_nodes = True
```

---


## The Compiler's Dual Personality

When a user builds their board, they don't have to choose between engineering and art. The compiler separates the logic based on the `--target` flag:

### 1. For the Manufacturer (--target pcb)

The compiler **ignores the render block**. It only looks at:
- `dimensions` - Physical size
- `pins` - Connection points
- `behavior` - Electrical limits

**Output**: 2D copper Gerber files for PCB manufacturing

**Example**:
```bash
hws build board.hw --target pcb

# Generates:
# build/gerber/board_top.gtl
# build/gerber/board_bottom.gbl
# build/gerber/board.drl
```

### 2. For the 3D Artist / Marketing (--target blender)

The compiler becomes a **scene generator**:
- It calculates and draws the exact physical copper traces using procedural geometry
- Whenever it hits a component, it reads the `render` block
- It automatically imports the high-fidelity 3D asset (`.glb`) and snaps it perfectly onto the copper traces at the exact mathematical coordinates

**Output**: Blender Python script with photorealistic components

**Example**:
```bash
hws build board.hw --target blender

# Generates:
# build/viz/sim.py
# build/viz/board.blend (optional)
```

### 3. For the Web Viewer (--target web) [Future]

The compiler generates a WebAssembly module and Three.js scene:
- Lightweight `.glb` models load instantly
- Interactive 3D viewer runs in browser
- No installation required

**Example**:
```bash
hws build board.hw --target web

# Generates:
# build/web/index.html
# build/web/board.wasm
# build/web/scene.json
```

---

## Why This Rethinks the Conventional System

In traditional EDA tools like KiCad or Altium, 3D viewing is an afterthought. It looks clunky, models are often misaligned, and it's strictly for checking if a component hits the casing.

By integrating this directly into the Hardware Script compiler, you achieve something massive:

### 1. The Digital Twin

The moment a user writes `add Component at [X, Y, Z]`, they are simultaneously:
- Designing the mathematical circuit
- Programming a cinematic 3D rendering

**No separate 3D modeling step required.**

### 2. Marketing & Documentation

A hardware startup can:
1. Write `.hw` code
2. Type `hws build --target viz`
3. Instantly have photorealistic marketing renders of their product

**Before they even manufacture it.**

### 3. The Web Viewer (Future)

Because we use the lightweight `.glb` format, you can easily build a WebAssembly (`hwc-web`) compiler later. Users could:
- Paste `.hw` code into a website
- A beautiful 3D model of their board instantly renders in their browser using Three.js
- Share designs with a simple URL

**No software installation required.**

### 4. No More Misalignment

Because the 3D model is explicitly anchored to the exact voxel coordinates (`offset: [0, 0, 0]`), pins will perfectly touch the copper traces every single time.

**Traditional tools**: Manual alignment, often wrong

**Hardware Script**: Mathematically perfect alignment, always

---


## The Architecture: Separation of Concerns

### The Core Principle

**The Sparse Voxel Array** (which handles physics, collisions, and routing mathematically) is completely separate from **The Scene Graph** (which handles how things look).

This separation achieves both perfect engineering AND perfect aesthetics:

### The Physics Engine is Happy

The router still sees the Transistor as a 3×3 block of `MaterialState::Body` voxels. It won't let you route a wire through the middle of the transistor.

**Collision detection**: O(1) array lookup

### The Manufacturer is Happy

The Gerber exporter only prints the `Copper` voxels. It ignores the 3D graphics entirely.

**Manufacturing files**: Pure mathematics, no graphics

### The Designer is Amazed

When they run `blender --python sim.py`, Blender doesn't draw blocky Minecraft-style components anymore. It seamlessly connects beautifully calculated copper voxel-wires to a photorealistic 3D transistor mesh.

**Visual output**: Photorealistic, cinema-quality

---

## Implementation Strategy

### Phase 1: Basic Procedural (v0.2)

**Goal**: Replace blocky voxel components with simple procedural shapes

**Implementation**:
```rust
// In hwc-export/src/blender.rs
match component.render.type {
    RenderType::Procedural => {
        generate_procedural_mesh(component.shape, component.color)
    }
}
```

**Output**: Clean, professional-looking boards with simple geometric components

### Phase 2: Asset Loading (v0.3)

**Goal**: Load `.glb` files for complex components

**Implementation**:
```rust
// In hwc-export/src/blender.rs
match component.render.type {
    RenderType::Asset => {
        let asset_path = resolve_asset_path(component.render.asset);
        generate_import_command(asset_path, component.position)
    }
}
```

**Output**: Photorealistic components mixed with procedural traces

### Phase 3: Full Scene Graph (v0.4)

**Goal**: Complete scene with lighting, materials, camera

**Implementation**:
```rust
// Generate complete Blender scene
let scene = SceneGraph::new();
scene.add_lighting();
scene.add_camera();
scene.add_materials();
scene.add_traces(voxel_engine);
scene.add_components(component_library);
scene.export_python();
```

**Output**: Cinema-ready renders, one command

### Phase 4: Web Viewer (v1.0)

**Goal**: Interactive 3D viewer in browser

**Implementation**:
```rust
// Compile to WebAssembly
#[wasm_bindgen]
pub fn render_board(hw_code: &str) -> SceneJSON {
    let ast = parse(hw_code);
    let scene = compile_to_scene(ast);
    scene.to_json()
}
```

**Output**: Share designs via URL, no installation

---


## The Component Library Strategy

### Standard Library Structure

```
~/.hw/packages/
├── passive/
│   ├── resistor_0805/
│   │   ├── resistor_0805.hwx
│   │   └── assets/
│   │       └── resistor.glb (optional, procedural is fine)
│   └── capacitor_0603/
│       ├── capacitor_0603.hwx
│       └── assets/
│           └── capacitor.glb (optional)
├── active/
│   ├── transistor_2n2222/
│   │   ├── transistor_2n2222.hwx
│   │   └── assets/
│   │       └── to92.glb
│   └── mosfet_irf540/
│       ├── mosfet_irf540.hwx
│       └── assets/
│           └── to220.glb
└── ics/
    ├── esp32_c3/
    │   ├── esp32_c3.hwx
    │   └── assets/
    │       ├── chip.glb
    │       └── footprint.png
    └── atmega328/
        ├── atmega328.hwx
        └── assets/
            └── dip28.glb
```

### Installation Modes

#### Lite Mode (Default)
```bash
hpm install ics/esp32_c3

# Downloads:
# - esp32_c3.hwx (math and physics only)
# - No 3D assets
# - Uses procedural fallback for visualization
```

**Use case**: Fast downloads, manufacturing only

#### Full Mode (With Assets)
```bash
hpm install ics/esp32_c3 --full

# Downloads:
# - esp32_c3.hwx (math and physics)
# - assets/chip.glb (photorealistic 3D model)
# - assets/footprint.png (silkscreen)
```

**Use case**: Marketing, documentation, presentations

### Community Contribution

**Creating a component package**:

1. Design the `.hwx` file (math and physics)
2. Create 3D model in Blender (export as `.glb`)
3. Test alignment with `hws build --target viz`
4. Publish to registry

```bash
# Create package
hws package create my_component.hwx

# Add 3D asset
hws package add-asset my_component.glb

# Test locally
hws build test_board.hw --target viz

# Publish
hpm publish my_component
```

---

## Render Targets: The Complete Picture

The core engine mathematically solves the board. Then, the **Exporter layer** translates those coordinates either into:

### 2D Manufacturing Instructions
- `.gtl` Gerber files (copper layers)
- `.drl` Drill files (vias and holes)
- `.csv` BOM (bill of materials)

### 3D Cinematic Scene Instructions
- `.glb` models (photorealistic components)
- Blender scripts (complete scenes)
- Three.js JSON (web viewer)

**The `.hwx` file is the ultimate bridge**: it holds the physics for the manufacturer, and the visual mapping for the renderer.

---


## Example: Complete Component Definition

### ESP32-C3 with Full 3D Mapping

```hw
define Component "ESP32_C3_WROOM_02":
    // ============================================
    // 1. PHYSICAL DIMENSIONS (The Bounding Box)
    // ============================================
    dimensions: 18mm by 25mm by 3mm
    grid: 180 by 250 by 30
    # Local resolution: 0.1mm per voxel
    
    // ============================================
    // 2. PIN INTERFACE (Where board connects)
    // ============================================
    pins:
        GND at [1, 0, 15]
        3V3 at [1, 0, 20]
        EN at [1, 0, 25]
        GPIO0 at [1, 50, 0]
        GPIO1 at [1, 55, 0]
        GPIO2 at [1, 60, 0]
        # ... 30+ more pins
    
    // ============================================
    // 3. ELECTRICAL BEHAVIOR (For validation)
    // ============================================
    behavior:
        type: Microcontroller
        
        # Electrical limits
        max_supply_voltage: 3.6V
        min_supply_voltage: 3.0V
        max_current: 500mA
        max_gpio_current: 40mA
        
        # Logic levels
        logic_high: 2.4V
        logic_low: 0.8V
    
    // ============================================
    // 4. THERMAL PROPERTIES
    // ============================================
    thermal:
        thermal_resistance: 50  # °C/W
        max_junction_temp: 125  # °C
        typical_power: 0.5W
    
    // ============================================
    // 5. 3D RENDERING (The Visual Body)
    // ============================================
    render:
        # High-fidelity 3D model
        asset: "assets/esp32_c3_photoreal.glb"
        
        # Align model origin to voxel grid
        # This ensures pins perfectly touch copper traces
        origin_offset: [0, 0, 0]
        
        # Scale factor (1.0 = exact match to dimensions)
        scale: 1.0
        
        # Fallback for lite mode (no asset downloaded)
        fallback_procedural:
            shape: chip
            body_color: "#1A1A1A"  # Dark gray
            pin_color: "#C0C0C0"   # Silver
            label: "ESP32-C3"
            label_color: "#FFFFFF"
    
    // ============================================
    // 6. METADATA (For package manager)
    // ============================================
    metadata:
        manufacturer: "Espressif"
        part_number: "ESP32-C3-WROOM-02"
        datasheet: "https://www.espressif.com/sites/default/files/documentation/esp32-c3-wroom-02_datasheet_en.pdf"
        package: "SMD-38"
        verified: true
```

---

## The Future: Parametric 3D Generation

### The Holy Grail

Instead of downloading pre-made 3D models, generate them on-the-fly based on parameters:

```hw
define Component "Resistor_Generic":
    dimensions: {length}mm by {width}mm by {height}mm
    
    render:
        type: parametric
        generator: "smd_resistor"
        parameters:
            length: {length}
            width: {width}
            height: {height}
            body_color: "#111111"
            endcap_color: "#C0C0C0"
            value_text: "{value}"
```

**Usage**:
```hw
add Resistor_Generic(
    length=2.0,
    width=1.2,
    height=0.5,
    value="10k"
) at [1, 10, 10]
```

**Result**: Perfect 3D model generated automatically, no asset download needed.

---

## Conclusion

The 3D Asset Mapping system bridges the gap between CAD (Computer-Aided Design) and CGI (Computer-Generated Imagery) seamlessly inside Hardware Script.

**Key Innovations**:
1. Components have dual identity (math + graphics)
2. Compiler separates concerns based on target
3. `.glb` format enables web viewing
4. Procedural generation for simple components
5. Perfect alignment guaranteed by mathematics

**This is how you break free from conventional EDA software.**

---

**Document Status**: 3D Asset Mapping Architecture  
**Version**: 1.0  
**Last Updated**: March 2026  
**This is how we make hardware design cinematic.**

