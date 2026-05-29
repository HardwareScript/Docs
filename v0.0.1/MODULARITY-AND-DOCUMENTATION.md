# Hardware Script - Modularity and Documentation Systems

**The Three Features That Make Hardware Script Production-Ready**

---

## Overview

This document describes the three critical language design features that transform Hardware Script from a "cool prototype" into a production-ready language:

1. **Libraries vs Components** - The sub-assembly concept
2. **Auto-Documentation Syntax** - `#` vs `##` comments
3. **Modularity** - The `main.hw` entry point and import system

These features are inspired by the best practices from Rust, Elixir, and Node.js ecosystems.

---

## 1. Libraries vs Components: The Sub-Assembly Concept

### The Problem

Nobody wants to place 15 tiny components just to regulate power. They want the "npm package" equivalent for hardware - a complete, tested, pre-wired module.

### The Solution: .hw Libraries

A `.hw` file can be either:
- **A board** - The final product you're designing
- **A library** - A reusable sub-assembly that can be imported

### Component (.hwx) vs Library (.hw)

**Component (.hwx)** - Single atomic part:
```yaml
# transistor_2n2222.hwx
name: "Transistor_2N2222"
type: "NPN_Transistor"
pins:
  - Collector
  - Base
  - Emitter
footprint: "TO-92"
```

**Library (.hw)** - Complete pre-routed module:
```hw
## A complete 5V voltage regulator module
## Input: 7-12V DC, Output: 5V at 1A
## WARNING: Requires heatsink above 500mA load
define Space "5V_Regulator":
    dimensions: 20mm by 15mm by 2mm
    grid: 40 by 30 by 2

# Internal components (pre-placed)
add IC_LM7805 named Regulator at [1, 20, 15]
add Capacitor (10uF) named C1 at [1, 10, 15]
add Capacitor (1uF) named C2 at [1, 30, 15]

# Internal routing (pre-wired)
route C1.Plus to Regulator.In:
    path:
        - [1, 10, 15]
        - [1, 20, 15]

route Regulator.Out to C2.Plus:
    path:
        - [1, 20, 15]
        - [1, 30, 15]

# Exposed pins for external connections
expose Regulator.In as VIN
expose Regulator.Out as VOUT
expose Regulator.GND as GND
```

### Using a Library

**In your main board**:
```hw
import Regulator from "@power/5v-regulator"

define Space "MyRobot":
    dimensions: 100mm by 100mm by 2mm
    grid: 200 by 200 by 2

# Place the entire pre-routed module as one piece
add Regulator named PowerSupply at [1, 50, 50]

# Connect to the exposed pins
route Battery.Plus to PowerSupply.VIN:
    path:
        - [1, 10, 10]
        - [1, 50, 50]

route PowerSupply.VOUT to ESP32.VIN:
    path:
        - [1, 50, 50]
        - [1, 150, 150]
```

### What Happens Internally

When the compiler encounters `add Regulator named PowerSupply at [1, 50, 50]`:

1. **Load the library**: Parse `@power/5v-regulator.hw`
2. **Get local grid**: The library has a 40×30×2 tensor grid
3. **Map to global coordinates**: Place the 40×30 block at position [1, 50, 50] in the main board
4. **Translate coordinates**: All internal coordinates become global:
   - Library's `[1, 20, 15]` → Main board's `[1, 70, 65]`
5. **Expose pins**: `PowerSupply.VIN`, `PowerSupply.VOUT`, `PowerSupply.GND` become connection points

### Why This Is Revolutionary

- **Reusability**: Design once, use everywhere
- **Abstraction**: Junior developers don't need to understand voltage regulation internals
- **Testing**: Libraries are tested once, trusted everywhere
- **AI-Friendly**: LLMs can reason about high-level architecture instead of individual resistors
- **npm for Hardware**: Same concept as software packages, but for physical circuits

### Package Registry Support

**Install a library**:
```bash
hpm install @power/5v-regulator
hpm install @motor/h-bridge
hpm install @sensors/imu-module
```

**Registry entry**:
```yaml
"@power/5v-regulator":
  type: "library"
  url: "https://github.com/hw-libraries/5v-regulator"
  version: "1.0.3"
  description: "Complete 5V regulator with LM7805, capacitors, and routing"
  dependencies:
    - "ics/lm7805"
    - "passive/capacitor_electrolytic"
```

---

## 2. Auto-Documentation Syntax: # vs ##

### Inspiration

- **Rust**: `//` vs `///` (regular vs doc comments)
- **Elixir**: `#` vs `@doc` (generates HexDocs)
- **Python**: `#` vs `"""docstring"""`

### The Two Comment Types

**Human comments (`#`)** - Ignored by compiler:
```hw
# This is a note for developers reading the code
# TODO: Replace with higher current regulator
add IC_LM7805 named Reg at [1, 20, 15]
```

**Documentation comments (`##`)** - Extracted by hwsd:
```hw
## A complete 5V voltage regulator module
## Input: 7-12V DC
## Output: Regulated 5V at up to 1A
## WARNING: Requires heatsink above 500mA load
define Space "5V_Regulator":
    dimensions: 20mm by 15mm by 2mm
    grid: 40 by 30 by 2
```

### Documentation Generation

**Run hwsd**:
```bash
hwsd generate 5v_regulator.hw
```

**Output**: Beautiful HTML documentation (like Rust's rustdoc or Elixir's HexDocs)

**Generated docs include**:
- Module description
- Input/output specifications
- Warnings and constraints
- Exposed pins with descriptions
- Usage examples
- Internal component list

### Example with Pin Documentation

```hw
## Power Distribution Module
## Provides regulated 5V and 3.3V from 12V input
define Space "PowerSystem":
    dimensions: 40mm by 30mm by 2mm
    grid: 80 by 60 by 2

    ## Main 5V regulator IC
    add IC_LM7805 named Reg5V at [1, 20, 15]
    
    ## 3.3V low-dropout regulator
    add IC_LM1117 named Reg3V3 at [1, 60, 15]
    
    ## Input power (7-12V DC)
    expose Reg5V.In as VIN
    
    ## Regulated 5V output (up to 1A)
    expose Reg5V.Out as VOUT_5V
    
    ## Regulated 3.3V output (up to 800mA)
    expose Reg3V3.Out as VOUT_3V3
    
    ## Common ground
    expose Reg5V.GND as GND
```

### Why This Is a Superpower

**For Humans**:
- Documentation lives with the code (never out of sync)
- Auto-generated, searchable API reference
- Clear warnings and constraints

**For AI/LLMs**:
- Structured documentation in JSON format
- LLMs can read hwsd output to learn how to use any library
- No need to parse code - just read the docs
- Instant understanding of inputs, outputs, and constraints

### LLM Integration

**hwsd JSON output**:
```json
{
  "module": "5V_Regulator",
  "description": "A complete 5V voltage regulator module",
  "specs": {
    "input": "7-12V DC",
    "output": "Regulated 5V at up to 1A"
  },
  "warnings": ["Requires heatsink above 500mA load"],
  "exposed_pins": {
    "VIN": "Input power (7-12V DC)",
    "VOUT": "Regulated 5V output (up to 1A)",
    "GND": "Common ground"
  }
}
```

**LLM workflow**:
1. User asks: "Add a 5V regulator to my board"
2. LLM runs: `hpm search voltage regulator`
3. LLM reads: `hwsd info @power/5v-regulator --format json`
4. LLM generates: Correct import and routing code
5. Result: First-try success

---

## 3. Modularity: The main.hw Entry Point

### Inspiration

- **Rust**: `main.rs` as the binary entry point
- **Node.js**: `index.js` as the package entry point
- **Python**: `__main__.py` for executable modules

### The Problem

Large projects cannot be written in a single 10,000-line file. You need:
- Separation of concerns
- Reusable modules
- Team collaboration (different people work on different modules)
- Version control friendly (smaller files = better diffs)

### The Solution: Import System

The compiler looks for `main.hw` (or `board.hw`) as the root entry point.

### Project Structure

```
my_robot_project/
├── hpm.json              # Package dependencies
├── main.hw               # Root entry point (the motherboard)
├── power_system.hw       # Local module (power distribution)
├── sensor_array.hw       # Local module (sensor cluster)
├── motor_controller.hw   # Local module (motor driver)
└── build/                # Compiled outputs (.hwx, .obj, gerbers)
```

### Main Entry Point

**main.hw** - The root motherboard:
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

### Local Module Example

**power_system.hw** - A local module:
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

### Import Syntax

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

**Standard library imports**:
```hw
import Transistor from standard.switches
import Resistor from passive/resistor_0805
```

### Compilation Process

When you run `hws build main.hw`:

1. **Parse main.hw**: Compiler starts at the entry point
2. **Resolve imports**: Find all imported .hw files (local or from hpm)
3. **Parse sub-modules**: Each imported file has its own local Space
4. **Map to global grid**: When you `add Power at [1, 50, 50]`:
   - Takes the Power module's local tensor grid (80×60×2)
   - Places it at position [1, 50, 50] in the main board's global grid (1000×1000×4)
   - Translates all internal coordinates to global coordinates
5. **Expose pins**: Exposed pins become connection points in the global space
6. **Validate**: Check for collisions, electrical issues, etc.
7. **Export**: Generate Gerber, 3D models, BOM, etc.

### Coordinate Translation Example

**power_system.hw** has a local coordinate:
```hw
add IC_LM7805 named Reg5V at [1, 20, 15]  # Local coordinates
```

**main.hw** places the module:
```hw
add Power at [1, 50, 50]  # Global placement
```

**Compiler translates**:
- Local `[1, 20, 15]` → Global `[1, 70, 65]`
- Calculation: `[1, 50+20, 50+15]`

### Benefits

**For Developers**:
- Modular design (separation of concerns)
- Reusable modules across projects
- Team collaboration (different files for different subsystems)
- Better version control (smaller diffs)

**For AI/LLMs**:
- Can generate complex systems by composing modules
- Understands high-level architecture
- Can work on one module without affecting others
- Easier to reason about large designs

---

## Summary: The Three Pillars

### 1. Libraries (.hw as sub-assemblies)
- **What**: Pre-routed modules that can be imported and placed as single units
- **Why**: Reusability, abstraction, npm-like ecosystem for hardware
- **Example**: `import Regulator from "@power/5v-regulator"`

### 2. Documentation (`##` comments)
- **What**: Auto-extracted documentation comments
- **Why**: Self-documenting code, AI-friendly, never out of sync
- **Example**: `## Input: 7-12V DC, Output: 5V at 1A`

### 3. Modularity (`main.hw` + imports)
- **What**: Entry point system with local and package imports
- **Why**: Large project support, team collaboration, separation of concerns
- **Example**: `import "power_system.hw" as Power`

---

## Implementation Roadmap

### v0.2 (Q2 2026)
- [ ] Parse `##` documentation comments
- [ ] Implement `import "file.hw" as Name` syntax
- [ ] Implement `expose Pin as Name` statement
- [ ] Coordinate translation for imported modules
- [ ] hwsd documentation generator (basic HTML output)

### v0.3 (Q3 2026)
- [ ] Package registry support for libraries
- [ ] Dependency resolution
- [ ] hwsd JSON output for LLMs
- [ ] Interactive documentation browser

### v1.0 (2027)
- [ ] Full ecosystem of community libraries
- [ ] AI-native documentation format
- [ ] Automatic library discovery and suggestions

---

**Document Status**: Language Design Specification  
**Last Updated**: March 2026  
**Part of**: Hardware Script Documentation Suite

