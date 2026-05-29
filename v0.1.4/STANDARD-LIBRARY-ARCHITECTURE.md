# Standard Library Architecture (v0.1.4)

## Decision: Standard Library File Approach ✅

After evaluating four options, we chose **Option 1: Standard Library File** as the implementation strategy.

## Why Standard Library File?

### The Problem
Hardcoding every unit (µF, nF, pF, %, ppm, mAh, dBm) in the compiler creates:
- Bloated binary
- Requires recompilation for new units
- Users can't customize
- Doesn't scale to infinite domain-specific units

### The Solution
Follow the F#/Nim/Rust pattern: **Compiler stays tiny, domain knowledge lives in standard library files**.

## Architecture

```
Hardware Script System
├── Compiler Core (hwc / Rust binary)
│   └── Only knows: mm, V, A, C (geometry + safety)
│
├── Standard Library (@std / ships with compiler)
│   ├── units.hw          # Defines: µF, nF, %, ppm, mAh, dBm
│   ├── materials.hw      # Raw elements: Copper, FR4, Silicon, Gold
│   ├── components.hw     # Generic primitives: Resistor_0805, Capacitor_0603
│   └── logic.hw          # Foundational gates: NAND, NOR, XOR
│
└── HPM Registry (cloud-hosted / community packages)
    ├── @espressif/esp32_wroom_32    # Vendor chips
    ├── @adafruit/motor_driver       # Community modules
    └── @rf/custom_antenna           # Specialized components
```

### The Critical Distinction

**Standard Library** = Bundled with compiler, works offline, irreducible baseline for physical reality

**HPM Registry** = Cloud-hosted (or self-hosted), millions of vendor chips, downloaded on-demand

This mirrors the architecture of world-class languages:
- Rust: `std` (built-in) vs `crates.io` (registry)
- Node.js: `node:fs` (core modules) vs `npm` (registry)
- Python: `sys` (built-in) vs `PyPI` (registry)

### File Locations (Priority Order)

1. **Project-local override**: `./stdlib/units.hw` (highest priority)
   - Per-project customization
   - Overrides all other locations

2. **User customization**: `~/.hw/stdlib/units.hw`
   - User's global customizations
   - Persists across projects

3. **Installed default**: `<install_dir>/stdlib/units.hw`
   - Ships with compiler
   - Official standard library

### Loading Strategy

```rust
// At compiler startup
let stdlib_loader = StdlibLoader::new();
let units = stdlib_loader.load()?;  // Loads from first available path
let registry = UnitRegistry::new(units);  // O(1) lookup

// During parsing
if registry.is_defined("µF") {
    // Known unit - validate and convert
} else {
    // Unknown unit - store as Custom(String)
}
```

## Implementation

### Crate Structure

```
hwc/
├── stdlib/
│   ├── units.hw          # Ships with compiler
│   └── README.md         # User documentation
│
├── crates/
│   └── hwc-stdlib/       # Standard library loader
│       ├── src/
│       │   ├── lib.rs        # Public API
│       │   ├── loader.rs     # File loading logic
│       │   └── registry.rs   # Fast lookup registry
│       └── Cargo.toml
```

### Key Components

**1. StdlibLoader** - Finds and loads `units.hw`
```rust
let loader = StdlibLoader::new();
let units: Vec<UnitDefinition> = loader.load()?;
```

**2. UnitRegistry** - Fast O(1) lookup
```rust
let registry = UnitRegistry::new(units);
registry.is_defined("µF");  // true
registry.to_base_si(100.0, "µF");  // Some(1e-4)
```

**3. UnitDefinition** - Parsed from `units.hw`
```rust
pub struct UnitDefinition {
    pub name: String,
    pub symbol: String,
    pub aliases: Vec<String>,
    pub base_si: Option<String>,
    pub multiplier: Option<f64>,
    pub dimension: String,
    // ...
}
```

## Unit Definition Format

```hw
define unit "Microfarad":
    symbol: "µF"
    aliases: ["uF", "microF"]
    base_si: "F"
    multiplier: 1e-6
    dimension: "capacitance"
    description: "Most common capacitor unit"
    examples: ["10µF", "100µF"]  # Note: No space between number and unit
```

**Critical Syntax Rule**: Units must be attached directly to numbers with NO space.
- ✅ Correct: `10µF`, `100nF`, `4.7kΩ`
- ❌ Incorrect: `10 µF`, `100 nF`, `4.7 kΩ`

This follows CSS (`10px`) and Rust (`10u32`) conventions for unambiguous parsing.

### Required Fields
- `symbol` - The unit symbol
- `dimension` - Unit category (capacitance, inductance, ratio, etc.)

### Optional Fields
- `aliases` - Alternative symbols
- `base_si` - Base SI unit for conversion
- `multiplier` - Conversion factor
- `description` - Human-readable description
- `note` - Additional notes
- `examples` - Usage examples

## Dimensions

The standard library supports these dimension categories:

| Dimension | Examples | Use Case |
|-----------|----------|----------|
| `capacitance` | F, µF, nF, pF | Capacitor values |
| `inductance` | H, µH, mH | Inductor values |
| `frequency` | Hz, kHz, MHz, GHz | Clock speeds, signals |
| `ratio` | %, ppm, ppb | Tolerance, precision |
| `power` | W, mW, kW | Power consumption |
| `energy` | J, Wh | Battery energy |
| `charge` | mAh, Ah | Battery capacity |
| `logarithmic_power` | dBm | RF signal strength |
| `time` | ms, µs, ns | Signal timing |
| `angle` | rad, mrad | Rotation |
| `wire_gauge` | AWG, SWG | Wire sizing |
| `custom` | User-defined | Domain-specific |

**Note**: The `@std` library provides commonly-used units. For specialized domains (RF engineering, nuclear sensors, automotive), use community packages from HPM:

```hw
# RF Engineering units
import "@rf-engineering/microwave-units"

# Nuclear sensor units
import "@nuclear/radiation-units"

# Automotive standards
import "@automotive/can-bus-units"
```

This prevents the standard library from becoming bloated with every possible unit while enabling infinite extensibility.

## Import Namespace & Security

### The `@std/` Namespace (Reserved & Hardcoded)

Hardware Script uses **scoped namespaces** for imports, with `@std/` (or `@standard/`) reserved exclusively for the standard library:

```hw
# Standard Library Import (bundled with compiler)
import Copper from "@std/materials"
import Resistor_0805 from "@std/components"
import NAND from "@std/logic"

# HPM Registry Import (downloaded from cloud)
import ESP32 from "@espressif/microcontrollers"
import MotorDriver from "@adafruit/drivers"
```

### Compiler Intercept (Security by Design)

When the compiler sees `@std/` or `@standard/`:

1. **Does NOT check** `hw.toml` dependencies
2. **Does NOT look** in `~/.hw/packages/` cache
3. **Routes directly** to `stdlib/` folder bundled with compiler installation

This is implemented in `hwc-compiler` Pass 1 (Symbol Registration):

```rust
fn resolve_import(import_path: &str) -> Result<ModulePath, ImportError> {
    // Hardcoded intercept for standard library
    if import_path.starts_with("@std/") || import_path.starts_with("@standard/") {
        return resolve_stdlib_import(import_path);
    }
    
    // Otherwise, resolve from hw.toml and HPM cache
    resolve_hpm_import(import_path)
}
```

### Registry Protection

The HPM Registry server **hard-blocks** the `@std` and `@standard` organization scopes:

```bash
$ hpm publish --name @std/virus
❌ Error: Reserved Namespace
   The @std and @standard scopes are reserved for the compiler's standard library.
   Choose a different organization name.
```

### Why This Architecture is Unspoofable

- **Zero Parser Changes**: Uniform syntax for all imports
- **Compiler-Level Security**: `@std/` intercept happens before network access
- **Visual Clarity**: Developers instantly know `@std/` has zero external dependencies
- **Prevents Dependency Confusion**: Malicious packages cannot shadow standard library

This follows the **Rust model** (hardcoded `std` crate) and fixes the **Node.js vulnerability** (early npm allowed shadowing core modules).

## User Workflow

### Using Standard Units (Zero Configuration)
```hw
# Parametric component with standard units (NO SPACES)
define component "Capacitor_0805" (cap: Measurement, tol: Measurement):
    electrical:
        capacitance: cap     # Direct parameter reference
        tolerance: tol       # Just works - defined in stdlib

# Usage with keyword arguments (NO SPACES between number and unit)
add Capacitor_0805 (cap: 10µF, tol: 10%) named C1 at [x:10, y:10, z:1]
```

**Important**: Units must be attached to numbers: `10µF` not `10 µF`

### Adding Custom Units
1. Create `~/.hw/stdlib/units.hw`
2. Add definition:
```hw
define unit "MyUnit":
    symbol: "MU"
    dimension: "custom"
    description: "My custom unit"
```
3. Use immediately - no recompilation!

### Project-Specific Units
1. Create `./stdlib/units.hw` in project
2. Add project-specific units
3. Overrides global stdlib for this project only

### Overriding Standard Library Definitions

If you disagree with the default definition of a unit or component, create a local override:

```
my_project/
├── stdlib/
│   ├── units.hw       # Override standard unit definitions
│   └── materials.hw   # Override standard material properties
└── main.hw
```

The compiler's loading priority (highest to lowest):
1. `./stdlib/` (project-local override)
2. `~/.hw/stdlib/` (user customization)
3. `<install_dir>/stdlib/` (compiler default)

This allows per-project customization without modifying the compiler installation.

## Benefits

### For Users
- ✅ Zero configuration - units just work
- ✅ Extensible - add units without recompiling
- ✅ Customizable - override per-project or globally
- ✅ Future-proof - new datasheets? Just edit text file

### For Compiler
- ✅ Tiny binary - only 4 core units (mm, V, A, C)
- ✅ Fast - O(1) unit lookup via hash map
- ✅ Maintainable - no unit whack-a-mole
- ✅ Scalable - infinite units without code changes

### For Language
- ✅ Professional - follows F#/Nim/Rust pattern
- ✅ Flexible - domain-specific units without language changes
- ✅ Documented - units.hw is self-documenting
- ✅ Testable - users can validate their custom units
- ✅ Scalable - community can extend without core team bottleneck
- ✅ Prevents maintainer burnout - no infinite PRs for niche units

## Comparison with Alternatives

| Approach | Binary Size | User Extensible | Recompile Needed | Configuration |
|----------|-------------|-----------------|------------------|---------------|
| **Standard Library** ✅ | Tiny | Yes | No | Zero |
| Compiler Internal | Large | No | Yes | Zero |
| Language Spec | Tiny | Yes | No | Every project |
| Independent File | Tiny | Yes | No | Every project |

## Future Enhancements

### Phase 2: Unit Validation
- Warn on undefined units
- Suggest similar units (typo detection)
- Suggest community packages that define the unit
- Dimension checking (can't add capacitance + resistance)

### Phase 3: Unit Conversion
- Automatic unit conversion in expressions
- `10µF + 1000nF` → `11µF`
- Dimensional analysis

### Phase 4: Custom Dimensions
- User-defined dimension categories
- Domain-specific physics
- Industry-specific units

### Phase 5: Community Unit Discovery
- `hwc suggest-unit dBm` → "Found in @rf-engineering/microwave-units"
- Automatic import suggestions in IDE
- Popular unit packages featured in registry

## Testing

```bash
# Test stdlib loading
cargo test --package hwc-stdlib

# Test unit parsing
cargo test --package hwc-parser --test unit_parsing_test

# Test full integration
cargo test --workspace
```

## Files Modified/Created

### New Files
- `hwc/stdlib/units.hw` - Standard library unit definitions
- `hwc/stdlib/README.md` - User documentation
- `hwc/crates/hwc-stdlib/` - Standard library loader crate
- `hwc/crates/hwc-parser/src/ast/unit.rs` - Unit AST
- `hwc/crates/hwc-parser/tests/unit_parsing_test.rs` - Tests

### Modified Files
- `hwc/crates/hwc-parser/src/ast/mod.rs` - Added Unit definition
- `hwc/crates/hwc-parser/src/lexer/token.rs` - Added Unit keyword
- `hwc/crates/hwc-parser/src/parser/definitions.rs` - Added unit parsing
- `hwc/crates/hwc-compiler/src/two_pass_compiler.rs` - Handle Unit definitions

## Conclusion

The Standard Library File approach provides the perfect balance:
- **Compiler stays tiny** (only 4 essential physics units: mm, V, A, C)
- **Users get flexibility** (add units without recompiling)
- **Zero configuration** (works out of the box)
- **Future-proof** (scales to infinite domain-specific units)
- **Community-driven** (domain experts maintain specialized units)
- **Prevents maintainer burnout** (no infinite PRs for niche units)

This is the industry-standard approach used by world-class languages, and it's now implemented in Hardware Script v0.1.4.

**The Ecosystem Strategy**:
- **Core Team**: Maintains compiler + 4 physics units + `@std` library
- **Community**: Builds domain-specific units (RF, nuclear, automotive, mechanical)
- **Users**: Explicitly import what they need
- **Promotion Path**: Popular community packages can be absorbed into `@std` in future versions

**See Also**: `GENERIC-MEASUREMENT-ARCHITECTURE.md` for the low-level compiler implementation details that enable the `Unit::Custom(String)` escape hatch used by this standard library system.

## Standard Library Contents

The standard library ships with the compiler at `hwc/stdlib/`:

```
hwc/stdlib/
├── units.hw          # Unit definitions (µF, nF, %, ppm, mAh, dBm, etc.)
├── materials.hw      # Standard materials (11 materials: Copper, FR4, Silicon, etc.)
├── profiles.hw       # Manufacturing profiles (5 profiles: PCB + ASIC)
├── components.hw     # Standard components (10 parametric components: resistors, caps, LEDs)
└── README.md         # User documentation
```

### Materials Library

The standard library includes 11 materials covering conductors, insulators, and semiconductors:

**Conductors:**
- Copper (universal PCB trace material)
- Aluminum (chip-level routing, heat sinks)
- Gold (contact pads, wire bonding)
- Silver (highest conductivity, RF applications)

**Insulators:**
- FR4 (standard PCB substrate)
- Air (ambient environment, spark gap calculations)
- SiliconDioxide (microchip insulator)
- Polyimide (flexible PCB substrate)

**Semiconductors:**
- Silicon (primary IC material)
- GalliumNitride (wide bandgap, power electronics)
- GalliumArsenide (high-frequency, optoelectronics)

Each material includes all required properties (resistivity, thermal_conductivity, density, etc.) sourced from Materials Project API, IPC standards, and NIST Chemistry WebBook.

### Components Library

The standard library includes 10 parametric components covering the most common passive parts:

**SMD Resistors:**
- `Resistor_0805` (val: Measurement, tol: Measurement) - 0.125W, 150V max
- `Resistor_0603` (val: Measurement, tol: Measurement) - 0.1W, 75V max
- `Resistor_1206` (val: Measurement, tol: Measurement) - 0.25W, 200V max

**SMD Capacitors:**
- `Capacitor_0805` (cap: Measurement, voltage: Measurement) - Ceramic
- `Capacitor_0603` (cap: Measurement, voltage: Measurement) - Ceramic
- `Capacitor_1206` (cap: Measurement, voltage: Measurement) - Ceramic

**SMD LEDs:**
- `LED_0805` (color: String) - 20mA max, 2.0V forward voltage
- `LED_0603` (color: String) - 20mA max, 2.0V forward voltage

**Through-Hole:**
- `Resistor_TH` (val: Measurement, tol: Measurement) - 1/4W axial resistor
- `LED_5mm` (color: String) - Standard 5mm LED

**Usage Examples:**
```hw
# Import from standard library (parser support, resolution pending)
import Resistor_0805 from "@std/components"
import LED_0805 from "@std/components"

# Use with keyword arguments
add Resistor_0805 (val: 10kΩ, tol: 5%) named R1 at [x:10, y:10, z:2]
add Capacitor_0805 (cap: 100nF, voltage: 16V) named C1 at [x:20, y:10, z:2]
add LED_0805 ("Red") named LED1 at [x:30, y:10, z:2]
```

All components include:
- Accurate physical dimensions from datasheets
- Pin positions using correct syntax: `PinName at [0mm, 0mm]`
- Electrical specifications (resistance, capacitance, voltage ratings)
- Render properties for 3D visualization

### Profiles Library

The standard library includes 5 profiles covering PCB and ASIC manufacturing:

**PCB Profiles:**
1. `PCB_Standard` - IPC-2221 Class 2 (100µm traces, 35µm copper, 85°C max)
2. `PCB_HighTemp` - Automotive/industrial (150µm traces, 70µm copper, 125°C max)
3. `PCB_HighVoltage` - Power electronics (200µm traces, 70µm copper, >150V)

**ASIC Profiles:**
4. `ASIC_5nm` - TSMC N5 process (28nm traces, 100nm copper, 105°C max)
5. `ASIC_180nm` - Mature node (220nm traces, 500nm copper, 125°C max)

Each profile includes all constraint blocks:

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
        low_voltage_threshold: 50V
        medium_voltage_threshold: 150V
    
    thermal:
        ambient_temp: 25C
        max_operating_temp: 85C
        max_temp_rise: 20C
        clustering_threshold: 5mm
    
    manufacturing:
        copper_thickness: 35µm
        ipc2221_k_external: 0.048
        ipc2221_k_internal: 0.024
```

**Profile Block Reference:**

- `trace:` - Minimum trace geometry (min_width, min_spacing)
- `via:` - Via constraints (min_diameter, min_annular_ring)
- `clearance:` - Voltage-dependent clearances (high_voltage, safety_factor, voltage thresholds)
- `thermal:` - Temperature limits (ambient, max operating, max rise, clustering threshold)
- `manufacturing:` - Process-specific constants (copper thickness, IPC-2221 k-values)

All values have sensible defaults. Users only override what's specific to their application.

## Profile Customization

### Manufacturing Constants

Profiles now support customizable manufacturing parameters that were previously hardcoded:

```hw
define profile "CustomPCB":
    manufacturing:
        copper_thickness: 70µm        # 2oz copper (default: 35µm = 1oz)
        ipc2221_k_external: 0.048     # IPC-2221 external layer constant
        ipc2221_k_internal: 0.024     # IPC-2221 internal layer constant
```

These values affect trace width calculations for current-carrying capacity. Different manufacturing processes (PCB vs ASIC) require different constants.

### Voltage Classification

Profiles support customizable voltage thresholds for clearance calculations:

```hw
define profile "CustomPCB":
    clearance:
        high_voltage: 3mm
        safety_factor: 2.5
        low_voltage_threshold: 50V     # Below this: low voltage rules
        medium_voltage_threshold: 150V # Below this: medium voltage rules
```

The compiler uses these thresholds to determine which clearance rules apply to each net. This allows different safety standards for different applications (consumer electronics vs industrial equipment).

### Use Cases

**Heavy Copper PCB** (2oz copper for high current):
```hw
define profile "PowerPCB":
    manufacturing:
        copper_thickness: 70µm
        ipc2221_k_external: 0.048
        ipc2221_k_internal: 0.024
```

**ASIC Process** (different thermal environment):
```hw
define profile "ASIC_Custom":
    manufacturing:
        copper_thickness: 500nm
        ipc2221_k_external: 0.024  # Different k-value for chip environment
        ipc2221_k_internal: 0.012
```

**High Voltage Application** (stricter safety margins):
```hw
define profile "IndustrialHV":
    clearance:
        high_voltage: 10mm
        safety_factor: 3.0
        low_voltage_threshold: 100V   # Higher threshold for industrial
        medium_voltage_threshold: 300V
```

All values have sensible defaults from the standard library, so users only need to override what's specific to their application.
