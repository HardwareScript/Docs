# Device Definitions in HardwareScript

**Version:** 0.2.1  
**Status:** Implementation Complete, SPICE Extraction In Progress  
**Date:** 2026-08-03

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

### The Device Solution

Native `device` declarations provide:

1. ✅ **Automatic DRC Exemption**: Compiler knows terminals are meant to overlap
2. ✅ **SPICE Extraction**: Calculates physical parameters (R = R_sheet × L/W)
3. ✅ **Material Contract Enforcement**: Validates physical layer requirements

---

## Device Definition Syntax

### Step 1: Define the Device Contract

At the top level (outside any `space` or `module`), define the device type:

```hw
device Resistor:
    terminals: [A, B]
    materials:
        A: Polysilicon
        B: Polysilicon
```

**Fields:**
- `terminals`: List of electrical terminals (connection points)
- `materials`: Material requirements for each terminal
- `tolerance` (optional): Parameter tolerances for LVS verification

**Example with Multiple Materials:**

```hw
device NMOS:
    terminals: [gate, source, drain, bulk]
    materials:
        gate: [Polysilicon, Aluminum]  # Either material allowed
        source: Silicon_N
        drain: Silicon_N
        bulk: Silicon_P
```

---

## Device Instantiation: Binding Pours to Terminals

Devices are instantiated by binding geometry (pours) to device terminals using the `device:` field.

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

device Resistor:
    terminals: [A, B]
    materials:
        A: Polysilicon
        B: Polysilicon

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
