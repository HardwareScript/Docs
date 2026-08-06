# Via Depth Control - Material-Based Penetration

**Document Type:** Feature Specification & Usage Guide  
**Version:** v0.2.1  
**Status:** Active  
**Date:** August 2026  
**Feature:** Material-Aware Via Penetration Depth Control

---

## Overview

HardwareScript v0.2.1 introduces **material-based via penetration control**, allowing IC designers to specify how deeply vias penetrate into conductive layers using percentages, absolute measurements, or material-specific rules.

### Why This Matters for ASIC Design

In IC fabrication, different materials require different via penetration depths:
- **Soft metals** (Aluminum): Shallow penetration (20-35%) prevents stress
- **Semiconductors** (Polysilicon): Deep penetration (60-80%) ensures ohmic contact
- **Refractory metals** (Tungsten): Moderate penetration (30-50%) balances contact resistance and mechanical integrity

---

## Quick Start

### Basic Usage: Percentage-Based Depth

```hardware
profile SimpleASIC:
    via:
        contact_depth: 50%    # All vias penetrate 50% of each layer
```

This means:
- Via penetrates **50% of the lower layer** (from top surface downward)
- Via penetrates **50% of the upper layer** (from bottom surface upward)
- If poly is 300nm thick and metal1 is 300nm thick, via penetrates 150nm into each

---

## Three Levels of Control

HardwareScript provides three levels of depth control with **clear precedence**:

```
┌─────────────────────────────────────────────────────────────┐
│ PRIORITY 1: Per-Instance Override (Highest)                │
│ ├─ Individual contact can specify custom depth             │
│ └─ Use when: Fixing problematic vias                       │
└─────────────────────────────────────────────────────────────┘
                            ▼ if not specified
┌─────────────────────────────────────────────────────────────┐
│ PRIORITY 2: Material-Specific PDK Rules                    │
│ ├─ PDK defines depth per material type                     │
│ └─ Use when: Different materials need different depths     │
└─────────────────────────────────────────────────────────────┘
                            ▼ if not specified
┌─────────────────────────────────────────────────────────────┐
│ PRIORITY 3: Global PDK Default (Fallback)                  │
│ ├─ Single depth value for all vias                         │
│ └─ Use when: Simple, uniform design                        │
└─────────────────────────────────────────────────────────────┘
```

---

## Level 1: Global PDK Default

The simplest approach: one depth value for all vias.

### Percentage Depth (Recommended for IC)

```hardware
profile UniformASIC:
    technology: "ASIC"
    
    stackup:
        poly:   [material: Polysilicon, thickness: 300nm, routable: true]
        d1:     [material: Silicon_Dioxide, thickness: 400nm, routable: false]
        metal1: [material: Aluminum, thickness: 300nm, routable: true]
    
    via:
        min_diameter: 200nm
        min_annular_ring: 50nm
        min_spacing: 300nm
        contact_depth: 50%    # All vias use 50% penetration
```

**Result:**
- Via from poly → metal1: penetrates 150nm into poly, 150nm into aluminum

### Absolute Depth

```hardware
profile AbsoluteASIC:
    via:
        contact_depth: 150nm    # All vias penetrate exactly 150nm
```

**Result:**
- Via from poly → metal1: penetrates 150nm into poly, 150nm into aluminum
- Works for uniform layer thicknesses, but doesn't adapt to thin layers

---

## Level 2: Material-Specific PDK Rules

Define different penetration depths for different materials. The compiler automatically selects the appropriate depth based on the layers being connected.

### Syntax

```hardware
profile MaterialAwareASIC:
    via:
        contact_depth: 50%    # Fallback default
        
        # Material-specific overrides
        material_contact_depths:
            Aluminum: 33%           # Soft metal, shallow penetration
            Polysilicon: 75%        # Need deep contact for ohmic junction
            Tungsten: 25%           # Very hard, shallow penetration
            Copper: 50%             # Standard penetration
            Silicon_N: 80%          # Nitride barrier, deep penetration
```

### How Material Selection Works

When a via connects two layers:
1. Compiler identifies materials of both layers (e.g., Polysilicon → Aluminum)
2. Looks up depth for lower layer material (Polysilicon: 75%)
3. Looks up depth for upper layer material (Aluminum: 33%)
4. Applies **each material's depth to its respective layer**

**Example:**

```hardware
# Stackup definition
stackup:
    poly:   [material: Polysilicon, thickness: 300nm]   # 300nm thick
    d1:     [material: Silicon_Dioxide, thickness: 400nm]
    metal1: [material: Aluminum, thickness: 300nm]      # 300nm thick

# Via placement
add contact(Tungsten) named V1 spanning layer: poly to metal1:
    diameter: 300nm
```

**Compiler resolution:**
- Lower layer (poly) material: **Polysilicon** → uses 75% depth → 225nm penetration
- Upper layer (metal1) material: **Aluminum** → uses 33% depth → 99nm penetration
- Via extends from Z = 75nm (300 - 225) to Z = 799nm (700 + 99)

### Mixing Percentages and Absolute Values

```hardware
profile MixedASIC:
    via:
        contact_depth: 50%
        material_contact_depths:
            Aluminum: 100nm         # Absolute value for aluminum
            Polysilicon: 75%        # Percentage for polysilicon
            Tungsten: 50nm          # Absolute value for tungsten
```

---

## Level 3: Per-Instance Override

Override depth for individual vias when needed (e.g., fixing a problematic connection or optimizing critical paths).

### Simple Override

```hardware
# Most vias use PDK defaults, but this one needs custom depth
add contact(Tungsten) named Critical_Via spanning layer: poly to metal1:
    diameter: 300nm
    contact_depth: 80%    # Override: use 80% instead of PDK default
```

### Asymmetric Depth (Directional Control)

```hardware
# Different penetration depths for lower vs upper layer
add contact(Tungsten) named Asymmetric_Via spanning layer: poly to metal1:
    diameter: 300nm
    contact_depth: {
        lower: 90%      # Penetrate 90% into poly (lower layer)
        upper: 20%      # Penetrate 20% into aluminum (upper layer)
    }
```

### Special Keywords: `through` and `touch`

#### `through` - Complete Penetration

```hardware
# Via penetrates completely through the lower layer
add contact(Tungsten) named Through_Via spanning layer: poly to metal1:
    diameter: 300nm
    contact_depth: {
        lower: through      # 100% through poly layer
        upper: 25%          # 25% into aluminum
    }
```

**Result:** Via starts at Z=0 (bottom of poly) and exits at Z=775nm (25% into aluminum)

#### `touch` - Minimal Contact

```hardware
# Via barely touches the layer (uses space resolution)
add contact(Tungsten) named Touch_Via spanning layer: poly to metal1:
    diameter: 300nm
    contact_depth: {
        lower: 50%          # 50% into poly
        upper: touch        # Minimal contact (1 resolution unit)
    }
```

**Result:** If `resolution: 10nm`, upper penetration is exactly 10nm

---

## Safety Bounds (Optional)

Prevent vias from penetrating too deep or too shallow, regardless of percentage calculations.

```hardware
profile SafeASIC:
    via:
        contact_depth: 50%
        min_contact_depth: 20nm     # Never less than 20nm
        max_contact_depth: 300nm    # Never more than 300nm
```

**Example with bounds:**
- Thin poly (100nm): 50% = 50nm, but clamped to **20nm minimum**
- Thick aluminum (800nm): 50% = 400nm, but clamped to **300nm maximum**

---

## Complete Example: Production-Ready ASIC PDK

```hardware
# production_pdk.hw
import * from @std/primitives/units
import Polysilicon, Aluminum, Tungsten, Silicon_Dioxide, Titanium_Silicide from "./materials"

export profile ProductionASIC:
    technology: "ASIC"
    
    stackup:
        poly:       [material: Polysilicon,     thickness: 300nm, routable: true]
        d1:         [material: Silicon_Dioxide, thickness: 400nm, routable: false]
        metal1:     [material: Aluminum,        thickness: 300nm, routable: true]
        d2:         [material: Silicon_Dioxide, thickness: 400nm, routable: false]
        metal2:     [material: Aluminum,        thickness: 300nm, routable: true]
    
    via:
        min_diameter: 200nm
        min_annular_ring: 50nm
        min_spacing: 300nm
        shape: SquareContact
        
        # Global default: 50% penetration
        contact_depth: 50%
        
        # Material-specific depths
        material_contact_depths:
            Aluminum: 33%           # Soft, avoid stress cracks
            Polysilicon: 75%        # Deep for ohmic contact
            Tungsten: 40%           # Refractory metal, moderate depth
        
        # Safety bounds
        min_contact_depth: 20nm     # Minimum for reliable contact
        max_contact_depth: 250nm    # Maximum to avoid over-etching
    
    trace:
        min_width: 200nm
        min_spacing: 200nm
```

---

## Usage in Space Definition

```hardware
import * from @std/primitives/units
import * from "./production_pdk"

space ProductionChip:
    dimensions: 10.0um by 10.0um by 1000nm
    resolution: 10nm
    profile: ProductionASIC
    
    nets:
        Signal_A: { classification: signal, potential: 1.8V }
    
    # Bottom pad (polysilicon)
    add pour(Polysilicon) named Poly_Pad on layer: poly:
        net: Signal_A
        dimensions: 2.0um by 2.0um
        at: [x: 3.0um, y: 3.0um]
    
    # Top pad (aluminum)
    add pour(Aluminum) named Metal_Pad on layer: metal1:
        net: Signal_A
        dimensions: 1.5um by 1.5um
        at: [x: 3.0um, y: 3.0um]
    
    # Via 1: Uses PDK material-specific depths
    # (Polysilicon: 75%, Aluminum: 33%)
    add contact(Tungsten) named Standard_Via spanning layer: poly to metal1:
        net: Signal_A
        diameter: 300nm
        align: center with Poly_Pad
    
    # Via 2: Custom override for critical path
    add contact(Tungsten) named Critical_Via spanning layer: poly to metal1:
        net: Signal_A
        diameter: 400nm
        contact_depth: 85%          # Override PDK defaults
        at: [x: 7.0um, y: 7.0um]
    
    # Via 3: Asymmetric depth for special case
    add contact(Tungsten) named Special_Via spanning layer: poly to metal1:
        net: Signal_A
        diameter: 300nm
        contact_depth: {
            lower: through          # Completely through poly
            upper: 25%              # Shallow aluminum penetration
        }
        at: [x: 5.0um, y: 5.0um]
```

---

## Compile-Time Evaluation

All depth expressions are evaluated **at compile time** during semantic lowering:

1. **Parse:** `contact_depth: 50%` → AST node `Expression::Percentage(0.5)`
2. **Resolve Layers:** Via connects poly (300nm) to metal1 (300nm)
3. **Apply Material Rules:** Polysilicon → 75%, Aluminum → 33%
4. **Calculate:**
   - Lower depth: `300nm * 0.75 = 225nm`
   - Upper depth: `300nm * 0.33 = 99nm`
5. **Apply Safety Bounds:** Clamp between min/max if defined
6. **Output to EntityGraph:** Via extends from Z=75nm to Z=799nm (fixed i64 values)

**Result:** Zero runtime computation. The 3D geometry is completely static.

---

## Migration Guide

### Before (v0.2.0 and earlier)

```hardware
profile OldStyle:
    via:
        contact_depth: 150nm    # Single global value
```

**Limitation:** All vias use 150nm regardless of material or layer thickness.

### After (v0.2.1)

```hardware
profile NewStyle:
    via:
        contact_depth: 50%      # Adapts to layer thickness
        material_contact_depths:
            Aluminum: 33%       # Material-aware
            Polysilicon: 75%
```

**Benefits:**
- Adapts to varying layer thicknesses
- Material-specific behavior
- Per-instance overrides when needed

---

## Best Practices

### 1. Start with Global Percentage
```hardware
via:
    contact_depth: 50%    # Simple default
```

### 2. Add Material Rules for Production
```hardware
via:
    contact_depth: 50%
    material_contact_depths:
        Aluminum: 33%
        Polysilicon: 75%
        Tungsten: 40%
```

### 3. Add Safety Bounds for Robustness
```hardware
via:
    contact_depth: 50%
    material_contact_depths:
        Aluminum: 33%
        Polysilicon: 75%
    min_contact_depth: 20nm
    max_contact_depth: 300nm
```

### 4. Override Only When Necessary
```hardware
# Most vias use PDK defaults
# Override only for special cases:
add contact(Tungsten) named Problematic_Via spanning layer: poly to metal1:
    diameter: 300nm
    contact_depth: 85%    # Needs deeper penetration
```

---

## Error Handling

### Missing `contact_depth`

```
ERROR: Contact 'Via_1' requires profile via.contact_depth but none is defined
HINT: Add 'contact_depth: 50%' or 'contact_depth: 150nm' to the 'via:' section of your profile.
```

### Invalid Depth Expression

```
ERROR: Contact depth must be a percentage (50%), measurement (150nm), or special keyword (through, touch)
FOUND: contact_depth: "invalid"
```

### Via Exceeds Space Bounds

```
ERROR: Via 'Via_1' extends below substrate base (Z=-50nm < 0nm)
Vias cannot penetrate below the wafer.
HINT: Reduce via.contact_depth (currently 75%) or adjust layer thicknesses.
```

---

## Reference: Depth Expression Types

| Expression | Example | Meaning |
|:-----------|:--------|:--------|
| **Percentage** | `50%` | 50% of layer thickness (adapts to each layer) |
| **Absolute** | `150nm` | Exact depth in nanometers (fixed for all layers) |
| **Through** | `through` | 100% penetration through entire layer |
| **Touch** | `touch` | Minimal contact (1 resolution unit, e.g., 10nm) |
| **Asymmetric** | `{ lower: 75%, upper: 33% }` | Different depths for lower vs upper layer |

---

## Summary

HardwareScript v0.2.1 via depth control provides:

✅ **Simple default:** `contact_depth: 50%`  
✅ **Material-aware:** Different depths per material  
✅ **Instance override:** Fix problematic vias  
✅ **Percentage or absolute:** Use what makes sense  
✅ **Safety bounds:** Prevent over/under penetration  
✅ **Zero new syntax:** Uses existing expression system  
✅ **Compile-time only:** No runtime computation  
✅ **ASIC-focused:** Designed for IC fabrication needs  

---

**Next Steps:**
- See `Authoritative-System-Specification.md` for expression evaluation rules
- See test cases in `hwc/tests/Via-Sandwich-Test/` for working examples
- Update your PDK profiles to use percentage-based depths
