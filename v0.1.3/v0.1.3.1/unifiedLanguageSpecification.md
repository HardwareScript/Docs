All references to fragmented file extensions (`.hwmat`, `.hwp`, `.hwx`, etc.) have been completely removed. The specification now reflects the **Unified Ecosystem** where everything is a top-level `define` block inside a `.hw` file, utilizing native SI unit parsing across the entire physical universe.

---

# Book 2: The Language Specification

**Hardware Script**  
**Target Audience**: Hardware engineers and LLMs writing .hw code  
**Last Updated**: March 2026

---

## Foundation

This document serves as the single source of truth for the Hardware Script language. It reflects the unified ecosystem architecture where **all physical reality is defined using a single syntax and a single file extension (`.hw`)**.

---

## Quick Reference Card

### Basic Syntax Patterns

```hw
# 1. Material Definition
define material "Copper":
    category: conductor
    properties:
        density: 8960 kg/m3
        resistivity: 1.68e-8 Ω/m

# 2. Profile Definition (Design Rules)
define profile "HighVoltage":
    trace:
        min_width: 254µm

# 3. Component Definition
define component "Resistor_0805":
    pins:
        Pin1
        Pin2
    layout:
        shape: Rectangle(2.0mm, 1.25mm, 0.5mm)

# 4. Space Definition (The Board/Chip)
define space "MainBoard":
    dimensions: 50mm by 50mm by 2mm
    grid: 500 by 500 by 2
    origin: tl by t
    profile: HighVoltage
    
    # Placement
    add Resistor_0805 named R1 at[x:10, y:15, z:1]
    
    # Routing
    route Power.Plus to R1.Pin1:
        path:
            - [x:10, y:10, z:1]
            -[x:10, y:15, z:1]
```

### The Unified File Extension

Hardware Script uses exactly **one file extension** for all logic, geometry, physics, and rules:
*   `.hw` - The Hardware Script source file.

*(Note: `hw.toml` and `hw.lock` are used strictly for package management and dependencies).*

### Coordinate System:[X, Y, Z] with Configurable Origin

**Coordinate Order**: Alphabetical order `[X, Y, Z]`
- **X** = Column (left to right)
- **Y** = Row (direction depends on XY origin)
- **Z** = Layer (direction depends on Z origin)

**Origin Points** (configurable per Space using unified `XY by Z` syntax):
- `tl by t` (default): Top-Left XY, Top-Down Z (like matrices/PCB design)
- `bl by t`: Bottom-Left XY, Top-Down Z (like CAD/CNC milling)

**Named (Declarative)**: Explicit labels, any order
```hw
add LED named Light at [x:10, y:15, z:2]
```

### Tri-Fold Case Sensitivity Rules

1. **Software Domain (Strictly Lowercase)**: All keywords (`define`, `space`, `material`, `add`, `route`, `tl`) must be lowercase.
2. **Physics Domain (SI Standard)**: Units must match standard SI casing (`mV`, `kΩ`, `MHz`).
3. **User Domain (Case-Sensitive)**: Identifiers and names (`ESP32`, `MainPower`) are exact-match.

---

## Part I: Language Overview

### The "Single-File" Superpower

Hardware Script is a **declarative, text-based hardware description language**. Unlike traditional EDA tools that use a fragmented mess of proprietary files and YAML databases, Hardware Script defines the entire physical universe in a unified syntax.

You can write an entire custom motherboard—including its custom materials, manufacturing rules, mechanical constraints, components, and routing—inside a single `.hw` file. This makes Hardware Script natively optimized for version control (Git), forum sharing, and LLM generation. 

While you *can* put everything in one file, best practice is to organize your project into modular `.hw` files (e.g., `materials.hw`, `profiles.hw`, `main.hw`) and `import` them.

### Coordinate System Abstraction: Solving the 40-Year Holy War

Hardware Script introduces a revolutionary approach to coordinate systems that eliminates the eternal debate between Top-Left vs Bottom-Left origins. 

**How It Works**:
1. **User Space**: You choose your preferred origin using unified `XY by Z` syntax (e.g., `origin: bl by t`).
2. **Absolute Space**: The compiler normalizes all coordinates to a universal format internally.
3. **Export Space**: Gerber/GDSII exporters translate to the required manufacturing standards.

### Engineering Standards: Native SI Unit Parsing

Because Hardware Script uses a unified parser, **it natively understands units everywhere**. You never need to write `min_width_nm: 254000`. You simply write `min_width: 254µm`, and the compiler translates it mathematically to fixed-point integers for the engine.

**Valid Examples**:
```hw
# Distance
50mm, 254µm, 5cm

# Electrical
12V, 500mV, 2A, 100mA, 4.7kΩ, 100nF, 2.4GHz
```

*(Note: Hardware Script explicitly rejects IEC 60062 notation like `4K7`. You must write `4.7kΩ` or `4.7kOhm` to ensure code is readable to non-experts).*

---

## Part II: Top-Level Definitions

A `.hw` file is composed of one or more `define` blocks. The compiler registers all definitions into its symbol table before executing spatial assembly.

### 1. Material Definition

Defines the physical, electrical, and thermal properties of conductors, insulators, and semiconductors natively using SI units.

**Syntax**:
```hw
define material "Name":
    category: <conductor|insulator|semiconductor>
    symbol: "Sym"
    properties:
        <property>: <value> <unit>
```

**Example**:
```hw
define material "Copper":
    category: conductor
    symbol: "Cu"
    description: "Universal PCB trace material"
    
    properties:
        density: 8960 kg/m3
        thermal_conductivity: 401 W/mK
        resistivity: 1.68e-8 Ω/m
        max_current_density: 35 A/mm2
        melting_point: 1085 C
        color: "#B87333"
```

### 2. Profile Definition (Manufacturing Rules)

Defines factory constraints and physical limits. The physics engine validates the `Space` against these rules.

**Example**:
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
        
    layer:
        min_thickness: 70µm
        allowed_conductors: [ Copper, Silver ]
        allowed_dielectrics: [ FR4, Air ]
```

### 3. Component Definition

The absolute physical and electrical blueprint for an atomic part. Components are built using abstraction blocks (Interface, Behavior, Layout, Electrical, Render) allowing cross-target compilation (PCB vs. Simulation vs. 3D render).

**Example**:
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
            Pin1 at [0, 0]
            Pin2 at[2.0mm, 0]
            
    electrical:
        max_voltage: 150V
        max_power: 0.125W
        
    render:
        type: procedural
        shape: smd_passive
        body_color: "#111111"
        endcap_color: "#C0C0C0"
```

### 4. Mechanical & Signal Constraints

Defines physical 3D boundaries, keep-out zones, and high-speed electrical rules.

**Example**:
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

### 5. Interface Definition (Hardware/Firmware Contract)

A declarative mapping file that binds physical hardware nets to software variables. Firmware developers can auto-generate `C`/`Rust` headers directly from this block.

**Example**:
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

Automated scripts for bare-metal CI/CD physics validation.

**Example**:
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

## Part III: Space Assembly (Building the Board)

The `define space` block is where physical placement and routing happen. It utilizes the materials, profiles, and components defined above.

**Syntax**:
```hw
define space "SpaceName":
    dimensions: <width>mm by <height>mm by <depth>mm
    grid: <x_cells> by <y_cells> by <z_cells>
    origin: <xy> by <z>
    profile: <ProfileName>
```

### 1. Substrate Spanning
Defines the base material layer that spans the entire space or specific regions.

```hw
add substrate(FR4) spanning [1, 1, 1] to[100, 100, 2]
```

### 2. Component Placement
Place components at specific grid coordinates within the space.

```hw
# Positional coordinates
add Battery (5V) named MainPower at[10, 10, 1]

# Declarative coordinates with rotation
add Resistor_0805 (10kΩ) named PullUp at [x:20, y:15, z:1] rotated 90
```

### 3. Routing (Manual Waypoints)
Defines electrical connections using explicit user-defined waypoints. Interpolation uses Bresenham's algorithm in 3D. 

*(Note: If the `path:` block is omitted, the compiler utilizes the Automatic Manhattan Router).*

```hw
route MainPower.Plus to PullUp.Pin1:
    path:
        - [x:10, y:10, z:1]
        -[x:20, y:10, z:1]
        - [x:20, y:15, z:1]
```

---

## Part IV: Import System and Organization

Because everything is a `.hw` file, importing is clean and unified. You do not need to specify what *type* of definition you are importing.

**Local File Imports**:
```hw
import "materials.hw"
import "profiles.hw"
import "components.hw"
import "mechanical.hw"
```

**Standard Library & Package Registry Imports**:
```hw
import Copper from "@standard/materials"
import Resistor_0805 from "@standard/components"
import ESP32_Module from "@wireless/esp32"
```

### Exposing Pins (Module System)
You can create reusable sub-assemblies (like a 5V Regulator circuit) inside a `define space` block, and expose specific pins to the outside world using the `expose` keyword.

```hw
define space "Regulator_5V":
    # ... components and routing ...
    
    expose Regulator.In as VIN
    expose Regulator.Out as VOUT
    expose Regulator.GND as GND
```

When someone imports this space, they can route directly to `PowerSupply.VIN`.

---

## Part V: Formal Grammar (EBNF)

```ebnf
program = { import_stmt } { definition_block }

import_stmt = "import" ( string | identifier "from" string )

definition_block = 
    | space_def
    | component_def
    | material_def
    | profile_def
    | mechanical_def
    | interface_def
    | test_def

# ==========================================
# DEFINITION BLOCKS
# ==========================================

material_def = "define" "material" STRING ":"
               INDENT material_body DEDENT

profile_def = "define" "profile" STRING ":"
              INDENT profile_body DEDENT

component_def = "define" "component" STRING ":"
                INDENT component_body DEDENT

space_def = "define" "space" STRING ":"
            "dimensions" ":" measure "by" measure "by" measure
            "grid" ":" number "by" number "by" number
            [ "origin" ":" origin_point ][ "profile" ":" identifier ]
            { space_statement }

space_statement = 
    | substrate_def 
    | component_placement 
    | route_def 
    | expose_def

# ==========================================
# SPACE STATEMENTS
# ==========================================

substrate_def = "add" "substrate" "(" identifier ")" "spanning" coordinate "to" coordinate

component_placement = "add" identifier [ "(" parameters ")" ] 
                      "named" identifier 
                      "at" coordinate 
                      [ "rotated" direction ]

route_def = "route" pin_ref "to" pin_ref ":"
            [ "path" ":" INDENT waypoint_list DEDENT ]

expose_def = "expose" pin_ref "as" identifier

# ==========================================
# PRIMITIVES
# ==========================================

waypoint_list = { "-" coordinate }

coordinate = "[" number "," number "," number "]" 
           | "[" coord_pair "," coord_pair "," coord_pair "]"

coord_pair = ( "x" | "y" | "z" ) ":" number

pin_ref = identifier [ "." identifier ]

direction = number [ "deg" ]

origin_point = ( "tl" | "bl" | "tr" | "br" ) [ "by" ( "t" | "b" ) ]

measure = number unit

unit = "mm" | "cm" | "µm" | "V" | "mV" | "A" | "mA" | "Hz" | "kHz" | "MHz" | "GHz" | "C" | "Ω" | "kΩ" | "MΩ" | "F" | "pF" | "nF" | "uF" | "kg/m3" | "W/mK"

identifier = LETTER { LETTER | DIGIT | "_" }
number = [ "-" ] DIGIT { DIGIT } [ "." DIGIT { DIGIT } ] [ ( "e" | "E" ) [ "-" ] DIGIT { DIGIT } ]
STRING = '"' { ANY_CHAR } '"'
```

---

## Part VI: Compilation Targets

Hardware Script uses explicit abstraction blocks inside `define component` to separate different levels of hardware description. This allows the exact same `.hw` codebase to compile to different targets based on the `--target` CLI flag.

| Target | Reads | Ignores | Primary Output |
|--------|-------|---------|----------------|
| `pcb` (Default) | `pins:`, `layout:` | `behavior:`, `electrical:` | Gerber files (`.gtl`), Drill files |
| `fpga` | `pins:`, `behavior:` | `layout:`, `render:` | Verilog/VHDL |
| `asic` | `pins:`, `behavior:`, `layout:` | `render:` | GDSII layout files |
| `spice` | `pins:`, `electrical:` | `layout:`, `render:` | SPICE netlist |
| `viz` | `pins:`, `layout:`, `render:` | `behavior:`, `electrical:` | OBJ/GLB 3D models, Blender scripts |

---

## Part VII: Error Handling

Hardware Script provides actionable error messages with memorable error codes (e.g., P16, R12) using `miette` diagnostics. 

**Syntax Errors (S-codes)**: Invalid keywords, wrong coordinate formatting, or tri-fold case violations (e.g., typing `Define Space` instead of `define space`).
**Compiler Errors (C-codes)**: Logical flaws like referencing a component that hasn't been placed.
**Routing Errors (R-codes)**: Physical placement collisions or out-of-bound grid coordinates.
**Physics Errors (P-codes)**: Laws of physics violated based on material properties (e.g., Dielectric Breakdown, Overheating).
**Manufacturing Errors (M-codes)**: Factory constraints violated (e.g., Trace thinner than Profile allows).

**Example P16 (Dielectric Breakdown)**:
```
Error[P16]: Dielectric Breakdown Risk
  ╭─[main.hw:42:1]
42 │  route Power_120V.Out to Relay.In:
   ·        ──────┬───────
   ·              ╰── High voltage net (120V)
45 │      -[x:10, y:50, z:2]
   ·        ──────┬──────
   ·              ╰── Approaches within 0.02mm of MCU_5V net here
   ╰────
  help: The voltage difference is 115V. Through Air, this requires a 
        minimum clearance of 0.08mm to prevent arcing.
  hint: Move the waypoint to [x:10, y:50, z:4] or change substrate material to FR4.
```

---

## Best Practices (Project Organization)

While the compiler accepts all definitions in a single file, professional Hardware Script projects use a modular structure modeled after Rust's crate organization.

**Recommended Structure**:
```text
my_project/
├── materials.hw       # Custom alloys and substrates
├── profiles.hw        # Factory rules (.hwp replacement)
├── components.hw      # Custom IC definitions (.hwx replacement)
├── constraints.hw     # Mechanical boundaries (.hwm replacement)
└── main.hw            # The assembly that imports everything
```

**Inside `main.hw`**:
```hw
import "materials.hw"
import "profiles.hw"
import "components.hw"

define space "Motherboard":
    # ... design rules and logic ...
```

By keeping the ecosystem unified under `.hw`, syntax highlighting, Language Servers (LSP), and LLM context windows perform flawlessly without context switching.