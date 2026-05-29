# Book 2: The Language Specification

**Hardware Script v0.1.4**  
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

# 3. Atomic Parts (Component Definition with Parameters)
define component "Resistor_0805" (val: Measurement, tol: Measurement):
    pins: Pin1, Pin2
    layout:
        shape: Rectangle(2.0mm, 1.25mm, 0.5mm)
    electrical:
        resistance: val      # Direct parameter reference
        tolerance: tol

# 4. Logical Schematic (Module Definition - NO Coordinates)
define module "LED_Driver":
    pins: VCC, GND
    add Resistor_0805 (val: 100Ω, tol: 5%) named R1
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

### File Extension

Hardware Script uses one file extension: `.hw`

### Coordinate System

**Order**: `[X, Y, Z]` (alphabetical)
- **X** = Column (left to right)
- **Y** = Row (direction depends on XY origin)
- **Z** = Layer (direction depends on Z origin)

**Origin** (configurable per Space using `XY by Z` syntax):
- `tl by t` (default): Top-Left XY, Top-Down Z
- `bl by t`: Bottom-Left XY, Top-Down Z

**Named coordinates** (explicit labels, any order):
```hw
add LED named Light at [x:10, y:15, z:2]
```

**Component Anchor Point: Top-Left Corner**

All component positions specify the **top-left-front corner** of the component's bounding box:
- When you place a component at `[x:100, y:200, z:2]`, the component's top-left corner is at that exact position
- The component extends positively: +X (right), +Y (down with `tl` origin), +Z (into board)
- Pin positions in the `layout:` block are offsets from this top-left corner

**Example:**
```hw
define component "NMOS":
    layout:
        shape: Rectangle(3mm, 2.5mm, 1mm)
        pin_positions:
            Gate at [0mm, 0mm]      # Top-left corner
            Drain at [1.5mm, 0mm]   # 1.5mm right of top-left
            Source at [3mm, 0mm]    # 3mm right (right edge)

# Component placed at grid [150, 200, 2]
add NMOS named Q1 at [x:150, y:200, z:2]
# Q1's top-left corner is at grid [150, 200, 2]
# Q1.Gate is at [150, 200, 2] + [0, 0, 0] = [150, 200, 2]
# Q1.Drain is at [150, 200, 2] + [15, 0, 0] = [165, 200, 2] (in 0.1mm grid)
```

This **Top-Left Anchor** system matches PCB CAD industry standards and makes pin position calculations trivial: `absolute_pin = component_anchor + pin_offset`.

### Tri-Fold Case Sensitivity

1. **Software Domain (Strictly Lowercase)**: Keywords (`define`, `space`, `material`, `add`, `route`) must be lowercase
2. **Physics Domain (SI Standard)**: Units must match SI casing (`mV`, `kΩ`, `MHz`)
3. **User Domain (Case-Sensitive)**: Identifiers (`ESP32`, `MainPower`) are exact-match

---

## Language Overview

Hardware Script is a declarative, text-based hardware description language. Everything is defined in `.hw` files using `define` blocks. 

The language is built on **Logical/Physical Duality**:
1. You define raw atoms and rules (`material`, `profile`).
2. You define pure electrical intent without coordinates (`module`).
3. You instantiate and physically place them (`space`).
4. You validate the physics programmatically (`test`).

You can write an entire custom motherboard—including materials, manufacturing rules, mechanical constraints, components, logical schematics, and physical routing—inside a single `.hw` file. Best practice is to organize into modular `.hw` files (e.g., `materials.hw`, `profiles.hw`, `modules.hw`, `main.hw`) and `import` them.

### Coordinate System Abstraction

Users choose their preferred origin using unified `XY by Z` syntax (e.g., `origin: bl by t`). The compiler normalizes all coordinates internally to absolute space, then export layers translate to manufacturing standards.

### Native SI Unit Parsing

Hardware Script natively understands units everywhere. Write `254µm` instead of `min_width_nm: 254000`.

**Valid units**:
```hw
# Distance
50mm, 254µm, 5cm

# Electrical
12V, 500mV, 2A, 100mA, 4.7kΩ, 100nF, 2.4GHz
```

Hardware Script rejects IEC 60062 notation like `4K7`. Write `4.7kΩ` or `4.7kOhm` for readability.

---

## Top-Level Definitions

A `.hw` file contains one or more `define` blocks. The compiler registers all definitions into a Symbol Table (Pass 1), then flattens modules and unrolls comptime logic during spatial assembly (Pass 2).

### 1. Material Definition

```hw
define material "Copper":
    category: conductor
    symbol: "Cu"
    description: "Universal PCB trace material"
    
    properties:
        # Blank lines and comments are allowed here
        density: 8960kg/m3
        thermal_conductivity: 401W/mK
        resistivity: 1.68e-8Ω/m
        max_current_density: 35A/mm2
        melting_point: 1085C
        color: "#B87333"
```

**Note**: Units must be attached to numbers with NO space (e.g., `8960kg/m3` not `8960 kg/m3`). This follows CSS and Rust conventions for unambiguous parsing.

**Blank Lines**: Blank lines are allowed anywhere in material definitions, including after `properties:` blocks. The parser automatically skips them.

### 2. Profile Definition

```hw
define profile "HighVoltage":
    description: "IPC-2221 compliant constraints for >150V"
    
    trace:
        min_width: 254µm
        min_spacing: 508µm
        
    via:
        min_diameter: 508µm
        min_annular_ring: 254µm
        
    clearance:
        high_voltage: 8.0mm
        safety_factor: 3.0
        low_voltage_threshold: 50V
        medium_voltage_threshold: 150V
        
    thermal:
        ambient_temp: 25C
        max_operating_temp: 85C
        max_temp_rise: 20C
        clustering_threshold: 5mm
        
    manufacturing:
        copper_thickness: 70µm
        ipc2221_k_external: 0.048
        ipc2221_k_internal: 0.024
        
    layer:
        min_thickness: 70µm
        allowed_conductors: [Copper, Silver]
        allowed_dielectrics: [FR4, Air]
```

### 3. Component Definition

Components are atomic, indivisible parts. They support parametric generics for reusability.

```hw
# Parametric component with keyword arguments
define component "Resistor_0805" (val: Measurement, tol: Measurement):
    metadata:
        manufacturer: "Yageo"
        package: "0805"
        
    pins:
        Pin1
        Pin2
        
    layout:
        shape: Rectangle(2.0mm, 1.25mm, 0.5mm)
        pin_positions:
            Pin1 at [0mm, 0.625mm]      # Centered vertically
            Pin2 at [2.0mm, 0.625mm]    # 2mm right, centered
            
    electrical:
        resistance: val        # Direct parameter reference (no {} needed)
        tolerance: tol
        max_voltage: 150V
        max_power: 0.125W
        
    render:
        type: procedural
        shape: smd_passive
        body_color: "#111111"
        endcap_color: "#C0C0C0"

# Usage with keyword arguments (self-documenting)
add Resistor_0805 (val: 10kΩ, tol: 1%) named R1 at [x:10, y:10, z:1]

# Positional arguments allowed for single-parameter components
add Capacitor_0805 (10µF, 16V) named C1 at [x:20, y:10, z:1]
```

**Standard Library Components:**

The standard library (`hwc/stdlib/components.hw`) includes 10 parametric components:

- `Resistor_0805`, `Resistor_0603`, `Resistor_1206` (val, tol)
- `Capacitor_0805`, `Capacitor_0603`, `Capacitor_1206` (cap, voltage)
- `LED_0805`, `LED_0603` (color)
- `Resistor_TH`, `LED_5mm` (through-hole)

Import: `import Resistor_0805 from "@std/components"` (parser support, resolution pending)

### 4. Mechanical & Signal Constraints

```hw
define mechanical "RobotEnclosure":
    dimensions: 150mm by 100mm by 50mm
    
    mounting_holes:
        - at [x:5, y:5] diameter 3mm
        - at [x:145, y:95] diameter 3mm
        
    keepout:
        - region [x:20, y:20] to [x:60, y:60] height 15mm

define signal_group "USB_Data":
    type: differential_pair
    target_impedance: 90Ω
    max_length_mismatch: 0.15mm
```

### 5. Interface Definition

```hw
define interface "RobotController":
    target: "ESP32_WROOM_32"
    
    bindings:
        Motor_PWM = DriverIC.Pin_4
        Status_LED = LED1.Anode
        Temp_Sensor = Thermistor.Out
        
    protocols:
        I2C_Bus_1:
            SDA: MCU.GPIO21
            SCL: MCU.GPIO22
            speed: 400kHz
```

### 6. Test Bench Definition

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
        Regulator.temperature < 100 C
        PowerSource.current < 2A
```

---

## Generative Architecture (Modules & Spaces)

To scale from simple LEDs to 64-bit GPUs, Hardware Script utilizes **Logical/Physical Duality** with **Fractal Encapsulation**.

### 7. Module Definition (The Logical Schematic)

A `module` represents purely electrical connections. **It cannot contain absolute coordinates (`at [x,y,z]`).**

```hw
define module "64Bit_ALU":
    pins:
        Bus_A[64]   # Array Pin (Bus)
        Bus_B[64]
        Bus_Out[64]
        CarryIn
        CarryOut
        
    # Comptime Parametric Generation
    for i in 0..63:
        add SingleBit_ALU named Bit[i]
        
        route Bus_A[i] to Bit[i].In_A
        route Bus_B[i] to Bit[i].In_B
        route Bit[i].Out to Bus_Out[i]
        
        # Comptime Conditional Routing
        if i == 0:
            route CarryIn to Bit[i].CarryIn
        else:
            route Bit[i-1].CarryOut to Bit[i].CarryIn
    
    # Connect final carry out
    route Bit[63].CarryOut to CarryOut
```

**Key Rules**:
- No `at [x, y, z]` coordinates allowed in modules
- Only logical connections (`route A to B`)
- Can contain `for` loops and `if` conditionals (evaluated at compile time)
- Can instantiate other modules (fractal nesting)
- Array pins (`Bus[64]`) for hardware buses

### 8. Space Definition (The Physical Layout)

A `space` is a physical boundary. Here, you instantiate components, modules, or even other spaces, and map them to absolute coordinates.

```hw
define space "GPU_Motherboard":
    dimensions: 300mm by 150mm by 2mm
    grid: 3000 by 1500 by 2
    origin: tl by t
    profile: HighDensity
    
    add substrate(FR4) spanning [1,1,1] to [3000, 1500, 2]
    
    # Place a logical module
    add 64Bit_ALU named CoreLogic
    
    # Map the logical module to physical reality
    layout CoreLogic:
        for i in 0..63:
            Bit[i] at [x: 50 + (i*2), y: 50, z: 1]
    
    # Comptime loop for physical placement
    for i in 0..7:
        add DDR6_RAM named VRAM[i] at [x: 50 + (i*20), y: 20, z:1]
        
        # Route hardware bus (all 32 bits in one line)
        route CoreLogic.Bus_Out to VRAM[i].DataBus
```

**The `layout:` Block**:
Maps the internal sub-components of a `module` to the local coordinate system of the current `space`. This is how logical intent becomes physical reality.

### Comptime Evaluation

`for` loops and `if` conditionals execute at **compile time**, not runtime:

```hw
# This generates 64 separate component placements before routing
for i in 0..63:
    add SingleBit_ALU named Bit[i]
```

**Range Syntax**:
- `0..63` is **inclusive** (generates 64 items: 0 to 63)
- Follows Ruby's range semantics for hardware engineers who think in bit indices

**Conditional Syntax**:
```hw
if i == 0:
    route GND to Bit[i].CarryIn
else:
    route Bit[i-1].CarryOut to Bit[i].CarryIn
```

### Polygons & Custom Shapes

For RF antennas, copper pours, and custom geometries:

```hw
define space "RF_Board":
    dimensions: 100mm by 100mm by 2mm
    grid: 1000 by 1000 by 2
    origin: tl by t
    
    # Copper pour (floods entire layer, auto-connects to net)
    add pour(Copper) named GND_Plane on z:2:
        boundary: [x:0, y:0] to [x:100mm, y:100mm]
        net: GND
        thermal_relief: true
        
    # Custom polygon (explicit point mapping for RF antenna)
    add polygon(Copper) named WiFi_Antenna at [x:10, y:10, z:1]:
        points:
            - [0, 0]
            - [15mm, 0]
            - [15mm, 2mm]
            - [5mm, 2mm]
            - [5mm, 10mm]
```

**Key Differences**:
- `pour` uses `on z:2` (floods entire layer)
- `polygon` uses `at [x, y, z]` (positioned shape)
- Points in `polygon` are relative to the origin position

---

## Space Assembly

The `define space` block is where physical placement and routing happen. It can instantiate both atomic components and logical modules.

```hw
define space "SpaceName":
    dimensions: <width>mm by <height>mm by <depth>mm
    grid: <x_cells> by <y_cells> by <z_cells>
    origin: <xy> by <z>
    profile: <ProfileName>
```

### 1. Substrate Spanning

```hw
add substrate(FR4) spanning [1, 1, 1] to [100, 100, 2]
```

### 2. Component Placement

```hw
# Atomic component with parameters
add Resistor_0805 (val: 10kΩ, tol: 1%) named PullUp at [x:20, y:15, z:1] rotated 90

# Logical module (no coordinates yet)
add LED_Driver named Driver1

# Map module's internal components to physical space
layout Driver1:
    R1 at [x:10, y:10, z:1]
    Light at [x:15, y:10, z:1]
```

### 3. Routing

**Automatic Routing** (clean syntax - recommended):
```hw
route MainPower.Plus to PullUp.Pin1
route R1.Pin2 to LED.Anode
route LED.Cathode to GND
```

**Manual Routing** (explicit path control):
```hw
route MainPower.Plus to PullUp.Pin1:
    path:
        - [x:10, y:10, z:1]
        - [x:20, y:10, z:1]
        - [x:20, y:15, z:1]
```

The compiler uses the Automatic Manhattan Router when no `path:` block is provided. Use manual routing only when you need precise control over trace paths (e.g., impedance-controlled traces, avoiding sensitive areas).

---

## Import System

```hw
# Local files
import "materials.hw"
import "profiles.hw"
import "components.hw"

# Standard library & packages
import Copper from "@standard/materials"
import Resistor_0805 from "@standard/components"
import ESP32_Module from "@wireless/esp32"
```

### Exposing Pins

```hw
define space "Regulator_5V":
    # ... components and routing ...
    
    expose Regulator.In as VIN
    expose Regulator.Out as VOUT
    expose Regulator.GND as GND
```

---

## Formal Grammar (EBNF)

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
for_stmt = "for" identifier "in" number ".." number ":" INDENT { statement } DEDENT
if_stmt = "if" condition ":" INDENT { statement } DEDENT [ "else" ":" INDENT { statement } DEDENT ]

# Module (Logic Only - No Coordinates)
module_def = "define" "module" STRING [ "(" parameters ")" ] ":"
             [ "pins" ":" INDENT { pin_decl } DEDENT ]
             { component_placement | route_def | for_stmt | if_stmt }

# Space (Physical Layout)
space_def = "define" "space" STRING ":"
            "dimensions" ":" measure "by" measure "by" measure
            "grid" ":" number "by" number "by" number
            [ "origin" ":" origin_point ]
            [ "profile" ":" identifier ]
            { space_statement }

space_statement = 
    | substrate_def 
    | component_placement 
    | route_def 
    | layout_block
    | expose_def
    | pour_def
    | polygon_def
    | for_stmt
    | if_stmt

# Component Definition with Parameters
component_def = "define" "component" STRING [ "(" parameters ")" ] ":"
                INDENT component_body DEDENT

parameters = parameter { "," parameter }
parameter = identifier ":" type_name

layout_block = "layout" identifier ":" INDENT { identifier "at" coordinate | for_stmt } DEDENT

pour_def = "add" "pour" "(" identifier ")" "named" identifier "on" coord_pair ":"
           INDENT pour_properties DEDENT

polygon_def = "add" "polygon" "(" identifier ")" "named" identifier "at" coordinate ":"
              INDENT "points" ":" INDENT { "-" coordinate } DEDENT DEDENT

pin_decl = identifier [ "[" number "]" ]
pin_ref = identifier [ "[" (number | identifier) "]" ] [ "." identifier [ "[" (number | identifier) "]" ] ]

coordinate = "[" number "," number "," number "]" 
           | "[" coord_pair "," coord_pair "," coord_pair "]"
coord_pair = ( "x" | "y" | "z" ) ":" (number | expression)

condition = expression comparison_op expression
comparison_op = "==" | "!=" | "<" | ">" | "<=" | ">="
expression = identifier | number | identifier "[" (number | identifier) "]"

# (Other primitives remain standard...)
```

---

## Compilation Targets

Hardware Script uses explicit abstraction blocks inside `define component` to separate different levels of hardware description. The same `.hw` codebase compiles to different targets based on the `--target` CLI flag.

| Target | Reads | Ignores | Primary Output |
|--------|-------|---------|----------------|
| `pcb` (Default) | `pins:`, `layout:` | `behavior:`, `electrical:` | Gerber files, Drill files |
| `fpga` | `pins:`, `behavior:` | `layout:`, `render:` | Verilog/VHDL |
| `asic` | `pins:`, `behavior:`, `layout:` | `render:` | GDSII layout files |
| `spice` | `pins:`, `electrical:` | `layout:`, `render:` | SPICE netlist |
| `viz` | `pins:`, `layout:`, `render:` | `behavior:`, `electrical:` | OBJ/GLB 3D models, Blender scripts |

---

## Error Handling

Hardware Script provides actionable error messages with memorable 3-character codes using `miette` diagnostics.

**Error Categories**:
- **S-codes**: Syntax errors (invalid keywords, wrong formatting, case violations)
- **C-codes**: Compiler errors (undefined references, logical flaws)
- **R-codes**: Routing errors (collisions, out-of-bounds)
- **P-codes**: Physics errors (dielectric breakdown, overheating)
- **M-codes**: Manufacturing errors (factory constraint violations)

**Example P16 (Dielectric Breakdown)**:
```
Error[P16]: Dielectric Breakdown Risk
  ╭─[main.hw:42:1]
42 │  route Power_120V.Out to Relay.In:
   ·        ──────┬───────
   ·              ╰── High voltage net (120V)
45 │      - [x:10, y:50, z:2]
   ·        ──────┬──────
   ·              ╰── Approaches within 0.02mm of MCU_5V net here
   ╰────
  help: The voltage difference is 115V. Through Air, this requires a 
        minimum clearance of 0.08mm to prevent arcing.
  hint: Move the waypoint to [x:10, y:50, z:4] or change substrate material to FR4.
```

---

## Best Practices

Professional Hardware Script projects use modular structure with Logical/Physical separation:

```
my_project/
├── materials.hw       # Custom alloys and substrates
├── profiles.hw        # Factory rules
├── components.hw      # Atomic parts with parameters
├── modules.hw         # Logical schematics (no coordinates)
├── constraints.hw     # Mechanical boundaries
└── main.hw            # Physical assembly that imports everything
```

**Inside modules.hw** (Pure Logic):
```hw
define module "PowerRegulator":
    pins: VIN, VOUT, GND
    # ... pure electrical connections, no coordinates ...
```

**Inside main.hw** (Physical Reality):
```hw
import "modules.hw"

define space "Motherboard":
    add PowerRegulator named Reg1
    
    layout Reg1:
        # Map module's components to physical space
        IC1 at [x:50, y:50, z:1]
        C1 at [x:45, y:50, z:1]
```

By keeping the ecosystem unified under `.hw` with clear Logical/Physical separation, syntax highlighting, Language Servers (LSP), and LLM context windows perform flawlessly without context switching.
