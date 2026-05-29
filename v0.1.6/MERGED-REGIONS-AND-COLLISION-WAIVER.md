# Merged Regions and Collision Waiver

**Version**: v0.1.6 Sprint 3.2  
**Status**: Complete  
**Philosophy**: NO IMPLICIT MAGIC - Explicit Intent Required

---

## Overview

Hardware Script provides explicit control over overlapping geometry through the `merge:` keyword. This feature is essential for multi-finger transistors, shared source/drain regions, and other advanced silicon structures where atoms must physically overlap.

## The Problem: Conflicting Safety Checks

Hardware Script has two collision detection systems:

1. **R15: Component Collision Check** - Prevents component bounding boxes from overlapping
2. **P12: Pour Geometry Check** - Detects overlapping internal pours without explicit merge intent

For multi-finger transistors, these checks conflict:
- **Pours MUST overlap** (shared source/drain regions)
- **But overlapping pours means overlapping components**
- **Result**: R15 blocks the design even though it's physically valid

## The Solution: Collision Waiver

The `merge:` keyword acts as a **Collision Waiver**. When you write `merge: [terminals]`, you explicitly declare:

> "I know these components will overlap. I authorize the geometry melting. Skip the collision check."

This resolves the conflict:
- **R15 check is WAIVED** for arrays with `merge:` keyword
- **P12 check is BYPASSED** for terminals in the merge list
- **Physical Reality is PRESERVED** through explicit intent

---

## How It Works

### 1. Normal Array (No Merge)

```hardware
add Transistor[3] named M_Array at [x: 10um, y: 20um, z: 2]:
    layout: horizontal_stack
    pitch: 5um
```

**Behavior**:
- ✅ R15: Component collision check ENABLED
- ✅ P12: Pour geometry check ENABLED
- ❌ If components overlap → R15 Error
- ❌ If pours overlap → P12 Error

### 2. Array with Merge Keyword

```hardware
add Transistor[3] named M_Array at [x: 10um, y: 20um, z: 2]:
    layout: horizontal_stack
    pitch: 1.5um
    merge: [source, drain]  # EXPLICIT INTENT
```

**Behavior**:
- ⚠️  R15: Component collision check **WAIVED** (skip_collision_check = true)
- ⚠️  P12: Pour geometry check **BYPASSED** for source and drain
- ✅ Components CAN overlap (collision waiver granted)
- ✅ Source/drain pours CAN overlap (explicit merge intent)
- ❌ Other pours (e.g., gate) still trigger P12 if they overlap

---

## Design Patterns

### Pattern 1: Realistic Component Sizes (Recommended)

**Problem**: Component shape is larger than pour, causing R15 collision.

**Solution**: Make component shape smaller, let pours extend beyond boundaries.

```hardware
component TransistorFinger:
    pins: [gate, source, drain]
    
    layout:
        ## Component shape: 1um wide (just the gate region)
        shape: Rectangle(1um, 10um, 0.5um)
        
        ## Source pour: 2um wide, extends BEYOND component boundaries
        ## Goes from -0.5um to 1.5um (relative to component origin)
        add pour(Silicon_N) named source on z:1:
            boundary: [x: -0.5um, y: 0um] to [x: 1.5um, y: 4um]
            net: GND
        
        ## Drain pour: 2um wide, also extends beyond
        add pour(Silicon_N) named drain on z:1:
            boundary: [x: -0.5um, y: 6um] to [x: 1.5um, y: 10um]
            net: VDD

## Array with pitch 1.5um
## Components: 1um wide, pitch 1.5um → NO overlap ✓
## Pours: 2um wide, pitch 1.5um → 0.5um overlap ✓
add TransistorFinger[3] named M_Array at [x: 10um, y: 20um, z: 2]:
    layout: horizontal_stack
    pitch: 1.5um
    merge: [source, drain]
```

**Result**:
- ✅ Components don't overlap (R15 passes even without waiver)
- ✅ Pours overlap by 0.5um (P12 bypassed for merged terminals)
- ✅ Merged regions created: `M_Array[0-2].source`, `M_Array[0-2].drain`
- ✅ Netlist annotates merged regions for parasitic extraction

### Pattern 2: Overlapping Components (Uses Collision Waiver)

**Problem**: Component shape equals pour size, causing unavoidable overlap.

**Solution**: Use `merge:` keyword to waive R15 collision check.

```hardware
component Finger:
    pins: [left, right]
    
    layout:
        ## Component shape: 2um wide
        shape: Rectangle(2um, 10um, 0.5um)
        
        ## Pour: 2um wide (fills entire component)
        add pour(Silicon_N) named source on z:1:
            boundary: [x: 0um, y: 0um] to [x: 2um, y: 10um]
            net: GND

## Array with pitch 1.5um
## Components: 2um wide, pitch 1.5um → 0.5um overlap!
## Pours: 2um wide, pitch 1.5um → 0.5um overlap!
add Finger[3] named F_Array at [x: 10um, y: 20um, z: 2]:
    layout: horizontal_stack
    pitch: 1.5um
    merge: [source]  # COLLISION WAIVER GRANTED
```

**Result**:
- ⚠️  Components overlap by 0.5um (R15 waived by merge keyword)
- ✅ Pours overlap by 0.5um (P12 bypassed for source)
- ✅ Merged region created: `F_Array[0-2].source`
- ✅ Build succeeds with explicit intent

---

## Netlist Annotations for Parasitic Extraction

When pours are merged, the netlist includes special annotations:

```spice
* ========================================
* NETS
* ========================================
* Net: GND (merged region: M_Array[0-2].source, 1 instances, total area: 20000000 nm², material: Silicon_N, layer: 1)
*   Parasitic extraction: Treat as single electrical node
* Net: VDD (merged region: M_Array[0-2].drain, 1 instances, total area: 20000000 nm², material: Silicon_N, layer: 1)
*   Parasitic extraction: Treat as single electrical node
```

**Key Information**:
- **Merged region name**: `M_Array[0-2].source` (all instances merged)
- **Total area**: Sum of all merged pour areas (for capacitance calculation)
- **Parasitic extraction note**: Treat as single electrical node (not 3 separate nodes)

This enables accurate parasitic extraction:
- **Resistance**: R_total = R_single / n (parallel combination)
- **Capacitance**: C_total = sum of all areas (combined surface area)

---

## Error Handling

### Error 1: Overlapping Pours Without Merge

```hardware
add Finger[3] named F_Array at [x: 10um, y: 20um, z: 2]:
    layout: horizontal_stack
    pitch: 1.5um
    # NO merge keyword!
```

**Result**:
```
Error: P12 (Geometric Collision)
  × Geometric collision in array 'F_Array': instances 0 and 1 have overlapping geometry
  help: Array instances have overlapping geometry without explicit merge intent.
        Physical Reality: Two pieces of material cannot occupy the same physical space.
        
        To fix this, either:
        1. Increase pitch to 2.5um or more
        2. Add explicit merge intent: merge: [source]
```

### Error 2: Overlapping Components Without Merge (Pattern 2 only)

If you use Pattern 2 (component size = pour size) without the merge keyword:

```hardware
add Finger[3] named F_Array at [x: 10um, y: 20um, z: 2]:
    layout: horizontal_stack
    pitch: 1.5um
    # NO merge keyword, but components overlap!
```

**Result**:
```
Error: R15 (Component Collision)
  × Component 'F_Array[1]' collides with existing component at position [0.012mm, 0.025mm, 2.000mm]
  help: Check component position, rotation, and ensure no collisions with other components.
        
        If this is intentional (e.g., multi-finger transistor), add explicit merge intent:
        merge: [source]
```

---

## Implementation Details

### Collision Waiver Mechanism

1. **Parser**: Sets `skip_collision_check: false` by default on all `ComponentPlacement`
2. **Array Unroller**: Sets `skip_collision_check: true` when `merge_terminals` is not empty
3. **Placer**: Checks `skip_collision_check` flag before running R15 collision detection
4. **P12 Guard**: Checks if terminal is in `merge_terminals` list before reporting collision

### Code Flow

```
User writes: merge: [source, drain]
    ↓
Parser: array_config.merge_terminals = ["source", "drain"]
    ↓
Array Unroller: 
    - For each instance:
        - Set skip_collision_check = !merge_terminals.is_empty()
        - Create ComponentPlacement with flag
    ↓
Component Placer:
    - Pass skip_collision_check to PlacementParams
    ↓
Placer (R15 Check):
    - if !skip_collision_check { check_collision() }
    - else { /* WAIVED */ }
    ↓
P12 Guard:
    - For each pour:
        - if pour.name in merge_terminals { continue }
        - else { check_collision() }
    ↓
Merge Logic:
    - Find overlapping pours for merged terminals
    - Create merged regions with combined bounding boxes
    - Set merged_region_id on PourMetadata
    ↓
Netlist Export:
    - Group pours by merged_region_id
    - Annotate with total area and parasitic extraction note
```

---

## Philosophy: NO IMPLICIT MAGIC

The `merge:` keyword embodies Hardware Script's core philosophy:

1. **Explicit Intent**: User must declare overlapping geometry
2. **No Hidden Behavior**: Collision waiver is visible in source code
3. **Physical Reality**: Overlapping atoms must be explicitly authorized
4. **Helpful Errors**: P12 and R15 guide users to correct syntax

**Bad (Implicit)**:
```hardware
# Compiler silently merges overlapping pours
add Transistor[3] at [x: 10um, y: 20um, z: 2]:
    pitch: 1um  # Pours overlap, but no explicit intent!
```

**Good (Explicit)**:
```hardware
# User explicitly declares merge intent
add Transistor[3] at [x: 10um, y: 20um, z: 2]:
    pitch: 1um
    merge: [source, drain]  # "I know they overlap. Melt them."
```

---

## Summary

| Feature | Without `merge:` | With `merge:` |
|---------|-----------------|---------------|
| R15 Component Collision | ✅ Enforced | ⚠️ **WAIVED** |
| P12 Pour Geometry Check | ✅ Enforced | ⚠️ Bypassed for merged terminals |
| Overlapping Components | ❌ Error | ✅ Allowed (explicit intent) |
| Overlapping Pours | ❌ Error | ✅ Allowed (explicit intent) |
| Merged Regions | ❌ Not created | ✅ Created with combined area |
| Parasitic Extraction | Individual nodes | **Single electrical node** |
| Netlist Annotation | Standard | **Merged region with total area** |

**Key Takeaway**: The `merge:` keyword is a **Permission Slip** that tells the compiler: "I know these will overlap. I have authorized the atom melting. Skip the safety checks."

---

## Test Files

- `test_merged_region_proper.hw` - Pattern 1 (realistic sizes, pours extend beyond)
- `test_merged_region_netlist.hw` - Pattern 2 (overlapping components, collision waiver)
- `test_p12_collision_no_merge.hw` - Error case (overlap without merge)
- `test_p12_multiple_terminals.hw` - Multiple merged terminals

---

## Future Enhancements

- [ ] Parasitic extraction API that uses merged_region_id
- [ ] Visual indication of merged regions in GLB export
- [ ] Merge validation (ensure merged pours actually overlap)
- [ ] Performance optimization for large merged arrays
