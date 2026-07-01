# Architectural Specification: Planar Island & Via Bridge (PIVB) Solver
**Document Type:** Engineering Architecture Specification  
**Status:** Approved for Implementation (v0.1.8)  
**Focus:** Connectivity Verification, Robust Polygon Geometric Welding, and Elimination of Vertical Coordinate-Snapping Errors

---

## 1. The Paradigm Shift: From Pixels to Manifolds

The legacy connectivity checker attempted to map continuous spatial coordinates onto a 1-based voxel grid or a brittle coordinate-snapping graph builder (`build_connectivity_graph`). This approach failed for sub-micron silicon where micro-sinking offsets (e.g., $500\text{nm}$ for Z-fighting mitigation) and via stack unrolling caused false-positive reports of physical gaps.

The **PIVB Solver** replaces coordinate-snapping with **Topological Connectivity Verification**. It recognizes that electrical continuity is defined by two fundamental geometric primitives:

1.  **Planar Islands (2D Nodes):** Continuous copper regions on a single layer, defined by pre-welded Clipper2 contours.
2.  **Vertical Bridges (Graph Edges):** Plated vias and contacts that act as vertical interconnects between these planar islands.

---

## 2. The PIVB Algorithmic Architecture

The PIVB solver operates in three deterministic passes, eliminating floating-point jitter and Z-depth sensitivity.

### Pass 1: Planar Island Extraction (Nodes)
Instead of walking raw trace segments, the solver queries the **Geometry Refinement Engine** for the pre-welded 2D contours. Because the middle-end already performs a Boolean Union (using `clipper2-rust`) to resolve Z-fighting and overlapping geometry, we already have a set of disjoint, manifold islands per layer.

*   Each unioned `Path64` contour is treated as a discrete **Planar Island**.
*   Islands are stored with their Z-interval metadata (e.g., `top_copper`, `metal1`).
*   This represents the "Electrical Truth" of that layer—everything within this contour is electrically contiguous.

### Pass 2: Vertical Bridge Mapping (Edges)
The solver iterates through all `ContactPlacement` entities (vias, contacts) assigned to the current Net ID. 

*   **Projection:** A via at $(X, Y)$ spanning layer $A$ to layer $B$ is a physical bridge.
*   **Intersection:** The solver performs a robust `Point-in-Polygon` check (using `clipper2-rust`) to identify which Planar Island on layer $A$ contains the via, and which Planar Island on layer $B$ contains the via.
*   **Graph Linking:** An undirected graph edge is added connecting the two identified islands.

### Pass 3: Connectivity Validation
The solver uses **Tarjan’s Strongly Connected Components (SCC)** algorithm on the resulting graph.

*   **Pass Condition:** The net is physically continuous if and only if the graph has exactly **one connected component**.
*   **Failure Condition:** If the graph contains $>1$ connected components, the net is fragmented. The solver generates a diagnostic report mapping the disconnected islands to their respective physical layers and $(X, Y)$ centers.

---

## 3. Engineering Advantages

| Feature | Legacy Voxel Checker | PIVB Solver |
| :--- | :--- | :--- |
| **Coordinate Sensitivity** | Fragile (snapping/jitter) | **Robust** (topological) |
| **Micro-Sinking Offset** | Causes false-positive gaps | **Transparent** (layer-interval mapping) |
| **Performance** | O(N² neighbors) | O(N log N + K) (sweep-line) |
| **Visual/Logic Sync** | Disconnected | **Unified** (shares Clipper2 contours) |
| **Via Tower Logic** | Fails on multi-segment unrolls | **Natively Handles** (via tower = bridge) |

---

## 4. Implementation Specification

### The Connectivity Graph Data Structure

```rust
pub struct PlanarIsland {
    pub layer_name: String,
    pub z_min: i64,
    pub z_max: i64,
    pub boundary: Paths64, // Pre-welded contour
    pub bbox: BoundingBox,  // Fast rejection
}
```

### The Verification Loop

```rust
// 1. Nodes = Welded Islands
for layer in stackup {
    for island in geometry.get_welded_islands(net_id, layer) {
        let idx = graph.add_node(island);
        // ...
    }
}

// 2. Edges = Via Vertical Bridges
for via in net_vias {
    let u = find_intersecting_island(via, layer_a);
    let v = find_intersecting_island(via, layer_b);
    if let (Some(u), Some(v)) = (u, v) {
        graph.add_edge(u, v, ());
    }
}

// 3. Validation
let scc = tarjan_scc(&graph);
if scc.len() > 1 {
    report_fragmentation(scc);
}
```

---

## 5. Diagnostic Reporting (P41: Disconnected Net)

When the PIVB Solver detects fragmentation, it no longer provides confusing, massive lists of thousands of "disconnected nodes." It provides **Electrical Island Reports**:

*   **Island Identification:** It groups disconnected components into numbered Islands (e.g., `Island 1`, `Island 2`).
*   **Bridge Detection:** Because the solver knows the physical gap between islands, it can suggest:
    *   *“Suggested fix: Add a via at (x, y) to bridge the gap on net [NetName].”*
*   **Visual Debugging:** The tool emits coordinates for every isolated island, allowing the **Hardware Script Monitor** to focus the 3D viewport directly on the orphaned geometry.

---

## 6. Migration and Integration Plan

1. **Deprecated:** The legacy `build_connectivity_graph()` coordinate-snapping routines in `connectivity_check.rs` are marked `#[deprecated]` and scheduled for removal.
2. **Unified Data:** The PIVB solver is injected into the post-synthesis pipeline. It consumes the same pre-welded 2D copper islands used by the GLB/DXF/GDSII export stage, ensuring that what you verify is exactly what you manufacture.
3. **Validation:** This architecture is now the single source of truth for the `P41: Disconnected Net` error variant, providing 100% fidelity to the physical reality of the board design.