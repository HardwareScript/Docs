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

### 3.1 Extension: Same-Net Copper Handshake (v0.1.8)
The standard precedence rule handles Copper-on-FR4 perfectly, but same-material interfaces (e.g., a Via Pad sitting on a Trace) require an additional rule.

#### 3.1.1 Bounding Box Match Guard
If two layers have the same material, same net, and same precedence:
- The **lower layer** in the Z-stack culls its **top face** ONLY if their XY bounding boxes match exactly (e.g., stacked planes).
- This prevents "Open Box" artifacts where a partial overlap (like a via on a trace) would otherwise delete the entire top face of the trace.

#### 3.1.2 Micro-Sinking (Z-Offset)
For partial intersections (vias landing on traces), Z-fighting is eliminated by sinking the via slightly (500nm) into the target copper volume. This hides the shared coplanar faces inside the solid copper volume without requiring mesh culling.

## 4. The Implementation: Strategy A (Physical Unioning)
The architectural goal for Hardware Script is now reality in v0.1.7: **Strategy A: 2D Co-Union / 3D Boolean Union**.

### 4.1 Why Physical Unioning?
-   **Engineering Fidelity**: Engineering solvers (FEA, Thermal, EM) require non-overlapping, manifold geometry.
-   **Additive Manufacturing**: 3D printers require unified contours to avoid over-deposition.
-   **Physical Truth**: In a real PCB, the copper of a via and a trace are a single contiguous lattice of atoms.

### 4.2 Material Synchronization (v0.1.7)
To enable perfect unioning, the compiler now performs **Automatic Material Synchronization**. 
-   **Manual Traces**: Inherit the material of the layer they are placed on (e.g., `ViaCopper` from the stackup).
-   **Binding Logic**: Pours and pads are grouped by `(Layer, Net, Material)`. By ensuring materials match, they are merged into a single manifold mesh, completely eliminating Z-fighting without rendering hacks.

## 5. Benefits
- **Zero Flickering**: Perfectly stable rendering in all GLB/glTF viewers (WebGL, WebGPU, Blender, etc.).
- **100% Accurate Coordinates**: 1.0000mm remains 1.0000mm. No "silent magic" offsets are added to the file.
- **Structural Integrity**: Via holes are carved exactly at the drill diameter with 0nm same-net clearance, ensuring physical truth.
