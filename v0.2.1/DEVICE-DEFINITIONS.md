# Device Definitions in HardwareScript

**Version:** 0.2.1  
**Status:** Implementation Complete, Multi-Pour Terminal Support  
**Date:** 2026-08-10

---

## Overview

This document describes how to create low-level semiconductor devices in HardwareScript using the native `device` keyword. Devices are multi-terminal physical structures where different electrical nets meet through a material channel (resistors, capacitors, diodes, transistors).

**Key Distinction:**
- **`pour`**: A single-net conductor or plane (wire, copper sheet)
- **`device`**: A multi-terminal component where different nets connect through physics

---

## Why Use Native Devices?

### The Problem with Raw Pours

Attempting to build devices from raw `add pour()` statements with manual net assignments causes:

1. **DRC Violations**: The compiler sees overlapping different nets as short circuits
2. **No SPICE Extraction**: Cannot calculate R, C, or transistor parameters automatically
3. **No Physical Validation**: Cannot enforce foundry material requirements
4. **Parasitic Extraction Errors**: Device bodies might be extracted as routing traces

### The Device Solution

Native `device` declarations provide:

1. ✅ **Automatic DRC Exemption**: Compiler knows terminals are meant to overlap
2. ✅ **SPICE Extraction**: Calculates physical parameters (R = R_sheet × L/W)
3. ✅ **Material Contract Enforcement**: Validates physical layer requirements
4. ✅ **Parasitic Exemption**: Device bodies automatically excluded from interconnect extraction

### Architectural Advantage: Semantic Parasitic Exemption

HardwareScript's device system provides a critical advantage over traditional EDA tools:

**Commercial Tools (Calibre, Assura, StarRC):**
- Operate on flat GDSII geometry
- Cannot distinguish device bodies from routing traces
- **Require manual blocker layers** to prevent double-counting device physics

**HardwareScript:**
- Device bodies defined as `add pour` with `device:` binding
- Parasitic extractor only processes `route` statements (`analytic_routes`)
- **No blocker layers needed** - architectural separation prevents double-counting

**Example:** A 4µm × 1µm polysilicon resistor body is automatically excluded from parasitic extraction. Only the aluminum routing traces are extracted, producing correct capacitance values (~0.19 fF for traces vs. ~15-20 fF if device body was incorrectly included).

See `NETLIST-ARCHITECTURE.md` Section 0 for detailed validation and comparison with commercial tools.

---

## Device Definition Syntax

### Step 1: Define the Device Contract

At the top level (outside any `space` or `module`), define the device type:

```hw
export device Resistor:
    terminals: [A, B, BULK]
    materials:
        A: [Polysilicon, Titanium_Silicide, Aluminum]
        B: [Polysilicon, Titanium_Silicide, Aluminum]
        BULK: [Air, P_Plus_Diffusion, Titanium_Silicide]
    metrics:
        L: clear_span(from: A, to: B)
        W: transverse_width(from: A, to: B)
    spice:
        prefix: X
        subcircuit: sky130_fd_pr__res_high_po
        terminal_order: [A, B, BULK]
        parameters: [W, L]
        parameter_style: named
```

**Required Fields:**
- `terminals`: Ordered list of electrical terminals (connection points)
- `materials`: Material requirements/alternatives for each terminal
- `metrics`: Declarative mathematical extraction operators (evaluated via 2D topological vector calculus)
- `spice`: SPICE netlist export metadata (`prefix`, `terminal_order`, `parameters`, `parameter_style`)
- `tolerance` (optional): Parameter tolerances for LVS verification

---

### Step 1.5: Universal Geometric & Physical Operators (`metrics:`)

HardwareScript enforces **Zero-Magic Device Extraction**. Instead of guessing terminal roles in Rust compiler code, the `.hw` device contract declares the exact geometric measurement rules:

```hw
export device NMOS:
    terminals: [D, G, S, B]
    materials:
        D: [N_Plus_Diffusion, Titanium_Silicide, Aluminum]
        G: [Polysilicon, Titanium_Silicide, Aluminum]
        S: [N_Plus_Diffusion, Titanium_Silicide, Aluminum]
        B: [P_Plus_Diffusion, Titanium_Silicide, Aluminum]
    
    # ── Universal Geometry Operators (Pure Declarative Math) ──
    metrics:
        L:  channel_length(from: S, to: D, control: G)
        W:  channel_width(from: S, to: D, control: G)
        AD: area(D, exclude: G)
        AS: area(S, exclude: G)
        PD: perimeter(D, exclude: G)
        PS: perimeter(S, exclude: G)

    spice:
        prefix: X
        subcircuit: sky130_fd_pr__nfet_01v8
        terminal_order: [D, G, S, B]
        parameters: [W, L, AD, AS, PD, PS]
        parameter_style: named
```

#### Supported Metric Operators

| Operator | Syntax | Description | Unit |
| :--- | :--- | :--- | :--- |
| **Channel Length** | `channel_length(from: S, to: D, control: G)` | Projects control electrode span along unit conduction flux vector $\hat{u}$ | $\mu m$ |
| **Channel Width** | `channel_width(from: S, to: D, control: G)` | Evaluates active diffusion / channel width along transverse normal $\hat{u}^\perp$ | $\mu m$ |
| **Clear Span** | `clear_span(from: A, to: B)` | Inner contact-to-contact clear span along flux vector | $\mu m$ |
| **Transverse Width** | `transverse_width(from: A, to: B)` | Conduction body width perpendicular to flux vector | $\mu m$ |
| **Overlap Area** | `overlap_area(terminal_a: C0, terminal_b: C1)` | 2D overlap surface area between two conductive plates | $\mu m^2$ |
| **Overlap Perimeter**| `overlap_perimeter(terminal_a: C0, terminal_b: C1)`| 2D overlap boundary perimeter | $\mu m$ |
| **Diffusion Area** | `area(terminal: D, exclude: G)` | Polygon surface area with optional gate electrode subtraction | $m^2$ |
| **Diffusion Perimeter**| `perimeter(terminal: D, exclude: G)` | Polygon boundary perimeter with optional gate electrode subtraction | $m$ |
| **Maxwell Resistance**| `resistance(from: A, to: B)` | Evaluates $R = \rho \cdot \frac{L}{W \cdot t}$ from geometry & stackup resistivity | $\Omega$ |
| **Plate Capacitance** | `capacitance(plate_a: C0, plate_b: C1)` | Evaluates $C = \varepsilon_0 \varepsilon_r \frac{A}{d}$ from overlap & dielectric stack | $F$ |

---

## Device Instantiation: Binding Pours to Terminals

Devices are instantiated by binding geometry (pours) to device terminals using the `device:` field.

**Key Feature: Multiple Pours Per Terminal**

A single terminal can have **multiple pours bound to it**. This is essential for real-world devices where one terminal includes both:
1. **Primary channel material** (e.g., Polysilicon resistor body)
2. **Contact materials** (e.g., Titanium Silicide or Aluminum pads)

### Multi-Terminal Binding Syntax

Bind multiple terminals to a single pour using comma-separated syntax:

```hw
add pour(Polysilicon) named Resistor_Body on layer: polyres:
    device: R1.A, R1.B  # ← Both terminals bound to this pour
    net: In
    dimensions: 4.0um by 1.0um
```

### Step 2: Create Geometry for Each Terminal

```hw
space Simple_Resistor_Space implements SimpleResistor:
    dimensions: 5.0um by 3.0um by 100nm
    resolution: 10nm
    profile: Resistor_Flattened

    # Polysilicon resistor body
    add pour(Polysilicon) named Resistor_Body on layer: polyres:
        device: R1.A  # Bind to terminal A of device R1
        dimensions: 1.0um by 1.0um
        at: [x: 2.0um, y: 1.5um]

    # Metal contact for terminal A
    add pour(Aluminum) named Contact_A on layer: metal1:
        device: R1.A  # Also bound to terminal A
        net: In       # Must specify net for routing
        dimensions: 400nm by 400nm
        at: [x: 1.7um, y: 1.5um]  # Overlaps resistor edge

    # Metal contact for terminal B
    add pour(Aluminum) named Contact_B on layer: metal1:
        device: R1.B  # Bound to terminal B
        net: Out      # Different net
        dimensions: 400nm by 400nm
        at: [x: 2.3um, y: 1.5um]  # Overlaps opposite edge
```

**Critical Rules:**

1. **Device Binding Format**: `device: <instance_name>.<terminal_name>`
2. **Multiple Pours Per Terminal**: Multiple pours can bind to the same terminal
3. **Net Assignment Required for Routing**: Pours with `device:` bindings need explicit `net:` for routing
4. **Overlapping is Intentional**: Contacts MUST physically overlap the device body

---

## Complete Resistor Example

### Full Working Example

```hw
import * from @std/primitives/units
import * from "./resistor_pdk"

# ========================================================================
# DEVICE DEFINITION (Top Level)
# ========================================================================

export device Resistor:
    terminals: [A, B]
    materials:
        A: Polysilicon
        B: Polysilicon
    metrics:
        R: resistance(from: A, to: B)
    spice:
        prefix: R
        terminal_order: [A, B]
        parameters: [R]
        parameter_style: positional

# ========================================================================
# MODULE (Logical Interface)
# ========================================================================

module SimpleResistor:
    pins: [input In, output Out]
    route In to Out

# ========================================================================
# SPACE (Physical Implementation)
# ========================================================================

space Simple_Resistor_Space implements SimpleResistor:
    dimensions: 5.0um by 3.0um by 100nm
    resolution: 10nm
    profile: Resistor_Flattened

    nets:
        In:  { classification: signal, potential: 1.8V, current: 1.0uA }
        Out: { classification: signal, potential: 0.0V, current: 1.0uA }

    # Resistor body (1μm × 1μm = ~400 Ohms @ 400 Ω/square)
    add pour(Polysilicon) named Resistor_Body on layer: polyres:
        device: R1.A
        dimensions: 1.0um by 1.0um
        at: [x: 2.0um, y: 1.5um]

    # Terminal A: Metal contact overlapping left edge
    add pour(Aluminum) named Contact_A on layer: metal1:
        device: R1.A
        net: In
        dimensions: 400nm by 400nm
        at: [x: 1.7um, y: 1.5um]

    # Terminal B: Metal contact overlapping right edge
    add pour(Aluminum) named Contact_B on layer: metal1:
        device: R1.B
        net: Out
        dimensions: 400nm by 400nm
        at: [x: 2.3um, y: 1.5um]

    # Vias connecting metal to polysilicon
    add contact(Titanium_Silicide) named Via_A spanning layer: polyres to metal1:
        device: R1.A
        net: In
        diameter: 200nm
        align: center with Contact_A

    add contact(Titanium_Silicide) named Via_B spanning layer: polyres to metal1:
        device: R1.B
        net: Out
        diameter: 200nm
        align: center with Contact_B

    # External connection pads
    add pour(Aluminum) named In_Pad on layer: metal1:
        net: In
        dimensions: 600nm by 600nm
        at: [x: 500nm, y: 1.5um]

    add pour(Aluminum) named Out_Pad on layer: metal1:
        net: Out
        dimensions: 600nm by 600nm
        at: [x: 3.5um, y: 1.5um]

    # Routing
    route In_Pad to Contact_A:
        net: In
        width: 300nm
        layer: metal1

    route Contact_B to Out_Pad:
        net: Out
        width: 300nm
        layer: metal1
```

---

## Physical Design Patterns

### Pattern 1: Overlapping Contacts

The contacts MUST physically overlap the device body:

```
Top View (must overlap):

     Contact_A                    Contact_B
  ┌─────────────┐            ┌─────────────┐
  │ Aluminum    │            │ Aluminum    │
  │  ┌──────────┼────────────┼──────────┐  │
  │  │ TiSi via │            │ TiSi via │  │
  └──┼──────────┘            └──────────┼──┘
     │                                   │
     └───────────────────────────────────┘
        Polysilicon Resistor Body
```

**Why Overlap?**
- Forms electrical contact between layers
- Via connects metal to polysilicon through the overlap region
- Without overlap: no electrical connection (floating)

### Pattern 2: Layer Stack

Typical resistor stack (bottom to top):

```
Layer 3: metal1 (Aluminum contacts) ← Z = 20-30nm
         ↕ via (Titanium Silicide)
Layer 2: dielectric (Silicon Dioxide) ← Z = 10-20nm
Layer 1: polyres (Polysilicon body)   ← Z = 0-10nm
```

---

## Device Types: The "Big Four"

### 1. Resistor (Resistive Channel)

```hw
device Resistor:
    terminals: [A, B]
    materials:
        A: Polysilicon
        B: Polysilicon
```

**Physics**: R = R_sheet × (L / W)  
**Use Case**: Biasing, pull-ups, current limiting

### 2. Capacitor (Dielectric Sandwich)

```hw
device Capacitor:
    terminals: [Top, Bottom]
    materials:
        Top: Aluminum
        Bottom: Aluminum
```

**Physics**: C = ε₀ε_r × (Area / Thickness)  
**Use Case**: Decoupling, filtering, timing

### 3. Diode (P-N Junction)

```hw
device Diode:
    terminals: [Anode, Cathode]
    materials:
        Anode: Silicon_P
        Cathode: Silicon_N
```

**Physics**: I = I_s(e^(V/nV_t) - 1)  
**Use Case**: Rectification, protection, logic

### 4. MOSFET (Gate-Controlled Channel)

```hw
device NMOS:
    terminals: [gate, source, drain, bulk]
    materials:
        gate: [Polysilicon, Aluminum]
        source: Silicon_N
        drain: Silicon_N
        bulk: Silicon_P
```

**Physics**: I_d = ½μC_ox(W/L)(V_gs - V_th)²  
**Use Case**: Switching, amplification, logic gates

---

## Topological Channel Continuity Verification

The compiler performs a physical topology check on every bound device after geometry binding. This check answers: **"Does the physical geometry actually form the continuous conductive path the user intended?"**

This catches silent failures that are invisible at the source-code level — for example, a serpentine loop with a missing vertical connector segment that looks syntactically correct but is physically broken.

### How Opt-In Works (Zero New Syntax)

Opt-in is determined purely by **how many terminals a single pour declares**. No new keywords or annotations are needed:

| Binding Style | Example | Compiler Behaviour |
|---|---|---|
| **Multi-Terminal Pour** | `device: R1.A, R1.B` | ✅ **Opted in.** The pour declares a shared conduction body connecting `A` to `B`. Compiler verifies that `A` can reach `B` through the physical geometry. |
| **Single-Terminal Pour** | `device: C1.c0` | ⬜ **Opted out.** The pour is an isolated physical plate. The compiler does **not** force it to connect to other plates. |

### Why It Works This Way

When a user writes `device: R1.A, R1.B` on a pour (or a loop of pours), they are making an **explicit physical declaration**: *"This geometry is a shared resistive/conductive body whose two ends are terminal A and terminal B."* The compiler honours that declaration by verifying it.

When a user writes `device: C1.c0` and `device: C1.c1` on separate pours, they are declaring **independent physical plates**. Capacitor plates are separated by a dielectric and intentionally do not share a DC conduction path. The compiler respects that.

### Validation Algorithm

Internally, `DeviceTopologyValidator` (in `crates/hwc-export/src/device_extractor/continuity.rs`) builds a 3D bounding-interval spatial graph:

1. **Collect** all pours bound to the device (channel body pours + contact pours + via plugs).
2. **Connect** pairs of elements whose 3D axis-aligned bounding boxes intersect.
3. **Run Union-Find** (Disjoint Set Forest) to compute connected islands.
4. **Check** whether every conduction terminal's contact pour lands in the same island.

If terminals end up in different islands, the device fails with a detailed diagnostic:

```
Invalid geometry for Resistor 'R1': Device channel fragmentation:
  Disconnected Terminal Pairs: 'A' ↮ 'B'
  The channel geometry is fragmented into 2 disjoint islands (open circuit).
    • Island 1: Seg0, Seg1, Seg2, Seg3, Vert0, Vert1, Vert2, Contact_A_LI
    • Island 2: Seg4, Contact_B_LI
```

### Practical Examples

#### ✅ Connected Serpentine Resistor (passes)

```hw
for i in 0..5:                         # 5 horizontal segments
    add pour(Polysilicon) named Seg{i} on layer: polyres:
        device: R1.A, R1.B             # Multi-terminal → opted in
        ...

for i in 0..4:                         # 4 vertical connectors (correct: N-1)
    add pour(Polysilicon) named Vert{i} on layer: polyres:
        device: R1.A, R1.B
        ...
```

**Result**: All 5 segments are joined by 4 connectors → `A` reaches `B` → ✅ passes.

#### ❌ Broken Serpentine Resistor (caught)

```hw
for i in 0..5:                         # 5 horizontal segments
    add pour(Polysilicon) named Seg{i} on layer: polyres:
        device: R1.A, R1.B
        ...

for i in 0..3:                         # ❌ Only 3 connectors — one turn is missing!
    add pour(Polysilicon) named Vert{i} on layer: polyres:
        device: R1.A, R1.B
        ...
```

**Result**: `Seg4` is isolated → compiler catches the open circuit with the exact island breakdown.

#### ✅ MIM Capacitor (correctly skipped)

```hw
add pour(CAPM) named Top_Plate on layer: capm:
    device: C1.c0        # Single-terminal → opted out
    net: VDD
    ...

add pour(Aluminum) named Bottom_Plate on layer: metal1:
    device: C1.c1        # Single-terminal → opted out
    net: GND
    ...
```

**Result**: No shared channel body declared → continuity check is skipped → ✅ passes.

---

## DRC Behavior with Devices

### Without Device Binding (Raw Pours)

```hw
# ❌ DRC ERROR: P46 Clearance Violation
add pour(Polysilicon) named Body on layer: polyres:
    net: In  # Net A
    
add pour(Aluminum) named Contact on layer: metal1:
    net: Out  # Net B - different net!
    # Compiler sees: Net A and Net B overlapping = SHORT CIRCUIT
```

**Error**: `Clearance violation: Pour on net 2 is 10nm from net 1 (required: 200nm)`

### With Device Binding

```hw
# ✅ DRC PASSES: Device terminals allowed to overlap
add pour(Polysilicon) named Body on layer: polyres:
    device: R1.A
    
add pour(Aluminum) named Contact on layer: metal1:
    device: R1.B
    net: Out
    # Compiler knows: R1 terminals A and B are meant to connect
```

**Result**: No clearance violation - DRC engine exempts device terminal boundaries

---

## Common Mistakes

### Mistake 1: Forgetting Net Assignment

```hw
# ❌ WRONG: Router cannot connect
add pour(Aluminum) named Contact_A on layer: metal1:
    device: R1.A
    # Missing: net: In
```

**Error**: `Entity 'Contact_A' has no associated net`

**Fix**: Add `net:` field for routing

```hw
# ✅ CORRECT
add pour(Aluminum) named Contact_A on layer: metal1:
    device: R1.A
    net: In  # Required for routing
```

### Mistake 2: Non-Overlapping Contacts

```hw
# ❌ WRONG: Contacts don't touch resistor body
Resistor_Body: 1500nm to 2500nm
Contact_A: 500nm to 900nm   # Too far left!
Contact_B: 3000nm to 3400nm # Too far right!
```

**Result**: No electrical connection - floating terminals

**Fix**: Overlap edges by 200-300nm

```hw
# ✅ CORRECT: Overlapping
Resistor_Body: 1500nm to 2500nm
Contact_A: 1500nm to 1900nm  # Overlaps left edge
Contact_B: 2100nm to 2500nm  # Overlaps right edge
```

### Mistake 3: Using `add device()` Syntax

```hw
# ❌ WRONG: Syntax not implemented
add device(Resistor) named R1:
    terminals: [A: In, B: Out]
```

**Error**: `Unexpected token: 'device'`

**Fix**: Use `device:` binding on pours

```hw
# ✅ CORRECT: Bind pours to device
add pour(Polysilicon) named Body on layer: polyres:
    device: R1.A
```

---

## Implementation Status

### ✅ Implemented (v0.1.6)

- `device` keyword parsing
- Device definition AST
- Device contract validation
- `device:` binding on pours
- DRC exemption for device terminals
- Symbol table registration

### 🚧 In Progress

- SPICE parameter extraction (R, C, MOSFET)
- Device extraction to netlist
- LVS verification with tolerance

### 📋 Planned

- Component-level device templates
- Parametric device PCells
- Advanced tolerance specifications
- Multi-finger transistor layouts

---

## Reference: SiliWiz Resistor Tutorial

This implementation follows the [TinyTapeout SiliWiz resistor tutorial](https://tinytapeout.com/siliwiz/resistors/):

**Key Points from Tutorial:**
1. 1μm × 1μm polysilicon square = ~400 Ohms (@ 400 Ω/square sheet resistance)
2. Metal1 contacts must overlap polysilicon edges
3. Metal1 vias bridge between polyres and metal1 layers
4. Contacts require "on top of" placement for electrical connection

---

## Next Steps

To create other low-level devices:

1. **Define Device Contract**: Specify terminals and material requirements
2. **Create PDK Profile**: Define layer stackup and via rules
3. **Build Test Space**: Instantiate device with bound pours
4. **Verify DRC**: Ensure no clearance violations
5. **Check Connectivity**: Verify nets route correctly
6. **Validate SPICE** (when implemented): Check extracted parameters

---

## See Also

- `tests/Resistor-Basics/simple_resistor_test.hw` - Working resistor example
- `tests/Resistor-Basics/resistor_pdk.hw` - PDK profile definition
- `tests/Resistor-Basics/materials.hw` - Material properties
- `crates/hwc-parser/src/parser/definitions/device.rs` - Parser implementation
- `crates/hwc-export/src/device_extractor/` - SPICE extraction (in progress)

---

**Document History:**
- 2026-08-03: Initial documentation based on resistor implementation
- 2026-08-11: Added parasitic exemption architectural advantage section, validated against commercial LPE standards
- 2026-08-17: Added Topological Channel Continuity Verification section documenting multi-terminal opt-in model, algorithm, and worked examples
