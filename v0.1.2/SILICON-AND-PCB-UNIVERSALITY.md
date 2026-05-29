# Hardware Script v0.1.2 - Silicon and PCB Universality

**Document Type**: Technical Deep Dive  
**Status**: Architectural Explanation  
**Last Updated**: March 2026

---

## Purpose of This Document

This document explains how Hardware Script's voxel grid architecture enables the same language to design both PCBs and silicon chips, and addresses the philosophical approach to behavioral logic compilation.

**Key insight**: The math is identical - only materials, scale, and manufacturing process differ.

---

## The Fundamental Question

**"Are microchips designed on fiberglass boards?"**

**Answer**: No. But mathematically, the way NVIDIA and Intel design chips is identical to how you design a PCB.

**The difference**: Materials, microscopic scales, and manufacturing process.

**The similarity**: 3D spatial topology, routing, layer stacking, via connections.

---

## How Microchips Are Actually Made

### The Silicon Manufacturing Process

**1. The Substrate**:
```
PCB: Fiberglass (FR4) board
Chip: Silicon wafer (crystalline slice)
```

**2. The Components**:
```
PCB: Pre-packaged chips soldered onto board
Chip: Transistors built directly into silicon via doping
```

**3. The Routing**:
```
PCB: Copper traces on 2-4 layers
Chip: 10-20 microscopic metal layers (Aluminum/Copper)
```

**4. The Vias**:
```
PCB: Drilled holes with plated copper
Chip: Laser-etched holes with deposited metal
```

**5. The Output File**:
```
PCB: Gerber files (.gtl, .gbl, .drl)
Chip: GDSII files (photolithography masks)
```

### The Manufacturing Difference

**PCB Manufacturing**:
1. Start with copper-clad fiberglass
2. Apply photoresist
3. Expose to UV light through mask
4. Etch away unwanted copper
5. Drill holes for vias
6. Plate holes with copper
7. Apply solder mask
8. Silkscreen component labels

**Silicon Manufacturing (Photolithography)**:
1. Start with pure silicon wafer
2. Grow oxide layer
3. Apply photoresist
4. Expose to UV light through GDSII mask
5. Etch patterns
6. Ion implantation (doping)
7. Deposit metal layers
8. Repeat 10-20 times for multiple layers
9. Dice wafer into individual chips

**The key**: Both use photolithography and layered manufacturing.

---

## How Hardware Script Handles Both

### The Universal Voxel Grid

**Because Hardware Script uses a discrete 3D tensor grid `[Z, X, Y]`, the compiler doesn't care if it's building**:
- A 50mm plastic PCB
- A 5nm silicon GPU

**The math is exactly the same.**

### What Changes

**Only three things change**:

1. **Space configuration** (dimensions and grid)
2. **Material assignments** (FR4 vs Silicon)
3. **Export target** (Gerber vs GDSII)

---

## Example: PCB vs Silicon

### PCB Design

```hw
define space "LED_Board":
    dimensions: 50mm × 50mm × 2mm
    grid: 500 × 500 × 2
    substrate: FR4
    
    # Voxel size: 50mm / 500 = 0.1mm per voxel
    
    add Battery at [1, 50, 50]
    add LED at [1, 450, 450]
    
    route Battery.Plus to LED.Anode:
        path:
            - [1, 50, 50]
            - [1, 450, 50]
            - [1, 450, 450]
```

**Compilation**:
```bash
hws build led_board.hw --target pcb
```

**Output**: Gerber files for PCB manufacturing

### Silicon Design

```hw
define Space "Custom_ALU_Silicon":
    dimensions: 2mm × 2mm × 0.1mm
    grid: 20000 × 20000 × 10
    substrate: Silicon
    
    # Voxel size: 2mm / 20000 = 100nm per voxel!
    
    # Base layer is pure silicon crystal
    add Substrate(Silicon) spanning [1, 1, 1] to [1, 20000, 20000]
    
    # Layers 2-10 are inter-layer dielectrics for metal routing
    add Substrate(SiliconDioxide) spanning [2, 1, 1] to [10, 20000, 20000]
    
    # Import raw silicon gate definitions
    import CMOS_NAND_Gate from standard.silicon.logic
    
    add CMOS_NAND_Gate named Gate1 at [1, 50, 50]
    add CMOS_NAND_Gate named Gate2 at [1, 150, 50]
    
    # Route on Metal Layer 1 (Z=2) and Metal Layer 2 (Z=3)
    route Gate1.out to Gate2.in:
        path:
            - [1, 50, 50]    # Start at silicon gate
            - [2, 50, 50]    # Move up to Metal Layer 1
            - [2, 150, 50]   # Route across Metal Layer 1
            - [1, 150, 50]   # Drop back down to silicon gate
```

**Compilation**:
```bash
hws build alu_silicon.hw --target asic
```

**Output**: GDSII files for silicon foundry (TSMC, Intel, Samsung)

---

## The Configuration Difference

### PCB Configuration

```toml
# hw.toml
[project]
name = "LED Board"
target = "pcb"

[fabrication]
format = "gerber"
manufacturer = "JLCPCB"
process = "standard_pcb"
layers = 2
copper_thickness = "1oz"
min_trace_width = "0.15mm"
```

### Silicon Configuration

```toml
# hw.toml
[project]
name = "Custom ALU"
target = "asic"

[fabrication]
format = "gdsii"
foundry = "TSMC"
process_node = "130nm"
metal_layers = 6
min_feature_size = "130nm"
```

---

## The Export Process

### PCB Export (Gerber)

**Compiler process**:
1. Look at Z=1 (top layer)
2. Find all COPPER cells
3. Convert to Gerber draw commands
4. Look at Z=2 (bottom layer)
5. Find all COPPER cells
6. Convert to Gerber draw commands
7. Find all HOLE cells
8. Convert to Excellon drill commands

**Output files**:
```
build/
├── board_top.gtl      # Top copper
├── board_bottom.gbl   # Bottom copper
├── board.drl          # Drill file
├── board_top.gts      # Top solder mask
├── board_bottom.gbs   # Bottom solder mask
└── board.gko          # Board outline
```

### Silicon Export (GDSII)

**Compiler process**:
1. Look at Z=1 (silicon layer)
2. Find all logic gates
3. Calculate 2D bounding boxes
4. Export as GDSII polygons (N-well, P-well masks)
5. Look at Z=2 (Metal Layer 1)
6. Find all COPPER cells
7. Export as GDSII polygons for metal layer
8. Find HOLE cells connecting layers
9. Export as GDSII contact/via masks

**Output files**:
```
build/
├── alu.gds            # GDSII stream file
├── alu_metal1.gds     # Metal layer 1
├── alu_metal2.gds     # Metal layer 2
├── alu_vias.gds       # Via masks
└── alu_nwell.gds      # N-well mask
```

---

## The Revolutionary Implication

### One Language, All Scales

**Currently**:
- Intel uses Cadence/Synopsys ($100,000+/year) for chips
- Hobbyists use KiCad (free) for PCBs
- Completely different tools, workflows, file formats

**With Hardware Script**:
- Same `.hw` syntax
- Same `[Z, X, Y]` routing logic
- Same compilation process
- Different targets and materials

**Result**:

```
A 10-year-old making a blinking LED board
    +
A senior engineer at NVIDIA designing an AI chip
    =
Using the exact same language
```

---

## The Behavioral Logic Question

### The Problem Statement

**How does the compiler take**:
```hw
if Eye.light_level < 50%:
    turn_on(LED)
```

**And convert it into physical silicon/copper logic gates?**

### The Philosophical Answer

**From the notes**:

> "I assume that in hardware systems, every logic gate has a fundamental logic behind it. The way we are writing it in English leaves too much space between what is in the logic versus what the English is saying."

**The decision**: Drop English-like syntax for behavioral logic.

**Use mathematically provable expressions instead.**

### The Mathematical Approach

**Instead of English-like**:
```hw
if Eye.light_level < 50%:
    turn_on(LED)
```

**Use mathematical logic**:
```hw
behavior:
    # Boolean algebra
    LED_on = NOT(Light_sensor_high)
    
    # Or gate-level
    LED_on = NAND(Light_sensor, Light_sensor)
```

### The Library Approach

**Over time, as people build libraries**:

```hw
import light_control from "stdlib/sensors"

# High-level function call
light_control.auto_brightness(
    sensor: Eye.light_level,
    output: LED,
    threshold: 50%
)
```

**Under the hood, the library provides the gate-level implementation.**

### The Compilation Process

**For behavioral blocks**:

1. **Parse boolean expressions**
2. **Convert to logic gates** (AND, OR, NOT, NAND, NOR, XOR)
3. **Map gates to physical components** (from stdlib)
4. **Place gates in voxel grid**
5. **Route connections**
6. **Validate physics**

**This is logic synthesis** - a well-understood process in EDA.

---

## The Scale Comparison

### PCB Scale

```
Dimensions: 50mm × 50mm × 2mm
Grid: 500 × 500 × 2
Voxel size: 0.1mm (100 micrometers)
Components: Pre-packaged chips
Routing: Copper traces
Layers: 2-4 typically
Manufacturing: PCB fab (JLCPCB, PCBWay)
Cost: $5-$50
Time: 2-7 days
```

### Silicon Scale

```
Dimensions: 2mm × 2mm × 0.1mm
Grid: 20000 × 20000 × 10
Voxel size: 100nm (0.1 micrometers)
Components: Individual transistors
Routing: Metal layers (Aluminum/Copper)
Layers: 10-20 metal layers
Manufacturing: Silicon foundry (TSMC, Intel)
Cost: $10,000-$1,000,000 (per wafer)
Time: 3-6 months
```

**Scale difference**: 1000× smaller voxels, 1000× more precision

---

## The Universality Proof

### Same Syntax, Different Scale

**LED board (hobbyist)**:
```hw
define Space "LED":
    dimensions: 50mm × 50mm × 2mm
    grid: 500 × 500 × 2
```

**GPU chip (NVIDIA)**:
```hw
define Space "GPU_Core":
    dimensions: 10mm × 10mm × 0.5mm
    grid: 100000 × 100000 × 20
```

**Same language. Same compiler. Different materials and export targets.**

### The Mathematical Truth

**The voxel grid is scale-invariant**:

```
Voxel size = Dimensions / Grid
```

**PCB**:
```
50mm / 500 = 0.1mm per voxel
```

**Silicon**:
```
10mm / 100000 = 100nm per voxel
```

**The compiler doesn't care about absolute size.**

**It only cares about**:
- Relative positions
- Connectivity
- Material properties
- Manufacturing constraints

---

## Consumer Electronics: The Complete Picture

### The Philosophical Boundary

Hardware Script has hit on the exact philosophical boundary of electronic systems:

**Hardware Script governs**: Electron Flow and Topology (The Nervous System and Brain)

**Hardware Script imports**: Physical Structure, Chemistry, and Acoustics (The Skin and Bones)

This is not a limitation - it's a precise semantic boundary that keeps the system blazing fast and perfectly optimized for its purpose.

### The Boundary Definition

**What Hardware Script Designs**:
```
✅ Electron flow
✅ Topology and connectivity
✅ Power distribution
✅ Signal routing
✅ Component placement
✅ Electromagnetic behavior
✅ Thermal management (electrical)
✅ Logic and control systems
```

**What Hardware Script Imports**:
```
⬜ Injection-molded plastic casings
⬜ Chemical compounds (battery chemistry)
⬜ Acoustic resonance (speaker cones)
⬜ Mechanical gears and structures
⬜ Optical lens systems
⬜ Pure material science
```

**The bridge**: The `.hwm` (Hardware Mechanical) file format.

---

## Real-World Consumer Electronics Applications

### How Hardware Script Builds Complete Products

Let's examine exactly how Hardware Script handles the consumer electronics you use every day.

### 1. TVs & Displays

**The Motherboard**: 100% Hardware Script
```hw
define Space "TV_Motherboard":
    dimensions: 300mm × 200mm × 10mm
    
    add HDMI_Controller named HDMI1 @ [1, 50, 50]
    add HDMI_Controller named HDMI2 @ [1, 50, 100]
    add VideoProcessor named GPU @ [1, 150, 100]
    add PowerSupply named PSU @ [1, 250, 100]
    
    route HDMI1.output -> GPU.input1
    route HDMI2.output -> GPU.input2
    route PSU.5V -> GPU.power
```

**The Screen (Pixel Matrix)**: Hardware Script can design the interconnects!

A modern OLED or LCD screen is a giant glass substrate with millions of microscopic transistors (TFTs) routed in a massive [X, Y] grid.

Because Hardware Script is scale-invariant (nanometers to meters), you can write a parametric loop to generate the exact wiring grid for a 4K TV display panel:

```hw
define Space "4K_Display_Panel":
    dimensions: 950mm × 540mm × 5mm
    grid: 3840 × 2160 × 10  # 4K resolution!
    
    # Generate 8.3 million pixel transistors
    for row in 0..2160:
        for col in 0..3840:
            add TFT_Pixel named Pixel[row][col] @ [1, col, row]
    
    # Row drivers
    for row in 0..2160:
        add RowDriver named Row[row] @ [2, 0, row]
        route Row[row].output -> Pixel[row][0].gate
    
    # Column drivers
    for col in 0..3840:
        add ColDriver named Col[col] @ [2, col, 0]
        route Col[col].output -> Pixel[0][col].source
```

**The Casing**: Designed in CAD (Blender/SolidWorks), imported via `.hwm`

```hw
import "tv_casing.hwm" as Enclosure

define Space "TV_Complete":
    # The compiler ensures the HDMI port aligns perfectly
    # with the hole in the plastic TV casing!
    constrain HDMI1.connector to Enclosure.hdmi_port_1
    constrain HDMI2.connector to Enclosure.hdmi_port_2
    constrain PSU.power_inlet to Enclosure.power_socket
```

### 2. Solar Panels

**A solar panel is a perfect target for Hardware Script!**

It's simply a flat substrate (glass/EVA) with components (photovoltaic diodes) placed in a grid and routed together with traces (silver busbars) in series and parallel.

```hw
define Space "400W_Solar_Panel":
    dimensions: 2000mm × 1000mm × 40mm
    substrate: Glass_EVA
    
    # 72 solar cells in 6×12 grid
    for row in 0..6:
        for col in 0..12:
            add PhotovoltaicCell named Cell[row][col] @ [1, col*150, row*150]
    
    # Series connection within each row (increases voltage)
    for row in 0..6:
        for col in 0..11:
            route Cell[row][col].positive -> Cell[row][col+1].negative {
                material: Silver_Busbar
                width: 2mm
                max_current: 10A
            }
    
    # Parallel connection between rows (increases current)
    for row in 0..5:
        route Cell[row][11].positive -> Cell[row+1][0].positive {
            material: Silver_Busbar
            width: 3mm
            max_current: 60A
        }
    
    # Output terminals
    add Terminal named Plus @ [1, 1900, 900]
    add Terminal named Minus @ [1, 1900, 100]
    
    route Cell[5][11].positive -> Plus
    route Cell[0][0].negative -> Minus
```

**You could write a 20-line `.hw` script to parametrically generate an entire 400W solar panel array and export it directly for manufacturing.**

### 3. Light Bulbs & Smart LEDs

**The Electronics**: 100% Hardware Script

```hw
define Space "Smart_LED_Bulb":
    dimensions: 60mm × 60mm × 10mm
    substrate: Aluminum_Core_PCB  # For heat dissipation
    
    # LED chips in circular pattern
    for angle in 0..360 step 30:
        add LED_Chip named LED[angle] @ [
            1,
            30 + 20*cos(angle),
            30 + 20*sin(angle)
        ]
    
    # Wi-Fi controller hidden in base
    add ESP32_WiFi named Controller @ [2, 30, 30]
    
    # Power management
    add AC_DC_Converter named PSU @ [2, 10, 10]
    
    # Connect all LEDs in parallel
    for angle in 0..360 step 30:
        route PSU.3V3 -> LED[angle].anode
        route LED[angle].cathode -> Controller.pwm_out[angle/30]
```

**The Bulb**: The glass dome and metal screw thread are mechanical components.

```hw
import "bulb_glass_dome.hwm" as Dome
import "e27_screw_base.hwm" as Base

define Space "Complete_Bulb":
    # When you render (hws build --target viz),
    # it looks like a photorealistic lightbulb,
    # even though the compiler only "did the math"
    # for the circuitry inside
    
    render:
        import Dome.glass_model  # 3D .glb model
        import Base.metal_thread  # 3D .glb model
```

### 4. Batteries & Battery Management Systems

**The Chemistry**: Material science handles the lithium cobalt oxide.

**The Pack & BMS**: 100% Hardware Script

Wiring 100 individual battery cells together into a Tesla car battery pack, calculating the heavy copper busbar thickness for 500 Amps of current, and designing the Battery Management System (BMS) PCB:

```hw
define Space "Tesla_Battery_Pack":
    dimensions: 2000mm × 1500mm × 200mm
    
    # 100 cylindrical 18650 cells in 10×10 grid
    for row in 0..10:
        for col in 0..10:
            add LithiumCell_18650 named Cell[row][col] @ [
                1,
                col * 20,
                row * 20
            ]
    
    # Series connection (increases voltage to 370V)
    for i in 0..99:
        route Cell[i].positive -> Cell[i+1].negative {
            material: Copper_Busbar
            thickness: 5mm  # Heavy copper for 500A!
            max_current: 500A
            
            # Compiler uses max_current to ensure
            # the busbars won't melt
        }
    
    # Battery Management System PCB
    add BMS_Controller named BMS @ [2, 750, 750]
    
    # Voltage monitoring for each cell
    for i in 0..100:
        route Cell[i].positive -> BMS.voltage_sense[i] {
            material: Thin_Wire
            max_current: 0.01A  # Just sensing, not power
        }
    
    # Temperature sensors
    for row in 0..10 step 2:
        for col in 0..10 step 2:
            add TempSensor named Temp[row][col] @ [1, col*20, row*20]
            route Temp[row][col].output -> BMS.temp_input
```

**The compiler uses the `max_current` constraints to ensure the busbars won't melt.**

### 5. Fans & Motors

**The Electromagnetics**: The physical winding of copper wire around the stator is usually an electromechanical task.

**The Controller**: 100% Hardware Script

The ESC (Electronic Speed Controller) that pulses the electricity to make the motor spin:

```hw
define Space "Brushless_Motor_ESC":
    dimensions: 40mm × 30mm × 5mm
    
    add Microcontroller named MCU @ [1, 20, 15]
    add MOSFET_HBridge named PhaseA @ [1, 10, 10]
    add MOSFET_HBridge named PhaseB @ [1, 20, 10]
    add MOSFET_HBridge named PhaseC @ [1, 30, 10]
    
    # PWM control signals
    route MCU.pwm_a -> PhaseA.gate
    route MCU.pwm_b -> PhaseB.gate
    route MCU.pwm_c -> PhaseC.gate
    
    # High current power traces
    route Battery.positive -> PhaseA.drain {
        material: Copper
        width: 5mm
        max_current: 50A
    }
```

---

## The .hwm (Hardware Mechanical) Bridge

### The Integration Point

Hardware Script doesn't design the casing, but it must obey it.

If an industrial designer makes a beautiful, curved TV casing in Blender, they export a `.hwm` file that defines:
- Keep-Out Zones
- Connector Alignments
- Screw Hole Positions
- Thermal Vents
- Physical Constraints

### Example .hwm File

```yaml
# tv_casing.hwm
mechanical_constraints:
  keep_out_zones:
    - name: "speaker_cavity"
      bounds: [0, 0, 0] to [50, 100, 30]
      reason: "Speaker magnet occupies this space"
    
    - name: "display_bezel"
      bounds: [0, 0, 0] to [950, 540, 5]
      reason: "Display panel mounting area"
  
  connector_alignments:
    - name: "hdmi_port_1"
      position: [1, 50, 0]
      orientation: "rear_facing"
      tolerance: ±0.5mm
    
    - name: "power_socket"
      position: [1, 150, 0]
      orientation: "rear_facing"
      tolerance: ±0.3mm
  
  screw_holes:
    - position: [10, 10, 0]
      diameter: 3mm
      thread: "M3"
    
    - position: [290, 10, 0]
      diameter: 3mm
      thread: "M3"
  
  thermal_vents:
    - position: [1, 200, 0]
      size: 50mm × 30mm
      airflow_direction: "exhaust"

render_model:
  format: "glb"
  file: "tv_casing_visual.glb"
  scale: 1.0
```

### How Hardware Script Uses .hwm

```hw
import "tv_casing.hwm" as Enclosure

define Space "TV_Motherboard":
    dimensions: 300mm × 200mm × 10mm
    
    # Respect mechanical constraints
    obey Enclosure.keep_out_zones
    
    add HDMI_Controller named HDMI1 @ [1, 50, 50]
    
    # The compiler will throw an error if your HDMI port
    # doesn't perfectly align with the hole in the plastic casing!
    constrain HDMI1.connector to Enclosure.hdmi_port_1 {
        tolerance: Enclosure.hdmi_port_1.tolerance
    }
    
    # Ensure screw holes are accessible
    for hole in Enclosure.screw_holes:
        keep_clear(hole.position, radius: hole.diameter/2 + 1mm)
```

**The compiler validates**:
- ✅ HDMI connector aligns with casing hole (within tolerance)
- ✅ No components placed in keep-out zones
- ✅ Screw holes remain accessible
- ✅ Thermal vents not blocked by components

**If validation fails, compilation error**:
```
Error: HDMI1 connector misaligned with Enclosure.hdmi_port_1
  Expected: [1, 50, 0] ± 0.5mm
  Actual: [1, 52, 0]
  Offset: 2mm (exceeds tolerance)
  
Suggestion: Move HDMI1 to [1, 50, 50] or adjust casing design
```

---

## Summary of the Ecosystem

### The Complete Manufacturing World

If you look at the entire physical world of manufactured goods, Hardware Script fits perfectly into its lane:

**Mechanical CAD (SolidWorks/Blender)**:
```
Designs the structural shells, cases, and gears
Exports .hwm constraint files
```

**Chemical/Material Engineering**:
```
Designs the battery chemistry and display phosphors
Provides material property databases
```

**Hardware Script**:
```
Acts as the universal nervous system
Routes the power
Calculates the physics of electron flow
Connects the logic
Generates manufacturing files for factories
Mathematically binds firmware to physical world
```

### The Semantic Boundary

By defining this strict semantic boundary—focusing purely on hardware intent, topology, and electron flow—Hardware Script avoids the bloat of trying to be a 3D mechanical modeling tool.

**The result**:
- Blazing fast compilation
- Text-based design
- Perfectly optimized for AI generation
- Clear separation of concerns
- Integration with existing mechanical CAD workflows

### The Scope is Actually Wider Than You Think

Hardware Script's scope in consumer electronics is more integrated than traditional EDA tools because:

1. **Scale-invariant voxel grid** - Handles microscopic TFTs to meter-scale solar panels
2. **Parametric generation** - Can generate millions of components with loops
3. **Physics-aware routing** - Automatically calculates busbar thickness for 500A
4. **Mechanical integration** - Imports and respects .hwm constraints
5. **Multi-domain** - PCBs, silicon, displays, power systems, all in one language

**Hardware Script is the master of "Electron Flow and Topology".**

**It leaves pure physical structure, chemistry, and acoustics to other tools.**

**But it bridges them together through the .hwm format.**

---

## Key Takeaways

1. **Same math, different materials** - PCB and silicon use identical spatial logic

2. **Voxel grid is scale-invariant** - Works from nanometers to meters

3. **Only three things change** - Dimensions, materials, export target

4. **Manufacturing process is similar** - Both use photolithography and layers

5. **One language for all scales** - Hobbyist LED to NVIDIA GPU

6. **Behavioral logic should be mathematical** - Not English-like

7. **Libraries provide abstraction** - High-level functions built on gate-level

8. **Export format determines manufacturing** - Gerber for PCB, GDSII for silicon

9. **The compiler is universal** - Same pipeline, different backends

10. **This is revolutionary** - Unifies PCB and chip design in one language

11. **Consumer electronics are fully supported** - TVs, solar panels, batteries, LEDs, motors

12. **The .hwm bridge is critical** - Mechanical CAD exports constraints, Hardware Script respects them

13. **Electron flow vs physical structure** - Clear semantic boundary keeps system fast

14. **Parametric generation at scale** - Generate millions of display pixels with loops

15. **Physics-aware compilation** - Automatically calculates trace thickness for current requirements

---

## Summary

**Hardware Script's voxel grid architecture enables true universality**:

**PCB design**:
- FR4 substrate
- 0.1mm voxels
- Gerber export
- PCB fab

**Silicon design**:
- Silicon substrate
- 100nm voxels
- GDSII export
- Foundry fab

**Consumer electronics**:
- TV motherboards and display panels
- Solar panel arrays
- Battery management systems
- Smart LED controllers
- Motor ESCs

**Same language. Same compiler. Same syntax.**

**The only differences**: Materials, scale, and export target.

**The integration**: .hwm files bridge mechanical CAD with Hardware Script, ensuring perfect alignment of electronic and physical designs.

**This unifies hardware design** from hobbyist projects to professional chip design to complete consumer products in a single, text-based, LLM-friendly language.

**The vision**: A 10-year-old and an NVIDIA engineer and a Tesla battery designer using the same `.hw` syntax.

---

**Document Status**: Technical Deep Dive  
**Last Updated**: March 2026  
**Part of**: Hardware Script v0.1.2 Documentation Suite
