# Hardware Script (.hw) Specification Overview

**Version**: 0.2 (Draft)  
**Paradigm**: Software-Defined Bare-Metal Hardware  
**Core Philosophy**: Deterministic Fractal Hardware - Hardware as pure, AI-readable, physical-abstracted text

---

## Documentation Structure

This specification has been divided into three focused documents for better organization:

1. **ARCHITECTURE.md** - System architecture, grid system, compilation pipeline, and implementation details
2. **LANGUAGE-SPEC.md** - Complete language syntax, grammar, keywords, and code examples
3. **TOOLING.md** - CLI tools, package management, IDE integration, and ecosystem

---

## Quick Start

For a complete hardware design example, see the end of this document.

For detailed information, refer to:
- **ARCHITECTURE.md** - System design and implementation
- **LANGUAGE-SPEC.md** - Complete syntax reference
- **TOOLING.md** - Development tools and ecosystem

---

## Core Concepts Summary

### The Universal Space

Every .hw file declares a **Space** - a bounded physical volume with discrete grid resolution:

```hw
define Space "BoardName":
    dimensions: 50mm by 50mm by 2mm
    grid: 100 by 100 by 2
```

The Synthesizer calculates voxel size: `50mm / 100 = 0.5mm` resolution

### 3D Tensor Coordinates [Z, X, Y]

All placement uses discrete coordinates:
- **Z**: Layer (1=top, 2=inner, 3=bottom)
- **X**: Column
- **Y**: Row

```hw
add Battery named Power at [1, 10, 10]
```

### Waypoint Routing

Define vertices; the Synthesizer interpolates paths:

```hw
route Power.out to LED.in:
    path:
        - [1, 10, 10]  # Start
        - [1, 50, 10]  # Waypoint
        - [2, 50, 10]  # Layer change (Via created)
        - [2, 90, 10]  # Route on layer 2
        - [1, 90, 10]  # Back to layer 1
```

### Explicit Component Logic

Logic is implemented through physical components from the standard library:

```hw
import AND_Gate from standard.logic
import NOT_Gate from standard.logic

add AND_Gate named Logic1 at [1, 50, 50]
add NOT_Gate named Logic2 at [1, 70, 50]

route SensorA.out to Logic1.Input_A:
    path:
        - [1, 30, 30]
        - [1, 50, 30]
        - [1, 50, 50]

route Logic1.Output to Logic2.Input:
    path:
        - [1, 50, 50]
        - [1, 70, 50]
```

---

## Complete Example: Automated Sprinkler Controller

---

**Document Status**: Draft Specification  
**Last Updated**: March 2026  
**Maintainers**: Hardware Script Core Team  
**License**: To Be Determined

---


This example demonstrates the complete Hardware Script workflow from imports to explicit logic implementation:

```hw
# 1. Imports from standard library
import Copper from standard.materials
import Battery from standard.power
import LightSensor from standard.sensors
import Comparator from standard.analog
import VoltageDivider from standard.analog
import Transistor from standard.switches

# 2. Define the physical workspace
define Space "Automated Sprinkler":
    dimensions: 50mm by 50mm by 2mm
    grid: 25 by 25 by 2
    
    # Add FR4 substrate across all layers
    add Substrate(FR4) spanning [1, 1, 1] to [2, 25, 25]
    
    # 3. Place components using 3D tensor coordinates
    add Battery (5V) named PowerSource at [1, 5, 5]
    add LightSensor named Eye at [1, 12, 12]
    add VoltageDivider (ratio: 0.5) named Threshold at [1, 15, 15]
    add Comparator named LogicGate at [1, 18, 18]
    add Transistor named Valve at [1, 20, 20]
    
    # 4. Route electrical connections with explicit waypoints
    route PowerSource.out to Eye.in:
        path:
            - [1, 5, 5]      # Start at battery
            - [1, 12, 5]     # Travel horizontally
            - [1, 12, 12]    # Arrive at sensor
    
    route PowerSource.out to Threshold.in:
        path:
            - [1, 5, 5]
            - [1, 15, 5]
            - [1, 15, 15]
    
    route Threshold.out to LogicGate.negative_pin:
        path:
            - [1, 15, 15]
            - [1, 18, 15]
            - [1, 18, 18]
    
    route Eye.out to LogicGate.positive_pin:
        path:
            - [1, 12, 12]    # Start at sensor
            - [2, 12, 12]    # Via to layer 2
            - [2, 18, 12]    # Route on bottom layer
            - [2, 18, 18]    # Position under comparator
            - [1, 18, 18]    # Via back to top layer
    
    route LogicGate.out to Valve.base:
        path:
            - [1, 18, 18]
            - [1, 20, 18]
            - [1, 20, 20]
    
    # Connect to external motor
    route Valve.collector to "EXTERNAL_SPRINKLER_MOTOR":
        path:
            - [1, 20, 20]
            - [1, 25, 20]
```

**What this creates:**
- A 50mm × 50mm PCB with 2 layers
- Discrete 2mm voxel resolution (50mm / 25 = 2mm per cell)
- Automatic via generation at layer transitions
- Physical logic using comparator and transistor components
- Ready for simulation and manufacturing

---

## CLI Usage

```bash
# Verify the design
hw verify sprinkler.hw

# Generate simulation
hw generate --target simulation

# Generate manufacturing files
hw generate --target manufacturing
```

---

**Document Status**: Overview and Quick Reference  
**Last Updated**: March 2026  
**Part of**: Hardware Script (.hw) Documentation Suite

For complete details, see:
- **ARCHITECTURE.md** - System architecture and grid mathematics
- **LANGUAGE-SPEC.md** - Full language syntax and grammar
- **TOOLING.md** - CLI tools, packages, and ecosystem

---
