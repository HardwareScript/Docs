


Here is the **fully updated, holistic Language Specification (Book 2)**. 

This version integrates the exact paradigms we just finalized—**Logical Modules (`define module`)**, **Fractal Spaces**, **Comptime Loops (`for` / `if`)**, and **Buses (`Array Pins`)**—while preserving all the existing constraints, interfaces, tests, and atomic components you've built.

This is the definitive "Unity of Hardware" specification.

---

# Book 2: The Language Specification

**Hardware Script v1.0-draft**  
**Target Audience**: Hardware engineers and LLMs writing .hw code  
**Last Updated**: March 2026

---

## Quick Reference

### Basic Syntax & The Fractal Workflow

```hw
# 1. Physics (Material Definition)
define material "Copper":
    category: conductor
    properties:
        resistivity: 1.68e-8 Ω·m

# 2. Rules (Profile Definition)
define profile "HighDensity":
    trace:
        min_width: 254µm

# 3. Atomic Parts (Component Definition)
define component "Resistor_0805":
    pins: Pin1, Pin2
    layout:
        shape: Rectangle(2.0mm, 1.25mm, 0.5mm)

# 4. Logical Schematic (Module Definition - NO Coordinates)
define module "LED_Driver":
    pins: VCC, GND
    add Resistor_0805 (100Ω) named R1
    add LED_0805 ("Red") named Light
    route VCC to R1.Pin1
    route R1.Pin2 to Light.Anode
    route Light.Cathode to GND

# 5. Physical Assembly (Space Definition - Absolute Coordinates)
define space "MainBoard":
    dimensions: 50mm by 50mm by 2mm
    grid: 500 by 500 by 2
    origin: tl by t
    profile: HighDensity
    
    # Instantiate the logical module
    add LED_Driver named Driver1
    
    # Map the module's logic to physical space
    layout Driver1:
        R1 at [x:10, y:10, z:1]
        Light at [x:15, y:10, z:1]
```

---

## Language Overview

Hardware Script is a declarative, text-based hardware description language. Everything is defined in `.hw` files using `define` blocks. 

The language is built on **Logical/Physical Duality**:
1. You define raw atoms and rules (`material`, `profile`).
2. You define pure electrical intent without coordinates (`module`).
3. You instantiate and physically place them (`space`).
4. You validate the physics programmatically (`test`).

### Native SI Unit Parsing
Hardware Script natively understands units. Write `254µm` instead of `min_width_nm: 254000`. No spaces are allowed between numbers and units.
* **Distance**: `mm`, `cm`, `µm`, `nm`
* **Electrical**: `V`, `mV`, `A`, `mA`, `Ω`, `kΩ`, `F`, `nF`, `Hz`, `GHz`

### Tri-Fold Case Sensitivity
1. **Software Domain (Lowercase)**: Keywords (`define`, `space`, `add`, `for`, `layout`) must be lowercase.
2. **Physics Domain (SI Standard)**: Units must match SI casing (`mV`, `kΩ`, `MHz`).
3. **User Domain (Exact Match)**: Identifiers (`ESP32`, `MainPower`) are case-sensitive.

---

## Part 1: Top-Level Definitions

### 1. Material Definition
Defines atomic physical, electrical, and thermal properties used by the simulation engine.

```hw
define material "Copper":
    category: conductor
    symbol: "Cu"
    properties:
        density: 8960 kg/m³
        thermal_conductivity: 401 W/mK
        resistivity: 1.68e-8 Ω·m
        max_current_density: 35 A/mm²
```

### 2. Profile Definition (Design Rules)
Defines factory capabilities and safety limits.

```hw
define profile "HighVoltage":
    trace:
        min_width: 254µm
        min_spacing: 508µm
    via:
        min_diameter: 508µm
        min_annular_ring: 254µm
    clearance:
        high_voltage: 8.0mm
        safety_factor: 3.0
```

### 3. Component Definition (Atomic Parts)
Defines the lowest-level, indivisible parts (like an 0805 resistor or an IC chip).

```hw
define component "Resistor_0805":
    metadata:
        manufacturer: "Yageo"
        package: "0805"
    pins:
        Pin1
        Pin2
    layout:
        shape: Rectangle(2.0mm, 1.25mm, 0.5mm)
        pin_positions:
            Pin1 at[0, 0]
            Pin2 at [2.0mm, 0]
    electrical:
        max_voltage: 150V
        max_power: 0.125W
```

### 4. Mechanical & Signal Constraints
Defines 3D boundaries, keep-out zones, and high-speed differential pair rules.

```hw
define mechanical "RobotEnclosure":
    dimensions: 150mm by 100mm by 50mm
    mounting_holes:
        - at[x:5, y:5, z:1] diameter 3mm
    keepout:
        - region[x:20, y:20, z:1] to[x:60, y:60, z:1] height 15mm

define signal_group "USB_Data":
    type: differential_pair
    target_impedance: 90Ω
    max_length_mismatch: 0.15mm
```

### 5. Interface Definition (Firmware Contract)
Binds physical hardware nets to software variables, allowing auto-generation of C/Rust header files.

```hw
define interface "RobotController":
    target: "ESP32_WROOM_32"
    bindings:
        Motor_PWM = DriverIC.Pin_4
        Status_LED = LED1.Anode
    protocols:
        I2C_Bus_1:
            SDA: MCU.GPIO21
            SCL: MCU.GPIO22
            speed: 400kHz
```

### 6. Test Bench Definition (Simulation)
Automated scripts for in-memory SPICE and physics validation before manufacturing.

```hw
define test "Short Circuit Protection":
    setup:
        apply 12V to PowerSource.VIN
        apply 0V to PowerSource.GND
    execute:
        short Regulator.VOUT to GND
        wait 1ms
    assert:
        Regulator.VOUT < 0.5V
        Regulator.temperature < 100C
        PowerSource.current < 2A
```

---

## Part 2: Generative Architecture (Modules & Spaces)

To scale from simple LEDs to 64-bit GPUs, Hardware Script utilizes **Fractal Encapsulation**.

### 7. Module Definition (The Logical Schematic)
A `module` represents purely electrical connections. **It cannot contain absolute coordinates (`at [x,y,z]`).**

```hw
define module "64Bit_ALU":
    pins:
        Bus_A[64]   # Array Pin (Bus)
        Bus_B[64]
        Bus_Out[64]
        
    # Comptime Parametric Generation
    for i in 0..63:
        add SingleBit_ALU named Bit[i]
        
        route Bus_A[i] to Bit[i].In_A
        route Bus_B[i] to Bit[i].In_B
        route Bit[i].Out to Bus_Out[i]
        
        # Conditional Routing
        if i > 0:
            route Bit[i-1].CarryOut to Bit[i].CarryIn
```

### 8. Space Definition (The Physical Layout)
A `space` is a physical boundary. Here, you instantiate components, modules, or even other spaces, and map them to absolute coordinates.

```hw
define space "GPU_Motherboard":
    dimensions: 300mm by 150mm by 2mm
    grid: 3000 by 1500 by 2
    origin: tl by t
    profile: HighDensity
    
    add substrate(FR4) spanning [1,1,1] to[3000, 1500, 2]
    
    # Place a logical module
    add 64Bit_ALU named CoreLogic
    
    # Map the logical module to physical reality
    layout CoreLogic:
        for i in 0..63:
            Bit[i] at[x: 50 + (i*2), y: 50, z: 1]
```

*(Note: The `layout:` block maps the internal sub-components of a `module` to the local coordinate system of the current `space`.)*

### Routing
If `path:` is omitted, the compiler uses the **Auto-Router**. If provided, the compiler forces deterministic Manhattan routing.

```hw
route Power.Plus to R1.Pin1:
    path:
        - [x:10, y:10, z:1]
        -[x:20, y:10, z:1]
        -[x:20, y:15, z:1]
```

---

## Part 3: Formal Grammar (EBNF)

*Updated to include Generative Hardware (Loops, Conditions, Buses).*

```ebnf
program = { import_stmt } { definition_block }

import_stmt = "import" ( string | identifier "from" string )

definition_block = 
    | space_def
    | module_def
    | component_def
    | material_def
    | profile_def
    | mechanical_def
    | interface_def
    | test_def

# Generative Control Flow
for_stmt = "for" identifier "in" number ".." number ":" INDENT { space_statement | route_def } DEDENT
if_stmt = "if" condition ":" INDENT { space_statement | route_def } DEDENT

# Module (Logic Only)
module_def = "define" "module" STRING ":"
             [ "pins" ":" INDENT { pin_decl } DEDENT ]
             { component_placement | route_def | for_stmt | if_stmt }

# Space (Physical Layout)
space_def = "define" "space" STRING ":"
            "dimensions" ":" measure "by" measure "by" measure
            "grid" ":" number "by" number "by" number[ "origin" ":" origin_point ]
            [ "profile" ":" identifier ]
            { space_statement }

space_statement = 
    | substrate_def 
    | component_placement 
    | route_def 
    | layout_block
    | expose_def
    | for_stmt
    | if_stmt

layout_block = "layout" identifier ":" INDENT { identifier "at" coordinate | for_stmt } DEDENT

pin_decl = identifier [ "[" number "]" ]
pin_ref = identifier [ "[" (number | identifier) "]" ] [ "." identifier [ "[" (number | identifier) "]" ] ]

coordinate = "[" number "," number "," number "]" 
           | "[" coord_pair "," coord_pair "," coord_pair "]"
coord_pair = ( "x" | "y" | "z" ) ":" (number | expression)

# (Other primitives remain standard...)
```

---

## Part 4: The Build System (`hwc`)

Hardware Script compiles to multiple targets based on the `--target` CLI flag, allowing you to use the exact same `.hw` codebase for PCB printing, Silicon lithography, or 3D rendering.

| Command | Target Output | Action |
|---------|---------------|--------|
| `hwc test` | Terminal | Runs all `define test` blocks via internal SPICE engine. Required before building. |
| `hwc serve` | WebGL Viewer | Compiles the `.hwx` binary and streams live 3D hot-reloading to VS Code / Browser. |
| `hwc build --target pcb` | Gerber / Drill | Outputs flat, factory-ready files for standard PCB manufacturing. |
| `hwc build --target silicon` | GDSII | Outputs layout files for TSMC / Intel lithography. |
| `hwc build --target viz` | Python / GLB | Outputs Blender scripting files for photorealistic marketing renders. |

---

## Project Organization Best Practices

Professional Hardware Script projects act like modern software packages:

```text
my_project/
├── hw.toml            # Project manifest & dependencies
├── hw.lock            # Cryptographic lockfile for geometric determinism
├── materials.hw       # Custom alloys and substrates
├── profiles.hw        # Factory rules
├── components.hw      # Custom IC definitions (Atomic)
├── logic.hw           # define module (Schematics & Buses)
├── constraints.hw     # define mechanical & define signal_group
├── interface.hw       # define interface (Firmware headers)
├── tests.hw           # define test (SPICE physics simulation)
└── main.hw            # define space (The Motherboard Assembly)
```