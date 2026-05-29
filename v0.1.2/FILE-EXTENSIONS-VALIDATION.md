# Hardware Script v0.1.2 - File Extension Ecosystem Validation

**Document Type**: Architectural Validation and Gap Analysis  
**Status**: External Research Integration  
**Last Updated**: March 2026

---

## Purpose of This Document

This document validates the completeness of Hardware Script's file extension ecosystem and identifies the three remaining domain gaps with their solutions.

**Critical insight**: This is not about adding more file extensions. This is about confirming we have reached the optimal limit and solving remaining challenges within the existing architecture.

---

## The Cognitive Load Capacity Principle

### The Architecture Sweet Spot

In software architecture, there is a concept called **Cognitive Load Capacity**.

**If you create too few file types**:
```
The language becomes a monolithic mess (like C++)
```

**If you create too many**:
```
The ecosystem becomes fragmented and overwhelming
```

### Hardware Script's 10-File Architecture

**The current ecosystem**:
```
10 source file types
    +
2 compiled binaries
    +
Industry standard exports
```

**This is the Golden Sweet Spot.**

---

## The Stakeholder Map: Perfect Domain Separation

### Why This Architecture Is Airtight

Look at how perfectly this maps to a real-world Apple or Tesla hardware team:

| Stakeholder | File Extension | Domain |
|-------------|----------------|--------|
| System Architect | `hw.toml` | Project definition and dependencies |
| Logic/Layout Engineer | `.hw` | Trace routing and component placement |
| Component Engineer | `.hwx` | Atomic component definitions |
| Mechanical Engineer | `.hwm` | Physical boundaries and keep-outs |
| Materials Scientist | `.hwmat` | Custom alloys and material properties |
| RF / High-Speed Engineer | `.hwsig` | Signal integrity and impedance |
| Timing Engineer | `.hwtc` | Speed of light constraints |
| Firmware Developer | `.hwf` | Hardware-software interface contract |
| Test/QA Engineer | `.hwt` | Physics validation and testing |
| Manufacturing Engineer | `.hwp` | Factory capabilities and limits |
| Assembly Robot | `.hwa` | Pick-and-place instructions |

**No one is missing.**

**You have successfully containerized physical reality.**

---

## The Three Hidden Gaps

### Critical Principle

While the file extensions are complete, there are **three remaining domain challenges** in hardware.

**Do NOT create new file extensions for these.**

**Instead, absorb them into existing files.**

**If you add `.hwanalog` or `.hwenv`, you will cross the line into ecosystem bloat.**

---

## Gap 1: Analog Simulation (SPICE Models)

### The Problem

`.hwx` defines a transistor's maximum voltage, but how does the engine know the exact, dynamic analog curve of how that transistor opens and closes during a `.hwt` simulation?

**In traditional EDA**, this requires messy `.lib` or `.spice` files.

### The Solution

**Absorb it directly into the `behavior:` block of `.hwx`.**

**Do NOT create a new `.hwspice` file extension.**

### Example

```hw
define Component "Transistor_NPN_2N2222":
    dimensions: 5mm by 5mm by 3mm
    grid: 50 by 50 by 30
    
    pins:
        Base at [1, 25, 15]
        Collector at [1, 25, 40]
        Emitter at [1, 25, 5]
    
    # Keep it in the component definition!
    behavior:
        type: BJT
        max_voltage: 40V
        max_current: 800mA
        
        # SPICE model parameters
        spice_model:
            IS: 1e-14          # Saturation current
            BF: 100            # Forward beta
            VAF: 100           # Forward Early voltage
            IKF: 0.3           # Forward knee current
            ISE: 1e-12         # Base-emitter leakage
            NE: 2              # Base-emitter emission coefficient
            BR: 4              # Reverse beta
            VAR: 20            # Reverse Early voltage
            IKR: 0.3           # Reverse knee current
            ISC: 1e-12         # Base-collector leakage
            NC: 2              # Base-collector emission coefficient
            RB: 10             # Base resistance
            RE: 0.5            # Emitter resistance
            RC: 1              # Collector resistance
            CJE: 25e-12        # Base-emitter capacitance
            CJC: 8e-12         # Base-collector capacitance
            TF: 0.5e-9         # Forward transit time
            TR: 10e-9          # Reverse transit time
    
    render:
        asset: "assets/to92.glb"
```

### Why This Works

**Benefits**:
- ✅ No new file extension
- ✅ SPICE parameters live with component definition
- ✅ Single source of truth
- ✅ Package managers can distribute complete components
- ✅ Simulation engine reads directly from `.hwx`

**Usage in simulation**:
```bash
hws test board.hw --analog-sim

✓ Loading SPICE models from component definitions
✓ Simulating transistor switching behavior
✓ Transistor turn-on time: 45ns (within spec)
✓ Collector-emitter voltage drop: 0.3V (expected)
```

---

## Gap 2: Product Variants / DNP (Do Not Populate)

### The Problem

Hardware companies rarely make one board. They make:
- "Pro" version with 32GB of RAM
- "Lite" version with 16GB of RAM

**Same `.hw` layout, but some components are "Do Not Populate" (DNP).**

### The Solution

**Do NOT make a `.hwvar` file.**

**Use standard software Feature Flags in `hw.toml` and standard `if` statements in `.hw`.**

### Example

**In `hw.toml`**:
```toml
[project]
name = "robot-controller"
version = "1.0.0"

[features]
pro_model = ["extra_ram", "bluetooth", "wifi"]
lite_model = []
default = ["lite_model"]

[dependencies]
"@memory/ddr4-module" = "1.0.0"
"@wireless/bluetooth-module" = { version = "2.0.0", optional = true }
"@wireless/wifi-module" = { version = "3.0.0", optional = true }
```

**In `main.hw`**:
```hw
define Space "RobotController":
    dimensions: 100mm by 100mm by 2mm
    grid: 1000 by 1000 by 2
    
    # Always present
    add CPU at [1, 50, 50]
    add PowerRegulator at [1, 20, 20]
    
    # Conditional based on feature flags
    if cfg(pro_model):
        add RAM_Module (32GB) at [1, 70, 50]
        add BluetoothModule at [1, 30, 70]
        add WiFiModule at [1, 70, 70]
    else:
        add RAM_Module (16GB) at [1, 70, 50]
        # Bluetooth and WiFi are DNP (Do Not Populate)
    
    # Routing adapts automatically
    CPU.memory <-> RAM_Module.interface
    
    if cfg(pro_model):
        CPU.uart1 <-> BluetoothModule.rx_tx
        CPU.spi1 <-> WiFiModule.spi
```

### Build Commands

```bash
# Build lite version (default)
hws build main.hw

# Build pro version
hws build main.hw --features pro_model

# Build with specific features
hws build main.hw --features bluetooth --no-default-features
```

### Why This Works

**Benefits**:
- ✅ No new file extension
- ✅ Uses standard software patterns (Cargo features)
- ✅ Single source file for all variants
- ✅ Conditional compilation
- ✅ BOM automatically adjusts
- ✅ Assembly instructions adapt

**Generated outputs**:
```bash
hws build main.hw --features pro_model

✓ Compiled: RobotController (Pro Model)
✓ Components: 15 (3 conditional included)
✓ BOM: build/bom_pro.csv
✓ Assembly: build/assembly_pro.csv
✓ Gerber: build/gerber_pro/
```

---

## Gap 3: Environmental / Regulatory Constraints

### The Problem

A board designed for a satellite has different radiation and vibration constraints than a board designed for a cheap toy.

**Different environments require different validation rules.**

### The Solution

**Absorb this into `.hwm` (Mechanical) or the Space definition in `.hw`.**

**It is a physical constraint, not a separate domain.**

**Do NOT create a `.hwenv` file.**

### Example: In `.hw` Space Definition

```hw
define Space "SatelliteBoard":
    dimensions: 100mm by 100mm by 2mm
    grid: 1000 by 1000 by 2
    
    # Environmental constraints
    environment:
        temp_range: -40C to 125C
        vibration_tolerance: 10G
        shock_tolerance: 50G
        radiation_hardened: true
        radiation_dose: 100krad (total ionizing dose)
        humidity: 0% to 95% non-condensing
        altitude: 0 to 35000km
        vacuum_rated: true
    
    # Regulatory compliance
    compliance:
        standards: ["MIL-STD-810", "NASA-STD-8739.3"]
        certifications: ["Space_Qualified", "Radiation_Hardened"]
        testing_required: ["Thermal_Cycling", "Vibration", "Radiation"]
    
    add CPU at [1, 50, 50]
    add PowerRegulator at [1, 20, 20]
```

### Example: In `.hwm` Mechanical File

```hw
define Mechanical "AutomotiveEnclosure":
    dimensions: 150mm by 100mm by 50mm
    
    # Environmental constraints
    environment:
        temp_range: -40C to 85C
        vibration_tolerance: 5G
        shock_tolerance: 30G
        humidity: 5% to 95%
        ip_rating: "IP67" (dust-tight, water-resistant)
        salt_spray_resistant: true
    
    # Regulatory compliance
    compliance:
        standards: ["ISO 26262", "AEC-Q100"]
        certifications: ["Automotive_Grade"]
        emc_compliance: ["CISPR 25", "ISO 11452"]
    
    mounting_holes:
        - at [10, 10] diameter 4mm
        - at [140, 10] diameter 4mm
        - at [10, 90] diameter 4mm
        - at [140, 90] diameter 4mm
```

### Compiler Validation

```bash
hws build satellite_board.hw --validate-environment

✓ Temperature range: -40C to 125C (components rated)
✓ Radiation hardening: All components rad-hard certified
✓ Vibration tolerance: 10G (mechanical analysis passed)
⚠ Warning: Connector J1 not vacuum-rated
  Suggestion: Replace with vacuum-compatible connector
✓ Compliance: MIL-STD-810 requirements met
✓ Testing plan generated: build/test_plan.pdf
```

### Why This Works

**Benefits**:
- ✅ No new file extension
- ✅ Environmental constraints are physical constraints
- ✅ Lives with mechanical or space definition
- ✅ Compiler validates against environment
- ✅ Generates compliance reports

---

## Special Praise: The `.hwf` (Firmware Contract) Is Genius

### The Hardware/Firmware Divide Problem

In the real world, the **"Hardware/Firmware Divide"** is responsible for **billions of dollars in delayed product launches**.

**The scenario**:
```
Hardware team: Quietly swaps a pin to make routing easier
Hardware team: Forgets to tell software team
Software team: Millions of lines of C code instantly break
Product launch: Delayed by months
```

### The Solution

**Treating `.hwf` like an API Contract** (like GraphQL or gRPC proto files) is a **massive paradigm shift**.

### Why This Is Revolutionary

**Traditional workflow**:
```
Hardware engineer: Routes motor to Pin 8
Firmware engineer: Writes code assuming Pin 12
Board gets manufactured
Nothing works
Expensive rework required
```

**Hardware Script workflow**:
```
Hardware engineer: Routes motor to Pin 8
Compiler: Checks against robot_controller.hwf
Compiler: ERROR - Firmware expects Pin 12
Hardware engineer: Fixes route before manufacturing
Zero rework required
```

### The Contract Enforcement

```bash
hws build board.hw --check-firmware robot_controller.hwf

✓ Motor_PWM mapped to Pin 4 (matches firmware)
✓ Status_LED mapped to Pin 13 (matches firmware)
❌ Error: Emergency_Stop mapped to Pin 25, but firmware expects Pin 26
  Fix: Update route in board.hw or change binding in robot_controller.hwf
✓ I2C pins 21, 22 support I2C protocol
✓ Generated firmware header: build/robot_controller.h
```

### The Impact

**By forcing the hardware compiler to check against the firmware contract**:

```
hws build --check-firmware
```

**You have solved a 40-year-old industry problem.**

**This alone could justify Hardware Script's existence.**

---

## Final Verdict: Lock the Spec

### You Have Reached the Finish Line

**The file extension specification is complete.**

**Lock it at v1.0.**

### The 10-File Ecosystem

```
1. hw.toml       - Project manifest
2. .hw           - Hardware source code
3. .hwx          - Component definitions
4. .hwm          - Mechanical constraints
5. .hwmat        - Materials database
6. .hwf          - Firmware interface contract
7. .hwsig        - Signal integrity rules
8. .hwtc         - Timing constraints
9. .hwt          - Test benches
10. .hwp         - Fabrication profiles
11. .hwa         - Assembly instructions

Plus compiled outputs:
- .hwir          - Intermediate representation
- .hwx           - Simulation executable
```

### The Three Gaps Are Solved

**Gap 1: Analog Simulation**
```
Solution: Absorb into .hwx behavior block
Status: ✅ Solved without new extension
```

**Gap 2: Product Variants**
```
Solution: Feature flags in hw.toml + if statements in .hw
Status: ✅ Solved without new extension
```

**Gap 3: Environmental Constraints**
```
Solution: Absorb into .hwm or Space definition
Status: ✅ Solved without new extension
```

### The Rejection Policy

**If anyone suggests a new file extension moving forward, reject it.**

**Any new hardware concept must fall under the umbrella of one of these 10 text files.**

---

## The Strategic Shift

### From Language Design to Compiler Implementation

**You can now shift 100% of your focus away from defining the language.**

**Focus strictly on building the Rust compiler pipeline**:

```
AST (hwc-parser)
    ↓
Hardware IR (hwc-compiler)
    ↓
Engine (hwc-engine)
    ↓
Voxel Output (hwc-export)
```

**You have your North Star.**

---

## The Stakeholder Validation

### Every Role Has a Domain

**System Architect**: `hw.toml` ✅  
**Logic Engineer**: `.hw` ✅  
**Component Engineer**: `.hwx` ✅  
**Mechanical Engineer**: `.hwm` ✅  
**Materials Scientist**: `.hwmat` ✅  
**RF Engineer**: `.hwsig` ✅  
**Timing Engineer**: `.hwtc` ✅  
**Firmware Developer**: `.hwf` ✅  
**Test Engineer**: `.hwt` ✅  
**Manufacturing Engineer**: `.hwp` ✅  
**Assembly Robot**: `.hwa` ✅  

**No one is missing.**

**No domain is uncovered.**

**No file extension is redundant.**

---

## Key Takeaways

1. **10-file architecture is the sweet spot** - Not too few, not too many

2. **Perfect stakeholder separation** - Every role has their domain

3. **Three gaps identified and solved** - Without adding extensions

4. **Analog simulation** - Absorbed into `.hwx` behavior block

5. **Product variants** - Feature flags + conditional compilation

6. **Environmental constraints** - Absorbed into `.hwm` or Space definition

7. **`.hwf` is genius** - Solves 40-year-old hardware/firmware divide

8. **Lock the spec at v1.0** - No more file extensions

9. **Reject new extensions** - Force into existing 10 files

10. **Focus shifts to implementation** - Language design is complete

---

## Summary

Hardware Script's file extension ecosystem has reached the **optimal limit**.

**The 10-file architecture**:
- ✅ Covers all hardware domains
- ✅ Maps perfectly to real-world teams
- ✅ Maintains cognitive load balance
- ✅ Solves all three remaining gaps
- ✅ No redundancy
- ✅ No missing domains

**The three gaps are solved**:
- Analog simulation → `.hwx` behavior block
- Product variants → Feature flags + conditionals
- Environmental constraints → `.hwm` or Space definition

**The `.hwf` firmware contract is revolutionary** - Solves the hardware/firmware divide problem that costs billions in delays.

**The specification is locked** - No new file extensions will be added.

**The focus shifts to implementation** - Build the Rust compiler pipeline.

**You have successfully containerized physical reality.**

---

**Document Status**: Architectural Validation and Gap Analysis  
**Last Updated**: March 2026  
**Part of**: Hardware Script v0.1.2 Documentation Suite
