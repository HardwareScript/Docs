# v0.1.7: Manufacturing Process DNA (Physical Behavior Abstraction)

## Overview
In Hardware Script v0.1.7, we have moved beyond simple material categorization (Conductor/Insulator) to a **Process-Aware Architecture**. Materials now carry "Manufacturing DNA" that dictates how they interact with the substrate during Z-axis transitions.

## The Problem: The "Ghost Hole" Bug
Previously, the compiler used a hardcoded list of materials (like "Copper") to decide if a via should "drill" through the substrate or simply "deposit" material. This caused failures when using custom materials or visual inspection materials (like `TransCopper`), where the system would place a conductive cylinder without carving the necessary hole in the FR4 or surrounding pours.

## The Solution: `process` Attribute
Materials now explicitly declare their physical manufacturing behavior using the `process` keyword.

### Syntax
```hw
material HighSpeedCopper:
    category: conductor
    process: drilled_plated    # <--- The DNA
    properties:
        resistivity: 1.68e-8 ohm_m
        ...
```

### Supported Processes
1. **`drilled_plated`** (PCB Default):
   - **Behavior**: Carves a physical void (hole) through all substrate and conductive layers it spans.
   - **Result**: Creates a Plated Through Hole (PTH) with an annular ring and internal plating.
   - **Usage**: Standard PCB vias, mounting holes.

2. **`deposited`** (CMOS/IC Default):
   - **Behavior**: Fills the specified volume without subtracting from the environment.
   - **Result**: A solid "plug" or "pillar" of material.
   - **Usage**: IC contacts, 3D-printed conductive traces, thick-film resistors.

3. **`etched`** (MEMS/Subtractive):
   - **Behavior**: Removes existing material in the specified volume.
   - **Usage**: Micro-channels, cavity resonators.

## Why this is significant for Hardware Scripts
This abstraction decouples **Physical Geometry** from **Manufacturing Intent**. An engineer can now define a material's behavior based on the target fabrication house (Foundry vs. PCB Shop) without changing the core layout logic.

It ensures that the **Voxel Engine** always maintains physical integrity, preventing "impossible geometry" where two materials occupy the same space without a defined manufacturing relationship.
