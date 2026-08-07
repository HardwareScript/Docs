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

## Physical & Mathematical Precision

HardwareScript uses **pure mathematical expressions** for via depth control. No special keywords—just percentages (0% to 100%) and absolute measurements.

### Why Percentages Are Superior

| Physical Intent | Expression | Z-Coordinate Behavior |
|:----------------|:-----------|:----------------------|
| **Surface Contact** | `0%` or `0nm` | Via terminates exactly on layer's top surface (Z = Z_top). Zero penetration. Perfect for printed electronics, 2D materials (graphene), or thick-film deposition. |
| **Partial Penetration** | `1%` to `99%` or `20nm` to `280nm` | Via penetrates N% down into layer thickness from Z_top. Perfect for silicide contacts, microvias, and diffused junctions. |
| **Complete Punch-Through** | `100%` | Via penetrates 100% of layer thickness, from Z_top to Z_bottom. Perfect for through-silicon vias (TSVs) and through-hole drilling. |

### Mathematical Continuity

Unlike string keywords, percentages support full arithmetic:

```hardware
# ✅ Parametric depth calculation
let base_depth = 50%
let safety_margin = 5%

add contact(Tungsten) named V1 spanning layer: poly to metal1:
    diameter: 300nm
    contact_depth: base_depth + safety_margin    # Evaluates to 55%

# ✅ Mixed absolute and percentage
add contact(Tungsten) named V2 spanning layer: poly to metal1:
    diameter: 300nm
    contact_depth: {
        lower: 100%         # Full penetration (mathematical)
        upper: 0% + 20nm    # Surface + 20nm offset
    }
```

### Compile-Time Evaluation

All depth expressions evaluate once during semantic lowering:

1. **Lower Layer Depth:**
   ```
   ΔZ_lower = Thickness_lower × Depth_lower
   ```
   For `lower: 100%` → `ΔZ_lower = 300nm × 1.0 = 300nm`

2. **Upper Layer Depth:**
   ```
   ΔZ_upper = Thickness_upper × Depth_upper
   ```
   For `upper: 0%` → `ΔZ_upper = 300nm × 0.0 = 0nm`

3. **Via Z-Span:**
   ```
   Via_bottom = Z_lower_top - ΔZ_lower
   Via_top = Z_upper_bottom + ΔZ_upper
   ```

**Result:** Pure i64 coordinates in picometers, no runtime computation.

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

### Special Cases: Full Penetration and Surface Contact

#### Complete Penetration (100%)

```hardware
# Via penetrates completely through the lower layer
add contact(Tungsten) named Through_Via spanning layer: poly to metal1:
    diameter: 300nm
    contact_depth: {
        lower: 100%         # Completely through poly layer
        upper: 25%          # 25% into aluminum
    }
```

**Result:** Via starts at Z=0 (bottom of poly) and exits at Z=775nm (25% into aluminum)

#### Surface Contact (0%)

```hardware
# Via makes surface contact only (no penetration)
add contact(Tungsten) named Surface_Contact spanning layer: graphene to metal1:
    diameter: 300nm
    contact_depth: {
        lower: 0%           # Surface contact only (no penetration)
        upper: 30%          # 30% into aluminum
    }
```

**Result:** Via sits exactly on top of graphene surface, perfect for 2D materials or printed electronics

#### Minimal Penetration (Absolute)

```hardware
# Via with minimal fixed penetration
add contact(Tungsten) named Minimal_Via spanning layer: poly to metal1:
    diameter: 300nm
    contact_depth: {
        lower: 50%          # 50% into poly
        upper: 10nm         # Exactly 10nm into aluminum
    }
```

**Result:** Upper penetration is always 10nm regardless of aluminum thickness

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
            lower: 100%             # Completely through poly
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

## Common Use Cases

### Use Case 1: Printed Electronics (0% Depth)

For additive manufacturing where vias are printed/deposited on surface:

```hardware
profile PrintedElectronicsPDK:
    via:
        contact_depth: 0%    # All vias sit on surface (no etching)
        material_contact_depths:
            Graphene: 0%     # 2D material, no penetration
            Silver: 5%       # Slight penetration for adhesion
            Copper: 10%      # Moderate penetration
```

### Use Case 2: Through-Silicon Vias (100% Depth)

For TSVs that penetrate completely through substrate layers:

```hardware
profile TSV_PDK:
    stackup:
        substrate:  [material: Silicon, thickness: 50um, routable: false]
        metal1:     [material: Copper, thickness: 1um, routable: true]
    
    via:
        contact_depth: {
            lower: 100%      # Completely through substrate (TSV)
            upper: 30%       # Partial penetration into metal
        }
```

### Use Case 3: Hybrid Process (Mixed Depths)

Different materials require different penetration strategies:

```hardware
profile HybridProcessPDK:
    via:
        contact_depth: 50%    # Default
        material_contact_depths:
            Graphene: 0%            # Surface contact only
            Aluminum: 25%           # Shallow etch
            Polysilicon: 80%        # Deep etch for ohmic contact
            Silicon_N: 100%         # Full punch-through
```

### Use Case 4: Microvia Stacks (Graduated Depths)

For HDI PCBs with laser-drilled microvias:

```hardware
profile MicroviaPDK:
    via:
        contact_depth: 33%           # Laser microvias (shallow)
        min_contact_depth: 10nm      # Minimum etch depth
        max_contact_depth: 50nm      # Maximum to prevent over-etch
```

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
| **Full Penetration** | `100%` | 100% penetration through entire layer |
| **Surface Contact** | `0%` | No penetration (surface contact only) |
| **Asymmetric** | `{ lower: 75%, upper: 33% }` | Different depths for lower vs upper layer |
| **Mixed** | `{ lower: 100%, upper: 10nm }` | Combine percentage and absolute values |

---

## Summary

HardwareScript v0.2.1 via depth control provides:

✅ **Pure mathematical expressions:** `0%` to `100%` (no keywords)  
✅ **Simple default:** `contact_depth: 50%`  
✅ **Material-aware:** Different depths per material  
✅ **Instance override:** Fix problematic vias  
✅ **Percentage or absolute:** Use what makes sense  
✅ **Safety bounds:** Prevent over/under penetration  
✅ **Zero new syntax:** Uses existing expression system  
✅ **Compile-time only:** No runtime computation  
✅ **ASIC-focused:** Designed for IC fabrication needs  
✅ **Parametric math:** `0% + 20nm`, `base_depth * 1.2` all work  

### Key Design Principles

1. **Mathematical Continuity:** `0%` = surface, `100%` = through—no special keywords
2. **Expression Arithmetic:** Full support for `+`, `-`, `*`, `/`, `mod` operations
3. **Zero Token Pollution:** No new reserved words; `through` can be a signal name
4. **Physical Precision:** Direct mapping to Z-coordinates and layer boundaries
5. **Compile-Time Safety:** All depths resolve to fixed i64 values before synthesis  

---

**Next Steps:**
- See `Authoritative-System-Specification.md` for expression evaluation rules
- See test cases in `hwc/tests/Via-Sandwich-Test/` for working examples
- Update your PDK profiles to use percentage-based depths
