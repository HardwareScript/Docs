# Hardware Script v0.1.2 - Core Problems and Fundamental Rethinks

**Document Type**: Strategic Architecture Analysis  
**Status**: External Research Integration  
**Last Updated**: March 2026

---

## Purpose of This Document

This document captures the 11 fundamental problems in hardware design and how Hardware Script approaches them differently from traditional EDA tools. Unlike the other documentation which describes what we've built, this document describes **what we're thinking about** and **why certain architectural decisions matter**.

This is not a feature list. This is a philosophical framework.

---

## The 11 Core Problems - Status Dashboard

### 🟢 CATEGORY 1: Okay As-Is (Just Expand & Integrate)

These problems are fundamentally solved by the current architecture. The Rust rewrite and syntax (space, @, ->) are correct. Do not overcomplicate them; just build them out.

#### 1. Hardware Description Language (Problem #1)

**Progress**: [█████████░] 90% Solved

**What is Done**:
- The AST, parser, and pure text-based syntax (as seen in stress_test.hw)
- Incredibly fast compilation
- Perfectly LLM-readable format
- Clean coordinate system [Z, X, Y]

**What Remains**:
- Formalizing the grammar for edge cases
- Locking in exact syntax for component imports
- Standard library path conventions

**Action**: Keep as-is. Just integrate standard library paths.

**First-Principle Rethink**:
- **Traditional EDA**: Separates the schematic (logic) from the PCB layout (physical)
- **The Trap**: Trying to build a "schematic mode"
- **The Rethink**: There is no schematic. The physical 3D tensor placement IS the logic. We are already doing this. Stick to it.

---

#### 2. Integrated Toolchain (Problem #7)

**Progress**: [████████░░] 80% Solved

**What is Done**:
- The single pipeline `hws build` concept
- Taking a .hw file and instantly turning it into Gerber, OBJ, and Blender outputs
- Single source of truth (the AST)

**What Remains**:
- Updating the Rust exporter to output standard Gerber X2
- .glb files instead of just basic OBJ/GTL
- Complete drill file generation

**Action**: Keep as-is. Just flesh out the exporter modules in Rust.

**First-Principle Rethink**:
- **Traditional EDA**: Forces you to export/import between 5 different programs
- **The Trap**: Building plugins for KiCad/Blender
- **The Rethink**: The compiler is the only source of truth. It directly outputs .gtl (Gerber) and .py (Blender) from the single AST in milliseconds.

---

### 🟠 CATEGORY 2: Problematic / Needs Fundamental Rethink

These are partially solved, but if we scale them using MVP logic, the Rust compiler will hit a brick wall. They require fundamental architectural changes.

#### 3. Component Knowledge Database (Problem #2)

**Progress**: [███░░░░░░░] 30% Solved

**What is Done**:
- Can place a placeholder component (like Transistor_NPN) on a grid
- Basic component positioning works

**The Critical Rethink - Nested Tensors**:

Do not load a component's internal micrometer geometry into the global motherboard grid, or you will run out of RAM.

**The Problem**:
- A 200mm motherboard with 0.5mm resolution = 400×400 grid = 160,000 cells
- A single IC chip with 0.01mm pin precision = 500×500 local grid = 250,000 cells
- If you load 10 chips into the global grid at full resolution = 2.5 million cells
- RAM explodes, compilation slows to a crawl

**The Solution - Nested Tensor Architecture**:

A component (.hwx) must be treated as a "solid mathematical bounding box" on the global grid, with a separate high-resolution local grid.

```
Global Grid (Motherboard):
  - 1 voxel = 0.5mm
  - Component occupies [50, 50] to [60, 60] (10×10 = 100 cells)

Local Grid (Inside Component):
  - 1 voxel = 0.01mm  
  - Component has 500×500 internal grid (250,000 cells)
  - Only loaded when routing interacts with a pin
```

**How the Compiler Handles This**:

1. **Placement Phase**: Component is a solid bounding box on the global grid
2. **Routing Phase**: When a route targets `IC1.Pin_5`, the compiler:
   - Looks up Pin_5's local coordinates in the component definition
   - Translates local [250, 300] to global [55, 56]
   - Routes to that global coordinate
3. **Never loads the full internal geometry into global memory**

**What Remains**:
- Building `hpm` (Hardware Package Manager) to pull lightweight JSON/YAML footprint definitions from a registry
- Implementing the nested tensor translation logic in Rust
- Defining the .hwx component file format

**Action**: This is HIGH PRIORITY. Without this, the system cannot scale beyond toy examples.

---

#### 4. Physics & Electrical Validation (Problem #3)

**Progress**: [████░░░░░░] 40% Solved

**What is Done**:
- Basic trace resistance math
- 3D spatial collision detection
- Materials database with physical properties

**The Critical Rethink - Strict Compilation**:

Do not build a "Warning Log." Treat hardware physics like software type-checking.

**Traditional EDA Approach**:
```
⚠️ Warning: Voltage mismatch (12V to 3.3V pin)
⚠️ Warning: Trace too thin for current
⚠️ Warning: Components overlapping
✅ Build successful (3 warnings)
```

**Hardware Script Approach**:
```
❌ ERROR: Voltage mismatch at Line 45
   12V net routes to IC1.Pin_3 (max 3.3V)
   Compilation FAILED
```

**Why This Matters**:

If 12V routes to a 3.3V pin, the compiler must explicitly panic and fail to build. Hardware bugs must trigger compilation failures, just like type-checking in Rust.

**The Three Types of Physics Validation**:

1. **Electrical Constraints** (Voltage/Current):
   - Voltage compatibility between nets
   - Current capacity of traces (ampacity)
   - Power dissipation limits

2. **Thermal Constraints**:
   - Component operating temperature ranges
   - Heat dissipation requirements
   - Thermal coupling between components

3. **Spatial Constraints**:
   - Physical collisions (components overlapping)
   - Clearance requirements (high voltage spacing)
   - Via clearance through copper planes

**What Remains**:
- Adding thermal limits to the Rust engine
- Current capacity checks based on trace width
- Via clearance logic for copper planes
- Strict compilation failures (not warnings)

**Action**: This is HIGH PRIORITY. Physics validation is what makes Hardware Script safe to manufacture.

---

### 🔵 CATEGORY 3: Untouched (But Actively Needed)

These haven't been built yet, but they are critical for moving from a "cool tool" to a "production-ready ecosystem."

#### 5. Parametric Reusable Modules (Problem #8)

**Progress**: [░░░░░░░░░░] 0% Solved

**The Concept**:

Allowing a fully routed .hw file (like a 5V regulator sub-assembly) to be imported and placed on the main board as a single unit.

**Example**:

```hw
# power_regulator.hw (the module)
space "5V_Regulator" {
    dimensions: 20mm × 15mm × 2mm
    grid: 40 × 30 × 2
    
    # Internal components at local coordinates
    add IC_LM7805 @ [1, 20, 15]
    add Capacitor @ [1, 10, 15]
    
    # Exposed pins
    expose VIN @ [1, 5, 15]
    expose VOUT @ [1, 35, 15]
    expose GND @ [1, 20, 5]
}

# main.hw (the motherboard)
space "Motherboard" {
    dimensions: 200mm × 200mm × 2mm
    grid: 400 × 400 × 4
    
    # Import and place the module
    import "power_regulator.hw" as PowerSupply @ [1, 50, 50]
    
    # Route to the module's exposed pins
    Battery.Plus -> PowerSupply.VIN
    PowerSupply.VOUT -> CPU.VCC
}
```

**The Challenge - Coordinate Translation**:

The Rust engine must be able to take a module's local [Z,X,Y] tensor and offset it seamlessly into the global [Z,X,Y] space.

**Translation Algorithm**:
```
Module local coordinate: [1, 20, 15]
Module placed at global: [1, 50, 50]
Translated coordinate:   [1, 70, 65]

Formula: global = module_origin + local
```

**What Needs to be Done**:
- Define the module file format (.hw with exposed pins)
- Implement coordinate translation in the Rust compiler
- Handle module rotation and mirroring
- Validate that modules don't overlap

**Why This Matters**:

This is the "npm for hardware" concept. Without reusable modules, every project starts from scratch. With modules, you can build a 5V regulator once and use it in 100 projects.

---

#### 6. Documentation System (Problem #9)

**Progress**: [░░░░░░░░░░] 0% Solved

**The Concept**:

The `##` auto-documentation syntax (inspired by Rust's `///` doc comments).

**Example**:

```hw
## A complete 5V voltage regulator module
## Input: 7-12V DC
## Output: Regulated 5V at up to 1A
## WARNING: Requires heatsink above 500mA load
space "5V_Regulator" {
    dimensions: 20mm × 15mm × 2mm
    grid: 40 × 30 × 2
    
    ## Main voltage regulator IC
    ## Datasheet: https://www.ti.com/lit/ds/symlink/lm7805.pdf
    add IC_LM7805 named Regulator @ [1, 20, 15]
    
    ## Input filter capacitor (reduces ripple)
    ## Value: 10µF electrolytic, 25V rated
    add Capacitor (10uF) named C1 @ [1, 10, 15]
    
    ## Input power (7-12V DC)
    expose VIN @ [1, 5, 15]
    
    ## Regulated 5V output (up to 1A)
    expose VOUT @ [1, 35, 15]
}
```

**What `hwsd` (Hardware Script Documentation) Should Generate**:

```json
{
  "module": "5V_Regulator",
  "description": "A complete 5V voltage regulator module",
  "specs": {
    "input": "7-12V DC",
    "output": "Regulated 5V at up to 1A"
  },
  "warnings": ["Requires heatsink above 500mA load"],
  "components": [
    {
      "name": "Regulator",
      "type": "IC_LM7805",
      "description": "Main voltage regulator IC",
      "datasheet": "https://www.ti.com/lit/ds/symlink/lm7805.pdf"
    },
    {
      "name": "C1",
      "type": "Capacitor",
      "value": "10uF",
      "description": "Input filter capacitor (reduces ripple)"
    }
  ],
  "exposed_pins": {
    "VIN": "Input power (7-12V DC)",
    "VOUT": "Regulated 5V output (up to 1A)",
    "GND": "Common ground"
  }
}
```

**Why This Matters**:

This is what LLMs will read to understand how to use community components. Documentation lives in the code and is never out of sync.

**What Needs to be Done**:
- Parse `##` comments in the Rust compiler
- Build `hwsd` CLI tool to extract documentation
- Generate JSON, HTML, and Markdown outputs
- Create a documentation website generator

---

#### 7. Design Rule Fragmentation (Problem #5)

**Progress**: [░░░░░░░░░░] 0% Solved

**The Problem**:

Every PCB manufacturer has different capabilities:
- JLCPCB: 0.15mm minimum trace width, 0.3mm minimum via diameter
- OSH Park: 0.13mm minimum trace width, 0.33mm minimum via diameter
- Advanced Circuits: 0.10mm minimum trace width, 0.25mm minimum via diameter

Traditional EDA makes the user manually type in 50 different factory limits.

**The Solution - Fabrication Profiles (.hwp)**:

```bash
hws build --profile jlcpcb_4layer
```

**Example Profile File** (`jlcpcb_4layer.hwp`):

```yaml
name: "JLCPCB 4-Layer Standard"
manufacturer: "JLCPCB"
url: "https://jlcpcb.com/capabilities"

constraints:
  trace_width_min: 0.15mm
  trace_spacing_min: 0.15mm
  via_diameter_min: 0.3mm
  via_drill_min: 0.2mm
  copper_to_edge_min: 0.5mm
  
  layers_max: 4
  board_thickness: 1.6mm
  copper_weight: 1oz
  
  max_board_size: 500mm × 500mm
  min_board_size: 10mm × 10mm

materials:
  substrate: FR4
  copper: Standard Copper
  surface_finish: [HASL, ENIG, OSP]
```

**How the Compiler Uses This**:

Before outputting Gerbers, the compiler checks the AST against the factory's exact physics limits:

```
❌ ERROR: Trace width violation
   Route at [1, 50, 60] is 0.12mm wide
   JLCPCB minimum: 0.15mm
   Suggestion: Increase trace width or use different manufacturer
```

**What Needs to be Done**:
- Define the .hwp file format
- Implement profile validation in Rust
- Create a library of manufacturer profiles
- Add `--profile` flag to CLI

---

#### 8. Verification & Testing Gap (Problem #6)

**Progress**: [░░░░░░░░░░] 0% Solved

**The Problem**:

In software, you write tests:
```rust
#[test]
fn test_voltage_regulator() {
    assert_eq!(regulator.output_voltage(), 5.0);
}
```

In hardware, you need an oscilloscope and a physical board.

**The Solution - Hardware Test Benches (.hwt)**:

```hw
## Test: 5V Regulator Output
test "regulator_output_voltage" {
    given:
        space "TestBoard"
        Battery (12V) named Power @ [1, 10, 10]
        5V_Regulator named Reg @ [1, 50, 50]
        
        Power.Plus -> Reg.VIN
        Power.GND -> Reg.GND
    
    assert:
        Reg.VOUT == 5V ± 0.1V
        Reg.current < 1.5A
        Reg.temperature < 85°C
}

## Test: Short Circuit Protection
test "regulator_short_circuit" {
    given:
        space "TestBoard"
        Battery (12V) named Power @ [1, 10, 10]
        5V_Regulator named Reg @ [1, 50, 50]
        
        Power.Plus -> Reg.VIN
        Reg.VOUT -> Reg.GND  # Intentional short
    
    assert:
        Reg.shutdown == true
        Reg.current < 0.1A
}
```

**How the Compiler Handles This**:

The compiler mathematically proves the state of the circuit before printing the board:

1. **Circuit Analysis**: Build a graph of all connections
2. **Voltage Propagation**: Calculate voltage at every node
3. **Current Flow**: Calculate current through every trace
4. **Thermal Modeling**: Calculate temperature rise
5. **Assertion Checking**: Verify all test conditions

**What Needs to be Done**:
- Define the .hwt test file format
- Implement circuit analysis in Rust
- Build `hws test` CLI command
- Create CI/CD integration

**Why This Matters**:

This enables Continuous Integration for physical hardware. You can catch bugs before manufacturing.

---

#### 9. Advanced Physics Validation (Problem #3 - Part B)

**Progress**: [░░░░░░░░░░] 0% Solved (Extension of Problem #3)

**What This Adds Beyond Basic Validation**:

1. **Thermal Calculations**:
   - Heat dissipation from components
   - Thermal coupling between components
   - Copper plane heat spreading
   - Required heatsink sizing

2. **Trace Width/Current Capacity**:
   - Automatic trace width calculation based on current
   - Temperature rise calculations
   - Via current capacity

3. **Signal Integrity**:
   - Impedance control for high-speed signals
   - Crosstalk between parallel traces
   - EMI/RF considerations

**The Rethink**:

Traditional tools use heavy, slow finite-element-analysis (FEA) solvers. Since we use discrete voxels, we can calculate current capacity algorithmically based on voxel cross-sections.

**Example - Trace Width Calculation**:

```
Required current: 5A
Copper thickness: 0.035mm (1oz)
Allowed temperature rise: 10°C

Formula: Width = (Current / (k × ΔT^0.44))^(1/0.725)
Result: Minimum trace width = 2.5mm

Compiler action: Route must be at least 3 voxels wide (if voxel = 1mm)
```

**What Needs to be Done**:
- Implement thermal modeling in Rust
- Add trace width auto-calculation
- Via current capacity checks
- Signal integrity analysis for high-speed nets

---

### 🔴 CATEGORY 4: The Traps (Ignore Completely For Now)

If we try to solve these now, the project will die. Let LLMs or humans handle these.

#### 10. Perfect Auto-Routing (Problem #4)

**Progress**: [░░░░░░░░░░] Ignore

**Why to Ignore**:

Auto-routing is an NP-hard traveling salesman problem. Companies have spent billions on it, and humans still route by hand because auto-routers make messy boards.

**The Rethink**:

Let the LLM do the routing. The syntax (`[1, 10, 20] -> [1, 50, 20]`) is literally perfect for an LLM to generate. Force explicit waypoints. Don't write a pathfinding algorithm yet.

**What We Do Instead**:

1. **Manual Waypoints**: User specifies exact path
2. **LLM Generation**: AI generates waypoints based on constraints
3. **Simple Interpolation**: Compiler just connects the dots

**Future Consideration** (v2.0+):

If we ever build auto-routing, it should be:
- Optional (not required)
- Constraint-driven (respects all physics rules)
- Deterministic (same input = same output)
- Explainable (shows why it chose that path)

---

#### 11. Supply Chain Uncertainty (Problem #10)

**Progress**: [░░░░░░░░░░] Ignore

**Why to Ignore**:

Finding out if DigiKey has a capacitor in stock requires complex API integrations that change daily. This is an integration problem, not a compiler problem.

**What We Do Instead**:

Focus on the design and compilation. Let external tools handle supply chain:
- Users can manually check DigiKey/Mouser
- Third-party tools can integrate with our BOM output
- Future plugins can add this functionality

---

#### 12. Manufacturing Automation (Problem #11)

**Progress**: [░░░░░░░░░░] Ignore

**Why to Ignore**:

Pushing files directly to a factory via API is cool, but a simple Gerber export is 100% fine. Users can upload the ZIP file to JLCPCB themselves for the next 2 years.

**What We Do Instead**:

Generate perfect Gerber files that work with any manufacturer. Let users handle the upload process.

---

## Summary Checklist for Current Development Sprint

### Immediate Priorities (Do Now):

1. ✅ Lock down the stress_test.hw syntax (Category 1 - Done)
2. 🔄 Implement Nested Tensors to prevent RAM explosion (Category 2 - HIGH PRIORITY)
3. 🔄 Build strict physics panics - make bad routing fail compilation (Category 2 - HIGH PRIORITY)
4. 🔄 Draft the .hwx component JSON/YAML structure (Category 3 - Next Up)

### Next Phase (3-6 Months):

5. Parametric reusable modules with coordinate translation
6. Documentation system (hwsd)
7. Fabrication profiles (.hwp)
8. Hardware test benches (.hwt)

### Future Considerations (6-12 Months):

9. Advanced physics validation (thermal, signal integrity)
10. Complete Gerber package with all layers
11. BOM generation and optimization

### Explicitly Ignore:

- Perfect auto-routing (let LLMs handle it)
- Supply chain integration (not a compiler problem)
- Manufacturing automation (Gerber export is enough)

---

**Document Status**: Strategic Framework  
**Last Updated**: March 2026  
**Next Document**: Material Database Architecture

