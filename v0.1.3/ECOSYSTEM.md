# Book 3: The Ecosystem & File Extensions

**Hardware Script v0.1.3**  
**Target Audience**: Project architects setting up workspaces  
**Last Updated**: March 2026

---

## Foundation

This document consolidates the following source materials:

**Source Files**:
- `Docs/v0.1.1/FILE-EXTENSIONS.md` — The master list of .hw, .hwx, .hwp, .hwf, and all file types
- `Docs/v0.1.2/FILE-EXTENSIONS-VALIDATION.md` — The explanation of how these replace traditional EDA gaps
- `Docs/v0.0.1/TOOLING.md` — The package registry, standard library paths, and CLI commands
- `Docs/v0.1.2/IDE-INTEGRATION-AND-LAUNCH.md` — VS Code extension setup and file icons

---

## Quick Reference: The 10-File Architecture

```
Project Root
├── hw.toml                    # Project manifest
├── hw.lock                    # Dependency lockfile
├── main.hw                    # Entry point
├── power_system.hw            # Module
├── sensor_array.hw            # Module
├── enclosure.hwm              # Mechanical constraints
├── robot_controller.hwf       # Firmware interface contract
├── materials/
│   └── custom.hwmat           # Custom materials database
├── constraints/
│   ├── usb_signals.hwsig      # Signal integrity rules
│   ├── ddr4_timing.hwtc       # Timing constraints
│   └── assembly.hwa           # Assembly instructions
├── tests/
│   └── power_test.hwt         # Test benches
├── profiles/
│   └── jlcpcb.hwp             # Fabrication profile
└── build/                     # Generated outputs
    ├── board.hwir             # Intermediate representation
    ├── board.hwx              # Simulation executable
    ├── robot_controller.h     # Auto-generated firmware header
    ├── gerber/                # PCB manufacturing files
    ├── assembly/              # Pick-and-place files
    ├── bom/                   # Bill of materials
    └── viz/                   # 3D models
```

---

## Part I: Philosophy and Design Principles

### The Cognitive Load Capacity Principle

In software architecture, there is a concept called **Cognitive Load Capacity**.

**If you create too few file types**:
```
The language becomes a monolithic mess (like C++)
```

**If you create too many**:
```
The ecosystem becomes fragmented and overwhelming
```

**Hardware Script's 10-file architecture is the Golden Sweet Spot.**

### Why This Architecture Matters

Hardware Script bridges the gap between digital software and physical reality. Unlike pure software, hardware requires file extensions that handle:

- **Physical boundaries** (Mechanicals)
- **Real-world testing** (Physics/Stimulus)
- **Factory rules** (Fabrication)
- **Hardware-firmware contracts** (Interface bindings)
- **High-speed physics** (Signal integrity, timing)

### The Stakeholder Map: Perfect Domain Separation

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

**No one is missing. You have successfully containerized physical reality.**

---

## Part II: The File Extensions

### 1. Project Configuration Files

#### hw.toml (The Manifest)

**What it is**: Project metadata, build scripts, and package dependencies.

**Analogy**: `package.json` (Node.js) or `Cargo.toml` (Rust)

**Why we need it**: Tells the `hpm` package manager what libraries to download, defines the project name, and sets the default target scale.

**Example**:
```toml
[project]
name = "robot-controller"
version = "1.0.0"
scale = "PCB"
author = "Your Name"
license = "MIT"

[dependencies]
"@power/5v-regulator" = "1.2.0"
"@sensors/imu-module" = "2.0.1"
"@motor/h-bridge" = "3.0.0"

[build]
default_target = "pcb"
optimization_level = 2

[features]
pro_model = ["extra_ram", "bluetooth", "wifi"]
lite_model = []
default = ["lite_model"]
```

**Feature Flags for Product Variants**:
```bash
# Build lite version (default)
hws build main.hw

# Build pro version
hws build main.hw --features pro_model
```

#### hw.lock (The Lockfile)

**What it is**: Auto-generated file that locks exact versions of component libraries.

**Analogy**: `package-lock.json` (Node.js) or `Cargo.lock` (Rust)

**Why we need it**: Hardware is expensive. If you compile a board today and re-compile it 5 years from now, you must generate the exact same geometry. The lockfile ensures a dependency update doesn't accidentally move a component by 0.1mm and ruin your board.

**Critical for**: Reproducible builds, manufacturing consistency, version control

**Example**:
```toml
# This file is @generated by hpm.
# It is not intended for manual editing.
version = 1

[[package]]
name = "@power/5v-regulator"
version = "1.2.0"
source = "registry+https://registry.hw-script.org"
checksum = "a3f8b9c2d1e4..."

[[package]]
name = "@sensors/imu-module"
version = "2.0.1"
source = "registry+https://registry.hw-script.org"
checksum = "7e2a1f5c8b3d..."
dependencies = [
    "@comms/i2c-interface 1.0.0",
]
```


---

### 2. Source Code Files

#### .hw (Hardware Source)

**What it is**: The main programming file. Defines spaces, places components, and routes traces.

**Analogy**: `.js`, `.ts`, or `.rs`

**Why we need it**: This is the core language. It can act as an entry point (`main.hw`) or an imported module (`power_system.hw`).

**Use cases**:
- Entry point: `main.hw`, `board.hw`
- Modules: `power_system.hw`, `sensor_array.hw`
- Libraries: Pre-routed sub-assemblies that can be imported

**Example**:
```hw
define space "MyBoard":
    dimensions: 100mm by 100mm by 2mm
    grid: 1000 by 1000 by 2

add Transistor at [50, 50, 1]
route Power.out to Transistor.base:
    path:
        - [10, 10, 1]
        - [50, 50, 1]
```

#### .hwx (Hardware Component Definition)

**What it is**: The absolute physical and electrical definition of a single atomic component (e.g., a 2N2222 Transistor or an ESP32 chip).

**Analogy**: `.d.ts` (TypeScript type definitions) or `.h` (C header files)

**Why we need it**: The compiler needs to know exactly how big a part is, where its pins are, and its maximum voltage. You usually don't write these; you download them from manufacturers via `hpm`.

**Package structure** (when including 3D assets):
```
~/.hw/packages/ics/esp32/
├── esp32.hwx          # The math, pins, and physics (The Brain)
└── assets/
    ├── chip.glb       # The 3D model and textures (The Body)
    └── footprint.png  # Optional silkscreen graphic
```

**Component Definition Syntax**: `.hwx` files use the five abstraction blocks (pins, behavior, layout, electrical, render) to define components. For complete syntax and examples, see Book 2 (Language Spec), Part III: Abstraction Blocks.

**Why SPICE models live here**: Analog simulation parameters belong with the component definition, not in a separate file. This keeps everything in one place and makes components self-contained.

---

### 3. Physical Constraint Files

#### .hwm (Hardware Mechanical / Boundary)

**What it is**: Defines the 3D physical constraints, keep-out zones, and enclosure boundaries.

**Analogy**: CSS (defines the visual box/boundaries the logic must live inside)

**Why we need it**: A motherboard must fit inside a specific plastic case. A `.hwm` file allows a mechanical engineer to define "You cannot route traces here, a screw goes here," and the `.hw` compiler will obey it.

**Use cases**:
- Enclosure dimensions
- Mounting hole positions
- Keep-out zones (where components can't go)
- Thermal zones (areas requiring heat dissipation)
- Connector positions (USB ports, power jacks)
- Environmental constraints (temperature, vibration, humidity)

**Example**:
```hw
define mechanical "RobotEnclosure":
    dimensions: 150mm by 100mm by 50mm
    
    # Mounting holes
    mounting_holes:
        - at [5, 5] diameter 3mm
        - at [145, 5] diameter 3mm
        - at [5, 95] diameter 3mm
        - at [145, 95] diameter 3mm
    
    # Keep-out zones
    keepout:
        # Area where battery sits
        - region [20, 20] to [60, 60] height 15mm
        # Area for cooling fan
        - region [100, 70] to [140, 95] height 10mm
    
    # External connectors
    connectors:
        USB_Port at [0, 50] facing West
        Power_Jack at [150, 50] facing East
    
    # Environmental constraints
    environment:
        temp_range: -40C to 85C
        vibration_tolerance: 5G
        shock_tolerance: 30G
        humidity: 5% to 95%
        ip_rating: "IP67" (dust-tight, water-resistant)
    
    # Regulatory compliance
    compliance:
        standards: ["ISO 26262", "AEC-Q100"]
        certifications: ["Automotive_Grade"]
        emc_compliance: ["CISPR 25", "ISO 11452"]
```

**Compiler validation**:
```bash
hws build board.hw --check-mechanical enclosure.hwm

✓ All components fit within enclosure
✓ Mounting holes clear of traces
✓ USB connector aligned with enclosure opening
❌ Error: Component C5 extends into keep-out zone
  Suggestion: Move C5 at least 2mm away from battery area
✓ Environmental constraints validated
```

#### .hwmat (Hardware Materials Database)

**What it is**: A materials library defining the physical and electrical properties of conductors, insulators, and semiconductors.

**Analogy**: JSON/YAML data files, or `.env` configuration files

**Why we need it**: The compiler needs to know material properties (resistivity, thermal conductivity, dielectric constants) to perform physics simulations, calculate trace widths, and validate thermal constraints.

**File locations**:
- Standard library: `~/.hw/data/standard-materials.hwmat`
- Project-specific: `./materials/custom.hwmat`
- User library: `~/.hw/materials/my_materials.hwmat`

**The 3-Tiered Material System**:

Hardware Script uses a three-level material hierarchy with override priority:

1. **Level 1: Core Standard Library** (immutable)
   - Ships with the compiler
   - Fundamental physics for common materials (copper, FR4, silicon, etc.)
   - Location: `~/.hw/data/standard-materials.hwmat`
   - ~50-100 materials covering 99% of use cases

2. **Level 2: Factory/PDK Library** (manufacturer-provided)
   - Specific composite materials and factory-specific grades
   - Downloaded from manufacturer websites (JLCPCB, PCBWay, TSMC PDKs)
   - Location: `./materials/factory_name.hwmat`
   - Examples: "JLCPCB_1oz_copper", "TSMC_N5_metal_stack"

3. **Level 3: User/Project Library** (fully custom)
   - Experimental materials, custom alloys, proprietary compounds
   - Location: `./materials/custom.hwmat` or `~/.hw/materials/my_materials.hwmat`
   - Examples: Custom silver alloys, exotic dielectrics

**Override priority**: User (Level 3) → Factory (Level 2) → Core (Level 1)

If a material is defined in multiple locations, the most specific definition wins. This allows users to override factory defaults or experiment with custom materials without modifying the standard library.

**Example**:
```yaml
# custom_materials.hwmat
conductors:
  silver:
    name: Silver
    symbol: Ag
    description: Highest conductivity metal, premium RF applications
    density_kg_m3: 10490
    thermal_conductivity_w_mk: 429
    color_hex: '#C0C0C0'
    resistivity_ohm_m: 1.59e-08
    max_current_density_a_mm2: 40
    melting_point_c: 962
    is_metal: true

insulators:
  polyimide:
    name: Polyimide (Kapton)
    symbol: PI
    description: Flexible PCB substrate, high-temp applications
    density_kg_m3: 1420
    thermal_conductivity_w_mk: 0.12
    color_hex: '#FFA500'
    relative_permittivity: 3.4
    dielectric_strength_kv_mm: 303
    glass_transition_temp_c: 360

semiconductors:
  gallium_nitride:
    name: Gallium Nitride
    symbol: GaN
    description: High-power, high-frequency transistors
    density_kg_m3: 6150
    thermal_conductivity_w_mk: 130
    color_hex: '#4169E1'
    band_gap_ev: 3.4
    electron_mobility_cm2_vs: 1000
    hole_mobility_cm2_vs: 30

metadata:
  version: '0.2'
  last_updated: '2026-03-14'
  author: 'Your Name'
  license: 'MIT'
```

**Usage in .hw files**:
```hw
# Import custom materials
import materials from "./materials/custom.hwmat"

define space "HighPowerBoard":
    dimensions: 50mm by 50mm by 2mm
    substrate: materials.ceramic_alumina
    
route PowerBus.Plus to Amplifier.VDD:
    material: materials.silver
    width: 2mm
    max_current: 10A
```

---

### 4. Hardware-Firmware Interface

#### .hwf (Hardware Firmware Interface / The Contract)

**What it is**: A declarative mapping file that binds physical hardware nets/pins to software variables and protocols. It acts as the "API Contract" between hardware and firmware.

**Analogy**: `.env` files, OpenAPI/Swagger specs, or GraphQL schemas

**Why we need it**: The line between hardware and firmware is where 99% of hardware projects fail. The hardware engineer routes a motor to Pin 8, but the firmware developer writes code assuming it's on Pin 12. The board gets manufactured, and nothing works. `.hwf` files prevent this by creating a binding contract that both sides must honor.

**Critical insight**: Firmware is just software (C/C++/Rust) that runs on bare metal. Your `.hw` file is the Frontend (React), the firmware is the Backend (Node.js), and `.hwf` is the API contract between them.

**Use cases**:
- Auto-generate C/Rust header files for firmware developers
- Validate that analog sensors aren't routed to digital-only pins
- Ensure PWM signals go to PWM-capable pins
- Document the hardware-software interface
- Prevent hardware/firmware integration failures

**Example**:
```hw
interface "RobotController":
    target: "ESP32_WROOM_32"
    
    bindings:
        # Software Variable = Physical Component Pin
        Motor_PWM = DriverIC.Pin_4 (type: PWM, freq: 10kHz)
        Status_LED = LED1.Anode (type: Digital_Out)
        Temp_Sensor = Thermistor.Out (type: Analog_In, range: 0-3.3V)
        Emergency_Stop = Button1.Out (type: Digital_In, pull: up)
        
    protocols:
        I2C_Bus_1:
            SDA: Chip.Pin_21
            SCL: Chip.Pin_22
            speed: 400kHz
            devices:
                - IMU_Sensor (address: 0x68)
                - OLED_Display (address: 0x3C)
        
        SPI_Bus_1:
            MOSI: Chip.Pin_23
            MISO: Chip.Pin_19
            SCK: Chip.Pin_18
            CS: Chip.Pin_5
            speed: 10MHz
    
    interrupts:
        Encoder_A: Chip.Pin_34 (trigger: rising_edge)
        Encoder_B: Chip.Pin_35 (trigger: rising_edge)
```

**Auto-generated C header** (`robot_controller.h`):
```c
// Auto-generated from robot_controller.hwf
// DO NOT EDIT - Regenerate with: hws build --gen-headers

#ifndef ROBOT_CONTROLLER_H
#define ROBOT_CONTROLLER_H

// Pin Definitions
#define MOTOR_PWM_PIN       4
#define STATUS_LED_PIN      13
#define TEMP_SENSOR_PIN     36  // ADC1_CH0
#define EMERGENCY_STOP_PIN  25

// I2C Configuration
#define I2C_SDA_PIN         21
#define I2C_SCL_PIN         22
#define I2C_FREQ_HZ         400000
#define IMU_I2C_ADDR        0x68
#define OLED_I2C_ADDR       0x3C

// SPI Configuration
#define SPI_MOSI_PIN        23
#define SPI_MISO_PIN        19
#define SPI_SCK_PIN         18
#define SPI_CS_PIN          5
#define SPI_FREQ_HZ         10000000

// Interrupt Pins
#define ENCODER_A_PIN       34
#define ENCODER_B_PIN       35

#endif // ROBOT_CONTROLLER_H
```

**Compiler validation**:
```bash
hws build board.hw --check-firmware robot_controller.hwf

✓ Motor_PWM mapped to Pin 4 (PWM capable)
✓ Temp_Sensor mapped to Pin 36 (ADC capable)
❌ Error: Emergency_Stop mapped to Pin 25, but .hw routes it to Pin 26
  Fix: Update route in board.hw or change binding in robot_controller.hwf
✓ I2C pins 21, 22 support I2C protocol
✓ Generated firmware header: build/robot_controller.h
```

**Why this is revolutionary**: By forcing the hardware compiler to check against the firmware contract, you solve a 40-year-old industry problem. This alone could justify Hardware Script's existence.


---

### 5. High-Speed Design Constraints

#### .hwsig (Hardware Signal Integrity / RF Rules)

**What it is**: Rules for high-speed physics. When electricity pulses billions of times per second (USB, Wi-Fi, PCIe), it stops acting like water in a pipe and starts acting like radio waves.

**Analogy**: TypeScript strict mode, but for physics

**Why we need it**: A USB data wire can't just be drawn however you want. It must be exactly 90 ohms of "impedance". The compiler needs to know this so it can automatically calculate the exact copper thickness based on `.hwmat` (materials) and `.hwp` (factory capabilities).

**Use cases**:
- USB, HDMI, Ethernet differential pairs
- RF antennas and transmission lines
- High-speed memory interfaces (DDR4/DDR5)
- PCIe, SATA, DisplayPort
- Impedance-controlled traces

**Example**:
```hw
signal_group "USB_Data":
    type: differential_pair
    target_impedance: 90 ohms (tolerance: ±5%)
    max_length: 50mm
    max_length_mismatch: 0.15mm
    route_together: true
    min_spacing: 0.2mm
    
apply "USB_Data" to:
    - route USB_Port.D_Plus to CPU.USB_DP
    - route USB_Port.D_Minus to CPU.USB_DN

signal_group "Ethernet_Gigabit":
    type: differential_pair
    target_impedance: 100 ohms (tolerance: ±10%)
    pairs: 4
    max_length: 150mm
    
apply "Ethernet_Gigabit" to:
    - route RJ45.TX_Plus[0] to PHY.TX_Plus[0]
    - route RJ45.TX_Minus[0] to PHY.TX_Minus[0]
    # ... (4 pairs total)

signal_group "WiFi_Antenna":
    type: single_ended
    target_impedance: 50 ohms (tolerance: ±2%)
    max_length: 30mm
    keepout_zone: 5mm (no copper pour)
    
apply "WiFi_Antenna" to:
    - route WiFi_Module.RF_Out to Antenna.Feed
```

**Compiler validation**:
```bash
hws build board.hw --check-signal-integrity

✓ USB_Data pair impedance: 90.2Ω (within tolerance)
✓ USB_Data length match: 0.12mm difference (within 0.15mm)
✓ Ethernet pairs impedance: 98-102Ω (within tolerance)
❌ Error: WiFi_Antenna trace impedance 45.3Ω (target: 50Ω ±2%)
  Suggestion: Increase trace width from 0.3mm to 0.35mm
  Or: Adjust substrate height in .hwp profile
✓ All differential pairs routed together (no splits)
```

#### .hwtc (Hardware Timing Constraints / Synchronization)

**What it is**: Rules for the speed of light. Used primarily for FPGAs, RAM chips, and high-speed CPUs.

**Analogy**: `async/await` race condition prevention

**Why we need it**: Electricity moves at roughly half the speed of light inside a PCB (~15 cm/ns). If you have 8 wires sending data to a RAM chip at 3 GHz, and one wire is 2mm longer than the others, that data arrives ~13 picoseconds too late. The data gets corrupted. The compiler reads `.hwtc` to know it must draw squiggly lines (serpentine meandering) to make all 8 wires the exact same physical length.

**Use cases**:
- DDR3/DDR4/DDR5 memory interfaces
- FPGA clock distribution
- High-speed parallel buses
- Clock domain crossing
- Setup and hold time requirements

**Example**:
```hw
timing_group "DDR4_RAM_Bus":
    clock: CPU.CLK_OUT (frequency: 2400MHz)
    
    constraints:
        length_match: within 0.1mm
        max_delay: 2.5ns
        min_delay: 0.5ns
        skew_tolerance: 50ps
        
    apply "DDR4_RAM_Bus" to:
        - route group CPU.Data[0..7] to RAM.Data[0..7]
        - route group CPU.Addr[0..15] to RAM.Addr[0..15]

timing_group "FPGA_Clock_Tree":
    clock: Oscillator.Out (frequency: 100MHz)
    
    constraints:
        length_match: within 0.05mm
        max_delay: 5ns
        fanout: 4
        
    apply "FPGA_Clock_Tree" to:
        - route Oscillator.Out to FPGA.CLK_IN
        - route Oscillator.Out to ADC.CLK
        - route Oscillator.Out to DAC.CLK
        - route Oscillator.Out to Memory.CLK
```

**Compiler validation**:
```bash
hws build board.hw --check-timing

✓ DDR4_RAM_Bus: All traces within 0.08mm (target: 0.1mm)
✓ DDR4_RAM_Bus: Max delay 2.3ns (within 2.5ns limit)
✓ DDR4_RAM_Bus: Skew 42ps (within 50ps tolerance)
✓ FPGA_Clock_Tree: Length matched within 0.03mm
⚠ Warning: Parallel_Display trace RGB[15] is 1.2mm longer than RGB[0]
  Suggestion: Add 1.2mm serpentine meander to RGB[0..14]
✓ All timing constraints satisfied
```

---

### 6. Testing and Validation

#### .hwt (Hardware Test Bench)

**What it is**: A test script that applies simulated voltages/data to your board and asserts expected results.

**Analogy**: `.test.js` or `.spec.ts` (Jest / Mocha)

**Why we need it**: Allows Continuous Integration (CI/CD) for hardware. You can mathematically prove your power regulator works before printing the board.

**Example**:
```hw
test "Power Regulator Voltage Output":
    # Setup
    apply 12V to PowerSource.VIN
    apply 0V to PowerSource.GND
    
    # Wait for stabilization
    wait 10ms
    
    # Assertions
    assert Regulator.VOUT == 5V within 0.1V
    assert Regulator.VOUT_current < 1A
    assert Regulator.temperature < 85C

test "Short Circuit Protection":
    apply 12V to PowerSource.VIN
    short Regulator.VOUT to GND
    
    wait 1ms
    
    # Should shut down, not burn out
    assert Regulator.VOUT < 0.5V
    assert Regulator.temperature < 100C
    assert PowerSource.current < 2A

test "Transistor Switching":
    apply 5V to Battery.Plus
    apply 0V to Battery.GND
    
    # Transistor should be off
    apply 0V to Transistor.Base
    assert Transistor.Collector to Transistor.Emitter resistance > 1M Ohms
    
    # Transistor should be on
    apply 3.3V to Transistor.Base
    assert Transistor.Collector to Transistor.Emitter resistance < 100 Ohms
```

**CI/CD Integration**:
```bash
# In your GitHub Actions workflow
hws test board.hw
# Runs all .hwt files, fails build if any test fails
```

---

### 7. Manufacturing Files

#### .hwp (Hardware Fabrication Profile)

**What it is**: A ruleset defining the physical limits of the factory you are using (e.g., `jlcpcb_4layer.hwp` or `tsmc_3nm.hwp`).

**Analogy**: `.babelrc`, `tsconfig.json`, or Webpack target profiles

**Why we need it**: It defines minimum trace widths, via drill sizes, and layer spacing. If your `.hw` code tries to draw a 0.05mm wire, but your imported `.hwp` says the factory's limit is 0.1mm, the compiler throws an error before you send it to the factory.

**Example - PCB Fabrication**:
```hw
profile "JLCPCB_4Layer_Standard":
    manufacturer: "JLCPCB"
    process: "PCB"
    
    constraints:
        min_trace_width: 0.127mm      # 5 mil
        min_trace_spacing: 0.127mm    # 5 mil
        min_drill_size: 0.3mm
        min_annular_ring: 0.15mm
        
        layers:
            max_count: 4
            copper_thickness: 35um    # 1 oz copper
            
        board:
            min_size: 10mm by 10mm
            max_size: 400mm by 500mm
            thickness: 1.6mm
        
        vias:
            min_diameter: 0.3mm
            max_aspect_ratio: 10:1
    
    cost_model:
        base_price: $5.00
        per_square_cm: $0.10
        setup_fee: $0
```

**Example - Silicon Fabrication**:
```hw
profile "TSMC_3nm_FinFET":
    manufacturer: "TSMC"
    process: "Silicon"
    node: "3nm"
    
    constraints:
        min_feature_size: 3nm
        min_gate_length: 5nm
        min_metal_pitch: 24nm
        
        layers:
            metal_layers: 15
            poly_layers: 2
        
        transistors:
            type: "FinFET"
            min_width: 6nm
            max_current_density: 1mA/um
    
    cost_model:
        wafer_price: $16000
        die_size_limit: 800mm²
```

**Usage**:
```bash
# Compile with fabrication profile
hws build board.hw --profile jlcpcb_4layer.hwp

# Compiler validates against constraints
❌ Error: Trace width 0.08mm violates profile minimum 0.127mm
  at route Battery.Plus to IC.VCC
  Suggestion: Increase trace_width to 0.127mm or use different profile
```

#### .hwa (Hardware Assembly Profile)

**What it is**: Instructions for the robotic pick-and-place machines that actually solder components onto the board after fabrication.

**Analogy**: Dockerfile / CI/CD deployment script

**Why we need it**: Fabrication (`.hwp`) makes the bare green board. Assembly (`.hwa`) glues the chips to it. A chip might look fine in 3D simulation, but if you place a giant connector next to a tiny resistor, the assembly robot's nozzle might smash the connector while trying to grab the resistor.

**Use cases**:
- Pick-and-place machine instructions
- Solder paste stencil specifications
- Reflow oven temperature profiles
- Component placement clearances
- Assembly process validation

**Example**:
```hw
assembly "Standard_SMT_Pick_And_Place":
    process: "Surface_Mount_Technology"
    
    solder_paste:
        alloy: "SAC305" (Lead-Free: 96.5% Sn, 3% Ag, 0.5% Cu)
        stencil_thickness: 0.12mm
        aperture_ratio: 0.8
        
    clearances:
        tool_nozzle_keepout: 2mm around components > 5mm height
        component_to_component: 0.5mm minimum
        component_to_edge: 2mm minimum
        
    reflow_oven:
        profile: "Lead_Free_Standard"
        preheat: 150-180C for 60-90s
        soak: 180-200C for 60-120s
        reflow: 230-250C for 30-60s
        peak_temp: 245C (max)
        max_time_above_liquidus: 60s
        cooling_rate: 3C/s max
        
    placement_order:
        1: "Small passives" (resistors, capacitors < 0805)
        2: "ICs and large components"
        3: "Connectors and mechanical parts"
        
    fiducials:
        count: 3 minimum
        type: "1mm copper circle"
        locations:
            - [5, 5] (bottom-left)
            - [95, 5] (bottom-right)
            - [50, 95] (top-center)
```

**Compiler validation**:
```bash
hws build board.hw --check-assembly robot_assembly.hwa

✓ All components have 2mm clearance for pick-and-place nozzle
✓ Fiducials placed at 3 locations for machine vision
✓ Solder paste stencil apertures calculated
❌ Error: Connector J1 (height: 12mm) is 1.5mm from Resistor R5
  Assembly nozzle requires 2mm clearance
  Suggestion: Move R5 at least 0.5mm away from J1
✓ Reflow profile compatible with all components
✓ Generated assembly files:
  - build/assembly/pick_and_place.csv
  - build/assembly/stencil.gbr
  - build/assembly/placement_drawing.pdf
```

---

### 8. Compiled Outputs

#### .hwir (Hardware Intermediate Representation)

**What it is**: A pure mathematical graph of your hardware, stripped of syntax, right before it is turned into physical geometry.

**Analogy**: LLVM IR, or Java `.class` bytecode

**Why we need it**: Advanced tools (like AI optimizers or auto-routers) can read the `.hwir` graph to optimize paths faster than reading the text-based `.hw` file.

**Use cases**:
- AI-powered auto-routing
- Design optimization
- Formal verification
- Cross-compiler targets

#### .hwx (Hardware Executable Simulation)

**What it is**: The compiled binary containing the 3D grid, material states, and physics data.

**Analogy**: `.wasm` (WebAssembly) or `.exe`

**Why we need it**: It is fed into the simulation engine (or a Blender plugin) to animate electron flow, generate heat maps, and visualize the board in real-time.

**Usage**:
```bash
# Generate simulation
hws build board.hw --target sim --output board.hwx

# Run in Blender
blender --python-expr "import hwc_plugin; hwc_plugin.load('board.hwx')"

# Or standalone viewer
hwc-viewer board.hwx
```

---

### 9. Industry Standard Exports

#### Gerber Files (.gtl, .gbl, .g1, .g2, etc.)

**Purpose**: PCB manufacturing

**Format**: RS-274X (industry standard)

**Files generated**:
- `.gtl` - Top copper layer
- `.gbl` - Bottom copper layer
- `.g1`, `.g2`, etc. - Inner layers
- `.gts` - Top soldermask
- `.gbs` - Bottom soldermask
- `.gto` - Top silkscreen
- `.gbo` - Bottom silkscreen
- `.gko` - Board outline

#### Drill Files (.drl)

**Purpose**: Via and hole positions for PCB manufacturing

**Format**: Excellon (industry standard)

#### GDSII (.gds)

**Purpose**: Silicon chip fabrication

**Format**: GDSII Stream Format

**Use case**: Sending chip designs to foundries (TSMC, Intel, Samsung)

#### 3D Formats (.obj, .step, .glb)

**Purpose**: 3D visualization and mechanical CAD

**Formats**:
- `.obj` - Simple 3D mesh (universal)
- `.step` - Mechanical CAD (precise dimensions)
- `.glb` - GLTF binary (modern, compressed, includes textures)

**Recommendation**: Use `.glb` for web/visualization, `.step` for mechanical engineering

#### Bill of Materials (.csv)

**Purpose**: Component ordering and assembly

**Format**: CSV spreadsheet

**Columns**:
- Reference (e.g., "R1", "C2", "U1")
- Value (e.g., "10kΩ", "100nF", "ESP32-C3")
- Package (e.g., "0805", "TO-92", "QFN-32")
- Quantity
- Manufacturer
- Part Number
- Supplier
- Unit Price
- Total Price


---

## Part III: Package Management (hpm)

### The Hardware Package Ecosystem

Hardware Script implements a package management system analogous to NPM (Node Package Manager) for software.

**Example imports**:
```hw
import BluetoothModule from @hardware/comms
import TemperatureSensor from @sensors/environmental
import LithiumBattery from @power/rechargeable
```

Each package includes:
- Complete electrical specifications
- 3D mechanical models and footprints
- Thermal characteristics
- Reference designs and usage examples
- Version history and compatibility information

### Package Structure

A standard hardware package contains:

```
package-name/
├── package.hw.json          # Package metadata
├── component.hw             # Component definition
├── models/
│   ├── mesh.glb            # 3D model
│   └── footprint.kicad     # PCB footprint
├── docs/
│   ├── README.md           # Usage documentation
│   └── datasheet.pdf       # Manufacturer datasheet
└── examples/
    └── basic-usage.hw      # Example implementations
```

### Package Metadata (package.hw.json)

```json
{
  "name": "@sensors/temperature-ds18b20",
  "version": "1.2.0",
  "description": "Digital temperature sensor with 1-Wire interface",
  "author": "Hardware Script Community",
  "license": "MIT",
  "keywords": ["sensor", "temperature", "1-wire"],
  "electrical": {
    "voltage_min": "3.0V",
    "voltage_max": "5.5V",
    "current_typical": "1.5mA",
    "current_max": "4mA"
  },
  "physical": {
    "footprint": "TO-92",
    "dimensions": "5.2mm x 4.1mm x 2.5mm"
  },
  "dependencies": {
    "@standard/materials": "^1.0.0"
  }
}
```

### Package Manager CLI (hpm)

**Installation**:
```bash
# Install Hardware Script CLI
curl -sSL https://hw-script.org/install.sh | bash

# Or using package managers
brew install hardware-script    # macOS
apt install hardware-script     # Ubuntu/Debian
choco install hardware-script   # Windows
```

**Package Commands**:
```bash
# Initialize new project
hpm init my-project

# Search for packages
hpm search temperature sensor

# Install package
hpm install @sensors/temperature-ds18b20

# Update packages
hpm update

# List installed packages
hpm list

# Publish package (requires authentication)
hpm publish
```

**Documentation Commands**:
```bash
# List all available documentation
hpm doc list

# Read specific documentation (streams to stdout)
hpm doc read language-spec
hpm doc read ecosystem
hpm doc read routing-physics
hpm doc read compiler-internals
hpm doc read vision

# Show where documentation is installed
hpm doc path
```

**Why `hpm doc` exists**: Hardware Script documentation is installed globally at `~/.hw/docs/` when you install the compiler. The `hpm doc read` command allows both humans and AI assistants to access documentation without cluttering project directories. This follows the same pattern as `rustup doc` (Rust) and `man` (Linux).

**For AI Assistants**: When an LLM reads the `.hw-llm-index.md` bootloader file in a project, it can execute `hpm doc read <name>` to stream full documentation into its context window. This enables zero-shot Hardware Script code generation without requiring the AI to have pre-training data on the language.

**Path Resolution**: The command searches for documentation in this order:
1. `$HW_DOCS_PATH` environment variable (for testing)
2. `~/.hw/docs/` (user installation)
3. `/usr/local/share/hw/docs/` (system-wide on Unix)
4. `C:\Program Files\Hardware Script\docs\` (system-wide on Windows)
5. `Docs/v0.1.3/` (development fallback)

**Project Initialization**:
```bash
hpm init sprinkler-controller

# Creates:
# sprinkler-controller/
# ├── hw.toml                # Project manifest
# ├── hw.lock                # Dependency lockfile (auto-generated)
# ├── main.hw                # Entry point
# ├── .hw-llm-index.md       # LLM bootloader (for AI assistants)
# ├── .gitignore             # Git ignore rules
# └── README.md              # Project documentation
```

### LLM Integration (AI-Assisted Development)

Hardware Script is designed to be **LLM-native** from the ground up. Every new project includes a `.hw-llm-index.md` file that serves as a "bootloader" for AI assistants.

**The Problem**: AI assistants (ChatGPT, Claude, GitHub Copilot, Cursor) have zero pre-training data on Hardware Script. Without guidance, they will hallucinate syntax from Verilog, VHDL, or Python.

**The Solution**: The `.hw-llm-index.md` file provides:
- Minimal syntax rules to get started
- Strict generation rules (no floating-point coordinates, Manhattan routing only)
- Knowledge index pointing to documentation sources
- Command references for querying full documentation

**How It Works**:

1. **AI reads bootloader**: When opening a Hardware Script project, the AI assistant reads `.hw-llm-index.md` first
2. **AI queries documentation**: If the user asks for advanced features, the AI executes `hpm doc read <name>` to stream full documentation
3. **AI generates code**: With syntax rules and documentation, the AI can write correct Hardware Script code

**Example AI Workflow**:
```
User: "Add USB signal integrity constraints"
AI: *Reads .hw-llm-index.md*
AI: *Sees: "For signal integrity, run: hpm doc read routing-physics"*
AI: *Executes: hpm doc read routing-physics*
AI: *Reads section on .hwsig files*
AI: *Generates correct .hwsig syntax*
```

**Benefits**:
- Zero project bloat (docs stored globally at `~/.hw/docs/`)
- Works offline (no internet required)
- Web fallback for ChatGPT/Claude (https://docs.hw-script.org)
- Always synced with compiler version
- Prevents AI hallucination

**For more details**, see `LLM-INTEGRATION.md` in the project root.

### Standard Library Reference

**standard.materials**:
- Conductors: `Copper`, `Silver`, `Gold`, `Aluminum`
- Insulators: `FR4`, `Rogers`, `Polyimide`, `Ceramic`
- Semiconductors: `Silicon`, `GalliumArsenide`, `SiliconCarbide`

**standard.power**:
- `Battery(voltage)`, `LithiumIon(capacity, voltage)`
- `VoltageRegulator(input, output)`, `BuckConverter`, `BoostConverter`

**standard.sensors**:
- Environmental: `TemperatureSensor`, `HumiditySensor`, `PressureSensor`, `LightSensor`
- Motion: `Accelerometer`, `Gyroscope`, `Magnetometer`, `IMU`
- Proximity: `UltrasonicSensor`, `InfraredSensor`, `LidarSensor`

**standard.logic**:
- Basic gates: `AND`, `OR`, `NOT`, `XOR`, `NAND`, `NOR`
- Complex: `Multiplexer`, `Demultiplexer`, `FlipFlop`, `Counter`

**standard.comms**:
- Serial: `UART`, `SPI`, `I2C`
- Wireless: `Bluetooth`, `WiFi`, `LoRa`, `Zigbee`
- Wired: `Ethernet`, `CAN`, `USB`

### Community Resources

**Official Registry**: https://registry.hw-script.org

**Package Categories**:
- Power Management
- Sensors
- Communication Modules
- Display Drivers
- Motor Controllers
- Audio Processing
- RF/Wireless
- Development Boards

**Community**:
- Documentation: https://docs.hw-script.org
- Forum: https://community.hw-script.org
- GitHub: https://github.com/hwsl-lang
- Discord: https://discord.gg/G9VBxKpW
- Twitter: @hwsl_lang
- Email: hardwarescript@gmail.com

---

## Part IV: IDE Integration

### VS Code Extension

**Features**:
- Syntax highlighting for .hw files
- IntelliSense autocomplete for components and properties
- Real-time error checking and linting
- Integrated terminal for `hws verify` and `hws generate`
- 3D preview panel for visual debugging
- Component library browser

**Installation**:
```bash
code --install-extension hardware-script.hw-language-support
```

### File Icons

Hardware Script provides custom file icons for all file types:

| Extension | Icon | Color |
|-----------|------|-------|
| `.hw` | Microchip | Blue |
| `.hwx` | Component | Green |
| `.hwm` | Mechanical | Orange |
| `.hwmat` | Material | Brown |
| `.hwf` | Interface | Purple |
| `.hwsig` | Signal | Yellow |
| `.hwtc` | Clock | Cyan |
| `.hwt` | Test | Red |
| `.hwp` | Factory | Gray |
| `.hwa` | Assembly | Pink |
| `.hwx` | Executable | Dark Purple |

### Syntax Highlighting

**Supported Editors**:
- Visual Studio Code
- Sublime Text
- Atom
- Vim/Neovim
- Emacs

**TextMate Grammar** (for editor integration):
```json
{
  "scopeName": "source.hw",
  "patterns": [
    {
      "name": "keyword.control.hw",
      "match": "\\b(define|import|from|add|route|expose)\\b"
    },
    {
      "name": "entity.name.type.hw",
      "match": "\\b(material|component|space)\\b"
    },
    {
      "name": "constant.numeric.hw",
      "match": "\\b\\d+(\\.\\d+)?(mm|cm|m|V|A|mA|Ohms|C|F)\\b"
    }
  ]
}
```

### Language Server Protocol (LSP)

The Hardware Script LSP provides:
- Semantic analysis and validation
- Go-to-definition for components and materials
- Find all references for routing and connections
- Hover documentation for standard library components
- Code actions for quick fixes

---

## Part V: CI/CD Integration

### GitHub Actions Workflow

```yaml
name: Hardware Verification

on: [push, pull_request]

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Install Hardware Script
        run: |
          curl -sSL https://hw-script.org/install.sh | bash
          echo "$HOME/.hw/bin" >> $GITHUB_PATH
      
      - name: Verify Design
        run: hws verify main.hw
      
      - name: Run Tests
        run: hws test tests/*.hw
      
      - name: Generate Manufacturing Files
        run: hws generate --target manufacturing
      
      - name: Upload Gerbers
        uses: actions/upload-artifact@v2
        with:
          name: gerber-files
          path: build/factory/
```

### GitLab CI Pipeline

```yaml
stages:
  - verify
  - test
  - build

verify_design:
  stage: verify
  script:
    - hws verify main.hw

run_tests:
  stage: test
  script:
    - hws test tests/*.hw

generate_outputs:
  stage: build
  script:
    - hws generate --target simulation
    - hws generate --target manufacturing
  artifacts:
    paths:
      - build/
```

---

## Part VI: Manufacturing Integration

### Supported Fabrication Services

- **PCBWay**: Direct API integration
- **JLCPCB**: Automated quote and order
- **OSH Park**: Community PCB service
- **Seeed Studio**: Fusion PCB service

### Order Workflow

```bash
# Generate manufacturing files
hws generate --target manufacturing

# Get fabrication quote
hws quote --service jlcpcb --quantity 10

# Place order (requires API key)
hws order --service jlcpcb --quantity 10 --shipping express
```

### Assembly Services

```bash
# Generate assembly files (BOM + CPL)
hws generate --target assembly

# Check component availability
hws check-stock build/factory/bom.csv

# Get assembly quote
hws quote --service jlcpcb --assembly --quantity 10
```

---

## Part VII: Advanced Tooling

### Design Rule Checker (DRC)

```bash
# Run design rule check
hws drc --rules oshpark main.hw

# Custom rules file
hws drc --rules custom-rules.yaml main.hw
```

**custom-rules.yaml**:
```yaml
trace_width_min: 0.15mm
trace_spacing_min: 0.15mm
via_diameter_min: 0.3mm
via_drill_min: 0.2mm
copper_to_edge_min: 0.5mm
```

### BOM Optimizer

```bash
# Optimize BOM for cost
hws optimize-bom --target cost build/factory/bom.csv

# Optimize for availability
hws optimize-bom --target availability build/factory/bom.csv

# Suggest alternatives
hws suggest-alternatives R1 build/factory/bom.csv
```

### 3D Viewer

```bash
# Launch interactive 3D viewer
hws view main.hw

# Export 3D model
hws export-3d --format step main.hw
hws export-3d --format glb main.hw
```

---

## Part VIII: Implementation Roadmap

### Phase 1 (v0.1 - Current) ✅

**Focus**: Prove the concept

**Files implemented**:
- `.hw` - Basic source files
- `.gtl`, `.gbl` - Gerber export
- `.obj` - Basic 3D export
- `sim.py` - Blender script

**Status**: Working MVP

### Phase 2 (v0.2 - Q2 2026)

**Focus**: Production-ready PCB compiler

**Files to implement**:
- `hw.toml` - Project configuration
- `hw.lock` - Dependency locking
- `.hwx` - Component definitions with render blocks
- `.hwmat` - Materials database (standard + custom)
- `.hwf` - Firmware interface contracts
- `.hwp` - Fabrication profiles (JLCPCB, PCBWay)
- `.hwa` - Assembly profiles (pick-and-place, reflow)
- `.drl` - Drill files
- `.csv` - BOM generation
- `.glb` - Modern 3D format (replace .obj)

**Goal**: Send boards to manufacturers

### Phase 3 (v0.3 - Q3 2026)

**Focus**: Testing and validation

**Files to implement**:
- `.hwt` - Test benches
- `.hwir` - Intermediate representation
- `.hwm` - Mechanical constraints
- `.hwsig` - Signal integrity rules (USB, Ethernet, RF)
- `.hwtc` - Timing constraints (DDR, FPGA clocks)

**Goal**: CI/CD for hardware

### Phase 4 (v0.4 - Q4 2026)

**Focus**: Advanced simulation

**Files to implement**:
- `.hwx` - Full simulation format
- GPU-accelerated physics
- Real-time electron flow

**Goal**: Interactive hardware design

### Phase 5 (v1.0 - 2027)

**Focus**: Silicon chip support

**Files to implement**:
- `.gds` - GDSII export
- `.hwp` profiles for TSMC, Intel, Samsung
- Nanometer-scale simulation

**Goal**: One tool from PCBs to custom chips

---

## Conclusion

Hardware Script's 10-file ecosystem provides complete coverage of the hardware design workflow from concept to manufacturing. The architecture is:

- **Complete**: Every stakeholder has their domain
- **Balanced**: Not too few, not too many file types
- **Validated**: Three remaining gaps solved without new extensions
- **Professional**: Follows established software patterns
- **Scalable**: Works from hobby projects to enterprise chips

The ecosystem includes:
- Project configuration (hw.toml, hw.lock)
- Source code (.hw, .hwx)
- Physical constraints (.hwm, .hwmat)
- Hardware-firmware interface (.hwf)
- High-speed design (.hwsig, .hwtc)
- Testing (.hwt)
- Manufacturing (.hwp, .hwa)
- Compiled outputs (.hwir, .hwx)
- Industry standards (Gerber, GDSII, BOM)

With package management (hpm), IDE integration, CI/CD support, and manufacturing integration, Hardware Script provides a complete, modern development experience for hardware design.

---

**Document Version**: 0.1.3 Consolidated Edition  
**Consolidated From**:
- FILE-EXTENSIONS.md (v0.1.1)
- FILE-EXTENSIONS-VALIDATION.md (v0.1.2)
- TOOLING.md (v0.0.1)
- IDE-INTEGRATION-AND-LAUNCH.md (v0.1.2)

**Last Updated**: March 2026  
**Status**: Ecosystem Reference  
**Part of**: Hardware Script v0.1.3 Documentation Suite

