# Hardware Script v0.1.6: Logical Behavior Guide

## Overview

In Hardware Script, **logical behavior** defines the structural netlist of your circuit - the devices that exist and how they're connected - without specifying physical placement. This is the "truth" that your physical layout must implement.

## Core Concept: Module Definitions

A `module` defines the logical behavior using three key elements:

1. **Pins** - The interface to the outside world
2. **Devices** - The components that make up the circuit (transistors, gates, etc.)
3. **Routes** - The connections between device terminals and pins

## Syntax Structure

```hw
module ModuleName:
    pins: [
        input InputPin,
        output OutputPin,
        power PowerPin,
        ground GroundPin
    ]
    
    add DeviceType named InstanceName
    route InstanceName.terminal to NetName
    route InstanceName.terminal to OtherInstance.terminal
```

## Pin Declarations

### Pin Roles

Hardware Script supports explicit pin directions for electrical correctness:

- `input` - Signal inputs
- `output` - Signal outputs  
- `power` - Power supply pins (VDD, VCC, etc.)
- `ground` - Ground pins (GND, VSS, etc.)
- `inout` - Bidirectional pins

### Syntax Options

**Bracket Style (Recommended):**
```hw
pins: [
    input A,
    input B,
    output OUT,
    power VDD,
    ground GND
]
```

**Property Style:**
```hw
pins:
    input: A, B
    output: OUT
    power: VDD
    ground: GND
```

**Array Pins:**
```hw
pins: [
    input Data[8],
    output Result[16]
]
```

## Device Instantiation

Use `add` to instantiate devices in your circuit:

```hw
add DeviceType named InstanceName
```

### Common Device Types

- `NMOS` - N-channel MOSFET
- `PMOS` - P-channel MOSFET
- `Resistor` - Resistor
- `Capacitor` - Capacitor
- Custom modules you've defined

### Device Terminals

Standard transistor terminals:
- `gate` - Control terminal
- `source` - Source terminal
- `drain` - Drain terminal
- `bulk` - Bulk/substrate connection (critical for CMOS!)

## Routing Connections

The `route` statement connects device terminals to nets or other terminals:

```hw
route From to To
```

### Routing Patterns

**Device to Net:**
```hw
route M1.drain to VOUT
```

**Device to Pin:**
```hw
route M1.gate to VIN
```

**Device to Device:**
```hw
route M1.source to M2.drain
```

**Internal Nodes:**
```hw
route M3.source to INTERNAL_NODE
route M4.drain to INTERNAL_NODE
```

## Complete Examples

### Example 1: CMOS Inverter

```hw
module Inverter_Logic:
    pins: [
        input VIN,
        output VOUT,
        power VDD,
        ground GND
    ]
    
    # NMOS transistor (pulls output LOW)
    add NMOS named M1
    route M1.drain to VOUT
    route M1.gate to VIN
    route M1.source to GND
    route M1.bulk to GND
    
    # PMOS transistor (pulls output HIGH)
    add PMOS named M2
    route M2.drain to VOUT
    route M2.gate to VIN
    route M2.source to VDD
    route M2.bulk to VDD
```

**Key Points:**
- Two transistors in complementary configuration
- Both gates connected to input
- Both drains connected to output
- NMOS source to ground, PMOS source to power
- Bulk connections prevent latch-up

### Example 2: 2-Input NAND Gate

```hw
module NAND_Logic:
    pins: [
        input A,
        input B,
        output OUT,
        power VDD,
        ground GND
    ]
    
    # PMOS transistors (parallel - pull-up network)
    add PMOS named M1
    route M1.drain to OUT
    route M1.gate to A
    route M1.source to VDD
    route M1.bulk to VDD
    
    add PMOS named M2
    route M2.drain to OUT
    route M2.gate to B
    route M2.source to VDD
    route M2.bulk to VDD
    
    # NMOS transistors (series - pull-down network)
    add NMOS named M3
    route M3.drain to OUT
    route M3.gate to A
    route M3.source to INTERNAL_NODE
    route M3.bulk to GND
    
    add NMOS named M4
    route M4.drain to INTERNAL_NODE
    route M4.gate to B
    route M4.source to GND
    route M4.bulk to GND
```

**Key Points:**
- PMOS in parallel: Either transistor can pull output HIGH
- NMOS in series: Both must be ON to pull output LOW
- Internal node connects the series NMOS chain
- Implements NAND truth table: OUT = !(A & B)

### Example 3: NOR Gate

```hw
module NOR_Logic:
    pins: [
        input A,
        input B,
        output OUT,
        power VDD,
        ground GND
    ]
    
    # PMOS transistors (series - pull-up network)
    add PMOS named M1
    route M1.drain to OUT
    route M1.gate to A
    route M1.source to INTERNAL_VDD
    route M1.bulk to VDD
    
    add PMOS named M2
    route M2.drain to INTERNAL_VDD
    route M2.gate to B
    route M2.source to VDD
    route M2.bulk to VDD
    
    # NMOS transistors (parallel - pull-down network)
    add NMOS named M3
    route M3.drain to OUT
    route M3.gate to A
    route M3.source to GND
    route M3.bulk to GND
    
    add NMOS named M4
    route M4.drain to OUT
    route M4.gate to B
    route M4.source to GND
    route M4.bulk to GND
```

**Key Points:**
- PMOS in series: Both must be ON to pull output HIGH
- NMOS in parallel: Either transistor can pull output LOW
- Implements NOR truth table: OUT = !(A | B)

## Advanced Features

### Parametric Modules with For Loops

```hw
module 8Bit_Bus_Buffer:
    pins: [
        input Data_In[8],
        output Data_Out[8],
        power VDD,
        ground GND
    ]
    
    for i in 0..7:
        add Inverter named Buf[i]
        route Data_In[i] to Buf[i].VIN
        route Buf[i].VOUT to Data_Out[i]
        route Buf[i].VDD to VDD
        route Buf[i].GND to GND
```

### Conditional Logic

```hw
module Configurable_Gate:
    pins: [
        input A,
        input B,
        output OUT,
        power VDD,
        ground GND
    ]
    
    if MODE == "NAND":
        add NAND_Gate named Gate
    else:
        add NOR_Gate named Gate
    
    route A to Gate.A
    route B to Gate.B
    route Gate.OUT to OUT
```

## Connecting to Physical Layout

After defining logical behavior, implement it in a `space` block:

```hw
space MyCircuit implements MyLogic:
    dimensions: 2mm by 2mm by 1mm
    grid: 2000 by 2000 by 10
    
    # Physical geometry using pours
    add pour(Silicon_N) named NMOS_Source on z:6:
        device: M1.source
        net: GND
        boundary: [x: 200um, y: 500um] to [x: 400um, y: 1300um]
```

**Key Binding Attributes:**
- `device: M1.source` - Binds this geometry to device M1's source terminal
- `net: GND` - Assigns this geometry to the GND net

## Best Practices

### 1. Always Connect Bulk Terminals

```hw
# ✅ CORRECT - Bulk connected
add NMOS named M1
route M1.bulk to GND

# ❌ WRONG - Bulk floating (causes latch-up!)
add NMOS named M1
# Missing bulk connection
```

### 2. Use Descriptive Instance Names

```hw
# ✅ GOOD
add NMOS named PullDown_Left
add PMOS named PullUp_Right

# ❌ BAD
add NMOS named M1
add PMOS named M2
```

### 3. Document Your Logic

```hw
module NAND_Logic:
    # Truth Table:
    #   A | B | OUT
    #   0 | 0 |  1   (both PMOS on)
    #   0 | 1 |  1   (one PMOS on)
    #   1 | 0 |  1   (one PMOS on)
    #   1 | 1 |  0   (both NMOS on in series)
    
    pins: [...]
```

### 4. Group Related Connections

```hw
# NMOS transistor
add NMOS named M1
route M1.drain to VOUT
route M1.gate to VIN
route M1.source to GND
route M1.bulk to GND

# PMOS transistor
add PMOS named M2
route M2.drain to VOUT
route M2.gate to VIN
route M2.source to VDD
route M2.bulk to VDD
```

## Common Patterns

### Complementary Pair (Inverter)
- 1 NMOS + 1 PMOS
- Gates tied together (input)
- Drains tied together (output)
- NMOS source to GND, PMOS source to VDD

### NAND Gate
- PMOS in parallel (pull-up)
- NMOS in series (pull-down)
- Requires internal node for series connection

### NOR Gate
- PMOS in series (pull-up)
- NMOS in parallel (pull-down)
- Requires internal node for series connection

### Transmission Gate
- 1 NMOS + 1 PMOS in parallel
- Complementary gate signals
- Bidirectional pass element

## Validation

The compiler performs alignment validation to ensure your physical layout matches the logical behavior:

```
✅ Physical netlist extracted: 4 devices
✅ Logical netlist synthesized: 4 devices
✅ Alignment validation passed: Layout matches schematic
```

This guarantees that what you designed logically is what you built physically!

## Summary

**Logical behavior in Hardware Script:**
1. Define the circuit structure (devices and connections)
2. No physical coordinates - purely logical
3. Serves as the "truth" for validation
4. Physical layout must implement this exactly

**Key syntax:**
- `module Name:` - Define logical behavior
- `pins: [...]` - Declare interface
- `add DeviceType named Instance` - Instantiate devices
- `route From to To` - Connect terminals

This separation of logical intent from physical implementation is what makes Hardware Script powerful - you can verify that your layout actually implements the circuit you designed!
