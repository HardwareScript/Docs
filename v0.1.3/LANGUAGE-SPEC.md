# Book 2: The Language Specification

**Hardware Script v0.1.3**  
**Target Audience**: Hardware engineers and LLMs writing .hw code  
**Last Updated**: March 2026

---

## Foundation

This document consolidates the following source materials:

**Source Files**:
- `Docs/v0.1/LANGUAGE-SPEC.md` — The v0.2 draft language specification
- `Docs/v0.0.1/SPECIFICATION.md` — The quick overview and Sprinkler example
- `Docs/v0.1/QUICK-REFERENCE.md` — Syntax cheat sheet
- `Docs/v0.1/MODULARITY-AND-DOCUMENTATION.md` — Imports, sub-assemblies, and documentation syntax
- `Docs/v0.1.2/FILE-EXTENSION-DEBATE-RESOLUTION.md` — Abstraction blocks syntax (pins, behavior, layout, electrical)
- `Docs/v0.0.1/SILICON-DESIGN.md` — Nanometer-scale syntax examples
- `Docs/v0.1.2/LANGUAGE-DESIGN-PHILOSOPHY.md` — Ruby-inspired syntax rationale, SI units enforcement, standards compliance

---

## Quick Reference Card

### Basic Syntax Patterns

```hw
# Space Definition
define space "BoardName":
    dimensions: <X>mm by <Y>mm by <Z>mm
    grid: <cols> by <rows> by <layers>

# Component Placement
add <Type> named <Name> at [X, Y, Z] rotated <Direction>

# Routing
route <Source> to <Destination>:
    path:
        - [X, Y, Z]
        - [X, Y, Z]

# Comments
# Human comment
## Documentation comment (extracted by hwsd)
```

### Coordinate System: [X, Y, Z] with Configurable Origin

**Coordinate Order**: Alphabetical order `[X, Y, Z]`
- **X** = Column (left to right)
- **Y** = Row (direction depends on XY origin)
- **Z** = Layer (direction depends on Z origin)
- All coordinates are 1-indexed

**Origin Points** (configurable per Space using unified `XY by Z` syntax):
- `tl by t` (default): Top-Left XY, Top-Down Z (like matrices/PCB design)
- `bl by t`: Bottom-Left XY, Top-Down Z (like CAD/CNC milling)
- `bl by b`: Bottom-Left XY, Bottom-Up Z (like 3D printing)
- Partial syntax `tl` defaults to `tl by t`
- `tr`: X decreases, Y increases downward
- `br`: X decreases, Y increases upward

**Named (Declarative)**: Explicit labels, any order
```hw
place LED at [x:10, y:15, z:2]  # Explicit, any order
place LED at [z:2, x:10, y:15]  # Layer-first when needed
```

### Keywords

**Actions**: `define`, `add`, `route`, `import`, `expose`  
**Types**: `space`, `component`, `material`, `substrate`  
**Properties**: `dimensions`, `grid`, `pins`, `behavior`, `layout`, `electrical`  
**Connectors**: `named`, `at`, `to`, `from`, `by`, `rotated`, `spanning`  
**Directions**: `North`, `South`, `East`, `West`

### Units

- **Distance**: `mm` (millimeters), `cm` (centimeters)
- **Electrical**: `V` (volts), `A` (amps), `mA` (milliamps)
- **Frequency**: `Hz`, `kHz`, `MHz`, `GHz`
- **Temperature**: `C` (celsius)
- **Resistance**: `Ω` (ohms), `kΩ`, `MΩ`
- **Capacitance**: `F`, `pF`, `nF`, `uF`

---

## Part I: Language Overview

### Design Principles

Hardware Script is designed with four core principles:

1. **Human Readability**: English-like syntax accessible to non-electrical engineers
2. **LLM Optimization**: Declarative structure optimized for AI generation and parsing
3. **Minimal Syntax Overhead**: Indentation-based, avoiding brackets and semicolons
4. **Deterministic Semantics**: Same input always produces same output

### Language Paradigm

Hardware Script is a **declarative, text-based hardware description language** that:
- Describes hardware intent, not implementation details
- Uses 3D tensor coordinates for spatial reasoning
- Supports multiple abstraction levels through explicit blocks
- Compiles to different targets (PCB, FPGA, ASIC, SPICE)

### Why Ruby-Inspired Syntax?

Hardware Script's syntax is inspired by Ruby's philosophy that code should read like natural English prose. This design choice was made after evaluating Python, SQL, and Ruby-inspired approaches.

**Ruby's Advantages for Hardware Description**:

1. **Reads Like Natural English**: Ruby was designed with the philosophy that code should read like prose, aligning perfectly with our goal of making hardware design accessible.

   ```hw
   add Battery (12V) named Power at [10,10,1]
   route Power.Plus to LED.Anode
   ```
   
   Reads as: "Add a 12-volt battery named Power at position [10,10,1]. Route Power's Plus pin to LED's Anode."

2. **Minimal Punctuation**: Ruby minimizes visual noise by making parentheses, semicolons, and braces optional in many contexts.

   ```hw
   add Resistor (4.7kΩ) named PullUp at [20,15,1] rotated 90
   ```

3. **Excellent for DSLs**: Ruby is famous for domain-specific languages like RSpec (testing), Rails routes, and Rake. Hardware Script follows this tradition.

4. **Prepositions as Keywords**: Ruby popularized using English prepositions as part of the syntax:

   ```hw
   add Battery at [10,10,1]
   route Power.Plus to LED.Anode
   dimensions: 50mm by 50mm by 4mm
   ```

**Why Not Python or SQL?**

- **Python**: Too permissive (multiple ways to do things), verbose method chaining, not optimized for declarative hardware descriptions
- **SQL**: Too verbose for hardware descriptions, awkward for hierarchical structures, limited expressiveness for complex logic

For complete rationale, see `Docs/v0.1.2/LANGUAGE-DESIGN-PHILOSOPHY.md`.

### Coordinate System Abstraction: Solving the 40-Year Holy War

Hardware Script introduces a revolutionary approach to coordinate systems that eliminates the eternal debate between TopLeft vs BottomLeft origins AND Top-Down vs Bottom-Up Z-axis directions.

**The Problem**: For 40 years, the EDA industry has been locked in multiple "holy wars":
- **Software Engineers** think in **TopLeft** (like matrices, screen pixels, text rendering)
- **Hardware Engineers** think in **BottomLeft** (like CAD tools, mechanical drawings)
- **Manufacturing** uses **BottomLeft** (Gerber standard, CNC machines)
- **PCB/Software** thinks **Top-Down Z** (Layer 1 is sky, Z increases downward)
- **3D Printing** thinks **Bottom-Up Z** (Layer 1 is ground, Z increases upward)
- **LLMs** naturally generate **TopLeft, Top-Down** (token-by-token, like reading text)

**Hardware Script's Solution**: Don't fight over the origin point. Let the compiler do the math.

**How It Works**:
1. **User Space**: You choose your preferred origin using unified `XY by Z` syntax
2. **Absolute Space**: Compiler normalizes all coordinates to a universal format
3. **Export Space**: Gerber/GDSII exporters translate to manufacturing standards

**The Unified Syntax**: Uses the same `by` keyword as dimensions and grid for perfect visual alignment:

```hw
dimensions: 100mm by 100mm by 2mm  # X by Y by Z
grid: 100 by 100 by 4              # X by Y by Z
origin: tl by t                    # XY by Z
```

**Example - Same Board, Different Mental Models**:

Software Engineer's perspective (PCB):
```hw
define space "SensorBoard":
    origin: tl by t
    add Sensor at [1, 1, 1]    # Top-left corner, top layer
    add LED at [3, 2, 1]       # Bottom-right corner, top layer
```

Hardware Engineer's perspective (Same PCB):
```hw
define space "SensorBoard":
    origin: bl by t
    add Sensor at [1, 2, 1]    # Same physical location!
    add LED at [3, 1, 1]       # Same physical location!
```

3D Printer Operator's perspective (Additive):
```hw
define space "PrintedEnclosure":
    origin: bl by b
    add Support at [1, 1, 1]   # Bottom-left corner, bottom layer
```

The compiler ensures all engineers are placing components at identical physical locations, despite using different mental models.

**Why This Matters**: When someone asks "Does your tool use Bottom-Left or Top-Left coordinates? Top-Down or Bottom-Up layers?", our answer is: **"Whichever ones your brain prefers. The compiler does the math."**

**Unified Syntax Benefits**:
- **Visual Alignment**: Mirrors the same `by` cadence as dimensions and grid
- **Self-Documenting**: `tl by t` instantly reads as "Top-Left by Top"
- **Lexer Optimization**: Parser splits by `by` keyword (already in lexer)
- **LLM Friendly**: Follows natural English grammar patterns
- **No Breaking Changes**: `origin: tl` defaults to `tl by t`
- **Graceful Fallbacks**: Omitted origin defaults to `tl by t`

**Origin Syntax Rules**:
- XY must be exactly 2 characters: `tl`, `bl`, `tr`, `br`
- Z must be exactly 1 character: `t` (top-down), `b` (bottom-up)
- Case-insensitive: `tl by t` and `TL by T` are equivalent
- Partial syntax `tl` defaults to `tl by t`
- If omitted entirely, defaults to `tl by t`
- Reserved keywords in lexer for fast parsing

For complete details, see `Docs/v0.1.2/COORDINATE-SYSTEM-ABSTRACTION.md`.

### Engineering Standards: SI Units Only

Hardware Script enforces strict SI (International System of Units) notation to ensure readability and eliminate ambiguity.

**Philosophy**: Exactly **two allowed formats** per unit:
1. **Unicode symbol**: `Ω`, `µF`, `°`
2. **Keyboard-friendly alias**: `Ohm`, `uF`, `deg`

**Standards Rejected**:
- **IEC 60062** (e.g., `4K7` = 4.7kΩ): Cryptic to non-experts, not self-documenting
- **SPICE notation** (e.g., `4.7k` = 4.7kΩ): Context-dependent, requires domain knowledge

**Valid Examples**:
```hw
# Distance
50mm, 5cm

# Voltage
12V, 500mV

# Current
2A, 100mA, 50µA

# Resistance (both valid)
4.7kΩ, 4.7kOhm

# Capacitance (both valid)
100µF, 100uF

# Frequency
60Hz, 2.4GHz
```

**Invalid Examples** (will produce helpful error messages):
```hw
add Resistor (4K7) named R1 at [10,10,1]   # ❌ IEC 60062 not allowed
add Resistor (4.7k) named R1 at [10,10,1]  # ❌ SPICE notation not allowed
```

**Error Message Example**:
```
❌ Syntax Error: Unrecognized unit format '4K7'
  ╭─[main.hw:5:15]
5 │ add Resistor (4K7) named R1 at [10,10,1]
  ·               ─┬─
  ·                ╰── Invalid unit formatting
  ╰────

Help: Hardware Script uses strict SI notation to ensure readability.
      Suggestion: Change '4K7' to '4.7kOhm' or '4.7kΩ'.
      
      IEC 60062 notation (4K7) is not permitted.
      Use explicit SI units for clarity.
```

This strict enforcement ensures that Hardware Script code is universally readable without requiring domain-specific knowledge of component marking standards.

### File Extension

`.hw` - Hardware Script source files

---

## Part II: Core Syntax

### 1. Space Definition

Every `.hw` file declares a **Space** - a bounded physical volume with discrete grid resolution and configurable coordinate origin.

**Syntax**:
```hw
define space "SpaceName":
    dimensions: <width>mm by <height>mm by <depth>mm
    grid: <x_cells> by <y_cells> by <z_cells>
    origin: <xy> by <z>  # Optional, defaults to tl by t
```

**Parameters**:
- `dimensions`: Physical size in millimeters (width × height × depth)
- `grid`: Discrete resolution (columns × rows × layers)
- `origin`: Coordinate system origin point (optional, defaults to `tl by t`)
  - XY: `tl`, `bl`, `tr`, `br` (controls XY-plane orientation)
  - Z: `t` (top-down), `b` (bottom-up) (controls Z-axis direction)

**Origin Points**:
- `tl by t` (default): (1,1) at top-left, Y increases downward, Z increases downward (software/PCB)
- `bl by t`: (1,1) at bottom-left, Y increases upward, Z increases downward (CAD/CNC)
- `bl by b`: (1,1) at bottom-left, Y increases upward, Z increases upward (3D printing)
- `tl by b`: (1,1) at top-left, Y increases downward, Z increases upward (software-driven 3D)
- Partial syntax `tl` defaults to `tl by t`

**Voxel Calculation**:
```
Voxel Size = Dimension / Grid Cells

Example:
  50mm / 100 cells = 0.5mm per voxel
  2mm / 20000 cells = 100nm per voxel (silicon scale)
```

**Example - PCB Scale with TL Origin**:
```hw
define space "SensorBoard":
    dimensions: 50mm by 50mm by 2mm
    grid: 100 by 100 by 2
    origin: tl by t  # Software-friendly (default)
    # Voxel size: 0.5mm × 0.5mm × 1mm
    # (1,1) = top-left corner, Y increases downward, Z increases downward
```

**Example - PCB Scale with BL Origin**:
```hw
define space "SensorBoard":
    dimensions: 50mm by 50mm by 2mm
    grid: 100 by 100 by 2
    origin: bl by t  # CAD-friendly
    # Voxel size: 0.5mm × 0.5mm × 1mm
    # (1,1) = bottom-left corner, Y increases upward, Z increases downward
```

**Example - Silicon Scale**:
```hw
define space "Custom_ALU_Silicon":
    dimensions: 2mm by 2mm by 0.1mm
    grid: 20000 by 20000 by 10
    origin: bl  # Traditional for silicon design
    # Voxel size: 100nm × 100nm × 10μm
```

**Rules**:
- Dimensions must include unit (`mm` or `cm`)
- Grid values must be positive integers
- Space definition must appear before components and routes
- Only one Space per file (use imports for modularity)
- Origin defaults to `tl` if not specified
- Compiler handles all coordinate transformations automatically


### 2. Substrate Definition

Define the base material layer that spans the entire space or specific regions.

**Syntax**:
```hw
add Substrate(<MaterialType>) spanning <region>
```

**Examples**:
```hw
# PCB substrate (fiberglass)
add Substrate(FR4) spanning [1, 1, 1] to [100, 100, 2]

# Silicon wafer substrate
add Substrate(Silicon) spanning [1, 1, 1] to [20000, 20000, 1]

# Dielectric layers for metal routing
add Substrate(SiliconDioxide) spanning [1, 1, 2] to [20000, 20000, 10]
```

**Common Substrate Materials**:
- `FR4` - Standard PCB fiberglass
- `Silicon` - Silicon wafer for chips
- `SiliconDioxide` - Insulating layer
- `Aluminum` - Metal chassis
- `Polyimide` - Flexible PCB substrate

---

### 3. Component Placement

Place components at specific grid coordinates within the space.

```hw
# PCB substrate (fiberglass)
add Substrate(FR4) spanning [1, 1, 1] to [100, 100, 2]

# Silicon wafer substrate
add Substrate(Silicon) spanning [1, 1, 1] to [20000, 20000, 1]

# Dielectric layers for metal routing
add Substrate(SiliconDioxide) spanning [1, 1, 2] to [20000, 20000, 10]
```Syntax**:
```hw
add <ComponentType> named <InstanceName> at [X, Y, Z] rotated <Direction>
```

**Parameters**:
- `ComponentType`: Type of component (from standard library or imported)
- `InstanceName`: Unique identifier for this instance
- `[X, Y, Z]`: Grid coordinates (1-indexed)
- `rotated`: Optional orientation (North, South, East, West)

**Examples**:
```hw
# Basic component placement
add Battery (5V) named PowerSource at [10, 10, 1]

# With rotation
add Transistor_NPN named Switch at [50, 50, 1] rotated North

# With parameters
add Resistor (10kΩ) named R1 at [30, 30, 1]
add Capacitor (100nF) named C1 at [35, 30, 1]

# Complex components
add ESP32_WROOM_32 named MCU at [50, 50, 1] rotated East
add LightSensor named Eye at [70, 70, 1]
```

**Coordinate System Details**:

```
Grid Coordinates (Discrete):
  [X, Y, Z] where:
    X ∈ [1, x_cells]     (1-indexed, column)
    Y ∈ [1, y_cells]     (1-indexed, row, direction depends on origin)
    Z ∈ [1, z_layers]    (1-indexed, layer)

Physical Coordinates (Continuous):
  physical_x = (grid_x - 1) * voxel_size_x
  physical_y = (grid_y - 1) * voxel_size_y  # After origin transformation
  physical_z = (grid_z - 1) * layer_thickness

Origin Transformation (handled by compiler):
  TL:     User_Y → Absolute_Y = (grid_height + 1) - User_Y
  BL:     User_Y → Absolute_Y = User_Y (no transformation)
  TR:     User_X → Absolute_X = (grid_width + 1) - User_X
          User_Y → Absolute_Y = (grid_height + 1) - User_Y
  BR:     User_X → Absolute_X = (grid_width + 1) - User_X
          User_Y → Absolute_Y = User_Y
```

**Rules**:
- Component names must be unique within the space
- Coordinates must be within grid bounds
- Components cannot overlap (compiler validates)
- Rotation affects pin positions

---

### 4. Manual Waypoint Routing

Define electrical connections between components using explicit user-defined waypoints.

**Book 2 Scope**: This section covers **Manual Waypoint Routing** only - where users explicitly define the path using waypoints. For **Automatic Routing** (when no waypoints are provided and the compiler calculates the path using A* pathfinding), see Book 5 (Routing & Physics).

**Syntax**:
```hw
route <Source> to <Destination>:
    path:
        - [X, Y, Z]
        - [X, Y, Z]
        - [X, Y, Z]
```

**Parameters**:
- `Source`: Component pin reference (e.g., `Battery.Plus`, `Switch.Collector`)
- `Destination`: Target pin reference or external connection
- `path`: List of waypoints defining the trace route

**Examples**:

**Straight Line**:
```hw
route PowerSource.out to LED.in:
    path:
        - [10, 10, 1]
        - [50, 10, 1]
```

**L-Shape**:
```hw
route Switch.Collector to Power.Out:
    path:
        - [5, 6, 1]
        - [15, 6, 1]
        - [15, 15, 1]
```

**Multi-Layer with Vias**:
```hw
route Eye.out to LogicGate.positive_pin:
    path:
        - [12, 12, 1]    # Start at sensor (layer 1)
        - [12, 12, 2]    # Via to layer 2
        - [18, 12, 2]    # Route on bottom layer
        - [18, 18, 2]    # Position under comparator
        - [18, 18, 1]    # Via back to top layer
```

**External Connections**:
```hw
route Valve.collector to "EXTERNAL_SPRINKLER_MOTOR":
    path:
        - [20, 20, 1]
        - [25, 20, 1]
```

**Manual Waypoint Routing Behavior**:
- User provides explicit waypoints in the `path:` section
- Compiler interpolates straight lines between consecutive waypoints using Bresenham's algorithm
- All intermediate cells marked as COPPER in tensor grid
- Layer changes (Z coordinate differences) automatically create vias
- User has complete control over the route geometry

**Automatic Routing Alternative**: If you omit the `path:` section, the compiler will use automatic routing algorithms (Manhattan routing with A* pathfinding). For complete details on automatic routing algorithms, constraints, and physics validation, see Book 5 (Routing & Physics).

**Rules for Manual Waypoint Routing**:
- Waypoints must be within grid bounds
- First waypoint should be near source pin
- Last waypoint should be near destination pin
- Layer changes (Z coordinate) create vias
- Traces cannot overlap (compiler validates)


---

### 5. Comments and Documentation

Hardware Script supports two types of comments with different purposes.

**Human Comments (`#`)**:

Comments starting with `#` are for developers and are ignored by the compiler.

```hw
# This is a comment for humans
define Space "Board":  # Inline comment
    dimensions: 50mm by 50mm by 2mm  # Physical size
```

**Documentation Comments (`##`)**:

Comments starting with `##` are extracted by `hwsd` (Hardware Script Documentation) to generate auto-documentation.

```hw
## A complete 5V voltage regulator module
## Input: 7-12V DC
## Output: Regulated 5V at up to 1A
## WARNING: Requires heatsink above 500mA load
define Space "5V_Regulator":
    dimensions: 20mm by 15mm by 2mm
    grid: 40 by 30 by 2
    
    ## Main voltage regulator IC
    add IC_LM7805 named Regulator at [20, 15, 1]
    
    ## Input filter capacitor (reduces ripple)
    add Capacitor (10uF) named C1 at [10, 15, 1]
```

**Multi-Line Comments (`#[...]#` and `##[...]##`)**:

For longer comments or disabling code blocks, use multi-line delimiters. Opening delimiters (`#[` or `##[`) must be followed by whitespace, and closing delimiters (`]#` or `]##`) must be preceded by whitespace.

```hw
#[ Multi-line comment for developers - ignored by compiler ]#

##[ Multi-line documentation extracted by hwsd for auto-docs ]##
```

**Documentation Generation**:
```bash
hwsd generate 5v_regulator.hw
# Outputs: HTML documentation with all ## comments formatted
```

**Why This Matters**:
- Auto-generated documentation (like Rust's rustdoc or Elixir's HexDocs)
- LLMs can read hwsd output to learn how to use libraries
- No separate docs to maintain (docs live in code)
- Community libraries are self-documenting

---

### 6. Import System

Import components, materials, and modules from the standard library or external packages.

**Syntax**:
```hw
import <Name> from <Source>
```

**Standard Library Imports**:
```hw
# Materials
import Copper from standard.materials
import FR4 from standard.materials
import Silicon from standard.materials

# Power components
import Battery from standard.power
import VoltageRegulator from standard.power

# Sensors
import LightSensor from standard.sensors
import TemperatureSensor from standard.sensors

# Logic components
import AND_Gate from standard.logic
import NOT_Gate from standard.logic
import Comparator from standard.analog

# Switches
import Transistor from standard.switches
```

**Package Registry Imports**:
```hw
# From hardware package registry
import Regulator from "@power/5v-regulator"
import IMU from "@sensors/imu-module"
import HBridge from "@motor/h-bridge"
import BluetoothModule from "@hardware/comms"
```

**Local File Imports**:
```hw
# Import local modules
import "power_system.hw" as Power
import "sensor_array.hw" as Sensors
import "./modules/motor_controller.hw" as Motors
```

**Silicon Library Imports**:
```hw
# For chip design
import CMOS_NAND_Gate from standard.silicon.logic
import CMOS_Inverter from standard.silicon.logic
import SRAM_Cell from standard.silicon.memory
```

---

### 7. Module System and Exposed Pins

Create reusable sub-assemblies by exposing internal pins for external connections.

**Syntax**:
```hw
expose <InternalPin> as <ExternalName>
```

**Example - Creating a Library Module**:
```hw
## A complete 5V voltage regulator module
define Space "5V_Regulator":
    dimensions: 20mm by 15mm by 2mm
    grid: 40 by 30 by 2

    # Internal components (pre-placed)
    add IC_LM7805 named Regulator at [20, 15, 1]
    add Capacitor (10uF) named C1 at [10, 15, 1]
    add Capacitor (1uF) named C2 at [30, 15, 1]

    # Internal routing (pre-wired)
    route C1.Plus to Regulator.In:
        path:
            - [10, 15, 1]
            - [20, 15, 1]

    route Regulator.Out to C2.Plus:
        path:
            - [20, 15, 1]
            - [30, 15, 1]

    # Exposed pins for external connections
    expose Regulator.In as VIN
    expose Regulator.Out as VOUT
    expose Regulator.GND as GND
```

**Using the Module**:
```hw
import Regulator from "@power/5v-regulator"

define Space "MyRobot":
    dimensions: 100mm by 100mm by 2mm
    grid: 200 by 200 by 2

    # Place the entire pre-routed module as one piece
    add Regulator named PowerSupply at [50, 50, 1]

    # Connect to the exposed pins
    route Battery.Plus to PowerSupply.VIN:
        path:
            - [10, 10, 1]
            - [50, 50, 1]

    route PowerSupply.VOUT to ESP32.VIN:
        path:
            - [50, 50, 1]
            - [150, 150, 1]
```

**What Happens Internally**:

When the compiler encounters `add Regulator named PowerSupply at [50, 50, 1]`:

1. Load the library: Parse `@power/5v-regulator.hw`
2. Get local grid: The library has a 40×30×2 tensor grid
3. Map to global coordinates: Place the 40×30 block at position [50, 50, 1]
4. Translate coordinates: All internal coordinates become global
5. Expose pins: `PowerSupply.VIN`, `PowerSupply.VOUT`, `PowerSupply.GND` become connection points


---

## Part III: Abstraction Blocks

Hardware Script uses explicit abstraction blocks to separate different levels of hardware description. This allows the same `.hw` file to be compiled for different targets (PCB, FPGA, ASIC, SPICE, visualization).

### The Five Abstraction Levels

1. **pins:** - Interface definition (universal across all targets)
2. **behavior:** - Logic/HDL level (for FPGA/ASIC synthesis)
3. **layout:** - Geometry/physical level (for PCB/ASIC placement)
4. **electrical:** - Circuit/SPICE level (for analog simulation)
5. **render:** - Visual/3D asset mapping (for visualization and rendering)

### Component Definition with Abstraction Blocks

**Syntax**:
```hw
define component "ComponentName":
    # 1. INTERFACE (The Pins/Ports)
    pins:
        <pin_definitions>
    
    # 2. BEHAVIOR (The Logic / HDL level)
    behavior:
        <logic_expressions>
    
    # 3. PHYSICAL (The Geometry / Layout level)
    layout:
        <geometry_definitions>
    
    # 4. ELECTRICAL (The Circuit / SPICE level)
    electrical:
        <electrical_characteristics>
    
    # 5. VISUAL (The 3D Asset Mapping)
    render:
        <3d_asset_mapping>
```

### Example: Custom ALU Component

```hw
define component "Custom_ALU":
    # 1. INTERFACE (The Pins/Ports)
    # Universal across all targets
    pins:
        A[8]           # 8-bit input A
        B[8]           # 8-bit input B
        Sum[8]         # 8-bit output Sum
        Carry          # Carry output
        VCC            # Power
        GND            # Ground
    
    # 2. BEHAVIOR (The Logic / HDL level)
    # Compiler synthesizes this into gates for FPGA/ASIC targets
    # Ignored for PCB target (treated as black box)
    behavior:
        Sum = A + B
        Carry = (A[7] AND B[7]) OR ((A[7] XOR B[7]) AND CarryIn)
    
    # 3. PHYSICAL (The Geometry / Layout level)
    # Used for PCB placement and mechanical constraints
    # Used for ASIC layout generation
    layout:
        shape: Rectangle(10mm, 10mm, 2mm)
        pin_spacing: 0.5mm
        pin_positions:
            A[0..7] at left_edge
            B[0..7] at right_edge
            Sum[0..7] at top_edge
            VCC at [0, 0]
            GND at [10, 10]
    
    # 4. ELECTRICAL (The Circuit / SPICE level)
    # Used for analog simulation
    # Optional for digital components
    electrical:
        max_voltage: 5V
        max_current: 100mA
        input_capacitance: 10pF
        output_resistance: 50Ω
    
    # 5. VISUAL (The 3D Asset Mapping)
    # Used for photorealistic rendering and visualization
    # Maps mathematical voxels to 3D assets
    render:
        asset: "assets/alu_chip.glb"
        origin_offset: [0, 0, 0]
        scale: 1.0
        fallback_procedural:
            shape: chip
            body_color: "#222222"
            label: "ALU"
```

### How Compiler Targets Interpret Blocks

**Target: PCB**
```
Reads: pins, layout
Ignores: behavior, electrical, render (treats as black box)
Output: Gerber files with component footprint
```

**Target: FPGA**
```
Reads: pins, behavior
Ignores: layout, render (FPGA tools handle placement)
Output: Verilog/VHDL for synthesis
```

**Target: ASIC**
```
Reads: pins, behavior, layout, electrical
Ignores: render (not needed for silicon)
Output: GDSII with synthesized gates
```

**Target: SPICE**
```
Reads: pins, electrical
Ignores: behavior, render (or converts to SPICE model)
Output: SPICE netlist for analog simulation
```

**Target: Simulation**
```
Reads: pins, behavior
Ignores: layout, electrical, render
Output: .hwx simulation executable
```

**Target: Visualization**
```
Reads: pins, layout, render
Ignores: behavior, electrical
Output: Blender scripts, 3D models (.obj, .glb)
```

### Pins Block Syntax

**Basic Pin Definition**:
```hw
pins:
    VCC
    GND
    Input
    Output
```

**Pin Arrays**:
```hw
pins:
    Data[8]        # 8-bit bus
    Address[16]    # 16-bit address bus
```

**Pin Positions** (for layout):
```hw
pins:
    VCC at [0, 0]
    GND at [10, 10]
    Input at [5, 0]
    Output at [5, 10]
```

### Behavior Block Syntax

**Boolean Expressions**:
```hw
behavior:
    Output = Input1 AND Input2
    LED_on = NOT(Light_sensor_high)
```

**Arithmetic**:
```hw
behavior:
    Sum = A + B
    Product = A * B
    Difference = A - B
```

**Conditional Logic**:
```hw
behavior:
    if Temperature > 85C:
        Fan_Enable = true
    else:
        Fan_Enable = false
```

### Layout Block Syntax

**Shape Definition**:
```hw
layout:
    shape: Rectangle(10mm, 10mm, 2mm)
    # or
    shape: Cylinder(radius: 5mm, height: 10mm)
```

**Pin Positions**:
```hw
layout:
    pin_positions:
        VCC at [0, 0]
        GND at [10, 10]
        Data[0..7] at left_edge
```

**Keep-Out Zones**:
```hw
layout:
    keepout:
        - region [2, 2] to [8, 8] height 5mm
```

### Electrical Block Syntax

**Voltage and Current**:
```hw
electrical:
    max_voltage: 5V
    min_voltage: 3.3V
    max_current: 100mA
    typical_current: 50mA
```

**Impedance and Capacitance**:
```hw
electrical:
    input_capacitance: 10pF
    output_resistance: 50Ω
    input_impedance: 1MΩ
```

**SPICE Model Reference**:
```hw
electrical:
    spice_model: "2N2222.lib"
    model_name: "Q2N2222"
```

### Render Block Syntax

The `render:` block maps the mathematical voxel representation to visual 3D assets for photorealistic rendering.

**Asset-Based Rendering** (for complex components):
```hw
render:
    asset: "assets/esp32_chip.glb"
    origin_offset: [0, 0, 0]
    scale: 1.0
```

**Procedural Rendering** (for simple components):
```hw
render:
    type: procedural
    shape: smd_passive
    body_color: "#111111"
    endcap_color: "#C0C0C0"
    label: "{value}"
```

**Complete Example with Fallback**:
```hw
render:
    # Primary: Use high-fidelity 3D asset
    asset: "assets/component.glb"
    origin_offset: [0, 0, 0]
    scale: 1.0
    
    # Fallback: If asset not available, generate procedurally
    fallback_procedural:
        shape: box
        color: "#222222"
        material: "matte_plastic"
        label: "COMPONENT"
```

**Available Procedural Shapes**:
- `smd_passive` - Standard SMD resistor/capacitor
- `box` - Simple rectangular box
- `cylinder` - Cylindrical (for through-hole components)
- `chip` - Generic IC package
- `connector` - Pin headers

**Render Block Properties**:

| Property | Type | Description | Required |
|----------|------|-------------|----------|
| `asset` | String | Path to `.glb` 3D model file | No |
| `origin_offset` | [X, Y, Z] | Offset from voxel grid origin | No (default: [0,0,0]) |
| `scale` | Float | Scale factor for 3D model | No (default: 1.0) |
| `type` | String | "procedural" or "asset" | No (inferred) |
| `shape` | String | Procedural shape type | For procedural |
| `body_color` | Hex | Body color for procedural | For procedural |
| `label` | String | Text label on component | No |
| `fallback_procedural` | Block | Fallback if asset unavailable | No |

**When to Use Each Approach**:

Use **asset-based rendering** for:
- Complex components (microcontrollers, sensors, modules)
- Components with intricate geometry
- Marketing and documentation renders
- When photorealism is important

Use **procedural rendering** for:
- Simple passive components (resistors, capacitors, LEDs)
- Standard packages (SOT-23, SOIC, QFP)
- Lightweight visualization
- When file size matters

### Complete Example: Component with All Five Blocks

Here's a comprehensive example showing a custom ESP32 component definition with all five abstraction blocks:

```hw
define Component "ESP32_WROOM_32":
    # 1. PINS - Interface definition (universal)
    pins:
        GND at [0, 5, 1]
        VCC at [0, 10, 1]
        EN at [0, 15, 1]
        GPIO0 at [50, 5, 1]
        GPIO2 at [50, 10, 1]
        GPIO4 at [50, 15, 1]
        TX at [100, 5, 1]
        RX at [100, 10, 1]
    
    # 2. BEHAVIOR - Logic/HDL level (for simulation)
    behavior:
        # WiFi state machine
        if EN == HIGH:
            WiFi_Active = true
            if GPIO0 == LOW:
                Boot_Mode = "Flash"
            else:
                Boot_Mode = "Normal"
        else:
            WiFi_Active = false
    
    # 3. LAYOUT - Physical geometry (for PCB/placement)
    layout:
        shape: Rectangle(18mm, 25mm, 3mm)
        pin_spacing: 1.27mm
        pin_positions:
            GND at [0, 5]
            VCC at [0, 10]
            EN at [0, 15]
            GPIO0 at [18, 5]
            GPIO2 at [18, 10]
            GPIO4 at [18, 15]
            TX at [9, 0]
            RX at [9, 25]
        keepout:
            - region [2, 2] to [16, 23] height 5mm  # Antenna clearance
    
    # 4. ELECTRICAL - Circuit characteristics (for SPICE)
    electrical:
        max_voltage: 3.6V
        min_voltage: 2.3V
        typical_voltage: 3.3V
        max_current: 500mA
        typical_current: 80mA
        sleep_current: 5uA
        input_capacitance: 5pF
        output_resistance: 40Ω
    
    # 5. RENDER - Visual/3D mapping (for visualization)
    render:
        # Use high-fidelity 3D model for photorealistic renders
        asset: "assets/esp32_wroom_32.glb"
        origin_offset: [0, 0, 0]
        scale: 1.0
        
        # Fallback for lightweight mode
        fallback_procedural:
            shape: chip
            body_color: "#C0C0C0"  # Silver shield
            label: "ESP32"
            material: "metallic"
```

**How different targets use this component**:

- `--target pcb`: Uses pins + layout, generates footprint
- `--target fpga`: Uses pins + behavior, synthesizes WiFi logic
- `--target spice`: Uses pins + electrical, simulates power consumption
- `--target sim`: Uses pins + behavior, runs behavioral simulation
- `--target viz`: Uses pins + layout + render, creates photorealistic 3D model


---

## Part IV: Complete Examples

### Example 1: Simple LED Circuit (PCB)

```hw
# Import components from standard library
import Battery from standard.power
import LED from standard.passive
import Resistor from standard.passive

# Define the physical workspace
define Space "LED_Board":
    dimensions: 20mm by 20mm by 2mm
    grid: 20 by 20 by 2
    origin: tl  # Default - software-friendly
    
    # Add FR4 substrate
    add Substrate(FR4) spanning [1, 1, 1] to [20, 20, 2]
    
    # Place components
    add Battery (5V) named Power at [5, 5, 1]
    add Resistor (220Ω) named R1 at [10, 10, 1]
    add LED (Red) named LED1 at [15, 15, 1]
    
    # Route connections
    route Power.Plus to R1.Pin1:
        path:
            - [5, 5, 1]
            - [10, 10, 1]
    
    route R1.Pin2 to LED1.Anode:
        path:
            - [10, 10, 1]
            - [15, 15, 1]
    
    route LED1.Cathode to Power.Minus:
        path:
            - [15, 15, 1]
            - [15, 5, 1]
            - [5, 5, 1]
```

### Example 2: Automated Sprinkler Controller

```hw
# Import components from standard library
import Copper from standard.materials
import Battery from standard.power
import LightSensor from standard.sensors
import Comparator from standard.analog
import VoltageDivider from standard.analog
import Transistor from standard.switches

# Define the physical workspace
define Space "Automated_Sprinkler":
    dimensions: 50mm by 50mm by 2mm
    grid: 25 by 25 by 2
    origin: tl  # Software-friendly coordinate system
    
    # Add FR4 substrate across all layers
    add Substrate(FR4) spanning [1, 1, 1] to [25, 25, 2]
    
    # Place components using XYZ tensor coordinates
    add Battery (5V) named PowerSource at [5, 5, 1]
    add LightSensor named Eye at [12, 12, 1]
    add VoltageDivider (ratio: 0.5) named Threshold at [15, 15, 1]
    add Comparator named LogicGate at [18, 18, 1]
    add Transistor named Valve at [20, 20, 1]
    
    # Route electrical connections with explicit waypoints
    route PowerSource.out to Eye.in:
        path:
            - [5, 5, 1]      # Start at battery
            - [12, 5, 1]     # Travel horizontally
            - [12, 12, 1]    # Arrive at sensor
    
    route PowerSource.out to Threshold.in:
        path:
            - [5, 5, 1]
            - [15, 5, 1]
            - [15, 15, 1]
    
    route Threshold.out to LogicGate.negative_pin:
        path:
            - [15, 15, 1]
            - [18, 15, 1]
            - [18, 18, 1]
    
    route Eye.out to LogicGate.positive_pin:
        path:
            - [12, 12, 1]    # Start at sensor
            - [12, 12, 2]    # Via to layer 2
            - [18, 12, 2]    # Route on bottom layer
            - [18, 18, 2]    # Position under comparator
            - [18, 18, 1]    # Via back to top layer
    
    route LogicGate.out to Valve.base:
        path:
            - [18, 18, 1]
            - [20, 18, 1]
            - [20, 20, 1]
    
    # Connect to external motor
    route Valve.collector to "EXTERNAL_SPRINKLER_MOTOR":
        path:
            - [20, 20, 1]
            - [25, 20, 1]
```med Valve at [20, 20, 1]
    
    # Route electrical connections with explicit waypoints
    route PowerSource.out to Eye.in:
        path:
            - [5, 5, 1]      # Start at battery
            - [12, 5, 1]     # Travel horizontally
            - [12, 12, 1]    # Arrive at sensor
    
    route PowerSource.out to Threshold.in:
        path:
            - [5, 5, 1]
            - [15, 5, 1]
            - [15, 15, 1]
    
    route Threshold.out to LogicGate.negative_pin:
        path:
            - [15, 15, 1]
            - [18, 15, 1]
            - [18, 18, 1]
    
    route Eye.out to LogicGate.positive_pin:
        path:
            - [12, 12, 1]    # Start at sensor
            - [12, 12, 2]    # Via to layer 2
            - [18, 12, 2]    # Route on bottom layer
            - [18, 18, 2]    # Position under comparator
            - [18, 18, 1]    # Via back to top layer
    
    route LogicGate.out to Valve.base:
        path:
            - [18, 18, 1]
            - [20, 18, 1]
            - [20, 20, 1]
    
    # Connect to external motor
    route Valve.collector to "EXTERNAL_SPRINKLER_MOTOR":
        path:
            - [20, 20, 1]
            - [25, 20, 1]
```

### Example 3: Silicon Chip Design (4-bit ALU)

```hw
# Import silicon components
import Silicon from standard.silicon.materials
import SiliconDioxide from standard.silicon.materials
import CMOS_NAND_Gate from standard.silicon.logic
import CMOS_XOR_Gate from standard.silicon.logic
import CMOS_Adder_Full from standard.silicon.logic

define space "ALU_4bit":
    dimensions: 1mm by 1mm by 0.05mm
    grid: 10000 by 10000 by 5
    origin: bl  # Traditional for silicon design
    # Voxel = 100 nanometers
    
    # Silicon substrate
    add Substrate(Silicon) spanning [1, 1, 1] to [10000, 10000, 1]
    
    # Dielectric layers for metal routing
    add Substrate(SiliconDioxide) spanning [1, 1, 2] to [10000, 10000, 5]
    
    # Place 4 full adders for 4-bit addition
    add CMOS_Adder_Full named Adder0 at [1000, 1000, 1]
    add CMOS_Adder_Full named Adder1 at [3000, 1000, 1]
    add CMOS_Adder_Full named Adder2 at [5000, 1000, 1]
    add CMOS_Adder_Full named Adder3 at [7000, 1000, 1]
    
    # Route carry chain on Metal Layer 1
    route Adder0.carry_out to Adder1.carry_in:
        path:
            - [1000, 1000, 1]
            - [1000, 1000, 2]  # Up to Metal1
            - [3000, 1000, 2]  # Route to next adder
            - [3000, 1000, 1]  # Down to silicon
    
    route Adder1.carry_out to Adder2.carry_in:
        path:
            - [3000, 1000, 1]
            - [3000, 1000, 2]
            - [5000, 1000, 2]
            - [5000, 1000, 1]
    
    route Adder2.carry_out to Adder3.carry_in:
        path:
            - [5000, 1000, 1]
            - [5000, 1000, 2]
            - [7000, 1000, 2]
            - [7000, 1000, 1]
```

### Example 4: Modular Robot Controller

**Main Entry Point (main.hw)**:
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
    add Power at [50, 50, 1]
    add Sensors at [150, 150, 1]
    add Motors at [100, 800, 1]

    # Connect modules together
    route Power.VOUT to Sensors.VIN:
        path:
            - [50, 50, 1]
            - [150, 150, 1]

    route Power.VOUT to Motors.VIN:
        path:
            - [50, 50, 1]
            - [100, 800, 1]
```

**Power Module (power_system.hw)**:
```hw
## Power Distribution System
## Provides regulated 5V and 3.3V rails from 12V input
define Space "PowerSystem":
    dimensions: 40mm by 30mm by 2mm
    grid: 80 by 60 by 2

    add IC_LM7805 named Reg5V at [1, 20, 15]
    add IC_LM1117 named Reg3V3 at [1, 60, 15]

    # Internal routing
    route Reg5V.In to Reg3V3.In:
        path:
            - [1, 20, 15]
            - [1, 60, 15]

    # Expose pins for external connections
    expose Reg5V.In as VIN
    expose Reg5V.Out as VOUT_5V
    expose Reg3V3.Out as VOUT_3V3
    expose Reg5V.GND as GND
```


---

## Part V: Grammar and Syntax Rules

### Formal Grammar (EBNF)

```ebnf
program = { import_stmt } space_def { component_def | route_def }

import_stmt = "import" identifier "from" source

space_def = "define" "Space" STRING ":"
            "dimensions" ":" measure "by" measure "by" measure
            "grid" ":" number "by" number "by" number
            { substrate_def }

substrate_def = "add" "Substrate" "(" identifier ")" "spanning" 
                coordinate "to" coordinate

component_def = "define" "Component" STRING ":"
                [ pins_block ]
                [ behavior_block ]
                [ layout_block ]
                [ electrical_block ]
                [ render_block ]

pins_block = "pins" ":" { pin_def }

behavior_block = "behavior" ":" { behavior_expr }

layout_block = "layout" ":" { layout_expr }

electrical_block = "electrical" ":" { electrical_expr }

render_block = "render" ":" { render_expr }

render_expr = ( "asset" ":" STRING )
            | ( "origin_offset" ":" coordinate )
            | ( "scale" ":" number )
            | ( "type" ":" STRING )
            | ( "shape" ":" identifier )
            | ( "body_color" ":" STRING )
            | ( "endcap_color" ":" STRING )
            | ( "label" ":" STRING )
            | ( "fallback_procedural" ":" { render_expr } )

component_placement = "add" identifier [ "(" parameters ")" ] 
                      "named" identifier 
                      "at" coordinate 
                      [ "rotated" direction ]

route_def = "route" pin_ref "to" pin_ref ":"
            "path" ":" waypoint_list

expose_def = "expose" pin_ref "as" identifier

waypoint_list = { "-" coordinate }

coordinate = "[" number "," number "," number "]"

pin_ref = identifier [ "." identifier ]

direction = "North" | "South" | "East" | "West"

measure = number unit

unit = "mm" | "cm" | "V" | "A" | "mA" | "Hz" | "kHz" | "MHz" | "GHz" | "C" | "Ω" | "kΩ" | "MΩ" | "F" | "pF" | "nF" | "uF"

identifier = LETTER { LETTER | DIGIT | "_" }

number = DIGIT { DIGIT } [ "." DIGIT { DIGIT } ]

STRING = '"' { ANY_CHAR } '"'
```

### Token Types

The lexer recognizes these token patterns:

| Token | Pattern | Example |
|-------|---------|---------|
| KEYWORD | Reserved words | `define`, `Space`, `add`, `route` |
| STRING | `"..."` | `"BoardName"` |
| COORD | `[n, n, n]` | `[1, 5, 10]` |
| MEASURE | number + unit | `20mm`, `5V`, `100kHz` |
| NUMBER | Integer or float | `100`, `3.14` |
| IDENTIFIER | Name or path | `Power.Out`, `LED1` |
| COLON | `:` | `:` |
| LIST_ITEM | `- [n, n, n]` | `- [1, 5, 6]` |
| COMMENT | `#...` | `# This is a comment` |
| DOC_COMMENT | `##...` | `## Documentation` |

### Reserved Keywords

**Actions**:
- `define` - Start definition block
- `add` - Place component or substrate
- `route` - Define trace
- `import` - Import module or component
- `expose` - Expose internal pin

**Types**:
- `space` - Define workspace
- `component` - Define component
- `material` - Define material
- `substrate` - Base material layer

**Properties**:
- `dimensions` - Physical size
- `grid` - Tensor resolution
- `pins` - Pin definitions
- `behavior` - Logic block
- `layout` - Geometry block
- `electrical` - Circuit block
- `render` - Visual/3D asset mapping block

**Connectors**:
- `named` - Assign name
- `at` - Specify position
- `to` - Route destination
- `from` - Import source
- `by` - Separator in dimensions/grid
- `rotated` - Set orientation
- `spanning` - Define region
- `as` - Alias or expose name

**Directions**:
- `North`, `South`, `East`, `West` - Rotation directions

### Indentation Rules

**Current Implementation**: Indentation is recommended but not strictly enforced in v0.1.

**Recommended Style**:
```hw
define Space "Board":
    dimensions: 20mm by 20mm by 2mm
    grid: 20 by 20 by 2
    
    add Battery at [1, 5, 5]
    
    route Battery.Plus to LED.Anode:
        path:
            - [1, 5, 5]
            - [1, 15, 15]
```

**Future**: Strict indentation-based scoping will be enforced in v0.2+

### Naming Conventions

**Component Instances**:
- Use PascalCase or snake_case: `PowerSource`, `power_source`
- Must be unique within the space
- Should be descriptive: `MainSwitch` not `S1`

**Pin References**:
- Use dot notation: `Component.Pin`
- Examples: `Battery.Plus`, `LED.Anode`, `MCU.GPIO_12`

**Spaces**:
- Use PascalCase with underscores: `Robot_Motherboard`
- Should describe the board's purpose

**Files**:
- Use snake_case: `power_system.hw`, `sensor_array.hw`
- Should match the Space name (lowercase)

### Line Interpolation Algorithm (Manual Waypoint Routing Only)

**Book 2 Scope**: This section covers the line interpolation algorithm used specifically for **Manual Waypoint Routing** when users provide explicit waypoints in the `path:` section.

When users provide explicit waypoints in the `path:` section, the compiler uses linear interpolation (Bresenham's algorithm) to draw straight lines between consecutive waypoints. 

**Algorithm**: Bresenham's line algorithm fills all voxels along the straight-line path between two waypoints, ensuring no gaps in the copper trace.

**Example**:
```hw
route A to B:
    path:
        - [10, 10, 1]  # Waypoint 1
        - [20, 15, 1]  # Waypoint 2
```

The compiler draws a straight line from [10,10,1] to [20,15,1], filling intermediate voxels: [11,10,1], [12,11,1], [13,11,1], etc.

**Automatic Routing Alternative**: This is different from the auto-router's Manhattan routing strategy used when no waypoints are provided. For automatic routing algorithms (A* pathfinding, Manhattan routing, physics constraints), see Book 5 (Routing & Physics).

---

## Part VI: Compilation Targets

Hardware Script compiles to different targets based on the `--target` flag.

### Official Target List

Hardware Script supports the following compilation targets:

| Target | Purpose | Primary Output |
|--------|---------|----------------|
| `pcb` | PCB manufacturing | Gerber files, drill files |
| `fpga` | FPGA synthesis | Verilog/VHDL |
| `asic` | ASIC/chip design | GDSII layout files |
| `spice` | Analog circuit simulation | SPICE netlist |
| `sim` | Behavioral simulation | Simulation executable |
| `viz` | 3D visualization & rendering | Blender scripts, 3D models |
| `web` | Web-based interactive viewer | WebAssembly, HTML viewer |

### Target: PCB (Default)

**Command**:
```bash
hws build board.hw
hws build board.hw --target pcb
```

**What it reads**:
- `pins:` blocks (for connectivity)
- `layout:` blocks (for footprints)
- Routing waypoints

**What it ignores**:
- `behavior:` blocks (treats components as black boxes)
- `electrical:` blocks (optional, used for validation)

**Output files**:
- Gerber files (`.gtl`, `.gbl`, `.g1`, `.g2`)
- Drill files (`.drl`)
- 3D models (`.glb`, `.step`)
- Bill of Materials (`.csv`)

### Target: FPGA

**Command**:
```bash
hws build design.hw --target fpga
```

**What it reads**:
- `pins:` blocks (for I/O)
- `behavior:` blocks (for logic synthesis)

**What it ignores**:
- `layout:` blocks (FPGA tools handle placement)

**Output files**:
- Verilog/VHDL (`.v`, `.vhd`)
- Constraint files (`.xdc`, `.sdc`)

### Target: ASIC

**Command**:
```bash
hws build chip.hw --target asic
```

**What it reads**:
- All blocks: `pins:`, `behavior:`, `layout:`, `electrical:`

**Output files**:
- GDSII (`.gds`)
- Verilog/VHDL (`.v`, `.vhd`)
- LEF/DEF files

### Target: SPICE

**Command**:
```bash
hws build circuit.hw --target spice
```

**What it reads**:
- `pins:` blocks (for connectivity)
- `electrical:` blocks (for SPICE models)

**What it ignores**:
- `behavior:` blocks (or converts to SPICE)

**Output files**:
- SPICE netlist (`.sp`, `.cir`)

### Target: Simulation

**Command**:
```bash
hws build system.hw --target sim
```

**What it reads**:
- `pins:` blocks (for connectivity)
- `behavior:` blocks (for behavioral simulation)

**Output files**:
- Simulation executable (`.hwx`)
- Waveform data (`.vcd`)

### Target: Visualization

**Command**:
```bash
hws build board.hw --target viz
```

**What it reads**:
- `pins:` blocks (for connectivity)
- `layout:` blocks (for geometry)
- `render:` blocks (for 3D assets)
- Routing waypoints

**What it ignores**:
- `behavior:` blocks (not needed for visualization)

**Output files**:
- Blender Python script (`.py`)
- OBJ 3D model (`.obj`)
- GLB 3D model (`.glb`)

**Purpose**: Generate photorealistic 3D renders for marketing, documentation, and visualization.

### Target: Web (Future)

**Command**:
```bash
hws build board.hw --target web
```

**What it reads**:
- `pins:` blocks (for connectivity)
- `layout:` blocks (for geometry)
- `render:` blocks (for 3D assets)
- Routing waypoints

**Output files**:
- WebAssembly module (`.wasm`)
- Three.js scene (`.json`)
- HTML viewer (`index.html`)

**Purpose**: Interactive 3D viewer that runs in web browsers without installation.


---

## Book Separation Reference

**This Book (Book 2: Language Spec) Covers**:
- Manual waypoint routing syntax (`path:` section with explicit waypoints)
- Bresenham line interpolation between user-defined waypoints
- Complete routing syntax and examples
- User-controlled trace geometry
- All language syntax and semantics

**Book 5 (Routing & Physics) Covers**:
- Automatic routing algorithms (A* pathfinding, Manhattan routing)
- Physics constraint generation (clearance, trace width, crosstalk)
- Material property translation to geometric constraints
- Design rule checking and validation
- Routing engine internals and algorithms

**The Strict Division**:
- **Book 2 owns**: Manual Waypoint Routing (what the user types: `path: -[1,1,1]`)
- **Book 5 owns**: Automatic Routing (what the A* algorithm does when user doesn't provide waypoints)

For automatic routing algorithms, physics constraints, and routing engine internals, see Book 5 (Routing & Physics).

---

## Part VII: Standard Library Reference

To see the full list of available standard library components and materials, see Book 3 (Ecosystem), Part III: Package Management.

---

## Part VIII: Error Handling and Validation

Hardware Script provides clear, actionable error messages with short, memorable error codes to help you understand and fix issues in your designs. This section shows what errors look like and how to interpret them. 

**Error Code Format**: `[Letter][Digit][Digit]` (e.g., P16, R12, S22)
- **S** - Syntax errors (you typed something wrong)
- **C** - Compiler errors (logic is flawed)
- **R** - Routing errors (physical placement failed)
- **P** - Physics errors (laws of physics violated)
- **M** - Manufacturing errors (factory can't build it)

For complete error handling philosophy and implementation details, see `Docs/v0.1.2/ERROR-HANDLING-PHILOSOPHY.md` and `Docs/v0.1.3/COMPILER-INTERNALS.md`.

### Compile-Time Errors

**Syntax Errors (S-codes)**:
```
Error[S21]: Invalid Coordinate Format
  ╭─[board.hw:12:1]
12 │  add Battery named Power at PowerPosition
   ·                             ──────┬──────
   ·                                   ╰── Expected coordinate format [X,Y,Z]
   ╰────
  hint: Use coordinate format like [1, 10, 10]
```

**Out of Bounds (R11)**:
```
Error[R11]: Coordinate Out of Bounds
  ╭─[board.hw:15:1]
15 │  add LED named Status at [1, 150, 50]
   ·                          ─────┬─────
   ·                               ╰── Coordinate exceeds grid bounds
   ╰────
  help: Grid size is 100 × 100 × 2
  hint: Reduce coordinate or increase grid size in Space definition
```

**Component Not Found (C11)**:
```
Error[C11]: Component Not Found
  ╭─[board.hw:20:1]
20 │  route PowerSource.VCC to Sensor.VCC:
   ·        ─────┬─────
   ·             ╰── Component 'PowerSource' not found in scope
   ╰────
  hint: Did you mean 'Power'? Check component names defined earlier.
```

**Component Collision (R12)**:
```
Error[R12]: Component Collision Detected
  ╭─[board.hw:15:1]
15 │  add Battery named Power at [1, 10, 10]
   ·  ────────┬────────
   ·          ╰── First component occupies [1,10,10] to [1,15,15]
17 │  add LED named Status at [1, 11, 10]
   ·  ─────────┬─────────
   ·           ╰── Second component overlaps here
   ╰────
  help: Components overlap by 3×3 voxels
  hint: Move LED to [1, 16, 10] or later in the file
```

### Physics Validation

**The Famous P16 (Dielectric Breakdown)**:
```
Error[P16]: Dielectric Breakdown Risk
  ╭─[board.hw:42:1]
42 │  route Power_120V.out to Relay.in:
   ·        ──────┬───────
   ·              ╰── High voltage net (120V)
45 │      - [1, 50, 60]
   ·        ─────┬─────
   ·             ╰── Approaches within 0.02mm of MCU_5V net here
   ╰────
  help: The voltage difference is 115V. Through Air, this requires a 
        minimum clearance of 0.08mm to prevent arcing.
  hint: Move the waypoint to [1, 50, 62] or change material to FR4.
```

**Trace Too Thin (P21)**:
```
Error[P21]: Trace Width Insufficient
  ╭─[board.hw:28:1]
28 │  route Motor.Power to Battery.Plus:
   ·        ─────┬─────
   ·             ╰── This net carries 10A
31 │      - [1, 80, 20]
   ·        ─────┬─────
   ·             ╰── Trace width: 0.5mm (too thin)
   ╰────
  help: For 10A through 1oz copper with 10°C temperature rise, minimum 
        trace width is 2.5mm.
  hint: Add 'width: 3mm' to the route block, or reduce current to 2A.
```

**Component Overheating (P22)**:
```
Error[P22]: Component Temperature Exceeds Maximum
  ╭─[board.hw:18:1]
18 │  add VoltageRegulator named Reg at [1, 20, 20]
   ·  ─────────┬─────────
   ·           ╰── Component temperature: 95°C (max: 85°C)
   ╰────
  help: Regulator is dissipating 2.5W with insufficient cooling.
  hint: Add heatsink or increase copper pour area around component.
```

### Manufacturing Errors

**Trace Too Thin for Factory (M11)**:
```
Error[M11]: Trace Thinner Than Factory Minimum
  ╭─[board.hw:35:1]
35 │  route Signal.Data to CPU.Pin5:
   ·        ──────┬──────
   ·              ╰── Trace width: 0.08mm
   ╰────
  help: Factory minimum trace width is 0.15mm for standard PCB process.
  hint: Increase trace width or select advanced PCB process.
```

### Runtime Validation

**Test Bench Failures**:
```
❌ Test Failed: Power Regulator Voltage Output
  Expected: 5V ± 0.1V
  Actual: 4.2V
  at test_power.hwt:15
```

### Common Error Codes Quick Reference

| Code | Description | Category |
|------|-------------|----------|
| S11  | Missing colon | Syntax |
| S21  | Invalid coordinate format | Syntax |
| S22  | Unknown unit format | Syntax |
| C11  | Component not found | Compiler |
| C12  | Pin does not exist | Compiler |
| C31  | Multiple Space definitions | Compiler |
| R11  | Coordinate out of bounds | Routing |
| R12  | Component collision | Routing |
| R21  | No route found | Routing |
| P16  | Dielectric breakdown (THE FAMOUS ONE) | Physics |
| P21  | Trace too thin for current | Physics |
| P22  | Component overheating | Physics |
| P31  | Impedance mismatch | Physics |
| M11  | Trace too thin for factory | Manufacturing |
| M12  | Via drill hole too small | Manufacturing |

**When you see an error**: Just say the code (e.g., "I'm getting P16") and search "Hardware Script P16" for documentation.

---

## Part IX: Best Practices

### Code Organization

**1. Use Modules for Reusability**:
```hw
# Bad: Everything in one file
define Space "ComplexBoard":
    # 500 lines of components and routing...

# Good: Modular design
import "power_system.hw" as Power
import "sensor_array.hw" as Sensors
import "motor_controller.hw" as Motors
```

**2. Document with ## Comments**:
```hw
## Power Distribution Module
## Provides regulated 5V and 3.3V from 12V input
## WARNING: Requires heatsink above 500mA load
define Space "PowerSystem":
    ...
```

**3. Use Descriptive Names**:
```hw
# Bad
add Battery named B1 at [1, 5, 5]
add LED named L1 at [1, 10, 10]

# Good
add Battery (5V) named MainPower at [1, 5, 5]
add LED (Red) named StatusIndicator at [1, 10, 10]
```

### Routing Best Practices

**1. Minimize Trace Length**:
```hw
# Bad: Unnecessary detour
route A to B:
    path:
        - [1, 5, 5]
        - [1, 5, 50]
        - [1, 50, 50]
        - [1, 50, 10]

# Good: Direct path
route A to B:
    path:
        - [1, 5, 5]
        - [1, 50, 10]
```

**2. Use Layer Changes Strategically**:
```hw
# Use bottom layer to avoid crossing traces
route Sensor.out to MCU.input:
    path:
        - [1, 10, 10]
        - [2, 10, 10]    # Via to bottom layer
        - [2, 50, 10]    # Route on bottom
        - [1, 50, 10]    # Via back to top
```

**3. Group Related Signals**:
```hw
# Route power traces first
route Power.VCC to IC1.VCC:
    ...
route Power.VCC to IC2.VCC:
    ...

# Then signal traces
route IC1.out to IC2.in:
    ...
```

### Component Placement

**1. Place Power Components First**:
```hw
# Power supply near board edge
add PowerJack named Input at [1, 5, 50]
add VoltageRegulator named Reg at [1, 10, 50]

# Then distribute to other components
add MCU named Controller at [1, 50, 50]
```

**2. Keep Related Components Close**:
```hw
# Sensor and its pull-up resistor
add Sensor named Temp at [1, 30, 30]
add Resistor (10kΩ) named R_Pullup at [1, 32, 30]
```

**3. Consider Thermal Management**:
```hw
# Keep hot components away from sensitive sensors
add VoltageRegulator named Reg at [1, 10, 10]  # Hot
add TemperatureSensor named Temp at [1, 90, 90]  # Far away
```

---

## Part X: Common Patterns

### Pattern 1: Power Distribution

```hw
## Power distribution to multiple components
define Space "PowerBus":
    dimensions: 50mm by 50mm by 2mm
    grid: 100 by 100 by 2
    
    add Battery (5V) named Power at [1, 10, 10]
    add IC1 at [1, 30, 30]
    add IC2 at [1, 50, 50]
    add IC3 at [1, 70, 70]
    
    # Star topology from power source
    route Power.VCC to IC1.VCC:
        path: [...]
    route Power.VCC to IC2.VCC:
        path: [...]
    route Power.VCC to IC3.VCC:
        path: [...]
```

### Pattern 2: Signal Bus

```hw
## 8-bit data bus routing
define Component "DataBus":
    pins:
        Data[8]
    
    # Route all 8 lines in parallel
    route CPU.Data[0] to RAM.Data[0]:
        path: [...]
    route CPU.Data[1] to RAM.Data[1]:
        path: [...]
    # ... repeat for all 8 bits
```

### Pattern 3: Differential Pair

```hw
## USB differential pair routing
route USB.D_Plus to MCU.USB_DP:
    path:
        - [1, 10, 10]
        - [1, 50, 10]

route USB.D_Minus to MCU.USB_DN:
    path:
        - [1, 10, 12]  # Parallel to D_Plus
        - [1, 50, 12]
```

### Pattern 4: Ground Plane

```hw
## Ground plane on bottom layer
add Substrate(Copper) spanning [2, 1, 1] to [2, 100, 100]

# Connect all ground pins to plane
route IC1.GND to "GROUND_PLANE":
    path:
        - [1, 30, 30]
        - [2, 30, 30]  # Via to ground plane
```

---

## Conclusion

This language specification provides the complete syntax and semantics for writing Hardware Script code. The language is designed to be:

- **Human-readable**: Clear, English-like syntax
- **LLM-friendly**: Structured, predictable patterns
- **Universal**: Same language from PCBs to silicon chips
- **Modular**: Reusable components and sub-assemblies
- **Deterministic**: Reproducible builds

For additional resources:
- **LANGUAGE-DESIGN-PHILOSOPHY.md**: Deep dive into syntax rationale and standards compliance
- **VISION.md**: Understand the philosophy and goals
- **GETTING-STARTED.md**: Step-by-step tutorials
- **ARCHITECTURE.md**: Compiler internals and implementation
- **API Reference**: Standard library documentation

---

**Document Version**: 0.1.3 Consolidated Edition  
**Consolidated From**:
- LANGUAGE-SPEC.md (v0.1)
- SPECIFICATION.md (v0.0.1)
- QUICK-REFERENCE.md (v0.1)
- MODULARITY-AND-DOCUMENTATION.md (v0.0.1)
- FILE-EXTENSION-DEBATE-RESOLUTION.md (v0.1.2) - Abstraction blocks
- SILICON-DESIGN.md (v0.0.1) - Silicon syntax examples
- LANGUAGE-DESIGN-PHILOSOPHY.md (v0.1.2) - Ruby syntax rationale, SI units, standards compliance

**Last Updated**: March 2026  
**Status**: Language Reference  
**Part of**: Hardware Script v0.1.3 Documentation Suite

