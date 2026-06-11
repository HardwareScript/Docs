# v0.1.7: Solder Mask Opening (Tented vs. Exposed Vias)

## Overview

Version 0.1.7 introduces **Profile-Driven Solder Mask Openings**, giving the compiler full awareness of the protective coating that covers a PCB. Every via that passes through a board face now automatically interacts with the solder mask — either punching a hole through it (exposed) or leaving it intact (tented) — without requiring the user to manually define mask layers or specify expansion values.

This completes the Physical Truth Migration for board-level vertical interconnects: vias, substrate drills, antipads, thermal reliefs, and now solder mask openings are all generated from a single profile-driven pipeline.

## 1. The Problem: The "Invisible Mask"

Before v0.1.7, the compiler had no concept of solder mask. Vias drilled through the substrate and copper planes, but the outermost protective layer — the green (or other color) coating that covers the entire board surface — was simply absent from the 3D model.

In real PCB fabrication, every via that reaches a board face interacts with the solder mask:

- **Exposed vias** have a circular opening punched through the mask, exposing the copper annular ring for soldering.
- **Tented vias** are sealed shut by the mask, preventing solder flow and protecting the via from environmental exposure.

Without this, the 3D model was physically incomplete and not manufacturing-ready.

## 2. The Solution: Zero Implicit Magic

The solder mask system follows the same architectural principle as the rest of v0.1.7: **no hardcoded constants in Rust source**. All values come from the user's profile or fall back to IPC-7351 industry defaults.

### 2.1 Profile-Driven Expansion

The mask expansion value — how far the opening extends beyond the copper pad — is stored in the profile's `manufacturing` block:

```hw
profile MyProfile:
    manufacturing:
        solder_mask_expansion: 75um    # IPC-7351 default
```

### 2.2 Formula

The solder mask opening diameter is computed as:

```
Opening Diameter = Pad Diameter + 2 × Profile Solder Mask Expansion
```

Where:

- `Pad Diameter = Drill Diameter + 2 × Min Annular Ring`
- `Min Annular Ring` comes from `profile.via.min_annular_ring` (default 150µm)
- `Solder Mask Expansion` comes from `profile.manufacturing.solder_mask_expansion` (default 75µm)

### 2.3 Auto-Created Mask Layers

Users do not need to define solder mask layers in their stackup. The compiler automatically creates two thin (20µm) `SolderMask` substrate layers at the board boundaries:

- **Top mask**: Z = `[stackup_height, stackup_height + 20µm]`
- **Bottom mask**: Z = `[-20µm, 0]`

The Z-positions are derived from `stackup_manager.board_thickness_nm()` — the actual physical thickness of the copper+dielectric stack — not from the user-defined space depth. This ensures the mask sits exactly on the board surface regardless of how the user defined their space dimensions.

If the user explicitly defines solder mask layers in their stackup, the auto-creation is skipped (no duplicates).

## 3. Syntax

### 3.1 Exposed Via (default)

```hw
add contact(ViaCopper) named MyVia at [x: 5mm, y: 5mm] spanning layer: top to bottom:
    net: SIG
    drill_diameter: 0.3mm
    is_tented: false                # default: false (exposed)
```

Result: A circular opening is cut through both the top and bottom solder mask layers, exposing the copper annular ring.

### 3.2 Tented Via

```hw
add contact(ViaCopper) named TentedVia at [x: 5mm, y: 5mm] spanning layer: top to bottom:
    net: GND
    drill_diameter: 0.3mm
    is_tented: true
```

Result: The solder mask remains intact over the via. No opening is cut.

### 3.3 Profile Configuration

```hw
profile MyProfile:
    stackup:
        top:    [material: Copper, thickness: 35um]
        d1:     [material: FR4, thickness: 200um]
        core:   [material: FR4, thickness: 1060um]
        bottom: [material: Copper, thickness: 35um]

    via:
        min_diameter: 0.1mm
        min_annular_ring: 0.05mm

    manufacturing:
        solder_mask_expansion: 75um    # Controls opening size
```

## 4. Engine Implementation

### 4.1 Substrate Layer Type

A new `SolderMask` variant was added to `SubstrateLayerType` in `crates/hwc-engine/src/voxel_grid/substrate_layers.rs`:

```rust
pub enum SubstrateLayerType {
    Pour,
    Contact,
    Substrate,
    SolderMask,  // NEW: solder mask coating on top/bottom board faces
}
```

### 4.2 Extended `drill_via_hole()`

The `drill_via_hole()` function in `crates/hwc-engine/src/voxel_grid/grid/substrate_ops.rs` was extended to accept solder mask parameters:

```rust
pub fn drill_via_hole(
    &mut self,
    hole_bbox: BoundingBox,
    diameter_nm: i64,
    via_net: NetId,
    clearance_nm: i64,
    is_tented: bool,           // NEW
    pad_diameter_nm: i64,      // NEW
    solder_mask_expansion_nm: i64,  // NEW
)
```

The `SolderMask` match arm computes the opening diameter and clamps the cutout bounding box to the mask layer's Z-range:

```rust
SubstrateLayerType::SolderMask => {
    if !is_tented {
        let opening_diameter = pad_diameter_nm + 2 * solder_mask_expansion_nm;
        let mask_cutout_bbox = BoundingBox::new(
            Point3D::new(
                hole_bbox.min.x.max(layer.bbox.min.x),
                hole_bbox.min.y.max(layer.bbox.min.y),
                layer.bbox.min.z,      // Use mask's own Z
            ),
            Point3D::new(
                hole_bbox.max.x.min(layer.bbox.max.x),
                hole_bbox.max.y.min(layer.bbox.max.y),
                layer.bbox.max.z,      // Use mask's own Z
            ),
        );
        layer.add_cylinder_cutout(mask_cutout_bbox, opening_diameter);
    }
}
```

### 4.3 Cutout BBox Z-Clamping

A critical detail: the via's bounding box Z-range `[0, stackup_height]` may only *touch* the solder mask layer's Z-range `[stackup_height, stackup_height + 20µm]` at a single boundary point. The export slicing logic uses strict inequality (`cutout.max.z > slice.z_start`), which would fail to match a boundary-touching cutout.

The fix clamps the cutout bounding box to the mask layer's own Z-range, ensuring the cutout fully spans the mask thickness and the export slicing logic can correctly identify it.

### 4.4 Auto-Creation in `program_to_space`

The solder mask layers are auto-created in `crates/hwc-compiler/src/ir/mod.rs` before the placement loop:

```rust
let stackup_height_nm = stackup_manager.board_thickness_nm();
let mask_thickness_nm = 20_000; // 20µm

// Top solder mask
let top_mask_bbox = BoundingBox::new(
    Point3D::new(0, 0, stackup_height_nm),
    Point3D::new(width_nm, height_nm, stackup_height_nm + mask_thickness_nm),
);

// Bottom solder mask
let bottom_mask_bbox = BoundingBox::new(
    Point3D::new(0, 0, -mask_thickness_nm),
    Point3D::new(width_nm, height_nm, 0),
);
```

### 4.5 ContactMetadata Extension

`ContactMetadata` in `crates/hwc-engine/src/space.rs` was extended with two new fields:

```rust
pub struct ContactMetadata {
    // ... existing fields ...
    pub is_tented: bool,
    pub mask_clearance_diameter_nm: Option<i64>,
}
```

These are populated from the contact's `properties` HashMap during IR placement and propagated through the auto-via inserter and unrolling systems.

## 5. Design Decisions

### Why Profile-Driven (Option A) over Explicit Values (Option B)?

The auto-via inserter generates vias without explicit mask parameters in `.hw` scripts. Profile-driven defaults ensure every via gets correct mask behavior automatically. Requiring explicit values (Option B) would force users to add boilerplate to every contact definition.

### Why Auto-Create Mask Layers?

Requiring users to manually add `SolderMask` layers in their stackup is pure boilerplate — every PCB has solder mask. Auto-creation follows the same pattern as auto-generated antipads and thermal reliefs: the compiler infers physical reality from the stackup and profile.

### Why `board_thickness_nm()` instead of `space.depth_nm`?

The user-defined space depth (e.g., `1.6mm`) is an arbitrary bounding box dimension. The actual physical thickness of the copper+dielectric stack (e.g., `1.365mm`) may differ. The solder mask must sit on the actual board surface, not the arbitrary bounding box boundary. Using `stackup_manager.board_thickness_nm()` ensures physical accuracy.

## 8. Dependencies

- Item 1 (Plated Through-Hole Via) — **DONE**
- Item 2 (Antipad / Plane Clearance) — **DONE**
- Item 3 (Thermal Relief) — **DONE**
- Manufacturing Process DNA — **DONE** (materials declare `process: drilled_plated`)
