# Hardware Script (.hw) Silicon Design Guide

**Version**: 0.2 (Draft)  
**Document Type**: Microchip and Integrated Circuit Design Reference  
**Core Principle**: The same mathematical framework for PCBs and silicon chips

---

## 1. Silicon vs PCB: The Mathematical Truth

### 1.1 Why "Space" Instead of "Board"

Hardware Script abandoned the term "Board" in favor of "Space" because of a fundamental mathematical truth: **microchip design and PCB design are identical processes at different scales**.

When Intel designs a CPU or Nvidia designs a GPU, they don't use fiberglass boards and drill bits. But the mathematical process—placing components, routing connections, managing layers—is exactly the same as PCB design. The only differences are:

- **Substrate Material**: Silicon wafer instead of FR4 fiberglass
- **Scale**: Nanometers instead of millimeters
- **Manufacturing Process**: Photolithography instead of CNC drilling
- **Output Format**: GDSII instead of Gerber

Because Hardware Script is built on the **Discrete 3D Tensor Grid [Z, X, Y]**, the Synthesizer doesn't care whether it's building a 50mm plastic PCB or a 5-nanometer silicon GPU. The math is exactly the same.

### 1.2 How Intel and Nvidia Actually Make Chips

**The Substrate**  
Instead of a fiberglass board (FR4), chip manufacturers start with a perfectly flat, microscopic slice of crystal called a **Silicon Wafer**.

**The Components**  
Instead of soldering pre-packaged black chips onto the board, they build transistors directly inside the silicon by injecting chemicals (**Doping**) to create P-Type and N-Type regions.

**The Routing (Copper Traces)**  
Above the silicon, they stack 10 to 20 microscopic layers of metal (Aluminum or Copper). Instead of drilling vias with physical drill bits, they use lasers and acid to etch microscopic holes between layers.

**The Output File**  
- PCB factories use **Gerber files**
- Silicon foundries (like TSMC, Intel, Samsung) use **GDSII files** (Graphic Data System)

A GDSII file is essentially a set of 2D stencils (called "Masks") that the factory uses to shine light onto the silicon to print the microscopic copper traces (**Photolithography**).

---

## 2. Designing Silicon in Hardware Script

### 2.1 The Silicon Space Definition

The key difference is the scale and substrate material. A microchip Space uses microscopic dimensions and Silicon substrate.

```hw
# Defining a microscopic space for an Integrated Circuit (IC)
define Space "Custom_ALU_Silicon":
    dimensions: 2mm by 2mm by 0.1mm
    grid: 20000 by 20000 by 10
    
    # MATHEMATICAL TRUTH: 2mm / 20,000 = 100 nanometers per Voxel!
    
    # The base layer is pure Silicon crystal, not fiberglass
    add Substrate(Silicon) spanning [1, 1, 1] to [1, 20000, 20000]
    
    # Layers 2 through 10 are inter-layer dielectrics (glass) for metal routing
    add Substrate(SiliconDioxide) spanning [2, 1, 1] to [10, 20000, 20000]
```

**Voxel Calculation**:
```
Dimensions: 2mm by 2mm by 0.1mm
Grid: 20000 by 20000 by 10

Voxel Width:  2mm / 20000 = 0.0001mm = 100 nanometers
Voxel Height: 2mm / 20000 = 100 nanometers
Voxel Depth:  0.1mm / 10 = 0.01mm = 10 micrometers
```

This gives you 100-nanometer resolution—suitable for modern chip design.

### 2.2 Placing "Naked" Silicon Components

Instead of importing packaged components like `Transistor_TO92`, chip designers import raw silicon gate definitions—the actual atomic structure of transistors and logic gates.

```hw
import CMOS_NAND_Gate from standard.silicon.logic
import CMOS_NOR_Gate from standard.silicon.logic
import CMOS_Inverter from standard.silicon.logic

# Placing raw microscopic logic gates at the nanometer scale
add CMOS_NAND_Gate named Gate1 at [1, 50, 50]
add CMOS_NAND_Gate named Gate2 at [1, 150, 50]
add CMOS_Inverter named Inv1 at [1, 250, 50]

# Routing on "Metal Layer 1" (Z = 2) and "Metal Layer 2" (Z = 3)
route Gate1.out to Gate2.in:
    path:
        - [1, 50, 50]   # Start at Silicon gate
        - [2, 50, 50]   # Move up to Metal Layer 1
        - [2, 150, 50]  # Route across Metal Layer 1
        - [1, 150, 50]  # Drop back down to Silicon gate

route Gate2.out to Inv1.in:
    path:
        - [1, 150, 50]  # Start at Gate2
        - [3, 150, 50]  # Move up to Metal Layer 2
        - [3, 250, 50]  # Route across Metal Layer 2
        - [1, 250, 50]  # Drop down to Inverter
```

### 2.3 Configuration for Silicon Manufacturing

To tell the Synthesizer to output microchip files instead of PCB files, change the `hw.config.yaml`:

```yaml
name: "CustomSilicon"
version: "1.0.0"

targets:
  - type: manufacturing
    format: gds_II          # <--- Outputs Silicon Foundry Stencils, NOT Gerber!
    process_node: "130nm"   # Tells the Synthesizer the TSMC manufacturing rules
    export_path: "./build/foundry/"
```

**Supported Process Nodes**:
- `"180nm"` - Older process, easier to manufacture
- `"130nm"` - Common for educational and hobbyist projects
- `"90nm"` - Mid-range performance
- `"65nm"` - Modern low-power designs
- `"45nm"` - High-performance computing
- `"28nm"` - Current generation mobile chips
- `"14nm"`, `"10nm"`, `"7nm"`, `"5nm"`, `"3nm"` - Cutting-edge processes

---

## 3. How the Synthesizer Translates to GDSII

### 3.1 GDSII Format Overview

GDSII (Graphic Data System) is a binary file format that represents integrated circuit layouts as a hierarchy of geometric shapes. Each layer of the chip (silicon, metal, vias) is represented as a separate set of polygons.

### 3.2 Translation Algorithm

When the Synthesizer reads `format: gds_II` in the configuration:

**Step 1: Silicon Layer (Z=1)**
- Locates all silicon logic gates in the tensor grid
- Calculates their 2D bounding boxes
- Exports as GDSII polygon structures representing:
  - N-well masks (for PMOS transistors)
  - P-well masks (for NMOS transistors)
  - Polysilicon gates
  - Diffusion regions

**Step 2: Metal Layers (Z=2+)**
- Scans each metal layer (Z=2, Z=3, etc.)
- Finds all cells marked as COPPER
- Merges adjacent cells into continuous polygons
- Exports as GDSII polygons for "Metal Layer 1", "Metal Layer 2", etc.

**Step 3: Via Masks**
- Locates all HOLE cells connecting different Z layers
- Instead of generating drill files (like PCB manufacturing), generates GDSII "Contact/Via Mask" layers
- The foundry uses these masks to etch holes with acid or lasers

**Step 4: Design Rule Checking**
- Validates against process node rules (e.g., minimum feature size for 130nm process)
- Checks metal spacing, via sizes, and layer-to-layer alignment

### 3.3 Example GDSII Output Structure

```
CustomSilicon.gds
├── Layer 0: N-Well
├── Layer 1: P-Well
├── Layer 2: Polysilicon (Gates)
├── Layer 3: N+ Diffusion
├── Layer 4: P+ Diffusion
├── Layer 10: Contact (Silicon to Metal1)
├── Layer 11: Metal Layer 1
├── Layer 12: Via1 (Metal1 to Metal2)
├── Layer 13: Metal Layer 2
├── Layer 14: Via2 (Metal2 to Metal3)
└── Layer 15: Metal Layer 3
```

---

## 4. Scale Comparison Examples

### 4.1 Microscopic (Silicon Chip)

```hw
define Space "Microprocessor":
    dimensions: 5mm by 5mm by 0.5mm
    grid: 5000 by 5000 by 5
    # Voxel = 1 micrometer (0.001mm)
    
    add Substrate(Silicon) spanning [1, 1, 1] to [1, 5000, 5000]
```

**Use Case**: Custom ALU, memory controller, signal processor

### 4.2 Standard (PCB Circuit)

```hw
define Space "SensorBoard":
    dimensions: 50mm by 50mm by 2mm
    grid: 500 by 500 by 2
    # Voxel = 0.1mm
    
    add Substrate(FR4) spanning [1, 1, 1] to [2, 500, 500]
```

**Use Case**: IoT devices, embedded systems, prototypes

### 4.3 Industrial (Server Rack)

```hw
define Space "DataCenter":
    dimensions: 2000mm by 1000mm by 500mm
    grid: 200 by 100 by 50
    # Voxel = 10mm
    
    add Substrate(Aluminum) spanning [1, 1, 1] to [50, 200, 100]
```

**Use Case**: Server chassis, industrial control panels, power distribution

---

## 5. Why This Makes Hardware Script Revolutionary

### 5.1 Unified Language Across Scales

Currently, the hardware industry is fragmented:

- **PCB Design**: KiCad (free), Altium Designer ($$$), Eagle
- **Silicon Design**: Cadence ($$$), Synopsys ($$$), Mentor Graphics ($$$)

These tools use completely different file formats, workflows, and interfaces. A PCB engineer cannot easily design a chip, and vice versa.

**Hardware Script unifies both domains**. A 10-year-old making a blinking LED board for a school project and a Senior Engineer at Nvidia designing the next AI microchip are using:
- The exact same `.hw` syntax
- The exact same `[Z, X, Y]` routing logic
- The exact same `hw verify` and `hw generate` commands

The only difference is the configuration file.

### 5.2 Democratizing Silicon Design

Traditionally, custom silicon design has been limited to:
- Large corporations with million-dollar tool licenses
- Universities with academic licenses
- Government research labs

Hardware Script makes silicon design accessible to:
- Hobbyists and makers
- Startups and small companies
- Students and educators
- Open-source hardware communities

### 5.3 AI-Native Silicon Development

Because Hardware Script uses text-based, declarative syntax, LLMs can:
- Generate complete chip designs from natural language specifications
- Iteratively optimize layouts based on performance constraints
- Suggest alternative routing strategies
- Automatically fix design rule violations

This enables **Agentic AI Loops** for silicon design—something impossible with traditional GUI-based tools.

---

## 6. Standard Silicon Library

### 6.1 standard.silicon.materials

```hw
import Silicon from standard.silicon.materials
import SiliconDioxide from standard.silicon.materials
import Polysilicon from standard.silicon.materials
import Aluminum from standard.silicon.materials
import Copper from standard.silicon.materials
import Tungsten from standard.silicon.materials
```

### 6.2 standard.silicon.logic

**Basic Gates**
```hw
import CMOS_Inverter from standard.silicon.logic
import CMOS_NAND_Gate from standard.silicon.logic
import CMOS_NOR_Gate from standard.silicon.logic
import CMOS_AND_Gate from standard.silicon.logic
import CMOS_OR_Gate from standard.silicon.logic
import CMOS_XOR_Gate from standard.silicon.logic
```

**Complex Logic**
```hw
import CMOS_FlipFlop_D from standard.silicon.logic
import CMOS_Latch_SR from standard.silicon.logic
import CMOS_Multiplexer from standard.silicon.logic
import CMOS_Adder_Full from standard.silicon.logic
```

### 6.3 standard.silicon.memory

```hw
import SRAM_Cell from standard.silicon.memory
import DRAM_Cell from standard.silicon.memory
import ROM_Cell from standard.silicon.memory
import Flash_Cell from standard.silicon.memory
```

### 6.4 standard.silicon.analog

```hw
import OpAmp from standard.silicon.analog
import Comparator from standard.silicon.analog
import VoltageReference from standard.silicon.analog
import CurrentMirror from standard.silicon.analog
```

---

## 7. Complete Silicon Design Example

```hw
# Custom 4-bit ALU (Arithmetic Logic Unit) in Silicon

import Silicon from standard.silicon.materials
import SiliconDioxide from standard.silicon.materials
import CMOS_NAND_Gate from standard.silicon.logic
import CMOS_XOR_Gate from standard.silicon.logic
import CMOS_Adder_Full from standard.silicon.logic

define Space "ALU_4bit":
    dimensions: 1mm by 1mm by 0.05mm
    grid: 10000 by 10000 by 5
    # Voxel = 100 nanometers
    
    # Silicon substrate
    add Substrate(Silicon) spanning [1, 1, 1] to [1, 10000, 10000]
    
    # Dielectric layers for metal routing
    add Substrate(SiliconDioxide) spanning [2, 1, 1] to [5, 10000, 10000]
    
    # Place 4 full adders for 4-bit addition
    add CMOS_Adder_Full named Adder0 at [1, 1000, 1000]
    add CMOS_Adder_Full named Adder1 at [1, 3000, 1000]
    add CMOS_Adder_Full named Adder2 at [1, 5000, 1000]
    add CMOS_Adder_Full named Adder3 at [1, 7000, 1000]
    
    # Route carry chain on Metal Layer 1
    route Adder0.carry_out to Adder1.carry_in:
        path:
            - [1, 1000, 1000]
            - [2, 1000, 1000]  # Up to Metal1
            - [2, 3000, 1000]  # Route to next adder
            - [1, 3000, 1000]  # Down to silicon
    
    route Adder1.carry_out to Adder2.carry_in:
        path:
            - [1, 3000, 1000]
            - [2, 3000, 1000]
            - [2, 5000, 1000]
            - [1, 5000, 1000]
    
    route Adder2.carry_out to Adder3.carry_in:
        path:
            - [1, 5000, 1000]
            - [2, 5000, 1000]
            - [2, 7000, 1000]
            - [1, 7000, 1000]
```

**Configuration (hw.config.yaml)**:
```yaml
name: "ALU_4bit"
version: "1.0.0"

targets:
  - type: simulation
    engine: blender
    export_path: "./build/sim.py"
  
  - type: manufacturing
    format: gds_II
    process_node: "130nm"
    export_path: "./build/foundry/alu.gds"
```

**Commands**:
```bash
# Verify the design
hw verify alu.hw

# Generate GDSII for foundry
hw generate --target manufacturing

# Generate 3D simulation
hw generate --target simulation
```

---

## 8. Future Silicon Features

### 8.1 Planned for v0.3

- **Analog Circuit Support**: Op-amps, comparators, voltage references
- **Mixed-Signal Design**: ADC/DAC integration
- **Power Grid Analysis**: IR drop calculation and optimization
- **Timing Analysis**: Critical path detection and optimization

### 8.2 Under Consideration

- **Standard Cell Libraries**: Pre-optimized logic gates for specific process nodes
- **Place and Route Optimization**: Automatic layout optimization algorithms
- **Parasitic Extraction**: Capacitance and resistance calculation
- **SPICE Integration**: Analog simulation integration

---

**Document Status**: Draft Specification  
**Last Updated**: March 2026  
**Part of**: Hardware Script (.hw) Documentation Suite

---
