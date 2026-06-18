
# Unified 2.5D/3D Routing and Placement Architecture

**Document Type:** Architectural Reference & Advanced Technical Specification  
**Status:** Canonical Reference (v0.1.7 / v0.1.8)  
**Focus:** Unified Pathfinder Core, Domain-Specific Routing (PCB vs. ASIC), Pin Escape Heuristics, Multi-Trace Junctions, and Deterministic Lock-Files.

---

## Section 1: The Unified Pathfinder Core (The Microkernel)

To prevent compiler bloat and maintain a single source of truth across printed circuit boards (PCBs) and silicon dies (ASICs), the compiler utilizes a **Unified 2.5D/3D Pathfinder Engine**. The core pathfinding mathematical routines remain entirely agnostic of the physical domain [MICROKERNEL-ARCHITECTURE.md]. 

Instead of hardcoding domain rules, the pathfinder acts as a raw mathematical engine that routes coordinate vectors in nanometers ($i64$ precision), while the active stackup profile dynamically configures the routing constraints [Z-AXIS-ABSTRACTION-IMPLEMENTATION.md].

### 1.1 Minkowski Sum Obstacle Inflation
Rather than allocating massive memory arrays to track pixel or voxel-level clearances, the pathfinder uses continuous geometric bounding boxes [ROUTING-AND-MANUFACTURING-ARCHITECTURE.md]. For any given routing trace of width $W_{\text{trace}}$ and clearance limit $C_{\text{clearance}}$ defined in the active profile, all layout obstacles (keepouts, component bodies, non-net copper) are inflated dynamically using the **Minkowski Sum**:

$$\text{Inflation} = \frac{W_{\text{trace}}}{2} + C_{\text{clearance}}$$

The router represents all obstacles as Axis-Aligned Bounding Boxes (AABBs). The pathfinder projects an infinitesimally thin mathematical ray around these inflated bounding boxes, guaranteeing that any path found maintains exact clearance bounds with $O(1)$ collision-evaluation overhead [Routing-&-Manufacturing-Roadmap.md].

---

## Section 2: Domain-Specific Routing Frameworks

When the router initializes, it reads the active stackup profile and configures its expansion algorithm into one of two primary domain behaviors.

### 2.1 The PCB Routing Domain (Continuous & Octilinear)
PCB layout is characterized by arbitrary continuous positioning, relative spacing, and signal integrity requirements that forbid sharp corners.

```
       PCB Octilinear Movement (45° / 90°)
       
                    [North]
               \       │       /
                 \     │     /
                   \   │   /
       [West] ───────┼─────── [East]
                   /   │   \
                 /     │     \
               /       │       \
                    [South]
```

*   **Octilinear Pathfinding**: The expansion tree evaluates 8 neighbors (North, South, East, West, and 45-degree diagonals: Northeast, Northwest, Southeast, Southwest).
*   **Gridless Spatial Routing**: Grid snapping is disabled or configured to a coarse grid (e.g., $100\mu\text{m}$). Clearance and track selection are entirely driven by the Minkowski Sum of the physical traces.
*   **Via Span Flexibility**: Vias can be routed as through-holes (spanning the entire Z-depth), blind vias (outer-to-inner), or buried vias (inner-to-inner) [Via-Engine-Implementation.md].

### 2.2 The Silicon (ASIC) Routing Domain (Gridded & Manhattan)
Silicon layouts require strictly orthogonal paths snapped to predefined manufacturing tracks.

```
       ASIC Manhattan Movement (90° only)
       
                    [North]
                       │
                       │
                       │
       [West] ───────┼─────── [East]
                       │
                       │
                       │
                    [South]
```

*   **Manhattan Pathfinding**: The expansion tree is restricted to 4 neighbors (North, South, East, West). 45-degree diagonals are strictly forbidden.
*   **Gridded Routing Tracks**: The pathfinder snaps all traces to the exact coordinate tracks dictated by the profile’s `track_pitch` and `grid_snapping` properties.
*   **Preferred-Direction Layers**: To maximize routing density and prevent wire deadlocks, the router penalizes directional deviation on alternating metal layers:
    *   **Odd Metal Layers (M1, M3, M5, ...)**: Direction is horizontally biased. Vertical steps incur a heavy path-cost penalty.
    *   **Even Metal Layers (M2, M4, M6, ...)**: Direction is vertically biased. Horizontal steps incur a heavy path-cost penalty.
*   **Via Towers**: Inter-layer connections cannot span multiple layers in a single pass. A route from M1 to M4 must be unrolled into a vertical tower of discrete, single-layer vias and intermediate landing pads [Via-Engine-Implementation.md].

---

## Section 3: The "Strict Boundary-Docking Box Model" (Pin Escape & Docking)

> **Implementation Status (v0.1.7):** ✅ Complete — Strict Interior Lockout enforced.
> - **3.1 Port Selection Heuristic**: Implemented in `port_escape.rs` with `smart_corner_clamp()`
> - **3.2 Escape Vector Enforcement**: Implemented in `calculate_rect_escape()` and `calculate_circular_escape()`
> - **3.3 Strict Interior Lockout**: Implemented in `router.rs` — All component interiors are impenetrable obstacles.

To prevent traces from colliding with the pins they are exiting, the router utilizes the **Origin-Facing Box Model**. Every landing pad, pin, or via is represented as a bounding box with four virtual cardinal **Docking Ports** (North, South, East, West).

### 3.1 Port Selection Heuristic
To determine which port to use for escape routing, the pathfinder evaluates the spatial direction to the target coordinate (or a designated system origin, such as the center of the board):

$$\vec{D}_{\text{target}} = \vec{P}_{\text{target}} - \vec{P}_{\text{pad\_center}}$$

The pathfinder calculates the dot product between $\vec{D}_{\text{target}}$ and the unit normal vectors $\vec{N}_{\text{port}}$ of each of the four ports:

$$\text{Score} = \vec{D}_{\text{target}} \cdot \vec{N}_{\text{port}}$$

The port yielding the highest positive score is selected as the active escape dock.

```
             North Port (Normal: [0, 1])
                    ┌─────────┐
                    │  ┌───┐  │
 West Port          │  │ O │  │          East Port
 (Normal: [-1, 0])  │  └───┘  │          (Normal: [1, 0])
                    └─────────┘
             South Port (Normal: [0, -1])
```

### 3.2 Escape Vector Enforcement
Once a port is selected:
1.  The router establishes the starting node at the exact boundary edge of that port.
2.  The router projects an orthogonal **Escape Step** along the port's normal vector. The length of this step must equal or exceed the minimum clearance distance before the A\* engine is permitted to evaluate any turns. This eliminates "acid trap" crevices and prevents routing paths from violating their own pad boundaries.

### 3.3 Virtual Sub-Ports (For Large Pads)
For large power/ground pads or thermal tabs where multiple parallel traces must connect to a single pad edge, the compiler calculates the maximum number of allowable parallel paths:

$$\text{Max Parallel Traces} = \left\lfloor \frac{L_{\text{edge}} - C_{\text{clearance}}}{W_{\text{trace}} + S_{\text{trace}}} \right\rfloor$$

Where:
*   $L_{\text{edge}}$ is the physical length of the pad boundary edge.
*   $W_{\text{trace}}$ is the width of the exiting trace.
*   $S_{\text{trace}}$ is the minimum trace-to-trace spacing.

If multiple traces are routed, the pad edge is partitioned into $N$ virtual sub-ports evenly offset along the boundary, ensuring that each exiting trace maintains precise clearance.

---

## Section 4: Multi-Trace Net Topologies & Steiner Trees

An electrical net is rarely a simple point-to-point connection. It is a logical tree connecting $N$ physical pins.

### 4.1 Steiner Minimum Tree (SMT) Routing
When routing multi-point nets, the pathfinder solves for a Steiner Minimum Tree (SMT) to minimize total trace length. Rather than treating each pin pair as an isolated routing pass, the router uses **Same-Net Trace Tapping**:

1.  The router routes the path between the first two pins (Pin 1 and Pin 2).
2.  The resulting copper segments are committed to the grid and tagged with the active `NetID`.
3.  When routing the third pin (Pin 3), the pathfinder treats the existing copper of `NetID` as a valid target destination.
4.  The pathfinder terminates the route the moment it intersects any point along the existing trace, creating a physical T-junction.

```
      Manual Segment Tapping (Shared Junction Coordinate)
      
                     [Trunk Segment]
      Pin A ─────────────────X───────────────── Pin B
                             │
                             │ [Tap Segment]
                             │
                           Pin C
```

### 4.2 Manual Segment Tapping Syntax
To declare a branching junction manually, the user does not need specialized node keywords. They simply write separate `route` statements that share an absolute coordinate on the same net:

```hardware
# Main Trunk Segment
route PinA to PinB:
    net: VCC_Rail
    width: 0.8mm
    path: [
        [10mm, 10mm],
        [20mm, 10mm]          # Shared Junction Point
    ]

# Tapped Branch Segment
route [20mm, 10mm] to PinC:   # Begins exactly at the junction coordinate!
    net: VCC_Rail
    width: 0.4mm
    path: [
        [20mm, 10mm],
        [20mm, 20mm]
    ]
```

### 4.3 The "Wide Junction" Taper (Mitered Joins)
When a thick trunk trace meets a thin branch trace, the sudden transition creates physical manufacturing stress and signal reflections. If `strategy: WideJunction` is enabled, the compiler’s `TeardropEngine` [teardrops.rs] automatically generates a triangular transition wedge (a mitered taper) at the junction point:

```
                  Thick Trunk
               =================┐
                                 \
                                  \───────── Thin Branch
                                  /─────────
                                 /
               =================┘
                 Mitered Taper (Wedge)
```

During GDSII/Gerber export, the 2D Clipper engine [2D-POLYGON-UNIONING-IMPLEMENTATION.md] welds these overlapping geometries into a single, continuous, non-overlapping copper manifold.

---

## Section 5: Standard Routing Patterns as Strategic Guides

Routing patterns (e.g., Zigzag, Trombone, Serpentine, Spiral) are not processed as rigid, unyielding paths. If they were, any obstacle in their path would cause a hard routing failure.

Instead, they act as **Heuristic Guides** that alter the A\* search cost dynamically.

### 5.1 The Heuristic Warping Cost
The A\* cost function evaluates nodes as follows:

$$f(n) = g(n) + h(n) + w(n)$$

Where:
*   $g(n)$ is the cost to reach node $n$.
*   $h(n)$ is the estimated distance to the target destination.
*   **$w(n)$ is the Pattern Warping Penalty**.

As the pathfinder expands, it compares the direction of the expansion step against the current step vector of the pattern template:
*   If the expansion step aligns with the pattern's active step, the penalty is zero: $w(n) = 0$.
*   If the expansion step deviates from the pattern, a heavy penalty is applied: $w(n) = \text{Penalty}_{\text{pattern}}$.

If an obstacle blocks the path, the cost of aligning with the pattern spikes to infinity ($Cost::INFINITE$). The pathfinder automatically selects the lower-cost option: deviating around the obstacle, then snapping back to the pattern once the clearance zone is passed.

### 5.2 Polar-to-Cartesian Trigonometric Rasterizer
For patterns that utilize non-orthogonal turns (such as the 45-degree steps in the `Spiral` pattern), the compiler translates polar steps into Cartesian coordinate offsets. To prevent floating-point rounding drift, the calculations are executed with $i64$ fixed-point scaling ($10^6$ multiplier):

$$x_{\text{new}} = x_{\text{old}} + \frac{d \cdot \cos(\theta) \cdot 10^6}{10^6}$$

$$y_{\text{new}} = y_{\text{old}} + \frac{d \cdot \sin(\theta) \cdot 10^6}{10^6}$$

All endpoints are rounded and snapped back to the nearest grid resolution coordinate in nanometers.

---

## Section 6: Profile Schema & Syntax Definitions

To prevent hardcoding of manufacturing and physical rules within the compiler binary, all constraints must be declared inside the stackup profile. 

### 6.1 Extended Profile Syntax
The `profile` block supports explicit declarations for layer routing directions, via-to-layer enclosures, and stacked via permissions:

```hardware
# @foundry/tsmc/180nm.hw

profile TSMC_180nm:
    # 1. Physical Stackup & Preferred Routing Directions
    stackup:
        sub: [material: Silicon_P, thickness: 300um]
        m1:  [material: Copper,    thickness: 500nm, routing_direction: horizontal]
        d1:  [material: SiO2,      thickness: 600nm]
        m2:  [material: Copper,    thickness: 600nm, routing_direction: vertical]
        d2:  [material: SiO2,      thickness: 600nm]
        m3:  [material: Copper,    thickness: 800nm, routing_direction: horizontal]

    # 2. Dynamic Via Enclosure & Stacking Rules
    via:
        min_diameter: 180nm
        default_via_fill: Tungsten
        
        # Enclosure (overhang) constraints per layer
        enclosures: [
            m1: 30nm,
            m2: 40nm,
            m3: 50nm
        ]
        
        # Multi-layer stacking rules
        allow_stacked_vias: true
        min_stagger_offset: 200nm

    # 3. Silicon Track Grid Definitions
    routing:
        track_pitch: 360nm
        grid_snapping: true
```

---

## Section 7: Determinism & The `.routes.lock` Schema

To ensure that layouts remain 100% reproducible across different builds, machines, and operating systems, all routed paths are locked into the `.routes.lock` file. The compiler reads this file to reconstruct layout paths instantly without running the pathfinder.

### 7.1 JSON Lock Schema (Nanometer Resolution)
Every coordinate in the lock-file is expressed as an integer in absolute nanometers ($i64$):

```json
{
  "version": "0.1.7",
  "board": "ProcessorSpace",
  "grid": {
    "dimensions_nm": [100000000, 100000000, 2000000],
    "resolution_nm": [100, 100, 50]
  },
  "routes": [
    {
      "net": "VCC_CORE",
      "material": "Copper",
      "segments": [
        {
          "layer": "m1",
          "width_nm": 400,
          "path": [
            [1200500, 2500000],
            [1500000, 2500000]
          ]
        },
        {
          "layer": "m2",
          "width_nm": 400,
          "path": [
            [1500000, 2500000],
            [1500000, 4800000]
          ]
        }
      ],
      "vias": [
        {
          "type": "Via1_Plug",
          "position": [1500000, 2500000],
          "start_layer": "m1",
          "end_layer": "m2"
        }
      ]
    }
  ]
}
```

---

## Section 8: Summary of Compiler Routing Pipeline

The compiler executes the routing and validation process in a strict 5-stage pipeline, fully decoupled from hardcoded physical assumptions:

```
Pass 1: Parse Profile (profile.hw)
  - Lower stackup, enclosures, and routing_direction constraints into memory.
                             │
                             ▼
Pass 2: Parse Layout (design.hw)
  - Lower netlists, spaces, component placements, and manual paths.
                             │
                             ▼
Pass 3: Obstacle Blitting (Minkowski)
  - Extract component AABBs, apply Minkowski inflation based on profile clearance.
                             │
                             ▼
Pass 4: 2.5D/3D Pathfinding (Leap-Frog Router)
  - Run A* with dynamic preferred-direction penalties and pattern-warping.
  - Stamp single-layer via towers with profile-defined enclosures.
                             │
                             ▼
Pass 5: Realization & Polygon Unioning (Clipper)
  - Group traces by net and layer, weld overlapping paths in 2D, and extrude meshes.
  - Write verified paths to 'project.routes.lock'.
```

By enforcing this architecture, Hardware Script ensures that the compiler remains a lean, high-performance mathematical engine, leaving the physical constraints of fabrication completely in the hands of the declarative stackup profile.
