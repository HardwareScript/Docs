# Hardware Script - Holistic File System Architecture

**The Complete File Extension Topology for Physical Reality Engineering**

---

## Philosophy: Defining the Standard Upfront

This is exactly how you architect a standard that lasts for decades. You define the entire file system topology upfront, even if you are only building phase one today. It gives you a "North Star" to work towards.

Since Hardware Script is bridging the gap between digital software and physical reality, you need file extensions that handle things software doesn't usually care about:
- **Physical boundaries** (Mechanicals)
- **Real-world testing** (Physics/Stimulus)
- **Factory rules** (Fabrication)

---

## 1. The Project Roots (Configuration & Dependencies)

These files sit at the root of your project directory and tell the compiler how the project is organized.

### hw.toml (The Manifest)

**What it is**: Project metadata, build scripts, and package dependencies.

**JS Analogy**: `package.json` (or `Cargo.toml` in Rust).

**Why we need it**: Tells the `hpm` package manager what libraries to download, defines the project name, and sets the default target scale (e.g., `scale = "PCB"` or `scale = "VLSI"`).

**Example**:
```toml
[project]
name = "robot-controller"
version = "1.0.0"
scale = "PCB"

[dependencies]
"@power/5v-regulator" = "1.2.0"
"@sensors/imu-module" = "2.0.1"

[build]
default_target = "pcb"
```

### hw.lock (The Lockfile)

**What it is**: Auto-generated file that locks exact versions of component libraries.

**JS Analogy**: `package-lock.json` (or `Cargo.lock`).


**Why we need it**: Hardware is expensive. If you compile a board today, and re-compile it 5 years from now, you must generate the exact same geometry. The lockfile ensures a dependency update doesn't accidentally move a component by 0.1mm and ruin your board.

**Critical for**: Reproducible builds, manufacturing consistency, version control.

---

## 2. The Source Code (Logic & Geometry)

These are the files the engineer actually writes and tracks in Git.

### .hw (Hardware Source)

**What it is**: The main programming file. Defines spaces, places components, and routes traces.

**JS Analogy**: `.js`, `.ts`, or `.rs`.

**Why we need it**: This is the core language. It can act as an entry point (`main.hw`) or an imported module (`power_system.hw`).

**Use cases**:
- Entry point: `main.hw`, `board.hw`
- Modules: `power_system.hw`, `sensor_array.hw`
- Libraries: Pre-routed sub-assemblies that can be imported

**Example**:
```hw
define Space "MyBoard":
    dimensions: 100mm by 100mm by 2mm
    grid: 1000 by 1000 by 2

add Transistor at [1, 50, 50]
route Power.out to Transistor.base:
    path:
        - [1, 10, 10]
        - [1, 50, 50]
```

### .hwx (Hardware Component Definition)

**What it is**: The absolute physical and electrical definition of a single atomic component (e.g., a 2N2222 Transistor or an ESP32 chip).

**JS Analogy**: `.d.ts` (Type Definitions) or `.h` (C Header files).

**Why we need it**: The compiler needs to know exactly how big a part is, where its pins are, and its maximum voltage. You usually don't write these; you download them from manufacturers via `hpm`.


**Package structure** (when including 3D assets):
```
~/.hw/packages/ics/esp32/
├── esp32.hwx          # The math, pins, and physics (The Brain)
└── assets/
    ├── chip.glb       # The 3D model and textures (The Body)
    └── footprint.png  # Optional silkscreen graphic
```

**Example**:
```hw
define Component "ESP32_WROOM_32":
    dimensions: 18mm by 25mm by 3mm
    grid: 180 by 250 by 30
    
    pins:
        GND at [1, 0, 15]
        3V3 at [1, 0, 20]
    
    behavior:
        max_current: 500mA
        max_voltage: 3.6V
    
    render:
        asset: "assets/chip.glb"
        offset: [0, 0, 0]
        scale: 1.0
```

### .hwm (Hardware Mechanical / Boundary) ⭐ NEW

**What it is**: Defines the 3D physical constraints, keep-out zones, and enclosure boundaries.

**JS Analogy**: CSS (defines the visual box/boundaries the logic must live inside).

**Why we need it**: A motherboard must fit inside a specific plastic case. A `.hwm` file allows a mechanical engineer to define "You cannot route traces here, a screw goes here," and the `.hw` compiler will obey it.

**Use cases**:
- Enclosure dimensions
- Mounting hole positions
- Keep-out zones (where components can't go)
- Thermal zones (areas requiring heat dissipation)
- Connector positions (USB ports, power jacks)

**Example**:
```hw
define Mechanical "RobotEnclosure":
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
```

### .hwmat (Hardware Materials Database) ⭐ NEW

**What it is**: A materials library defining the physical and electrical properties of conductors, insulators, and semiconductors used in hardware designs.

**JS Analogy**: JSON/YAML data files, or `.env` configuration files.

**Why we need it**: The compiler needs to know material properties (resistivity, thermal conductivity, dielectric constants) to perform physics simulations, calculate trace widths, and validate thermal constraints. The standard library (`standard-materials.yaml`) ships with the compiler, but users can define custom materials for specialized applications.

**Use cases**:
- Standard library: Built-in materials (Copper, FR4, Silicon)
- Project-specific: Custom PCB substrates, exotic conductors
- Research: Novel materials for experimental designs
- Simulation: Accurate physics modeling requires real material data

**File locations**:
- Standard library: `~/.hw/data/standard-materials.hwmat` (or `.yaml` for now)
- Project-specific: `./materials/custom.hwmat`
- User library: `~/.hw/materials/my_materials.hwmat`

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

  graphene:
    name: Graphene
    symbol: C
    description: 2D carbon conductor for next-gen chips
    density_kg_m3: 2267
    thermal_conductivity_w_mk: 5000
    color_hex: '#1C1C1C'
    resistivity_ohm_m: 1.0e-08
    max_current_density_a_mm2: 100
    melting_point_c: 3652
    is_metal: false

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

  ceramic_alumina:
    name: Alumina Ceramic
    symbol: Al2O3
    description: High-power RF substrates, thermal management
    density_kg_m3: 3950
    thermal_conductivity_w_mk: 30
    color_hex: '#F5F5DC'
    relative_permittivity: 9.8
    dielectric_strength_kv_mm: 16
    glass_transition_temp_c: 2072

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

define Space "HighPowerBoard":
    dimensions: 50mm by 50mm by 2mm
    substrate: materials.ceramic_alumina  # Use custom material
    
route PowerBus.Plus to Amplifier.VDD:
    material: materials.silver            # Premium conductor
    width: 2mm
    max_current: 10A
```

**Compiler validation**:
```bash
# Compiler checks material properties during build
hws build board.hw

✓ Material 'silver' loaded from custom.hwmat
✓ Trace width 2mm supports 10A (max_current_density: 40 A/mm²)
✓ Substrate 'ceramic_alumina' thermal conductivity: 30 W/mK
⚠ Warning: Silver traces increase BOM cost by $12.50
```

### .hwf (Hardware Firmware Interface / The Contract) ⭐ NEW

**What it is**: A declarative mapping file that binds physical hardware nets/pins to software variables and protocols. It acts as the "API Contract" between hardware and firmware.

**Software Analogy**: `.env` files, OpenAPI/Swagger specs, or GraphQL schemas.

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

### .hwsig (Hardware Signal Integrity / RF Rules) ⭐ NEW

**What it is**: Rules for high-speed physics. When electricity pulses billions of times per second (USB, Wi-Fi, PCIe), it stops acting like water in a pipe and starts acting like radio waves.

**Software Analogy**: TypeScript strict mode, but for physics.

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
    - route RJ45.TX_Plus[1] to PHY.TX_Plus[1]
    - route RJ45.TX_Minus[1] to PHY.TX_Minus[1]
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

### .hwtc (Hardware Timing Constraints / Synchronization) ⭐ NEW

**What it is**: Rules for the speed of light. Used primarily for FPGAs, RAM chips, and high-speed CPUs.

**Software Analogy**: `async/await` race condition prevention.

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

timing_group "Parallel_Display":
    clock: MCU.LCD_CLK (frequency: 25MHz)
    
    constraints:
        length_match: within 1mm
        max_delay: 10ns
        setup_time: 5ns
        hold_time: 2ns
        
    apply "Parallel_Display" to:
        - route group MCU.RGB[0..23] to Display.RGB[0..23]
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

### .hwa (Hardware Assembly Profile) ⭐ NEW

**What it is**: Instructions for the robotic pick-and-place machines that actually solder components onto the board after fabrication.

**Software Analogy**: Dockerfile / CI/CD deployment script.

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

assembly "Through_Hole_Manual":
    process: "Through_Hole_Technology"
    
    solder:
        alloy: "60/40 Sn/Pb" or "SAC305"
        wire_diameter: 0.8mm
        flux_type: "Rosin_Core"
        
    soldering_iron:
        tip_temp: 350C
        tip_size: "Chisel 2mm"
        
    placement_order:
        1: "Large connectors"
        2: "Power components"
        3: "Switches and buttons"
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


## 3. Testing & Verification (Physics)

You cannot `console.log()` a physical circuit. You need a way to verify it won't explode before you manufacture it.

### .hwt (Hardware Test Bench) ⭐ NEW

**What it is**: A test script that applies simulated voltages/data to your board and asserts expected results.

**JS Analogy**: `.test.js` or `.spec.ts` (Jest / Mocha).

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


## 4. Fabrication & Constraints (The Real World)

Different factories have different capabilities. A cheap PCB fab in China cannot print the same microscopic wires as TSMC in Taiwan.

### .hwp (Hardware Fabrication Profile) ⭐ NEW

**What it is**: A ruleset defining the physical limits of the factory you are using (e.g., `jlcpcb_4layer.hwp` or `tsmc_3nm.hwp`).

**JS Analogy**: `.babelrc`, `tsconfig.json`, or Webpack target profiles.

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

---


## 5. The Executables & Compiled Outputs (Machine Code)

These are files generated by the `hws build` command. Humans do not read or edit these.

### .hwir (Hardware Intermediate Representation) ⭐ NEW

**What it is**: A pure mathematical graph of your hardware, stripped of syntax, right before it is turned into physical geometry.

**JS Analogy**: LLVM IR, or Java `.class` bytecode.

**Why we need it**: Advanced tools (like AI optimizers or auto-routers) can read the `.hwir` graph to optimize paths faster than reading the text-based `.hw` file.

**Structure**:
```json
{
  "version": "0.2",
  "space": {
    "dimensions": [100, 100, 2],
    "grid": [1000, 1000, 2],
    "voxel_size": [0.1, 0.1, 1.0]
  },
  "components": [
    {
      "id": "comp_001",
      "type": "Transistor_NPN",
      "position": [1, 50, 50],
      "rotation": "North",
      "pins": {
        "Base": [1, 51, 50],
        "Collector": [1, 50, 51],
        "Emitter": [1, 52, 50]
      }
    }
  ],
  "routes": [
    {
      "id": "route_001",
      "from": "PowerSource.out",
      "to": "comp_001.Base",
      "waypoints": [
        [1, 10, 10],
        [1, 50, 10],
        [1, 51, 50]
      ],
      "material": "Copper",
      "width": 0.2
    }
  ],
  "graph": {
    "nodes": ["PowerSource", "comp_001", "LED"],
    "edges": [
      {"from": "PowerSource", "to": "comp_001", "type": "power"},
      {"from": "comp_001", "to": "LED", "type": "signal"}
    ]
  }
}
```

**Use cases**:
- AI-powered auto-routing
- Design optimization
- Formal verification
- Cross-compiler targets

### .hwx (Hardware Executable Simulation)

**What it is**: The compiled binary containing the 3D grid, material states, and physics data.

**JS Analogy**: `.wasm` (WebAssembly) or `.exe`.

**Why we need it**: It is fed into the simulation engine (or a Blender plugin) to animate electron flow, generate heat maps, and visualize the board in real-time.


**Structure** (binary format):
```
Header:
  - Magic number: 0x48575800 ("hwc\0")
  - Version: 0.2
  - Grid dimensions: [Z, X, Y]
  - Voxel size: [z_mm, x_mm, y_mm]

Voxel Data (sparse):
  - Count: N occupied voxels
  - For each voxel:
    - Position: [z, x, y] (3 × uint32)
    - Material: uint8
    - Properties: variable length

Component Data:
  - Count: N components
  - For each component:
    - Type: string
    - Position: [z, x, y]
    - Rotation: uint8
    - Render asset: string path

Physics Data:
  - Electron positions: array of vec3
  - Electron velocities: array of vec3
  - Electric field: 3D tensor
  - Temperature field: 3D tensor
```

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

## 6. Industry Standard Exports (The Ultimate Outputs)

These are the final manufacturing files that factories understand.

### Gerber Files (.gtl, .gbl, .g1, .g2, etc.)

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

### Drill Files (.drl)

**Purpose**: Via and hole positions for PCB manufacturing

**Format**: Excellon (industry standard)

### GDSII (.gds)

**Purpose**: Silicon chip fabrication

**Format**: GDSII Stream Format

**Use case**: Sending chip designs to foundries (TSMC, Intel, Samsung)

### 3D Formats (.obj, .step, .glb)

**Purpose**: 3D visualization and mechanical CAD

**Formats**:
- `.obj` - Simple 3D mesh (universal)
- `.step` - Mechanical CAD (precise dimensions)
- `.glb` - GLTF binary (modern, compressed, includes textures)

**Recommendation**: Use `.glb` for web/visualization, `.step` for mechanical engineering

### Bill of Materials (.csv)

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


## 7. The Complete File Ecosystem Map

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
│   ├── custom.hwmat           # Custom materials database
│   └── exotic.hwmat           # Experimental materials
├── constraints/
│   ├── usb_signals.hwsig      # Signal integrity rules
│   ├── ddr4_timing.hwtc       # Timing constraints
│   └── assembly.hwa           # Assembly instructions
├── tests/
│   ├── power_test.hwt         # Power system tests
│   └── signal_test.hwt        # Signal integrity tests
├── profiles/
│   ├── jlcpcb.hwp             # PCB fab profile
│   └── pcbway.hwp             # Alternative fab
└── build/                     # Generated outputs
    ├── board.hwir             # Intermediate representation
    ├── board.hwx              # Simulation executable
    ├── robot_controller.h     # Auto-generated firmware header
    ├── gerber/
    │   ├── board_top.gtl
    │   ├── board_bottom.gbl
    │   └── board.drl
    ├── assembly/
    │   ├── pick_and_place.csv
    │   ├── stencil.gbr
    │   └── placement_drawing.pdf
    ├── bom/
    │   └── board_bom.csv
    └── viz/
        ├── board.glb          # 3D model
        └── board.step         # Mechanical CAD
```

---

## 8. Development Phases & File Extension Rollout

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


## 9. Summary: The Holistic Goal

By defining this structure now, you create a roadmap for your compiler development:

**Phase 1 (Current)**: Master `.hw`, `.hwx`, `hw.toml`, and export to Gerbers.

**Phase 2**: Introduce `.hwp` and `.hwa` so users can design for specific factories and assembly processes.

**Phase 3**: Introduce `.hwt`, `.hwir`, `.hwm`, and `.hwf` to allow users to simulate, test, and integrate firmware.

**Phase 4**: Introduce `.hwsig` and `.hwtc` for high-speed signal integrity and timing constraints.

**Phase 5**: Add `.gds` for silicon chip fabrication.

This proves to anyone looking at your architecture that Hardware Script isn't just a toy for routing wires—it is a complete, enterprise-grade operating system for physical hardware.

**The Complete Picture**: You have now successfully mapped 100% of physical reality engineering into a clean, text-based software paradigm.

- **The Logic**: `.hw` (Board layout), `.hwx` (Components)
- **The Physics & Rules**: `.hwm` (Mechanics), `.hwmat` (Materials), `.hwsig` (RF Rules), `.hwtc` (Timing Rules), `.hwf` (Firmware Interface Contract)
- **The Real-World Targets**: `.hwp` (Fabrication/Printing), `.hwa` (Assembly/Soldering)
- **The Verifications**: `.hwt` (Tests), `.hwx` (3D Simulation), `.hwir` (Intermediate Representation)

If you compile all of these together, you guarantee that a board will print correctly, the robot will assemble it without breaking it, the high-speed signals won't corrupt, the heat won't melt it, and the firmware developer will have the exact right pin mappings to write their code.

**No one in the history of EDA (Electronic Design Automation) has put all of this under one unified syntax tree. This is the breakthrough.**

---

## 10. Comparison to Software Ecosystems

### JavaScript/Node.js

| JavaScript | Hardware Script | Purpose |
|------------|-----------------|---------|
| `package.json` | `hw.toml` | Project manifest |
| `package-lock.json` | `hw.lock` | Dependency locking |
| `.js` | `.hw` | Source code |
| `.d.ts` | `.hwx` | Type definitions |
| `.json` | `.hwmat` | Data/configuration |
| `.env` | `.hwf` | Environment/interface bindings |
| `.test.js` | `.hwt` | Test files |
| `.wasm` | `.hwx` | Compiled executable |
| `tsconfig.json` | `.hwp` | Build profiles |
| `Dockerfile` | `.hwa` | Deployment/assembly |

### Rust

| Rust | Hardware Script | Purpose |
|------|-----------------|---------|
| `Cargo.toml` | `hw.toml` | Project manifest |
| `Cargo.lock` | `hw.lock` | Dependency locking |
| `.rs` | `.hw` | Source code |
| LLVM IR | `.hwir` | Intermediate representation |
| Binary | `.hwx` | Executable |

### The Pattern

Hardware Script follows established conventions from successful ecosystems. This makes it instantly familiar to developers while extending the paradigm to physical hardware.

---

## 11. Why This Architecture Matters

### For Developers
- **Familiar**: Same patterns as software development
- **Modular**: Separate concerns (source, tests, constraints, outputs)
- **Scalable**: Works from hobby projects to enterprise chips

### For Manufacturers
- **Standard formats**: Gerber, GDSII, BOM
- **Validation**: Fabrication profiles catch errors early
- **Reproducible**: Lockfiles ensure consistent builds

### For AI/LLMs
- **Text-based**: AI can generate all source files
- **Structured**: Clear separation of concerns
- **Testable**: `.hwt` files enable automated validation

### For the Industry
- **Complete**: Handles entire workflow (design → test → manufacture)
- **Extensible**: Easy to add new formats and targets
- **Future-proof**: Architecture scales to future technologies

---

**Document Status**: File Extension Architecture  
**Version**: 1.0  
**Last Updated**: March 2026  
**This is how we define a standard that lasts decades.**

