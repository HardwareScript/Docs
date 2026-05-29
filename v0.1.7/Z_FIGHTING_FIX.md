# v0.1.7: Manifold Surface Handshake (Z-Fighting Fix)

In previous versions, rendering coplanar surfaces (e.g., a copper pour sitting exactly on top of an FR4 substrate) resulted in **Z-Fighting**—a visual flickering caused by the GPU's inability to determine which surface is closer due to floating-point precision limits.

Version 0.1.7 introduces the **Manifold Surface Handshake**, a clean architectural solution that eliminates flickering without modifying physical coordinates or using rendering hacks like depth bias.

## 1. The Problem: The "Blind Tie"
When two materials share the exact same Z-coordinate (e.g., `Z=5.000mm`), the GPU's depth buffer receives identical values for both. As the camera moves, microscopic rounding errors cause the pixels to alternate between the two materials, creating the "flicker" effect.

## 2. The Solution: Automatic Face Culling
Instead of "shaving off" material, the hwc-compiler now performs a **Surface Handshake** during the mesh generation phase.

### How it Works:
1. **Material Precedence**: Every material is assigned a precedence value (1 = High, 4 = Low). For example, Gold and Copper have higher precedence than FR4 (Substrate).
2. **Contact Detection**: During export, the Scene Graph analyzes all substrate layers to find shared boundaries.
3. **The Handshake**: If a High-Precedence material touches a Low-Precedence material:
   - The High-Precedence mesh **culls its own shared face**.
   - The result is that only **one face** exists at that specific coordinate in the final GLB file.
4. **Volume Preservation**: The system is "manifold-aware," meaning it only culls faces that are in direct contact with another material, preserving the internal integrity of the 3D model.

## 3. Implementation Details

### Scene Graph Analysis
The logic is implemented in `hwc-export/src/scene_graph/substrate.rs`. It iterates through all layers and compares their bounding boxes and precedences:

```rust
// Logic snippet from substrate.rs
if my_precedence < other_precedence {
    let touching_bottom = (layer.bbox.min.z - other.bbox.max.z).abs() < 1000; // 1um tolerance
    if touching_bottom && xy_overlap {
        base_culling.bottom = true;
    }
}
```

### Propagated Culling
The `FaceCulling` bitmask is now a first-class parameter in all mesh generation functions:
- `create_box_mesh`
- `create_cylinder_mesh`
- `create_box_with_holes_mesh` (Manifold Rule)

## 4. Benefits
- **Zero Flickering**: Perfectly stable rendering in all GLB/glTF viewers (WebGL, WebGPU, Blender, etc.).
- **100% Accurate Coordinates**: 1.0000mm remains 1.0000mm. No "silent magic" offsets are added to the file.
- **Scalable Architecture**: The system scales automatically. If you add a new material (e.g., Solder Mask), it simply needs a precedence value to participate in the handshake.
- **Manifold Geometry**: The resulting files are cleaner for downstream tools like 3D slicers or thermal simulators, as they contain no redundant internal faces.
