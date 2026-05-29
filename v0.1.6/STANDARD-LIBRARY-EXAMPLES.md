# Standard Library Examples (v0.1.6)

**Base Documentation**: [v0.1.5 LANGUAGE-SPEC.md](../v0.1.5/LANGUAGE-SPEC.md)  
**Status**: Standard library rewritten with unified syntax  
**Version**: 0.1.6

---

## Overview

This document shows how the Hardware Script Standard Library has been transformed using the v0.1.6 Unified Grammar. By applying the three universal rules, the standard library becomes:

- **Simpler to parse**: The compiler uses the same logic for all definitions
- **Easier to read**: Consistent punctuation and structure
- **More extensible**: Users can add custom units, components, and modules using the same patterns

---

## Unit Definitions

The unit system demonstrates the power of the Type-as-Keyword paradigm. Units are no longer buried in `define` statements—they're first-class language constructs.

### Before (v0.1.5 - Clunky)

```hw
define unit "Microfarad":
    symbol: "µF"
    aliases: ["uF"]
    base_si: "F"
    multiplier: 1e-6
    dimension: "capacitance"

define unit "Megahertz":
    symbol: "MHz"
    base_si: "Hz"
    multiplier: 1e6
    dimension: "frequency"
```

### After (v0.1.6 - Professional & Clean)

```hw
##[
    Standard Library - v0.1.6 Unified Unit Definitions
    This file defines the mathematical relationships of non-core units.
]##

# Frequency Group
unit Hertz:
    symbol: "Hz"
    base_si: "Hz"
    dimension: frequency

unit Kilohertz:
    symbol: "kHz"
    base_si: "Hz"
    multiplier: 1e3
    dimension: frequency

unit Megahertz:
    symbol: "MHz"
    base_si: "Hz"
    multiplier: 1e6
    dimension: frequency

unit Gigahertz:
    symbol: "GHz"
    base_si: "Hz"
    multiplier: 1e9
    dimension: frequency

# Capacitance Group
unit Farad:
    symbol: "F"
    base_si: "F"
    dimension: capacitance

unit Millifarad:
    symbol: "mF"
    base_si: "F"
    multiplier: 1e-3
    dimension: capacitance

unit Microfarad:
    symbol: "µF"
    aliases: ["uF"]
    base_si: "F"
    multiplier: 1e-6
    dimension: capacitance

unit Nanofarad:
    symbol: "nF"
    base_si: "F"
    multiplier: 1e-9
    dimension: capacitance

unit Picofarad:
    symbol: "pF"
    base_si: "F"
    multiplier: 1e-12
    dimension: capacitance

unit Femtofarad:
    symbol: "fF"
    base_si: "F"
    multiplier: 1e-15
    dimension: capacitance

# Resistance Group
unit Ohm:
    symbol: "Ω"
    aliases: ["ohm"]
    base_si: "Ω"
    dimension: resistance

unit Kiloohm:
    symbol: "kΩ"
    aliases: ["kohm"]
    base_si: "Ω"
    multiplier: 1e3
    dimension: resistance

unit Megaohm:
    symbol: "MΩ"
    aliases: ["Mohm"]
    base_si: "Ω"
    multiplier: 1e6
    dimension: resistance

# Inductance Group
unit Henry:
    symbol: "H"
    base_si: "H"
    dimension: inductance

unit Millihenry:
    symbol: "mH"
    base_si: "H"
    multiplier: 1e-3
    dimension: inductance

unit Microhenry:
    symbol: "µH"
    aliases: ["uH"]
    base_si: "H"
    multiplier: 1e-6
    dimension: inductance

unit Nanohenry:
    symbol: "nH"
    base_si: "H"
    multiplier: 1e-9
    dimension: inductance

# Tolerance Group
unit Percent:
    symbol: "%"
    multiplier: 0.01
    dimension: ratio
```

**Key Changes**:
1. `unit` is now a keyword (no `define`)
2. Unit names are bare identifiers (no quotes)
3. `dimension` uses bare identifiers (not strings)
4. Colons (`:`) for all static properties
5. Universal list syntax `[]` for aliases

**Why This Makes the Compiler Simpler**:
- The parser only needs to recognize one keyword: `unit`
- The same property parser handles all fields
- No special cases for unit definitions
- The compiler stays "dumb and fast"—it doesn't need to know what a "Microfarad" is, it just parses the multiplier and stores it

---

## The "Verilog Killer" Stress Test

This section demonstrates the most complex logic constructs in Hardware Script v0.1.6. These examples prove that the unified syntax can handle professional-grade hardware design.

### Basic D Flip-Flop

```hw
##[
    Standard Library - Register & FSM Complexity v0.1.6
    This file tests nested matching, register arrays, and handshake protocols.
]##

# STATIC WORLD: Definitions use Colons (:) and Brackets ([])
enum RegMode:
    values: [Hold, Load, Increment, Decrement]

struct FifoStatus:
    full[1]
    empty[1]
    count[3]
    overflow[1]

# BEHAVIORAL WORLD: Actions use Equals (=)

# ============================================================================
# DFlipFlop - Basic D flip-flop
# ============================================================================
module DFlipFlop:
    pins: [input D, input Clock, input Reset, output Q]
    
    logic:
        # Use '=' for dynamic register binding
        let state = reg(clock: Clock, reset: Reset, init: 0)
        
        # State updates are dynamic actions (single = for comparison)
        state.next = if Reset = 1: 0 else: D
        
        Q = state
```

### 8-bit Register with Mode Control

```hw
# ============================================================================
# Register8 - 8-bit register with wrap-around logic
# ============================================================================
module Register8:
    pins: [
        input Data[8],
        input Clock,
        input Reset,
        input Enable,
        input Mode[2],
        output Out[8]
    ]
    
    logic:
        let reg_val = reg(clock: Clock, reset: Reset, init: 0)
        
        # Nested logic flows like a clean software script (single = for comparison)
        reg_val.next = if Reset = 1:
            0
        else if Enable = 1:
            match Mode:
                0x0: reg_val                                # Hold
                0x1: Data                                   # Load
                0x2: if reg_val = 0xFF: 0 else: reg_val + 1  # Increment with wrap
                else: if reg_val = 0: 0xFF else: reg_val - 1 # Decrement with wrap
        else:
            reg_val
        
        Out = reg_val
```

**Key Features**:
- Nested `if`/`else` expressions
- `match` with multiple cases
- Inline conditionals for wrap-around logic
- All using the `=` operator for behavioral logic

### Shift Register with Parallel Load

```hw
# ============================================================================
# ShiftRegister8 - Using Universal List syntax for concatenation
# ============================================================================
module ShiftRegister8:
    pins: [
        input SerIn,
        input ParIn[8],
        input Clk,
        input Rst,
        input Load,
        input En,
        output ParOut[8]
    ]
    
    logic:
        let sreg = reg(clock: Clk, reset: Rst, init: 0)
        
        # Single = for comparison
        sreg.next = if Rst = 1:
            0
        else if Load = 1:
            ParIn
        else if En = 1:
            [sreg[6..0], SerIn]  # Universal [] used for concatenation
        else:
            sreg
        
        ParOut = sreg
```

**Key Features**:
- Bit-slice syntax `sreg[6..0]`
- List concatenation `[high_bits, low_bit]`
- Multiple control signals
- Clean priority encoding with nested `if`

### Register File with Forwarding

```hw
# ============================================================================
# RegisterFile4x8 - Complex Forwarding & Address Decoding
# ============================================================================
module RegisterFile4x8:
    pins: [
        input WriteData[8],
        input WriteAddr[2],
        input ReadAddr1[2],
        input ReadAddr2[2],
        input Clock,
        input Reset,
        input WriteEnable,
        output ReadData1[8],
        output ReadData2[8]
    ]
    
    logic:
        # Array-style register definitions
        let rf = [
            reg(clock: Clock, reset: Reset, init: 0),
            reg(clock: Clock, reset: Reset, init: 0),
            reg(clock: Clock, reset: Reset, init: 0),
            reg(clock: Clock, reset: Reset, init: 0)
        ]
        
        # Address decoding using comptime-style unrolling (single = for comparison)
        for i in 0..3:
            rf[i].next = if Reset = 1:
                0
            else if (WriteEnable = 1) & (WriteAddr = i):
                WriteData
            else:
                rf[i]
        
        # Read port 1 with Bypass/Forwarding logic
        ReadData1 = if (WriteEnable = 1) & (WriteAddr = ReadAddr1):
            WriteData  # Forward from write data
        else:
            match ReadAddr1:
                0: rf[0]
                1: rf[1]
                2: rf[2]
                else: rf[3]
        
        # Read port 2 with Bypass/Forwarding logic
        ReadData2 = if (WriteEnable = 1) & (WriteAddr = ReadAddr2):
            WriteData  # Forward from write data
        else:
            match ReadAddr2:
                0: rf[0]
                1: rf[1]
                2: rf[2]
                else: rf[3]
```

**Key Features**:
- Array of registers using `[]` syntax
- Compile-time `for` loop for register updates
- Write forwarding (bypass) logic
- Address decoding with `match`
- Demonstrates professional CPU design patterns

### Pipeline Register with Handshake

```hw
# ============================================================================
# PipelineRegister8 - Valid/Ready Handshake Protocol
# ============================================================================
module PipelineRegister8:
    pins: [
        input DataIn[8],
        input ValidIn,
        output ReadyOut,
        input Clock,
        input Reset,
        output DataOut[8],
        output ValidOut,
        input ReadyIn
    ]
    
    logic:
        let data_reg = reg(clock: Clock, reset: Reset, init: 0)
        let valid_reg = reg(clock: Clock, reset: Reset, init: 0)
        
        # Handshake control wires
        let transfer = ValidIn & ReadyOut
        let consume = ValidOut & ReadyIn
        
        # Single = for comparison
        data_reg.next = if Reset = 1:
            0
        else if transfer = 1:
            DataIn
        else:
            data_reg
        
        valid_reg.next = if Reset = 1:
            0
        else if transfer = 1:
            1
        else if consume = 1:
            0
        else:
            valid_reg
        
        DataOut = data_reg
        ValidOut = valid_reg
        ReadyOut = if valid_reg = 1: ReadyIn else: 1
```

**Key Features**:
- Valid/Ready handshake protocol (industry standard)
- Backpressure handling
- Demonstrates how to build pipeline stages
- Clean separation of control logic and data path

---

## Why the v0.1.6 Logic Unification Wins

### 1. The End of the `define` Bloat

By simply typing `module Name:`, we treat hardware modules as first-class citizens. The code looks like a modern programming language (Rust/F#), not a 1990s config file.

### 2. Explicit Time Control

The `.next` keyword on registers makes it impossible to have Verilog-style race conditions. If you use `=`, it's a wire. If you use `.next`, it's a clock-driven memory update.

The lowercase `reg` primitive signals that this is a built-in hardware primitive, not an imported component.

### 3. Concatenation Clarity

Using `[high_bits, low_bits]` for hardware concatenation leverages the Universal List rule. It makes bit-shuffling (the most common task in hardware) beautiful.

### 4. Complex State Management

In the `Register8` example, notice the nested `match` and `if` inside the `reg_val.next` assignment. This allows the designer to express **Intent** (I want to increment, decrement, or load) in one place, rather than scattering logic across five files.

The single `=` operator for comparison eliminates the `==` vs `=` confusion from C/Verilog.

### 5. Professional Patterns

The `RegisterFile4x8` and `PipelineRegister8` examples show that Hardware Script can handle professional CPU design patterns:
- Write forwarding (bypass logic)
- Pipeline handshakes (Valid/Ready)
- Address decoding
- Backpressure handling

---

## Material Definitions

Materials also benefit from the unified syntax.

```hw
# ✅ v0.1.6 - Clean and consistent
material Copper:
    category: conductor
    properties:
        resistivity: 1.68e-8Ω·m
        density: 8960kg/m³
        thermal_conductivity: 401W/(m·K)
        temp_coefficient: 3.9e-3/°C

material FR4:
    category: insulator
    properties:
        dielectric_constant: 4.5
        loss_tangent: 0.02
        thermal_conductivity: 0.3W/(m·K)
        breakdown_voltage: 20kV/mm

material Silicon:
    category: semiconductor
    properties:
        bandgap: 1.12eV
        electron_mobility: 1400cm²/(V·s)
        hole_mobility: 450cm²/(V·s)
        intrinsic_carrier_density: 1.5e10/cm³
```

---

## Profile Definitions

```hw
# ✅ v0.1.6 - Flexible and extensible
profile StandardPCB:
    allowed_materials:
        conductors: [Copper, Silver, Gold]
        dielectrics: [FR4, Rogers4350B, Air]
        soldermask: [LPI_Green, LPI_Blue, LPI_Red]
    
    constraints:
        min_trace_width: 0.1mm
        min_trace_spacing: 0.1mm
        min_via_diameter: 0.3mm
        min_drill_diameter: 0.2mm
        min_annular_ring: 0.15mm
    
    # Custom fields for manufacturing
    manufacturer_code: "FAB-001"
    process_node: "Standard PCB"
    quality_grade: "IPC Class 2"

profile HighSpeedPCB:
    allowed_materials:
        conductors: [Copper]
        dielectrics: [Rogers4350B, Rogers4003C]
    
    constraints:
        min_trace_width: 0.075mm
        min_trace_spacing: 0.075mm
        impedance_tolerance: 10%
        max_trace_length_mismatch: 0.5mm
    
    # Custom fields
    controlled_impedance: true
    differential_pairs: true
```

---

## Component Definitions

```hw
# ✅ v0.1.6 - Parametric components
component Resistor_0805 (resistance: Measurement, tolerance: Measurement):
    pins: [A, B]
    
    electrical:
        resistance: resistance
        tolerance: tolerance
        power_rating: 0.125W
        max_voltage: 150V
    
    layout:
        shape: Rectangle(2.0mm, 1.25mm, 0.6mm)
        pin_positions:
            A at [0mm, 0.625mm]
            B at [2.0mm, 0.625mm]
        pad_shapes:
            A: Rectangle(0.9mm, 1.2mm)
            B: Rectangle(0.9mm, 1.2mm)
    
    metadata:
        manufacturer: "Yageo"
        series: "RC0805"
        package: "0805"
        datasheet: "https://www.yageo.com/upload/media/product/productsearch/datasheet/rchip/PYu-RC_Group_51_RoHS_L_12.pdf"

component Capacitor_0805 (capacitance: Measurement, voltage: Measurement):
    pins: [A, B]
    
    electrical:
        capacitance: capacitance
        voltage_rating: voltage
        tolerance: 10%
        dielectric: "X7R"
    
    layout:
        shape: Rectangle(2.0mm, 1.25mm, 0.6mm)
        pin_positions:
            A at [0mm, 0.625mm]
            B at [2.0mm, 0.625mm]
        pad_shapes:
            A: Rectangle(0.9mm, 1.2mm)
            B: Rectangle(0.9mm, 1.2mm)
    
    metadata:
        manufacturer: "Murata"
        series: "GRM21"
        package: "0805"
```

---

## Conclusion

By applying the v0.1.6 Unified Grammar to the Standard Library, we achieve:

1. **Consistency**: Same syntax patterns across all definition types
2. **Simplicity**: The compiler uses one parser for all blocks
3. **Extensibility**: Users can add custom definitions following the same patterns
4. **Readability**: Code reads like a professional engineering document

The standard library is now **half the size but twice as powerful**. The compiler can use a single recursive expression-parser for both the properties (colons) and the logic (equals), making the Rust backend extremely lean.

**This is the foundation for the most robust logic synthesizer in the industry.**
