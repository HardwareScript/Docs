# Material Integrity and Vertical Interconnects

## The Foundational Principle: One Voxel, One Identity

Hardware Script enforces a strict physical law: **every voxel in 3D space can contain exactly one material**. This "Brutal Precision" approach eliminates ambiguity and ensures that your hardware design has deterministic, verifiable physics.

## The Surface-to-Surface Mandate

When connecting materials across different Z-layers (vertical interconnects), Hardware Script follows the **Surface-to-Surface Mandate**:

> **Contacts (vias) must occupy the dielectric layers BETWEEN the conductive layers they connect.**

This mirrors how real semiconductor fabrication works: vias don't interpenetrate with metal layers—they touch them at boundaries.

## The Layer Architecture

Hardware designs follow a strict layering model:

```
Layer 5: Metal/Conductive (Aluminum traces, pads)
         ↑ face-to-face contact
Layer 4: Dielectric (Tungsten/Polysilicon vias) ← CONTACTS LIVE HERE
         ↑ face-to-face contact
Layer 3: Active/Conductive (Silicon regions, gates)
         ↑
Layer 1-2: Substrate
```

### Key Rules

1. **Pours occupy conductive layers** - These are your active regions, metal traces, and pads
2. **Contacts occupy dielectric layers** - These are the vertical interconnects between conductive layers
3. **Connectivity through face-touching** - Materials connect at Z-boundaries, not by overlapping
4. **No horizontal interpenetration** - Different materials cannot occupy the same XY space on the same layer

## Syntax: Single-Layer Contacts

To connect Layer 3 to Layer 5, your contact lives **entirely on Layer 4**:

```hardware-script
## Layer 3: Silicon source region
add pour(Silicon_N) named Source on z:3:
    net: GND
    boundary: [x: 100um, y: 100um] to [x: 300um, y: 300um]

## Layer 5: Aluminum metal trace
add pour(Aluminum) named GND_Rail on z:5:
    net: GND
    boundary: [x: 100um, y: 100um] to [x: 300um, y: 150um]

## Layer 4: Tungsten via connecting them
## Note: spanning z:4 to z:4 (single layer)
add contact(Tungsten) named Via_Source net: GND at [x: 200um, y: 125um] spanning z:4 to z:4
```

### Why `spanning z:4 to z:4`?

The contact occupies **only Layer 4** (the dielectric layer). Its:
- **Bottom face** (z:4 start) touches the **top** of Layer 3 (Source)
- **Top face** (z:4 end) touches the **bottom** of Layer 5 (GND_Rail)

This creates electrical connectivity through **face-to-face contact** without any material interpenetration.

## The Physics Guarantee

The compiler enforces these rules through:

1. **P43 Material Integrity Check** - Prevents horizontal interpenetration on the same layer
2. **Sparse-Voxel Handshake** - Verifies that every voxel has exactly one material
3. **Connectivity Checker** - Validates electrical connections through face-touching

If you violate these rules, the build fails immediately with a clear error message.

## Example: NMOS Transistor with Vertical Interconnects

```hardware-script
space NMOS_Layout:
    dimensions: 1mm by 1mm by 1mm
    grid: 1000 by 1000 by 10
    origin: tl by t
    
    ## Substrate (Layers 1-2)
    add substrate(Silicon_P) spanning [x:1um, y:1um, z:1] to [x:999um, y:999um, z:2]
    
    ## Layer 3: Active regions
    add pour(Silicon_N) named Source on z:3:
        net: GND
        boundary: [x: 200um, y: 100um] to [x: 300um, y: 200um]
    
    add pour(Silicon_N) named Drain on z:3:
        net: VOUT
        boundary: [x: 200um, y: 300um] to [x: 300um, y: 400um]
    
    ## Layer 5: Metal interconnect
    add pour(Aluminum) named GND_Metal on z:5:
        net: GND
        boundary: [x: 100um, y: 100um] to [x: 300um, y: 200um]
    
    ## Layer 4: Vertical interconnect (dielectric layer)
    add contact(Tungsten) named Via_Source net: GND at [x: 250um, y: 150um] spanning z:4 to z:4
```

## Design Philosophy

This architecture embodies three core principles:

1. **Hardware-First Design** - The language models physical reality, not abstract concepts
2. **Brutal Precision** - No ambiguity, no guessing, no "mushing" of materials
3. **Foundry Boundary** - You describe the target state (mask), not the manufacturing process

The compiler acts as a **sentient guardian of physical truth**, catching violations before they become fabrication errors.

## Migration from Multi-Layer Spanning

If you have existing code with contacts spanning multiple layers (e.g., `spanning z:3 to z:5`), update them to occupy only the dielectric layer:

**Before:**
```hardware-script
add contact(Tungsten) named Via1 at [x: 200um, y: 150um] spanning z:3 to z:5
```

**After:**
```hardware-script
add contact(Tungsten) named Via1 at [x: 200um, y: 150um] spanning z:4 to z:4
```

The connectivity remains identical—the contact still connects Layer 3 to Layer 5—but now the physics is explicit and verifiable.

---

**See Also:**
- [Language Specification](LANGUAGE-SPEC.md) - Complete syntax reference
- [Architecture](ARCHITECTURE.md) - Compiler design and voxel engine
- [Getting Started](GETTING-STARTED.md) - Your first Hardware Script project
