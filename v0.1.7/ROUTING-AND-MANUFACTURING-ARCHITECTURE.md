# Hardware Script v0.1.7: The Unified 2.5D Routing, Stacking, and Manufacturing Architecture

**Document Type:** Architectural Authoritative Reference & Unified Specification  
**Status:** Canonical Reference (v0.1.7)  
**Focus:** 2.5D Shape-Based Routing, Z-Axis Stackups, Symmetrical/Length-Matching Patterns, and Dynamic Fabricator-Rule Enforcement  
**Core Philosophy:** Zero Implicit Magic. Reality as Code.

---

## Section 1: The Core Paradigm (Physical Truth vs. Visual Presentation)

Hardware Script v0.1.7 resolves the conflict between mathematical correctness (which a fabrication house requires) and rendering capabilities (which standard 3D/2D viewports require) by strictly isolating the compiler’s physical engine from the visual export layer.

1.  **The Voxel Grid / Space Database** holds the absolute, uncompromised physical truth. If a copper trace ends at X = 10.0mm, it ends exactly at 10,000,000nm in the database. No vertex coordinates are ever shifted, shaved, or scaled for visual convenience.
2.  **The Exporter Layer** uses non-destructive rendering and mathematical scene graph attributes to present this data to the GPU without introducing visual Z-fighting, and without altering a single nanometer of raw layout data.

## Section 2: The Z-Axis Abstraction & Stackup Manager

Raw integer layer indices (e.g., `z: 1`) are removed in favor of a dual-paradigm elevation system. All Z-coordinates are managed by the `StackupManager`.

### 2.1 The Two Paradigms

-   **The Assembly Paradigm (Continuous Physics)**: Used for custom or bare-metal layouts. Coordinates require explicit, absolute physical units (nm, µm, mm).
    ```hardware
    add pour(Silicon_N) named N_Well at z: 150um:
        thickness: 50um
        boundary: [x: 100um, y: 100um] to [x: 400um, y: 400um]
    ```
-   **The High-Level Paradigm (Semantic Intent)**: Used for standard PCB/IC stackups. Coordinates are abstracted using the `layer:` keyword, which references named layers in a Profile Stackup.
    ```hardware
    profile SimpleStackup:
        stackup:
            top_copper: [material: Copper, thickness: 35um]
            dielectric:    [material: FR4,    thickness: 1.5mm]
            bottom_copper: [material: Copper, thickness: 35um]

    space MyBoard:
        profile: SimpleStackup
        add pour(Copper) named Trace1 on layer: top_copper
    ```

### 2.2 Mathematical Resolution of the Stackup

The `StackupManager` calculates starting heights ($Z_{start}$) and ending heights ($Z_{end}$) relative to the Z-origin specified in the space.

-   For `origin: bl by b` (Bottom-Up), the first layer defined in the profile is assigned to Z = 0.
-   For `origin: tl by t` (Top-Down), the first layer is at the maximum height of the stackup.

The compiler converts all positions to absolute nanometers (i64) before passing them to the Voxel Engine.

## Section 3: The Zero-Flicker GPU Handshake (Depth Bias)

To prevent visual Z-fighting (flickering) when two distinct physical layers share an exact mathematical boundary plane, we implement a **Zero-Fudge Depth Bias Handshake**.

We explicitly reject "silent shaving" of geometries, which would alter physical files. Instead, we use the GPU’s hardware rasterizer unit to resolve depth ties dynamically.

### 3.1 The Culling & Precedence Rules

1.  **Face Culling**: When a conductive pour or component pad rests flush on a substrate layer, the compiler’s mesh generator automatically sets `culling.bottom = true` on the copper, completely omitting the redundant, invisible bottom face of the copper mesh.
2.  **The Layer Index Precedence**: The compiler calculates the depth-bias dynamically based on the Stackup index (Conductors adjacent to Dielectrics get a priority offset) and writes it to the glTF `extras` block.

### 3.2 The glTF Metadata Schema

```json
"materials": [
  {
    "name": "Copper",
    "extras": {
      "polygonOffset": true,
      "polygonOffsetFactor": -1.0,
      "polygonOffsetUnits": -1.0,
      "renderOrder": 10
    }
  }
]
```

## Section 4: The 2.5D Shape-Based Analytic Pathfinder

The routing engine operates as a 2.5D Shape-Based Pathfinder operating on continuous, nanometer-precision i64 coordinates. It replaces discrete voxel-crawling with analytic geometry.

### 4.1 The Planar Lock
The router constraints the A* pathfinder to run in 2D (X, Y) on a specific plane determined by the `StackupManager`. Z-movements are restricted to vertical layer-to-layer transitions (vias) which have a distinct, high-penalty routing cost.

### 4.2 Minkowski Obstacle Inflation
Instead of allocating billions of voxels to enforce clearances, the router utilizes **Minkowski Sum / Obstacle Inflation**:
1.  Queries the `BoundingBoxTracker` for all obstacle AABBs on the active layer.
2.  Inflates each obstacle's bounding box in X and Y by: $Inflation = \frac{Trace Width}{2} + Clearance$.
3.  The pathfinder routes an infinitesimally thin mathematical ray around these inflated boundaries, guaranteeing exact clearance with O(1) collision overhead.

### 4.3 Escape Exemption
To prevent the router from colliding with the very pins it is starting from, the router accepts `exempt_component_names` in its parameters. The source and destination component bounding boxes are treated as `Cost::SAFE` (0) during the initial and final steps, allowing the trace to escape the perimeter cleanly.

### 4.4 Swept-Volume Collision Checking
When evaluating a path segment with width $W$, the collision checker constructs a 3D bounding box for the swept volume and queries the 2D Spatial Index (R-Tree/Coarse Grid) for intersections.

## Section 5: The "Bouncer vs. Butler" Ohmic Bridge Paradigm

To bridge the gap between abstract logical routing and exact material chemistry:

-   **Assembly Level (The Bouncer)**: The compiler does not automatically inject intermediate contact materials. Manual placement of incompatible materials (e.g., Silicon to Metal without Silicide) triggers **Error P45: Forbidden Junction**.
-   **Router Level (The Butler)**: The router automatically reads the active Profile's bridge table and stamps the necessary compound via stacks at material transitions.

### The Via-Stamping "Sandwich" Rule
The auto-via inserter creates a **Compound Via** by stamping exactly one voxel layer of the specified bridge material (e.g., `Titanium_Silicide`) at the exact boundary, filling the rest of the vertical column with the low-resistance fill material (e.g., `Tungsten`).

---

## Section 6: Compiler Execution Pipeline

The `hwc` compiler executes a strict 5-stage pipeline for high-performance builds:

1.  **Pass 1: Topological Sort & Structural Placement**
    - Build the Spatial Dependency Graph and resolve all relative coordinate anchors.
2.  **Pass 2: Obstacle Blitting & Keepout Map**
    - Extract 3D bounding boxes and create the high-speed 2.5D obstacle map.
3.  **Pass 3: Parallel 2.5D Auto-Routing**
    - Lock Z-planes and execute Minkowski-inflated A* with auto-via stamping.
4.  **Pass 4: Verification & Validation**
    - Run P45 Forbidden Junction and R16 Analytic DRC checks. Execute parallel dummy fill if enabled.
5.  **Pass 5: Realization & Zero-Flicker Export**
    - Export mathematical vectors to DXF/SPICE and generate GLB with `polygonOffset` metadata.
