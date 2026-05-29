# Book 4: The Compiler Internals

**Hardware Script v0.1.4**  
**Target Audience**: Rust systems programmers contributing to the compiler  
**Last Updated**: March 2026

---

## Architecture

Hardware Script is a hardware synthesis engine that transforms `.hw` files into manufacturing outputs (Gerber, GDSII, 3D models).

```
.hw Source → Unified AST → Hardware IR → Voxel Grid → Manufacturing Files
```

Everything is defined in `.hw` files using `define` blocks. The compiler uses a two-pass architecture with a Symbol Table to resolve all definitions before spatial assembly.

---

## The Multi-Level IR Pipeline

```
Layer 1: Intent (.hw source)
         ↓
Layer 2: Logical IR (hwc-compiler) — Two-Pass Compilation
         ↓
Layer 3: Physical IR (hwc-engine) — Voxel Grid + Routing
         ↓
Layer 4: Physics IR (hwc-physics) — Validation
         ↓
Layer 5: Manufacturing (hwc-export) — File Generation
```

---

## Layer 2: Two-Pass Compilation

The `hwc-compiler` uses a two-pass architecture:

**Pass 1 (Symbol Registration)**: Registers all `define material`, `define profile`, `define component`, `define module`, `define mechanical` blocks into a Symbol Table.

**Pass 2 (Space Unrolling & Assembly)**: Flattens all `define module` logic, unrolls `for` loops and `if` conditionals (Comptime evaluation), maps relative coordinates to absolute coordinates, and builds the Hardware IR.

### Implementation

```rust
pub fn compile_to_ir(program: Program) -> Result<HardwareIR, CompileError> {
    let mut ir = HardwareIR::new();
    let mut symbol_table = SymbolTable::new();
    
    // Pass 1: Symbol Registration
    for def in program.definitions {
        match def {
            Definition::Material(m) => symbol_table.register_material(m)?,
            Definition::Profile(p) => symbol_table.register_profile(p)?,
            Definition::Component(c) => symbol_table.register_component(c)?,
            Definition::Module(m) => symbol_table.register_module(m)?,
            Definition::Mechanical(m) => symbol_table.register_mechanical(m)?,
            Definition::Space(s) => ir.space_ast = Some(s),
            _ => {}
        }
    }
    
    let space_ast = ir.space_ast.take()
        .ok_or(CompileError::NoSpaceDefined)?;

    // Pass 2: Space Unrolling & Assembly
    ir.space_name = space_ast.name;
    ir.dimensions_nm = convert_mm_to_nm(space_ast.dimensions);
    ir.grid = space_ast.grid;
    ir.voxel_size_nm = calculate_voxel_size_nm(ir.dimensions_nm, ir.grid);
    
    if let Some(profile_name) = space_ast.profile {
        ir.constraints = symbol_table.get_profile(&profile_name)?;
    }
    
    // Unroll for loops and if conditionals (Comptime evaluation)
    let unrolled_statements = unroll_comptime_logic(&space_ast.statements)?;
    
    for statement in unrolled_statements {
        match statement {
            Statement::ComponentPlacement(component_ast) => {
                // Check if it's a module (logical schematic)
                if let Some(module_def) = symbol_table.get_module(&component_ast.type_name) {
                    // Flatten the module into the current space
                    let flattened = flatten_module(
                        &module_def,
                        &component_ast,
                        &symbol_table
                    )?;
                    ir.components.extend(flattened.components);
                    ir.nets.extend(flattened.nets);
                } else {
                    // Regular component
                    let component_def = symbol_table.get_component(&component_ast.type_name)?;
                    
                    let placed = PlacedComponent {
                        id: ComponentId::new(),
                        name: component_ast.name,
                        position: component_ast.position,  // Top-Left Anchor
                        pins: transform_pins_to_global(
                            &component_def.pins,
                            component_ast.position,  // Anchor point
                            component_ast.rotation
                        ),
                        bounding_box: calculate_bounding_box(&component_def, component_ast.position),
                    };
                    ir.components.push(placed);
                }
            }
            Statement::Route(route_ast) => {
                let from_pin = resolve_pin_reference(&route_ast.from, &ir.components)?;
                let to_pin = resolve_pin_reference(&route_ast.to, &ir.components)?;
                
                let net = NetRoute {
                    id: NetId::new(),
                    name: route_ast.name,
                    from_pin,
                    to_pin,
                    waypoints: route_ast.waypoints,
                    width_voxels: calculate_width_voxels(route_ast.width, ir.voxel_size_nm),
                    material: resolve_material(&route_ast.material, &symbol_table)?,
                    constraints: resolve_constraints(&route_ast.constraints)?,
                };
                ir.nets.push(net);
            }
            Statement::Layout(layout_block) => {
                // Map module's internal components to physical coordinates
                apply_layout_mapping(&layout_block, &mut ir.components)?;
            }
            _ => {}
        }
    }
    
    Ok(ir)
}

/// Unrolls for loops and evaluates if conditionals at compile time
fn unroll_comptime_logic(statements: &[Statement]) -> Result<Vec<Statement>, CompileError> {
    let mut result = Vec::new();
    
    for stmt in statements {
        match stmt {
            Statement::For(for_loop) => {
                // Evaluate range at compile time
                for i in for_loop.start..=for_loop.end {
                    // Substitute loop variable in body
                    let instantiated = substitute_loop_var(&for_loop.body, &for_loop.var, i)?;
                    result.extend(instantiated);
                }
            }
            Statement::If(if_stmt) => {
                // Evaluate condition at compile time
                if evaluate_condition(&if_stmt.condition)? {
                    result.extend(unroll_comptime_logic(&if_stmt.then_body)?);
                } else if let Some(else_body) = &if_stmt.else_body {
                    result.extend(unroll_comptime_logic(else_body)?);
                }
            }
            _ => result.push(stmt.clone()),
        }
    }
    
    Ok(result)
}

/// Flattens a logical module into physical components and nets
fn flatten_module(
    module: &ModuleDefinition,
    placement: &ComponentPlacement,
    symbol_table: &SymbolTable
) -> Result<FlattenedModule, CompileError> {
    let mut flattened = FlattenedModule::new();
    
    // Recursively flatten nested modules
    for component in &module.components {
        if let Some(nested_module) = symbol_table.get_module(&component.type_name) {
            let nested = flatten_module(nested_module, component, symbol_table)?;
            flattened.components.extend(nested.components);
            flattened.nets.extend(nested.nets);
        } else {
            flattened.components.push(component.clone());
        }
    }
    
    // Copy routes from module
    flattened.nets.extend(module.routes.clone());
    
    Ok(flattened)
}
```

### Hardware IR Structure

```rust
pub struct HardwareIR {
    pub space_name: String,
    pub dimensions_nm: (i64, i64, i64),  // Fixed-point nanometers
    pub grid: (usize, usize, usize),
    pub voxel_size_nm: i64,
    pub components: Vec<PlacedComponent>,
    pub nets: Vec<NetRoute>,
    pub materials: MaterialDatabase,  // Populated from Symbol Table
    pub constraints: ConstraintSet,
}

pub struct ModuleDefinition {
    pub name: String,
    pub pins: Vec<PinDeclaration>,
    pub components: Vec<ComponentPlacement>,
    pub routes: Vec<RouteDefinition>,
    pub parameters: Vec<Parameter>,  // For generics
}

pub struct Parameter {
    pub name: String,
    pub param_type: ParameterType,
}

pub enum ParameterType {
    Measurement,
    String,
    Number,
}

pub struct FlattenedModule {
    pub components: Vec<PlacedComponent>,
    pub nets: Vec<NetRoute>,
}
```

---

## Layer 3: Physical IR — Data-Oriented Design

The `hwc-engine` uses Data-Oriented Design with fixed-point math and Morton encoding for deterministic, cache-friendly spatial operations.

### NetlistArena (ECS)

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub struct ComponentId(u32);

#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub struct PinId(u32);

#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub struct NetId(u32);

pub struct NetlistArena {
    pub components: Vec<ComponentData>,
    pub pins: Vec<PinData>,
    pub nets: Vec<NetData>,
}
```

Strongly-typed IDs enable instant array lookups: `arena.pins[pin_id.0 as usize].connected_net`.

### Morton Encoding (Z-Curve)

```rust
fn morton_encode(x: u32, y: u32, z: u32) -> u64 {
    let mut result = 0u64;
    for i in 0..21 {
        result |= ((x & (1 << i)) as u64) << (2 * i);
        result |= ((y & (1 << i)) as u64) << (2 * i + 1);
        result |= ((z & (1 << i)) as u64) << (2 * i + 2);
    }
    result
}

struct VoxelGrid {
    materials: Vec<u8>,
    net_ids: Vec<u32>,
    collision_mask: Vec<u64>,
}
```

Voxels close in physical space are close in RAM. 100× performance improvement over HashMap.

### Fixed-Point Math

```rust
// All coordinates use i64 nanometers for determinism
let clearance_nm = 5_200_000 * 15 / 10;  // Always 7_800_000
// Identical on x86, ARM, RISC-V
```

### Top-Left Anchor Coordinate System

**Component Placement**: All component positions specify the **top-left-front corner** of the component's bounding box.

```rust
// Component at position [15mm, 20mm, 1mm] with size 3mm × 2.5mm × 1mm
let position = Point3D::new(15_000_000, 20_000_000, 1_000_000);  // Top-left corner
let size = (3_000_000, 2_500_000, 1_000_000);  // Width, height, depth

// Bounding box extends positively from anchor
let bbox = BoundingBox::new(
    position,  // Min corner (top-left-front)
    Point3D::new(
        position.x + size.0,  // Max X (right edge)
        position.y + size.1,  // Max Y (bottom edge)
        position.z + size.2,  // Max Z (back edge)
    )
);
```

**Pin Position Calculation**: Pins are defined as offsets from the top-left corner.

```rust
// Pin at [1.5mm, 0mm, 0mm] in component layout
let pin_offset = Point3D::new(1_500_000, 0, 0);

// Absolute pin position = component anchor + pin offset
let pin_absolute = Point3D::new(
    position.x + pin_offset.x,  // 15mm + 1.5mm = 16.5mm
    position.y + pin_offset.y,  // 20mm + 0mm = 20mm
    position.z + pin_offset.z,  // 1mm + 0mm = 1mm
);
```

This system matches PCB CAD industry standards (KiCad, Altium) and eliminates complex center-based coordinate transformations.

For detailed geometry structures (Point3D, BoundingBox, TraceSegment), see `Docs/v0.1.3/COMPILER-INTERNALS.md`.

---

## Layer 4: Physics Validation

The `hwc-physics` crate validates electrical, thermal, electromagnetic properties and clearance requirements before manufacturing.

---

## Layer 5: Manufacturing Export

The `hwc-export` crate converts validated voxel grids into industry-standard formats using custom emitters (no bloated dependencies). See Book 6 for implementation.

---

## The 8 Rust Crates

```
hwc-cli        - Pipeline orchestration
hwc-parser     - Lexer, parser, unified AST (native SI unit parsing)
hwc-compiler   - Two-pass compilation with Symbol Table & Import Resolution
hwc-engine     - Voxel grid with Morton encoding
hwc-physics    - Physics validation
hwc-export     - Manufacturing file generation
hwc-materials  - Material property structures (ConductorProperties, InsulatorProperties)
hwc-stdlib     - Standard library loader & registry
```

### Material System Architecture

Materials are defined in the standard library (`hwc/stdlib/materials.hw`) and can be overridden by user code:

**Standard Library Materials** (`hwc/stdlib/materials.hw`):
```hw
define material "Copper":
    category: conductor
    symbol: "Cu"
    description: "Universal PCB trace material"
    
    properties:
        resistivity: 1.68e-8Ω·m
        thermal_conductivity: 401W/mK
        density: 8960kg/m³
        max_current_density: 35A/mm²
        melting_point: 1085C
        reference_temp: 20C
```

**User Override**:
```hw
import FR4 from standard.materials  # Use standard FR4

# Override Copper with custom properties
define material "Copper":
    category: conductor
    properties:
        resistivity: 2.0e-8Ω·m
        thermal_conductivity: 380W/mK
        density: 8900kg/m³
        max_current_density: 30A/mm²
        melting_point: 1085C
        color: "#B87333"
```

**Material Database Population** (`hwc-compiler/src/conversions.rs`):

During compilation, materials from the Symbol Table are converted to the `MaterialDatabase`:

```rust
pub fn populate_material_database(
    symbol_table: &SymbolTable,
) -> Result<MaterialDatabase, ConversionError> {
    let mut database = MaterialDatabase::empty();

    for (name, material_def) in symbol_table.materials() {
        match material_def.category {
            MaterialCategory::Conductor => {
                // Extract required properties (no hardcoded defaults)
                let mut resistivity_ohm_m = None;
                let mut thermal_conductivity_w_mk = None;
                let mut density_kg_m3 = None;
                let mut max_current_density_a_mm2 = None;
                let mut melting_point_c = None;
                
                // Parse properties from material definition
                for prop in &material_def.properties {
                    match prop.key.as_str() {
                        "resistivity" => resistivity_ohm_m = Some(convert_to_base_unit(...)),
                        "thermal_conductivity" => thermal_conductivity_w_mk = Some(...),
                        // ... other properties
                    }
                }
                
                // Validate all required properties are present
                let resistivity_ohm_m = resistivity_ohm_m.ok_or_else(|| {
                    ConversionError::MissingProperty {
                        material: name.clone(),
                        property: "resistivity".to_string(),
                    }
                })?;
                
                // Create conductor with validated properties
                let conductor = ConductorProperties {
                    name: name.clone(),
                    resistivity_ohm_m,
                    thermal_conductivity_w_mk,
                    // ...
                };
                
                database.conductors.insert(name.to_lowercase(), conductor);
            }
            MaterialCategory::Insulator => { /* similar */ }
            MaterialCategory::Semiconductor => { /* similar */ }
        }
    }
    
    Ok(database)
}
```

**Key Design Decisions**:

1. **No Hardcoded Defaults**: All material properties must come from material definitions (either stdlib or user-defined). This eliminates hidden assumptions.

2. **Standard Library as Defaults**: The `hwc/stdlib/materials.hw` file provides scientifically-sourced defaults from Materials Project API, IPC standards, and NIST data.

3. **User Override Priority**: User-defined materials with the same name override stdlib materials during Symbol Table registration.

4. **Required Properties Validation**: Missing required properties (resistivity, thermal_conductivity, density, etc.) cause clear compile-time errors.

5. **Future-Ready for Internet Lookup**: The architecture supports fetching material properties from online databases (Materials Project API) while maintaining offline-first operation with stdlib defaults.

### Profile System Architecture

Profiles define fabrication constraints AND physics limits (thermal, electrical). Like materials, profiles come from the standard library and can be overridden:

**Standard Library Profiles** (`hwc/stdlib/profiles.hw`):
```hw
define profile "PCB_Standard":
    description: "Standard PCB fabrication (IPC-2221 Class 2)"
    
    trace:
        min_width: 100µm
        min_spacing: 100µm
    
    via:
        min_diameter: 300µm
        min_annular_ring: 150µm
    
    clearance:
        high_voltage: 2mm
        safety_factor: 2.0
    
    thermal:
        ambient_temp: 25C
        max_operating_temp: 85C
        max_temp_rise: 20C
        clustering_threshold: 5mm
```

**User Override**:
```hw
import PCB_Standard from standard.profiles  # Use standard profile

# Override for high-temperature application
define profile "PCB_HighTemp":
    description: "High-temperature automotive application"
    
    trace:
        min_width: 150µm
        min_spacing: 150µm
    
    thermal:
        ambient_temp: 85C
        max_operating_temp: 125C
        max_temp_rise: 15C
```

**Profile to ConstraintSet Conversion** (`hwc-compiler/src/conversions.rs`):

```rust
pub fn profile_to_constraints(
    profile: &ProfileDefinition,
) -> Result<ConstraintSet, ConversionError> {
    // Extract thermal constraints (no hardcoded defaults)
    let thermal = profile.thermal.as_ref().map(|thermal_def| {
        ThermalConstraints {
            ambient_temp_c: measurement_to_celsius(&thermal_def.ambient_temp),
            max_operating_temp_c: measurement_to_celsius(&thermal_def.max_operating_temp),
            max_temp_rise_c: measurement_to_celsius(&thermal_def.max_temp_rise),
            clustering_threshold_nm: thermal_def
                .clustering_threshold
                .as_ref()
                .map(measurement_to_nm),
        }
    });
    
    Ok(ConstraintSet {
        name: profile.name.clone(),
        trace,
        via,
        clearance,
        layer,
        thermal,  // Optional - if None, physics validation skips thermal checks
    })
}
```

**Key Design Decisions**:

1. **No Hardcoded Thermal Limits**: Compiler has ZERO hardcoded temperature limits. All values come from profiles.

2. **Standard Library as Defaults**: `hwc/stdlib/profiles.hw` provides industry-standard defaults (IPC-2221, automotive, ASIC processes).

3. **User Override Priority**: User-defined profiles override stdlib profiles during Symbol Table registration.

4. **Optional Thermal Constraints**: If profile has no `thermal:` block, physics validation skips thermal checks (allows pure geometric validation).

5. **Physics Validation Uses Constraints**: The physics engine reads `constraints.thermal` for ambient temp, max operating temp, and temp rise limits - no magic numbers.
```

### Import Resolution (hwc-compiler)

The compiler implements **namespace-based security** for imports:

```rust
// Pass 1: Symbol Registration includes import resolution
fn resolve_import(import_path: &str, symbol_table: &mut SymbolTable) -> Result<(), ImportError> {
    // Hardcoded intercept for standard library
    if import_path.starts_with("@std/") || import_path.starts_with("@standard/") {
        // Route to bundled stdlib/ folder (no network access)
        return resolve_stdlib_import(import_path, symbol_table);
    }
    
    // Otherwise, resolve from hw.toml and HPM cache
    resolve_hpm_import(import_path, symbol_table)
}

fn resolve_stdlib_import(import_path: &str, symbol_table: &mut SymbolTable) -> Result<(), ImportError> {
    let stdlib_path = get_compiler_stdlib_path()?;
    let module_path = import_path.strip_prefix("@std/")
        .or_else(|| import_path.strip_prefix("@standard/"))
        .ok_or(ImportError::InvalidStdlibPath)?;
    
    let file_path = stdlib_path.join(format!("{}.hw", module_path));
    let module_ast = parse_file(&file_path)?;
    
    // Register all definitions from stdlib module
    for def in module_ast.definitions {
        symbol_table.register(def)?;
    }
    
    Ok(())
}
```

This **compiler-level intercept** prevents dependency confusion attacks and ensures `@std/` imports never touch the network.

---

## Error Handling

3-character error codes using `miette` and `thiserror`:

- **S** - Syntax errors
- **C** - Compiler errors
- **R** - Routing errors
- **P** - Physics errors
- **M** - Manufacturing errors

Example:
```
❌ Error[P16]: Dielectric Breakdown Risk
   ╭─[main.hw:42:1]
 42 │     route Power_120V.out to Relay.in:
   ·           ──────┬───────
   ·                 ╰── High voltage net (120V)
 45 │             -[x:10, y:50, z:2]
   ·               ──────┬──────
   ·                     ╰── Approaches within 0.02mm of MCU_5V net
   ╰────
  help: Voltage difference is 115V. Through Air, requires 
        minimum 0.08mm clearance to prevent arcing.
```

See `Docs/v0.1.2/ERROR-HANDLING-PHILOSOPHY.md` for complete philosophy.

---

## Hyper-Lean Dependency Stack

4 external dependencies:

- **logos** — Fast lexing
- **miette & thiserror** — Diagnostics
- **rayon** — Multi-threading

No `serde_yaml` (native SI unit parsing eliminates YAML). No parser combinators (hand-written parser). No generic 3D math (custom integer-based). No graph libraries (custom NetlistArena).

```
Dependencies: 4 libraries
Compile time: <10 seconds
Binary size: 5MB
Control: Complete
```

---

## Design Principles

### Logical/Physical Duality (from HardwareScript Philosophy)

The compiler distinguishes between logical intent (`define module`) and physical reality (`define space`):

- **Modules** contain pure electrical connections with no coordinates
- **Spaces** contain absolute physical placement and routing
- **Pass 2** flattens modules into spaces, mapping logic to geometry

This enables true Zero-Cost Abstractions and infinite component reusability.

### Comptime Evaluation (from Zig)

`for` loops and `if` conditionals execute at compile time, not runtime:

```hw
for i in 0..63:
    add SingleBit_ALU named Bit[i]
```

The compiler unrolls this into 64 separate component placements before spatial assembly.

### Parametric Generics (from Rust)

Components accept parameters for reusability:

```hw
define component "Resistor_0805" (val: Measurement, tol: Measurement):
    electrical:
        resistance: val
        tolerance: tol

add Resistor_0805 (val: 10kΩ, tol: 1%) named R1
```

Parameters are substituted during Pass 1 symbol registration.

### Electrical Borrow Checker (from Rust)

A net can have infinite `Input` pins but **exactly one** `Output` pin. Multiple outputs cause short circuits.

```
Error[C42]: Multiple Drivers on Net
Pin A is an Output, and Pin B is an Output. This will cause a short circuit.
```

### No Hidden Control Flow (from Zig)

Every connection must be explicit. No implicit power routing.

```hw
add IC_74HC00 named AND_Gate at [x:10, y:10, z:1]

# Must explicitly route power
route MainPower.5V to AND_Gate.VCC
route MainPower.GND to AND_Gate.GND
```

### Universal Target Compilation (from C)

Same `.hw` file compiles for different factories. Profile defines manufacturing constraints.

```bash
hwc build main.hw --profile JLCPCB_4Layer
hwc build main.hw --profile TSMC_5nm
```

### Zero-Cost Abstractions (from C++)

Modular imports flatten to raw copper traces during compilation. No runtime overhead.

### Interface-Driven Design (from Java)

Components implement interfaces for drop-in replacements:

```hw
define interface "I2C_TempSensor":
    pins: VCC, GND, SDA, SCL

define component "Sensor_A" implements I2C_TempSensor:
    # expensive but accurate

define component "Sensor_B" implements I2C_TempSensor:
    # cheap alternative

# Swap at compile-time
hwc build main.hw --use Sensor_B
```

### Physical Object Model (from JavaScript)

CSS-style selectors for bulk operations:

```hw
apply profile HighCurrent to nets:
    - match: "*_VCC"
    - match: "Motor_*"

apply mechanical HeatsinkKeepout to components:
    - match: class(PowerIC)
```

---

## Contributing

**Architecture Guidelines**:
- Parser builds unified AST
- Compiler resolves symbols and builds IR
- Engine manipulates voxels
- Physics validates
- Export generates files

**Areas for Contribution**:
- Symbol table caching and incremental builds
- Advanced routing algorithms
- GPU acceleration for voxel grid
- Additional export formats (STEP, simulation outputs)
- Comptime optimization for large loop unrolling
- Module flattening performance improvements

---

## Conclusion

Hardware Script's compiler uses a two-pass symbol resolution architecture with Comptime evaluation for `for` loops and `if` conditionals. The Logical/Physical Duality (`define module` vs `define space`) enables fractal encapsulation where modules flatten into spaces during Pass 2. Native SI unit parsing, Data-Oriented Design with Morton encoding, and fixed-point math ensure deterministic, cache-friendly execution. The unified `.hw` ecosystem scales from LED circuits to silicon chips.
