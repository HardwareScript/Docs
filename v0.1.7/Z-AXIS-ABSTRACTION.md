# Hardware Script v0.1.7: The Z-Axis Abstraction Fix

**Document Type**: Architectural Authoritative Reference & Migration Guide  
**Status**: Breaking Change (v0.1.6 → v0.1.7)  
**Focus**: Z-Axis Coordinates, Semantic Layers, and Profile Stackups

## 1. Executive Summary

Hardware Script v0.1.7 introduces a critical architectural fix to how the Z-axis is handled in spatial design.

In earlier versions, Z-coordinates were defined using raw integers (e.g., `z: 1`), which implicitly leaked the internal 0-based array indices of the Voxel Engine to the user. This created a "Z-Axis Abstraction Leak," forcing engineers to align their 1-based mental models (Layer 1, Layer 2) with the compiler's 0-based memory arrays.

**The Fix**: Hardware Script now strictly separates Voxel Reality (internal memory) from Human Intent.

As of v0.1.7, raw integers are strictly forbidden for Z-coordinates. You must now choose between two paradigms:

1.  **The Assembly Paradigm**: Pure physical units (e.g., `z: 1.5mm`).
2.  **The High-Level Paradigm**: Semantic layer names defined by a Profile (e.g., `layer: l1`).

---

## 2. The Breaking Change: Why Raw Integers Are Dead

### The Problem (v0.1.6)

```hscript
# ❌ v0.1.6 (Deprecated)
add pour(Copper) on z: 1:
    boundary: [x: 0mm, y: 0mm] to [x: 10mm, y: 10mm]
```

Notice the inconsistency: `x` and `y` require physical units (`mm`), but `z` does not. Because `z: 1` represented a voxel array index, changing the grid resolution of your space would silently destroy the physical thickness of your board.

### The Solution (v0.1.7)

The compiler's Voxel Engine has been refactored to speak exclusively in **nanometers (nm)**. The Voxel Engine no longer knows what a "layer" is.

To interface with the engine, the user must use either Physical Units (Assembly) or Semantic Names (High-Level).

---

## 3. Paradigm A: The Assembly Format (Continuous Physics)

**Philosophy**: Brutal precision. You are the master of the atom. You describe exactly where material exists in 3D space using physical units.

**Syntax Rule**: `z` is treated exactly like `x` and `y`. It requires a measurement unit (`nm`, `um`, `mm`).

### Example: Bare-Metal Silicon Design

In Assembly mode, you manually calculate the physical height and thickness of every element.

```hscript
space BareMetal_SSD_Controller:
    dimensions: 5mm by 5mm by 1mm
    grid: 1000 by 1000 by 100
    origin: bl by b

    # 1. The Substrate (3D Bounding Box in absolute units)
    add substrate(Silicon_P) spanning [x: 0, y: 0, z: 0] to [x: 5mm, y: 5mm, z: 200um]
    
    # 2. Pours use a physical Z-height and a physical thickness
    add pour(Silicon_N) named N_Well at z: 150um:
        thickness: 50um
        boundary: [x: 100um, y: 100um] to [x: 400um, y: 400um]
        
    add pour(Copper) named M1_Trace at z: 200um:
        thickness: 20um
        boundary: [x: 150um, y: 150um] to [x: 800um, y: 150um]

    # 3. Contacts/Vias span between precise physical heights
    add contact(Tungsten) named Via_1 at [x: 150um, y: 150um] spanning z: 150um to 200um:
        diameter: 50nm
```

**How the Compiler handles this**: The IR Builder converts `z: 150um` to `150,000nm`. It then divides by the space's internal Z-voxel size to find the correct array index. The user never sees or cares about the array.

### 3.1 Net-Aware Geometric Termination (v0.1.8)
A critical rule for physical accuracy: **Vias do not drill through their target net.**
- **Through-Hole Vias**: Span the entire board (Top surface to Bottom surface).
- **Blind/Buried Microvias**: Must terminate at the **top surface** (for contacts from above) or **bottom surface** (for contacts from below) of the target inner copper layer. 
- **Micro-Sinking**: To prevent visual Z-fighting, the via terminates 500nm *inside* the target copper layer. This ensures a solid volumetric overlap and a stable render without "Phantom Plates" appearing underneath the trace.

---

## 4. Paradigm B: The High-Level Format (Semantic Intent)

**Philosophy**: The compiler is the Butler. You think in logical manufacturing layers (`l1`, `top`, `m1`), not nanometers.

**Syntax Rule**: The Z-axis is completely abstracted. You use the `layer:` keyword instead of `z:`, and the compiler calculates absolute Z-heights using a **Stackup** defined in your Profile.

### Step 1: The Profile Stackup

A `stackup` block must be defined in the active profile. We highly recommend using short, ergonomic, lowercase names (e.g., `l1`, `l2`, `d1`) to maximize typing speed.

```hscript
# @std/profiles/pcb_standard.hw
profile HighSpeedPCB:
    # The compiler calculates cumulative absolute Z-heights top-to-bottom
    stackup:
        l1: [material: Copper, thickness: 35um]  # Top layer
        d1: [material: FR4,    thickness: 0.2mm] # Dielectric 1
        l2: [material: Copper, thickness: 35um]  # Inner layer 1
        d2: [material: FR4,    thickness: 1.0mm] # Core
        l3: [material: Copper, thickness: 35um]  # Bottom layer
```

### Step 2: Semantic Placement in Space

Notice that in High-Level mode, the `z` axis vanishes entirely from the space definition. The user only works in 2D (`x`, `y`), and the compiler handles the 3D reality.

```hscript
import HighSpeedPCB from @std/profiles/pcb_standard

space SSD_Motherboard:
    profile: HighSpeedPCB
    dimensions: 80mm by 22mm   # No Z-dimension needed! Stackup calculates it.
    
    # 1. Pours sit on named layers
    add pour(Copper) named HighSpeed_RX on layer: l1:
        boundary: [x: 0, y: 5mm] to [x: 10mm, y: 5mm]
        
    add pour(Copper) named GND_Plane on layer: l2:
        boundary: [x: 0, y: 0] to [x: 80mm, y: 22mm]

    # 2. Vias span between named layers
    add contact(Copper) named GND_Via at [x: 5mm, y: 5mm] spanning layer: l1 to l2:
        diameter: 0.3mm

    # 3. Components are placed on named layers
    add NAND_Flash_Chip named U1 at [x: 40mm, y: 11mm] on layer: l1
    
    # 4. Routing automatically generates necessary vias through the stackup
    route Controller.tx to U1.rx
```

**How the Compiler handles this**: When the IR Builder sees `layer: l2`, it checks the Profile. It calculates the thickness of `l1 + d1`, realizes that `l2` physically exists at exactly `Z = 235um`, and quietly passes `235um` to the Voxel Engine.

---

## 5. Migration Guide (v0.1.6 to v0.1.7)

Updating your existing `.hw` files to v0.1.7 is a quick, deterministic process.

### Migrating Pours & Components

If you want to use **Assembly (Physical)**:
```hscript
# Old
add pour(Copper) on z: 2:
# New (multiply your old z-index by your old layer thickness)
add pour(Copper) at z: 2mm:
```

If you want to use **High-Level (Semantic)**:
```hscript
# Old
add pour(Copper) on z: 1:
# New
add pour(Copper) on layer: l1:
```

### Migrating Contacts (Vias)

If you want to use **Assembly (Physical)**:
```hscript
# Old
spanning z: 1 to z: 2
# New
spanning z: 1mm to 2mm
```

If you want to use **High-Level (Semantic)**:
```hscript
# Old
spanning z: 1 to z: 2
# New
spanning layer: l1 to l2
```

---

## 6. Internal Architecture Notes (For Contributors)

For those working on the `hwc` compiler source code in Rust, the AST has been updated to reflect this dual-paradigm approach.

In `hwc-parser/src/ast.rs`, elevation logic is now strictly typed:

```rust
pub enum Elevation {
    /// Represents Assembly Paradigm: e.g., `z: 1.5mm`
    Physical(Measurement),
    
    /// Represents High-Level Paradigm: e.g., `layer: l1`
    Semantic(Identifier),
}
```

The `hwc-engine` (Voxel Grid) has been purged of all `layer_index` references. Methods like `stamp_voxel` and `route_path` now exclusively accept `Point3D { x_nm: i64, y_nm: i64, z_nm: i64 }`.

The translation between `Elevation` and `z_nm` happens exactly once inside `hwc-compiler/src/ir/space_builder.rs` via the `StackupManager`.
