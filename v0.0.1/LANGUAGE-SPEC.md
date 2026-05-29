# Hardware Script (.hw) Language Specification

**Version**: 0.2 (Draft)  
**Paradigm**: Software-Defined Bare-Metal Hardware  
**Document Type**: Language Syntax and Grammar Reference

---

## 1. Language Philosophy

### 1.1 Design Principles

Hardware Script (.hw) is designed with the following principles:

1. **Human Readability**: English-like syntax accessible to non-electrical engineers
2. **LLM Optimization**: Declarative structure optimized for AI generation and parsing
3. **Minimal Syntax Overhead**: Indentation-based, avoiding brackets, semicolons, and complex memory management
4. **Declarative Semantics**: Focus on "what" rather than "how"

### 1.2 Structural Rules

To eliminate friction and allow LLMs native fluency, .hw enforces three strict structural rules:

1. **Indentation-Based**: No brackets `{}` or semicolons `;`. Scope is defined purely by spaces/tabs.
2. **Colon Properties**: Physical parameters are declared with a colon `:` (e.g., `dimensions: 10mm by 10mm`)
3. **Comments**: Anything following a `#` is ignored by the Synthesizer

---

## 2. Reserved Keywords

### 2.1 Actions (Verbs)

- **define**: Creates a new blueprint from scratch (Material, Component, Space)
- **add**: Physically instantiates a component onto the workspace
- **route**: Manually defines waypoint-based copper routing
- **import**: Pulls a component library into the file

### 2.2 Structure (Prepositions)

- **from**: Targets a library (e.g., `import X from Y`)
- **named**: Assigns a local reference variable
- **to**: Targets a routing destination
- **at**: Specifies 3D coordinate placement
- **spanning**: Defines a volume range in the grid
- **rotated**: Specifies component orientation

### 2.3 Property Keywords

- **dimensions**: Physical size specification
- **grid**: Mathematical tensor subdivision
- **layers**: Z-axis depth specification
- **material**: Material assignment
- **shape**: Geometric definition
- **sockets**: Connection point definitions
- **pins**: Component connection points
- **body**: Component keep-out zone
- **path**: Waypoint routing definition
- **all**: Shortcut for entire Space dimensions (spanning all)
- **expose**: Make internal pins accessible externally (for modules/libraries)

### 2.4 Layer Keywords

- **layer**: Specific Z-axis layer reference
- **top**: Alias for Z=1 (component layer)
- **bottom**: Alias for Z=max (component layer)
- **inner**: Reference to middle layers (Z=2 to Z=max-1)

### 2.5 Measurement Units

- **Distance**: `mm`, `cm`, `m`, `inches`
- **Electrical**: `V`, `A`, `mA`, `Ohms`, `F`
- **Temperature**: `C`, `F`, `K`
- **Angle**: `degrees`, `radians`

---

## 3. The 3D Tensor Coordinate System

### 3.1 Coordinate Format: [Z, X, Y]

All placement, movement, and routing within a Space uses a strict, AI-native 3D Array format: **[Z, X, Y]**

- **Z (Depth/Layer)**: The vertical cross-section of the space (layer index)
  - `1` = Top Layer
  - `2` = Inner Layer (for multi-layer boards)
  - `3` = Bottom Layer
- **X (Horizontal)**: The grid column
- **Y (Vertical)**: The grid row

Example: `[1, 5, 10]` targets Layer 1 (Top), Column 5, Row 10

### 3.2 Coordinate Ranges

For defining volumes or spans:

```hw
spanning [1, 1, 1] to [2, 100, 100]
```

This defines a rectangular prism from coordinate [1,1,1] to [2,100,100].

---

## 4. Space Definition

### 4.1 Core Concept

Every .hw file begins by declaring a **Space**. A Space dictates the absolute laws of physics, real-world constraints, and structural resolution for the project.

### 4.2 Syntax

```hw
define Space "SpaceName":
    dimensions: <width> by <height> by <depth>
    grid: <x_cells> by <y_cells> by <z_cells>
```

### 4.3 Voxel Calculation

The Synthesizer divides **Dimensions** by **Grid** to determine the exact physical size of the "Smallest Possible Unit" (1 Voxel).

```
Voxel Size = Dimensions / Grid

Example:
  Dimensions: 10mm by 10mm by 2mm
  Grid: 100 by 100 by 2
  
  Voxel Width:  10mm / 100 = 0.1mm
  Voxel Height: 10mm / 100 = 0.1mm
  Voxel Depth:  2mm / 2 = 1mm
```

### 4.4 Examples at Different Scales

**Microscopic (Silicon Chip)**
```hw
define Space "Microprocessor":
    dimensions: 5mm by 5mm by 0.5mm
    grid: 5000 by 5000 by 5
    # Voxel = 1 micrometer (0.001mm)
```

**Standard (PCB Circuit)**
```hw
define Space "SensorBoard":
    dimensions: 50mm by 50mm by 2mm
    grid: 500 by 500 by 2
    # Voxel = 0.1mm
```

### 4.5 Substrate and Material Layers

#### 4.5.1 Enhanced Spanning Syntax

Hardware Script supports geometric macros for professional PCB and silicon design:

**Entire Space Coverage**
```hw
define Space "ProfessionalPCB":
    dimensions: 50mm by 50mm by 1.6mm
    grid: 500 by 500 by 4
    
    # Base substrate across all layers
    add Substrate(FR4) spanning all
    # Expands to: [1, 1, 1] to [4, 500, 500]
```

**Single Layer Coverage (Ground/Power Planes)**
```hw
    # Solid copper ground plane
    add Copper named GroundPlane spanning layer 2
    # Expands to: [2, 1, 1] to [2, 500, 500]
    
    # Solid copper power plane  
    add Copper named PowerPlane spanning layer 3
    # Expands to: [3, 1, 1] to [3, 500, 500]
```

**Multi-Layer Coverage**
```hw
    # Thick dielectric block
    add Substrate(SiliconDioxide) spanning layers 2 to 4
    # Expands to: [2, 1, 1] to [4, 500, 500]
```

**Deterministic Expansion**: All spanning shortcuts expand to exact coordinates before Engine A physics calculations.

### 4.5 Adding Substrates

```hw
define Space "MicroSensor":
    dimensions: 10mm by 10mm by 2mm
    grid: 100 by 100 by 2
    
    # Add FR4 substrate spanning entire space
    add Substrate(FR4) spanning all
```

**Spanning Shortcut**: The `spanning all` keyword automatically fills the entire 3D grid defined by the Space dimensions and grid resolution. Engine A translates this to the exact coordinates `[1, 1, 1] to [2, 100, 100]` based on the Space definition above.

### 4.6 Multi-Layer Board Design

#### 4.6.1 Layer Assignment Strategy

**4-Layer Board Example**:
```hw
define Space "ComplexBoard":
    dimensions: 100mm by 100mm by 2mm
    grid: 1000 by 1000 by 4
    
    # Base substrate
    add Substrate(FR4) spanning all
    
    # Layer assignments:
    # Z=1: Top component layer
    # Z=2: Ground plane (solid copper)
    # Z=3: Power plane (5V distribution)  
    # Z=4: Bottom component layer
```

#### 4.6.2 Power and Ground Planes

**Ground Plane** (dedicated inner layer):
```hw
# Solid copper sheet connected to GND
add CopperPlane(GND) at layer 2 spanning all
```

**Power Plane** (dedicated inner layer):
```hw
# Solid copper sheet carrying 5V
add CopperPlane(5V) at layer 3 spanning all
```

**Benefits**:
- Instant ground/power access via vias
- Massive current capacity  
- Built-in heat dissipation
- Reduced electromagnetic interference

#### 4.6.3 Layer Usage Rules

**Component Layers** (Z=1, Z=max):
- ✅ Components, pins, copper traces, vias
- **Purpose**: Physical component mounting

**Inner Layers** (Z=2 to Z=max-1):
- ✅ Copper traces, copper planes, vias
- ❌ Physical components (enforced by Engine A)
- **Purpose**: Signal routing, power distribution

**Engine A Validation**:
```
❌ Error: Cannot place Transistor at [2, 50, 50]
  Physical components only permitted on outer layers
  Available positions: Z=1 or Z=4
```

---

## 5. Material Definition

### 5.1 Syntax

```hw
define Material "MaterialName":
    type: <Conductor|Insulator|Semiconductor>
    <property>: <value>
```

### 5.2 Required Properties by Type

**Conductor**
```hw
define Material "StandardCopper":
    type: Conductor
    max_current: 10 Amps
    melting_point: 1085 C
    thermal_conductivity: 401 W/mK
    resistivity: 1.68e-8 Ohm·m
    color: Copper
```

**Insulator**
```hw
define Material "FR4":
    type: Insulator
    dielectric_strength: 20 kV/mm
    thermal_resistance: 0.3 K/W
    color: Green
```

**Semiconductor**
```hw
define Material "Silicon":
    type: Semiconductor
    bandgap: 1.12 eV
    carrier_mobility: 1400 cm²/V·s
    color: Gray
```

---

## 6. Component Definition

### 6.1 Components as Nested Local Spaces

Components are simply **nested local spaces** with:
- **Local Grid**: The physical footprint in grid cells
- **Body (Keep-Out Zone)**: The cells the component physically blocks
- **Pins**: The specific cells where external copper is permitted to connect
- **Render Mesh**: The string reference for 3D visualization

### 6.2 Syntax

```hw
define Component "ComponentName":
    grid: <width> by <height>
    
    body:
        spans: [<x1>, <y1>] to [<x2>, <y2>]
    
    pins:
        <PinName> at [<x>, <y>]
        <PinName> at [<x>, <y>]
    
    render_mesh: "<mesh_identifier>"
```

### 6.3 Simple Component Example

```hw
define Component "Transistor_NPN":
    grid: 3 by 3
    
    body:
        spans: [0, 0] to [2, 2]
    
    pins:
        Collector at [0, 1]
        Base      at [1, 0]
        Emitter   at [2, 1]
    
    render_mesh: "standard_transistor_TO92"
```

### 6.4 Complex Component Example

```hw
define Component "MicroController_ATmega328":
    grid: 10 by 10
    
    body:
        spans: [0, 0] to [9, 9]
    
    pins:
        VCC    at [0, 2]
        GND    at [0, 7]
        TX     at [3, 0]
        RX     at [4, 0]
        D0     at [5, 0]
        D1     at [6, 0]
        RESET  at [9, 5]
    
    render_mesh: "atmega328_dip28"
```

### 6.5 Legacy Socket Definition

For simpler components without explicit grid definitions:

```hw
define Component "PowerPin":
    material: StandardCopper
    shape: Cylinder (radius: 1mm, height: 5mm)
    
    sockets:
        In is top
        Out is bottom
```

---

### 7. Component Instantiation with Rotation

### 7.1 Adding Components to Space

```hw
add <ComponentType> [(<parameters>)] named <LocalName> at [Z, X, Y] [rotated <direction>]
```

### 7.2 Rotation Support

Components can be rotated to face specific directions:

```hw
add Transistor_NPN named Switch at [1, 50, 50] rotated East
add Battery (5V) named PowerSource at [1, 5, 5] rotated North
```

**Supported Directions**:
- `North` - Component faces toward decreasing Y
- `South` - Component faces toward increasing Y  
- `East` - Component faces toward decreasing X
- `West` - Component faces toward increasing X

### 7.3 Examples

```hw
add Battery (5V) named PowerSource at [1, 5, 5]
add LED named Light at [1, 90, 90] rotated South
add Transistor_NPN named Switch at [1, 50, 50] rotated East
add Sensor named Eye at [2, 10, 10]  # Inner layer
```

---

## 8. Routing and Connections

### 8.1 Pin Connection (Logical Adjacency Only)

```hw
connect <source>.<socket> to <destination>.<socket>
```

**IMPORTANT**: `connect` is only valid for **logical pin connections** within component definitions where pins are mathematically adjacent (zero trace length). For physical board routing, use `route` with explicit waypoints.

Example (valid - internal component definition):
```hw
define Component "NOT_Gate":
    # Internal connections (zero physical distance)
    connect R_Pullup.out to T1.Collector
```

### 8.2 Physical Routing (Explicit Waypoints Only)

```hw
route <source>.<socket> to <destination>.<socket>:
    path:
        - [Z, X, Y]  # Start point
        - [Z, X, Y]  # Waypoint 1
        - [Z, X, Y]  # Waypoint 2
        - [Z, X, Y]  # End point
```

**CRITICAL**: All physical copper traces on the board MUST use explicit `route` statements with waypoints. No automatic routing algorithms are used.

### 8.3 Explicit Routing Example

```hw
route Power.out to Switch.Base:
    path:
        - [1, 5, 5]       # Start at Battery
        - [1, 50, 5]      # Travel across X
        - [1, 50, 49]     # Travel down Y
```

### 8.4 Via Generation (Layer Changes)

A Via is automatically generated when Z coordinate changes:

```hw
route Switch.Emitter to Light.in:
    path:
        - [1, 52, 51]     # Start at top layer
        - [2, 52, 51]     # Drill down to layer 2 (Via created)
        - [2, 90, 51]     # Route beneath other components
        - [1, 90, 51]     # Drill back up to layer 1 (Via created)
        - [1, 90, 90]     # Arrive at LED
```

### 8.5 Advanced Routing Parameters with Via Clearance

```hw
route PowerSource.out to Motor.in:
    path:
        - [1, 5, 5]
        - [1, 10, 5]
        - [3, 10, 10]       # Via through Layer 2 copper plane
    trace_width: 2mm
    max_current: 5A
    impedance: 50 Ohms
    clearance: 1mm          # CRITICAL: Clear copper around via
```

#### Via Clearance for Copper Planes

When routing through solid copper planes (ground/power planes), vias require clearance to prevent short circuits:

**With Clearance (Correct)**:
```hw
route Battery.VCC to PowerPlane:
    path:
        - [1, 10, 10]       # Start on component layer
        - [3, 10, 10]       # Via through ground plane (layer 2)
    clearance: 0.5mm        # Creates anti-pad around via
```

**Without Clearance (Error)**:
```hw
route Battery.VCC to PowerPlane:
    path:
        - [1, 10, 10]
        - [3, 10, 10]       # Via contacts ground plane = SHORT CIRCUIT
    # Missing clearance parameter = Fatal Error
```

**Engine A Response**:
```
❌ Fatal Electrical Error: Dead Short Detected
  Via at [2, 10, 10] contacts GroundPlane copper
  Solution: Add 'clearance: Xmm' parameter to route block
```

---

## 9. Import System and Modularity

### 9.1 The main.hw Entry Point

**Inspired by**: Rust's `main.rs` and Node.js's `index.js`

**The Problem**: Large projects cannot be written in a single 10,000-line file. You need modularity.

**The Solution**: The compiler looks for `main.hw` (or `board.hw`) as the root entry point, which can import other .hw files.

### 9.2 Project Structure

```
my_robot_project/
├── hpm.json              # Package dependencies
├── main.hw               # Root entry point (the motherboard)
├── power_system.hw       # Local module (power distribution)
├── sensor_array.hw       # Local module (sensor cluster)
├── motor_controller.hw   # Local module (motor driver)
└── build/                # Compiled outputs (.hwx, .obj, gerbers)
```

### 9.3 Main Entry Point Example

**main.hw** (the root motherboard):
```hw
## Robot Motherboard - Main Entry Point
## This is the primary PCB that integrates all subsystems
define Space "Robot_Motherboard":
    dimensions: 200mm by 200mm by 2mm
    grid: 1000 by 1000 by 4

# Import local modules
import "power_system.hw" as Power
import "sensor_array.hw" as Sensors
import "motor_controller.hw" as Motors

# Place imported modules on the main board
add Power at [1, 50, 50]
add Sensors at [1, 150, 150]
add Motors at [1, 100, 800]

# Connect modules together
route Power.VOUT to Sensors.VIN:
    path:
        - [1, 50, 50]
        - [1, 150, 150]

route Power.VOUT to Motors.VIN:
    path:
        - [1, 50, 50]
        - [1, 100, 800]
```

**power_system.hw** (local module):
```hw
## Power Distribution System
## Provides regulated 5V and 3.3V rails from 12V input
define Space "PowerSystem":
    dimensions: 40mm by 30mm by 2mm
    grid: 80 by 60 by 2

add IC_LM7805 named Reg5V at [1, 20, 15]
add IC_LM1117 named Reg3V3 at [1, 60, 15]

# Expose pins for external connections
expose Reg5V.In as VIN
expose Reg5V.Out as VOUT_5V
expose Reg3V3.Out as VOUT_3V3
expose Reg5V.GND as GND
```

### 9.4 Import Syntax

**Local file imports**:
```hw
import "power_system.hw" as Power
import "sensor_array.hw" as Sensors
import "./modules/motor_controller.hw" as Motors
```

**Package imports** (from hpm registry):
```hw
import Regulator from "@power/5v-regulator"
import IMU from "@sensors/imu-module"
import HBridge from "@motor/h-bridge"
```

### 9.5 Component vs Library Imports

**Single Component** (.hwx):
```hw
import Transistor from standard.switches
import Resistor from passive/resistor_0805

# Use like a primitive
add Transistor named Q1 at [1, 10, 10]
add Resistor (220Ohm) named R1 at [1, 15, 10]
```

**Library/Sub-Assembly** (.hw):
```hw
import Regulator from "@power/5v-regulator"

# Place entire pre-routed module as one unit
add Regulator named PowerSupply at [1, 50, 50]

# Connect to exposed pins
route Battery.Plus to PowerSupply.VIN:
    path:
        - [1, 10, 10]
        - [1, 50, 50]
```

### 9.6 How the Compiler Handles Imports

1. **Parse main.hw**: Compiler starts at the entry point
2. **Resolve imports**: Find all imported .hw files (local or from hpm)
3. **Parse sub-modules**: Each imported file has its own local Space
4. **Map to global grid**: When you `add Power at [1, 50, 50]`, the compiler:
   - Takes the Power module's local tensor grid (80×60×2)
   - Places it at position [1, 50, 50] in the main board's global grid (1000×1000×4)
   - Translates all internal coordinates to global coordinates
5. **Expose pins**: Exposed pins become connection points in the global space

**Result**: Modular, reusable hardware design with proper encapsulation

---

## 10. Import System (Standard Library)

### 10.1 Import Syntax

```hw
import <ComponentName> from <package.path>
```

### 10.2 Standard Library Paths

```
standard.materials      # Copper, Gold, Silicon, FR4, etc.
standard.power          # Battery, Regulator, Capacitor
standard.sensors        # LightSensor, TempSensor, Gyroscope
standard.switches       # Transistor, Relay, MOSFET
standard.logic          # AND, OR, NOT, XOR gates
standard.comms          # UART, SPI, I2C, Bluetooth
```

### 10.3 Import Examples

```hw
import Copper from standard.materials
import Battery from standard.power
import LightSensor from standard.sensors
import Transistor from standard.switches
```

### 10.4 Third-Party Packages

```hw
import BluetoothModule from @hardware/comms
import GPS from @sensors/location
import TemperatureSensor from @sensors/environmental
```

Package naming convention: `@namespace/category`

---

## 11. Complete Language Example

```hw
# 1. Imports
import Copper from standard.materials
import Battery from standard.power
import LightSensor from standard.sensors
import Comparator from standard.analog
import VoltageDivider from standard.analog
import Transistor from standard.switches

# 2. Professional 4-Layer PCB Design
define Space "Automated Sprinkler":
    dimensions: 50mm by 50mm by 1.6mm
    grid: 500 by 500 by 4
    
    # Standard 4-layer stackup
    add Substrate(FR4) spanning all
    add Copper named TopLayer spanning layer 1      # Component layer
    add Copper named GroundPlane spanning layer 2   # Solid ground
    add Copper named PowerPlane spanning layer 3    # Solid +5V  
    add Copper named BottomLayer spanning layer 4   # Component layer
    
    # 3. Component Instantiation
    add Battery (5V) named PowerSource at [1, 50, 50]
    add LightSensor named Eye at [1, 200, 200]
    add VoltageDivider (ratio: 0.5) named Threshold at [1, 300, 200]
    add Comparator named LogicGate at [1, 400, 200]
    add Transistor named Valve at [4, 450, 450] rotated South
    
    # 4. Power Distribution Through Planes
    route PowerSource.VCC to PowerPlane:
        path:
            - [1, 50, 50]
            - [3, 50, 50]       # Via to power plane
        clearance: 1mm          # Clear around via
    
    route PowerSource.GND to GroundPlane:
        path:
            - [1, 50, 50]
            - [2, 50, 50]       # Via to ground plane
        clearance: 1mm
    
    # 5. Signal Routing with Layer Changes
    route Eye.out to LogicGate.positive_pin:
        path:
            - [1, 200, 200]     # Start at sensor
            - [4, 200, 200]     # Via through planes to bottom
            - [4, 400, 200]     # Route on bottom layer
            - [1, 400, 200]     # Via back to top
        clearance: 0.5mm        # Clear around all vias
    
    route Threshold.out to LogicGate.negative_pin:
        path:
            - [1, 300, 200]
            - [1, 400, 200]
    
    route LogicGate.out to Valve.base:
        path:
            - [1, 400, 200]
            - [4, 400, 200]     # Via to bottom layer
            - [4, 450, 400]     # Route to valve
            - [4, 450, 450]
        clearance: 0.5mm
    
    # 6. Load Connection to Planes
    route PowerPlane to Valve.collector:
        path:
            - [3, 450, 450]     # From power plane
            - [4, 450, 450]     # Via to valve
        clearance: 0.5mm
    
    route Valve.emitter to GroundPlane:
        path:
            - [4, 450, 450]
            - [2, 450, 450]     # Via to ground plane
        clearance: 0.5mm
```

---

## 12. Grammar (EBNF Notation)

```ebnf
program = { import_stmt | define_stmt }

import_stmt = "import" ( string_literal "as" identifier | identifier "from" package_path )

define_stmt = "define" type identifier ":" block

type = "Material" | "Component" | "Space"

block = INDENT { statement } DEDENT

statement = property_stmt | add_stmt | route_stmt | expose_stmt

property_stmt = identifier ":" value

add_stmt = "add" identifier [ "(" params ")" ] "named" identifier 
         [ position_clause ] [ rotation_clause ] [ spanning_clause ]

position_clause = "at" coordinate

rotation_clause = "rotated" direction

spanning_clause = "spanning" spanning_target

spanning_target = "all" 
                | "layer" number
                | "layers" number "to" number
                | coordinate "to" coordinate

expose_stmt = "expose" socket_ref "as" identifier

direction = "North" | "South" | "East" | "West"

coordinate = "[" number "," number "," number "]"

route_stmt = "route" socket_ref "to" socket_ref ":" INDENT route_block DEDENT

route_block = path_stmt [ clearance_stmt ] [ trace_params ]

path_stmt = "path" ":" INDENT { coordinate } DEDENT

clearance_stmt = "clearance" ":" distance_value

trace_params = { trace_param }

trace_param = "trace_width" ":" distance_value
            | "max_current" ":" current_value
            | "impedance" ":" impedance_value

socket_ref = identifier "." identifier

identifier = letter { letter | digit | "_" }

string_literal = '"' { any_char } '"'

package_path = identifier { "." identifier } | "@" identifier "/" identifier

value = number [ unit ] | string | boolean

distance_value = number unit

current_value = number ("A" | "mA")

impedance_value = number "Ohms"

unit = "mm" | "cm" | "m" | "V" | "A" | "mA" | "Ohms" | "C" | "F" | "%"
```

---

**Document Status**: Draft Specification  
**Last Updated**: March 2026  
**Part of**: Hardware Script (.hw) Documentation Suite

---
