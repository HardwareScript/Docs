# Netlist Architecture: Device Extraction & SPICE Export

**Version**: v0.2.1+  
**Philosophy**: Zero compiler magic. User controls everything. Fail loudly.  
**Architectural Boundary**: Geometry extraction vs. semiconductor physics

---

## The Golden Boundary Line

HardwareScript enforces a strict three-layer architecture to stay lightweight, fast, and extensible:

```
┌─────────────────────────────────────────────────────────────────────────┐
│ 1. CORE COMPILER (hwc-compiler)                                        │
│    • Layout Netlist Topology & Terminal Mapping                         │
│    • Geometric Extraction: Width (W), Length (L), Area (A), Gap (d)     │
│    • Linear Passives: R = ρL/A, C = ε₀εᵣA/d, L_via, C_couple           │
│    • Output: Standard SPICE Netlist (.sp)                               │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │ Passes extracted W, L, R, C + Model Cards
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 2. PDK / STANDARD LIBRARY (diode_pdk.hw, tsmc180nm.hw)                 │
│    • spice_model blocks (.model / .subckt cards)                        │
│    • Semiconductor Physics: IS, VTO, LAMBDA, GAMMA                      │
│    • Process Corners (TT, FF, SS), Temperature Coefficients             │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │ Imports into Simulation Engine
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 3. SIMULATION & TESTING LAYER (hsm / hwc-test / LTSpice)               │
│    • Transient, AC, DC Operating Point, Noise, Monte Carlo              │
│    • Newton-Raphson Matrix Solvers (ngspice.wasm / Xyce)                │
│    • Interactive Probing & Waveform Viewing (uPlot)                     │
└─────────────────────────────────────────────────────────────────────────┘
```

### What This Means

| Feature | Compiler Calculates | PDK Declares | Simulator Solves |
|---------|-------------------|--------------|------------------|
| **Terminal Connectivity** | ✅ Extracts | ❌ | ❌ |
| **R, C, L (Linear Passives)** | ✅ Calculates | ❌ | ❌ |
| **W, L (Transistor Geometry)** | ✅ Extracts | ❌ | ❌ |
| **IS, VTO, LAMBDA (Semiconductor)** | ❌ | ✅ Declares | ❌ |
| **Transient/AC/DC Analysis** | ❌ | ❌ | ✅ Solves |

---

## Overview

The netlist system translates physical hardware layouts into SPICE circuit descriptions for simulation. The architecture is **metadata-driven** - the compiler extracts geometry and linear physics, while complex semiconductor models come from PDK files.

### Core Principle

**The compiler is a pure geometric extraction engine + linear physics calculator.**

**What the Compiler DOES:**
- ✅ Extract topology and terminal connectivity
- ✅ Calculate W, L, Area, thickness from physical geometry
- ✅ Calculate R = ρL/A, C = ε₀εᵣA/d, L_via from geometry + material properties
- ✅ Format standard SPICE netlists

**What the Compiler DOES NOT DO:**
- ❌ Calculate semiconductor physics (IS, VTO, LAMBDA, etc.)
- ❌ Solve non-linear differential equations
- ❌ Hardcode device-specific behavior
- ❌ Provide process-specific model cards

**Why This Boundary?**
1. **Keeps compiler lightweight** (<20MB, <100ms builds)
2. **Avoids obsolescence** - New process nodes don't require Rust code changes
3. **Standard interoperability** - SPICE output runs in LTSpice, Xyce, ngspice, hsm
4. **User control** - PDK physics declared in `.hw` files, not buried in compiler

---

## Architecture Flow

```
1. Device Definition (user declares structure)
   ↓
2. Material Properties (user declares physics)
   ↓
3. Geometry Binding (user binds pours to device terminals)
   ↓
4. Parameter Extraction (compiler calculates from geometry + materials)
   ↓
5. SPICE Generation (compiler formats using device metadata)
```

---

## 1. Device Definitions

### Purpose
Declare the **contract** for a physical device: terminals, materials, and SPICE export format.

### Structure

```hw
device Capacitor:
    terminals: [top, bottom]
    materials:
        top: Aluminum
        bottom: Polysilicon
    spice:
        prefix: C
        terminal_order: [top, bottom]
        parameters: [C]
        parameter_style: positional
```

### Required Fields

#### `terminals: [...]`
List of terminal names. Must match geometry bindings.

#### `materials: { terminal: material }`
Expected materials for each terminal. Enforces physical constraints.

#### `spice:` block
**ALL fields are REQUIRED** - no defaults, fail loudly if missing.

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `prefix` | char | SPICE device prefix | `C` for capacitor, `R` for resistor, `M` for MOSFET |
| `terminal_order` | [string] | Order of terminals in SPICE card | `[top, bottom]` → `CC1 In Out <value>` |
| `parameters` | [string] | Parameter names to extract | `[C]` for capacitance, `[R]` for resistance |
| `parameter_style` | enum | Formatting style: `positional` or `named` | See below |

#### `parameter_style` Values

- **`positional`**: Value as positional argument
  ```spice
  CC1 In Out 1.73e-13
  RR1 A B 1000
  ```
  Used by: R, C, L, V, I, E, G, H, F

- **`named`**: Value as named parameter
  ```spice
  MM1 drain gate source bulk NMOS W=1.5u L=0.18u
  ```
  Used by: M, Q, J, X (transistors, subcircuits)

### Why `parameter_style` is Required

The user must explicitly declare formatting because:
1. **No compiler guessing** - adding a new device type shouldn't require Rust code changes
2. **SPICE syntax varies** - passives use positional, active devices use named
3. **User control** - you decide the output format, not the compiler

---

## 1.5. SPICE Model Cards (PDK-Provided)

### Purpose
Declare **process-specific semiconductor physics** that cannot be calculated from geometry alone.

### When to Use spice_model vs. parameters

| Device Type | Compiler Calculates | PDK Declares (spice_model) |
|-------------|-------------------|---------------------------|
| **Resistor** | ✅ R from geometry | ❌ (none needed) |
| **Capacitor** | ✅ C from geometry | ❌ (none needed) |
| **Inductor** | ✅ L from geometry | ❌ (none needed) |
| **Diode** | ✅ W, L (if applicable) | ✅ IS, N, RS, BV, IBV, CJO |
| **MOSFET** | ✅ W, L from gate geometry | ✅ VTO, KP, LAMBDA, GAMMA, PHI |

### Structure

```hw
spice_model DMOD:
    type: diode
    parameters:
        IS: 1e-12    # Saturation current (cannot calculate from geometry!)
        N: 1.0       # Ideality factor (process-dependent)
        RS: 0.1      # Series resistance (Ohms)
        BV: 40.0     # Breakdown voltage (process-dependent)
        IBV: 1e-10   # Breakdown current
```

### Output Format

The compiler emits this as a standard SPICE `.model` card:

```spice
.model DMOD D (IS=1e-12 N=1.0 RS=0.1 BV=40.0 IBV=1e-10)
```

### Why Not Calculate IS, VTO, etc.?

These parameters depend on:
- **Doping concentration profiles** (not just geometry)
- **Junction depth and grading**
- **Process technology node** (180nm vs. 65nm vs. 5nm)
- **Temperature coefficients**
- **Process variations** (FF, TT, SS corners)

**Attempting to calculate these would require:**
1. TCAD-level process simulation (hours of compute)
2. Doping profile measurement data (not available from layout)
3. Temperature-dependent models (different for each foundry)

**Result:** The compiler would become:
- ❌ 500,000+ lines of physics code
- ❌ Obsolete every process node
- ❌ Slower than LTSpice itself

**Instead:** Let PDKs declare these values, just like real EDA tools do.

---

## 2. Device Definitions with Geometry vs. Model Parameters

### Resistor (Geometry-Derived)

```hw
device Resistor:
    terminals: [A, B]
    materials:
        A: Polysilicon
        B: Polysilicon
    spice:
        prefix: R
        terminal_order: [A, B]
        parameters: [R]              # ✅ Compiler calculates R = ρL/A
        parameter_style: positional
```

**SPICE Output:**
```spice
RR1 In Out 1000.00
```

### Diode (Model-Provided)

```hw
device Diode:
    terminals: [Anode, Cathode]
    materials:
        Anode: Silicon_P
        Cathode: Silicon_N
    spice:
        prefix: D
        terminal_order: [Anode, Cathode]
        parameters: []               # ❌ No geometry-derived parameters
        model: DMOD                  # ✅ References spice_model card
        parameter_style: positional
```

**SPICE Output:**
```spice
.model DMOD D (IS=1e-12 N=1.0 RS=0.1 BV=40.0)
DD1 Anode_In Cathode_Out DMOD
```

### MOSFET (Hybrid: Geometry W,L + Model VTO,LAMBDA)

```hw
device NMOS:
    terminals: [drain, gate, source, bulk]
    materials:
        gate: [Polysilicon, Aluminum]
        source: Silicon_N
        drain: Silicon_N
        bulk: Silicon_P
    spice:
        prefix: M
        terminal_order: [drain, gate, source, bulk]
        parameters: [W, L]           # ✅ Compiler extracts from gate geometry
        model: NMOS_180              # ✅ References process model
        parameter_style: named
```

**SPICE Output:**
```spice
.model NMOS_180 NMOS (VTO=0.7 KP=120u LAMBDA=0.01 GAMMA=0.4)
MM1 drain gate source bulk NMOS_180 W=1.5u L=0.18u
```

---

## 3. Material Properties

### Purpose
Declare **physical properties** needed for parameter extraction and physics validation.

### Structure

```hw
material Silicon_Dioxide:
    category: insulator
    properties:
        relative_permittivity: 3.9
        opacity: 0.6
```

### Required Properties by Device Type

#### Capacitors
- `relative_permittivity` (εᵣ) - for C = ε₀εᵣ(A/d)

#### Resistors
- `resistivity` (ρ) - for R = ρ(L/A)
- `thickness` - for cross-sectional area

### Property Access

The compiler looks up properties dynamically:

```rust
// NO hardcoded property names in extraction logic
let epsilon_r = material_props.get("relative_permittivity")
    .ok_or("Material missing 'relative_permittivity' property")?;
```

**Fail loudly if missing** - the user sees exactly what's needed.

---

## 4. Geometry Binding

### Purpose
Explicitly bind physical geometry (pours) to device terminals.

### Syntax

```hw
add pour(Polysilicon) named Bottom_Plate on layer: polyres:
    device: C1.bottom    # <-- Explicit binding
    net: Out
    dimensions: 10um by 10um
    at: [x: 7.5um, y: 7.5um]
```

### Format: `device: <instance>.<terminal>`

- `C1` - device instance name
- `bottom` - terminal name (must match device definition)

### Discovery Process

The compiler scans all pours, groups by device instance, validates all terminals are bound:

```
✓ C1.top    → Top_Plate (Aluminum on metal1)
✓ C1.bottom → Bottom_Plate (Polysilicon on polyres)
```

Missing terminal → **compilation error** with clear message.

---

## 5. Parameter Extraction

### Purpose
Calculate device parameters from physical geometry and material properties.

### Architecture: Registry-Based

```rust
pub struct ParameterExtractionRegistry {
    extractors: FxHashMap<String, ExtractionFn>,
}

// Register extraction functions dynamically
registry.register("Capacitor", extract_capacitor_parameters);
registry.register("Resistor", extract_resistor_parameters);
// Add more without touching compiler core!
```

### Extraction Process

#### Capacitor Example

```rust
fn extract_capacitor_parameters(
    terminal_pours: &FxHashMap<CompactString, PourMetadata>,
    space: &HardwareSpace,
) -> Result<FxHashMap<CompactString, f64>, String> {
    // 1. Get geometry from terminal pours
    let area = calculate_overlap_area(bbox1, bbox2)?;
    let thickness = (z2 - z1).abs();
    
    // 2. Get dielectric properties from stackup (NOT hardcoded!)
    let epsilon_r = find_dielectric_permittivity(z1, z2, space)?;
    
    // 3. Apply physics: C = ε₀εᵣ(A/d)
    let capacitance = EPSILON_0 * epsilon_r * (area / thickness);
    
    // 4. Return parameter map
    Ok(hashmap!["C" => capacitance])
}
```

#### Key Points

1. **No device type checks** - function called via registry lookup
2. **Material properties from stackup** - looks at layers between plates
3. **Pure physics calculation** - no heuristics or magic numbers
4. **Fail loudly** - missing properties → clear error

### Dielectric Property Lookup

**OLD (WRONG)**: Search pours for insulators between plates  
**NEW (CORRECT)**: Search **stackup layers** for insulators

```rust
// Find dielectric layer from profile stackup
for layer in &space.stackup_layers {
    if layer.overlaps_z_range(z_bottom, z_top) {
        if space.material_registry.is_insulator(layer.material_id) {
            return layer.get_property("relative_permittivity");
        }
    }
}
```

Why? Dielectric layers (d1, d2, etc.) are in the stackup, not user-placed pours.

---

## 6. SPICE Generation

### Purpose
Format extracted devices into SPICE netlists using device metadata.

### Process

```rust
// 1. Get device metadata from user definition (NOT hardcoded!)
let spice_info = device_def.spice_info()?;

// 2. Build SPICE card using metadata
netlist.push(spice_info.prefix);           // 'C' from device definition
netlist.push_str(&device.name);            // 'C1'

// 3. Add terminals in user-declared order
for terminal in &spice_info.terminal_order {
    let net = device.terminals.get(terminal)?;
    netlist.push_str(net);                 // 'In Out'
}

// 4. Format parameters using user-declared style
match spice_info.parameter_style {
    Positional => netlist.push_str(&format!(" {:.2e}", value)),  // 1.73e-13
    Named => netlist.push_str(&format!(" {}={:.2}u", name, value)),  // W=1.5u
}
```

### Output Format

#### Positional (Capacitor)
```spice
CC1 In Out 1.73e-13
```

#### Named (MOSFET)
```spice
MM1 drain gate source bulk NMOS W=1.5u L=0.18u
```

### Formatting Rules

- **Small values** (< 1e-3): Scientific notation (1.73e-13)
- **Large values** (> 1e6): Scientific notation (5.60e+06)
- **Normal values**: Fixed-point (1000.00)

**No device type checks** - formatting based on value magnitude.

---

## Adding New Device Types

### Step 1: Define the Device

```hw
device Inductor:
    terminals: [A, B]
    materials:
        A: [Copper, Aluminum]
        B: [Copper, Aluminum]
    spice:
        prefix: L
        terminal_order: [A, B]
        parameters: [L]
        parameter_style: positional  # MUST declare!
```

### Step 2: Register Parameter Extractor

```rust
// In parameter_extraction.rs
fn extract_inductor_parameters(
    terminal_pours: &FxHashMap<CompactString, PourMetadata>,
    space: &HardwareSpace,
) -> Result<FxHashMap<CompactString, f64>, String> {
    // Calculate L = μ₀μᵣN²A/l from geometry
    // ...
}

// Register it
registry.register("Inductor", extract_inductor_parameters);
```

### Step 3: Use It

```hw
space My_LC_Circuit:
    add pour(Copper) named Inductor_A:
        device: L1.A
        # ...
```

**No compiler changes needed!** Everything is user-declared.

---

## Error Handling Philosophy

### Fail Loudly with Clear Messages

**BAD (old way)**:
```
Error: Missing parameter
```

**GOOD (new way)**:
```
Error: Device 'C1' missing REQUIRED parameter 'C' (device type: Capacitor)

The parameter extraction system could not calculate capacitance.

Possible causes:
1. Terminal geometry missing bounding boxes
2. No dielectric material found between plates
3. Dielectric material missing 'relative_permittivity' property

Add 'relative_permittivity: <value>' to your dielectric material definition.
```

### No Silent Defaults

**NEVER**:
```rust
let style = parameter_style.unwrap_or(Positional);  // BAD!
```

**ALWAYS**:
```rust
let style = parameter_style.ok_or_else(||
    "SPICE block missing REQUIRED field 'parameter_style'. \
     Add 'parameter_style: positional' or 'parameter_style: named'"
)?;
```

---

## Testing Checklist

When adding a new device type:

- ✅ Device definition with ALL required fields
- ✅ Material definitions with ALL required properties
- ✅ Parameter extractor registered in registry
- ✅ Test with valid geometry → correct SPICE output
- ✅ Test with missing terminal → clear error
- ✅ Test with missing material property → clear error
- ✅ Test with missing `parameter_style` → clear error

---

## Migration Notes

### What Changed (v0.2.1)

#### Before (WRONG)
- Compiler guessed parameter style from device prefix
- `matches!(prefix, 'M' | 'Q' | ...)` - hardcoded matching
- Defaults everywhere (`unwrap_or_default()`)
- Searched pours for dielectric materials

#### After (CORRECT)
- User declares `parameter_style` in device definition
- Zero hardcoded device types
- No defaults - fail loudly if missing
- Searches stackup layers for dielectric materials

### Required Changes

All device definitions MUST add `parameter_style`:

```hw
# Before
device Capacitor:
    spice:
        prefix: C
        terminal_order: [top, bottom]
        parameters: [C]

# After
device Capacitor:
    spice:
        prefix: C
        terminal_order: [top, bottom]
        parameters: [C]
        parameter_style: positional  # NEW - REQUIRED!
```

---

## Design Principles

1. **User Control**: Everything declared in `.hw` files, nothing hardcoded
2. **Fail Loudly**: Missing metadata = clear compilation error
3. **No Magic**: Zero heuristics, guessing, or defaults
4. **Extensible**: Adding devices = writing `.hw` files, not Rust code
5. **Physics-Based**: Parameters calculated from real geometry + materials
6. **Metadata-Driven**: Compiler executes user declarations, doesn't interpret

---

## Summary

The netlist system is a **pure execution engine** for user-declared metadata:

- **Device definitions** declare structure and SPICE format
- **Material properties** declare physics
- **Geometry bindings** declare layout
- **Parameter extraction** calculates from geometry + materials
- **SPICE generation** formats using device metadata

**The compiler knows nothing about resistors, capacitors, or MOSFETs.** It only executes user declarations.

Adding new device types = writing `.hw` files. No compiler changes. Ever.
