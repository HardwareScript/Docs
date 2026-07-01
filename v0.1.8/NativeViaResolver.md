# Architectural Specification: Native Via Resolver (v0.1.8)
**Document Type:** Engineering Architecture Specification  
**Status:** Approved for Implementation  
**Focus:** Conductive Horizon Traversal, PDK-Driven Bridge Resolution, and O(log N) Spatial Collision Detection

---

## 1. The Architectural Paradigm Shift

The legacy `AutoViaInserter` treated the board as a monolithic "voxel stack," requiring the compiler to iterate through every layer index in a stack, regardless of material properties. This led to "Layer Blindness"—the compiler would attempt to drill through insulators (e.g., `SiO2`, `gate_oxide`), failing or producing physically impossible geometry.

The **Native Via Resolver** implements a "Conductive Horizon" philosophy:

1.  **Material-Awareness**: The resolver understands that only conductive layers (Metals, Semiconductors, Silicides) carry current. Insulators are treated as spatial boundaries, not "layers to be drilled."
2.  **PDK-Driven Execution**: The resolver does not contain hardcoded logic for how to stack a via. It queries the **Bridge Registry** (defined in the Profile) to determine the precise interface materials required for a transition.
3.  **Spatial-Index Routing**: By abandoning voxel-grids for collision detection, the resolver performs spatial lookups directly against the `EntityGraph` using logarithmic R-tree queries.

---

## 2. Core Operational Principles

### 2.1 Conductive Horizon Traversal
Instead of iterating through every layer index $i$ to $j$, the Resolver fetches the **Conductive Horizon List**:

$$H = \{ L_i \mid \text{Material}(L_i).category \in \{\text{Conductor, Semiconductor}\} \}$$

The Resolver only generates via instructions for the transitions between these horizons. If a transition involves jumping over an insulator (e.g., $M1 \to M2$ over an $SiO_2$ layer), the Resolver automatically creates a single continuous via structure, skipping the intermediate index.

### 2.2 The Bridge Registry (PDK Enforcement)
The Resolver executes a "Fail-Fast" protocol based on the Profile's `BridgeTable`.

*   **Rule Lookup**: `resolve_bridge(Mat_A, Mat_B)`
*   **Success**: Returns a stack of interface materials (e.g., `[Titanium_Silicide, Tungsten]`).
*   **Failure**: If no bridge rule exists for the transition, the compiler immediately halts with `IrError::MissingPdkBridge(Mat_A, Mat_B)`.

This ensures that "illegal" material transitions (like Copper directly touching Silicon) are physically impossible to route.

---

## 3. The Resolver Execution Pipeline

The `ViaResolver` operates as a discrete compiler pass, transforming logical net transitions into a physical instruction set (`ViaInstruction`).

### Pass 1: Transition Discovery
The resolver extracts all layer transitions from the logical `NetlistArena`. For a net transitioning from `Metal1` to `Metal3`, it generates a `LayerTransition` object that defines the start/end physical coordinates and the material stack.

### Pass 2: Spatial Intersection (rstar/geo-index)
Rather than checking voxels, the Resolver performs a spatial query:
*   It fetches all `PlanarIslands` (2D contours) at the transition coordinates $(X, Y)$ on all intermediate layers.
*   It ensures that every transition in the stack has a valid planar connection. If a transition lands in a "keepout" or "empty" region, it triggers an `IrError::ViaCollision`.

### Pass 3: PDK Bridge Stamping
The Resolver assembles the via stack based on the Bridge Registry:
1.  **Interface Resolution**: Retrieves the necessary bridge materials.
2.  **Geometry Generation**: Creates the `ViaPlacement` instruction, including the specific hole drill, interface material stamping, and core fill material.
3.  **Instruction Set Emission**: The instructions are passed to the `GeometryRefinement` engine to generate the final 3D Manifold Mesh.

---

## 4. Engineering Impact

| Metric | Legacy `AutoViaInserter` | Native `ViaResolver` |
| :--- | :--- | :--- |
| **Logic** | Pancake/Layer Iteration | Conductive Horizon Traversal |
| **Material Handling** | Hardcoded/Voxel-default | PDK-Driven `BridgeRegistry` |
| **Collision Detection** | $O(N \cdot L)$ (Iterative Voxel) | $O(\log S)$ (Spatial R-tree) |
| **PDK Violation** | Late-stage DRC check | **Fail-Fast (Build-Time Error)** |
| **Z-Axis Handling** | Brittle Grid Indices | Absolute Nanometer `i64` |

---

## 5. Migration Strategy: "Delete and Replace"

To implement the `ViaResolver`, follow this "Delete and Replace" workflow:

1.  **Deprecation Phase**:
    *   Delete `crates/hwc-compiler/src/auto_via_inserter/`.
    *   Delete all references to `AutoViaInserter` and `collision.rs`.
    *   Remove `LayerTransition` and associated legacy types.

2.  **Implementation Phase**:
    *   Create `crates/hwc-compiler/src/via_resolver/`.
    *   Implement `ViaResolver` trait:
        *   `resolve_bridge(from, to)`
        *   `get_conductive_horizons(start, end)`
    *   Implement the `EntityGraph` query logic using `rstar` to replace voxel collision checks.

3.  **Verification Phase**:
    *   Run `test_complex_hybrid_pcb.hw`.
    *   Verify that TSV and via stacks generate the correct interface materials (Silicide/Tungsten) based on the PDK rules.
    *   Verify build-time halt on forbidden material junctions.

---

## 6. Conclusion
By deleting the legacy via system and implementing the **Native Via Resolver**, we align the compiler with its v0.1.8 goals: **Total PDK rule enforcement, sub-millisecond pathfinding, and the elimination of magic heuristics.** The compiler will now be a deterministic engine that executes the PDK's manufacturing truth, rather than guessing at it.