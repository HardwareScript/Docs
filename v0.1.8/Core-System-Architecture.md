# Architectural Specification: Hardware Script v0.1.8 Synthesis Engine

**Document Type:** Core System Architecture Reference  
**Status:** Feature Freeze (v0.1.8-alpha)  
**Focus:** Dedicated Specifications for the 22 Core Compilation, Routing, Verification, and Physical-Awareness Subsystems  

---

## Subsystem 1: Entity Graph (The Canonical Database)

### 1. Core Abstraction
The **Entity Graph** serves as the single, authoritative, and decentralized source of truth for the physical and logical design state. In older architectures, layouts were represented as raw voxel grids, which led to high memory overhead and quantization noise. In other systems, design states are stored as flat, indexed arrays where adding or deleting a single element causes index-shifting cascades that invalidate pointers throughout the compiler memory.

The Entity Graph resolves this by implementing a directed graph of decoupled design entities. Every physical component instance (stored as a FixedTransform2D referencing a `ComponentStamp`), pad, via, logical net, trace segment, and polygon is stored canonically as a node in the graph. Component geometry is never duplicated—each stamp is stored once in local coordinate space as an **Oriented Bounding Volume Hierarchy (OBVH)**, and instances reference it via lightweight transforms (see Subsystem 19: Zero-Stamping Scene Graph). Solder mask openings are modeled as first-class subtractive mask keepouts in the database, merged dynamically during Gerber generation—never linked to component pads. Same-net topological verification ensures no floating antennas or spurious inductive loops. 

### 2. Algorithmic Engine
Instead of using raw memory pointers or array indices, nodes are identified and linked using **Stable, Cryptographically Unique Identifiers**:
*   `ComponentId`: Unique hash of the component type and its placement statement.
*   `PinId`: Represents a physical connection point.
*   `NetId`: The logical net binding.
*   `RouteId`: Identifies a resolved, continuous path connecting pins.
*   `GeometryId`: Identifies the extruded, welded copper solid.

These IDs are generated using a stable hashing algorithm of the entity's relative properties and semantic dependencies:

$$\text{EntityId} = \text{hash}(\text{Type} + \text{Semantic Path} + \text{ParentId})$$

This prevents "index-shifting" when elements are added or deleted. The R\*-Tree index and the 3D meshes are completely volatile and can be reconstructed from this graph on-the-fly at any time.

```
       Entity Graph Structure (Topological Node Map)
       
              [ ComponentNode (R1) ]
                 │
                 ▼ (contains)
              [ PinNode (R1.A) ] 
                 │
                 ▼ (bound to)
              [ NetNode (VOUT) ] ◄────► [ RouteNode (Segment) ] ◄────► [ GeometryNode (Welded) ]
```

### 3. Data Flow & Pipeline Integration
*   **Input:** Receives parsed and lowered AST declarations during the semantic translation phase.
*   **Output:** Serves as the master registry for the A* pathfinder, the DRC verifier, and the mesh exporters.
*   **Incremental Update:** When a component is edited, only its direct node and its immediate children are modified or flagged as dirty. The rest of the graph remains untouched.

---

## Subsystem 2: Soft Corridor Planner

### 1. Core Abstraction
Traditional EDA routers use hard boundary walls to restrict detailed pathfinding within pre-allocated global corridors (G-cells). While this limits the search space, hard boundaries frequently cause routing deadlocks in dense silicon areas where multiple traces must cross, and force unnecessary layer-changing vias on PCBs when a local channel is temporarily blocked.

The **Soft Corridor Planner** resolves this by replacing rigid walls with dynamic **Cost Fields**. A corridor is not a hard physical keepout; it is a soft, preferred routing envelope.

```
                         Soft Corridor Cost Field
                         
       [ Outer Boundary ]  ──────────────────────────────── [Cost: +100]
         [ Preferred ]     - - - - - - - - - - - - - - - -  [Cost: +10]
           [ Center ]      ==============================  [Cost: 1]
```

### 2. Algorithmic Engine
The planner translates the coarse, global routing channels into Z-locked cost fields over the R\*-Tree. For any given coordinate step $(x, y, z)$ evaluated by the pathfinder:
*   A step along the exact center line of the planned corridor has a base cost of `1`.
*   A step deviating into the preferred envelope incurs a minor penalty of `+10`.
*   A step exiting the corridor into a non-allocated region incurs a moderate penalty of `+100`.

$$\text{Step Cost}(n) = \text{Base Cost} + \text{Congestion Penalty}(n) + \text{Corridor Penalty}(n)$$

If an unexpected physical barrier completely blocks the corridor, the detailed router is not blocked. The pathfinder simply routes around the barrier through a non-allocated region (paying the `+100` penalty) and snaps back into the corridor once the obstacle is passed.

### 3. Data Flow & Pipeline Integration
*   **Input:** Receives coarse, negotiated paths from the Partition Stage.
*   **Output:** Feeds dynamic step costs to the Topological Line-Search Router during detailed expansion.

---

## Subsystem 3: Partition Stage (Global Planning)

### 1. Core Abstraction
Routing a dense board with 100,000 nets using a flat, global pathfinder requires evaluating a massive search space, leading to extremely slow compile times.

The **Partition Stage** resolved this by implementing spatial decomposition. It divides the 3D layout area into a coarse 3D grid of G-cells (tiles) before detailed routing begins. By resolving net allocations on a global, coarse scale first, the compiler partitions a single, complex routing problem into thousands of localized detailed routing runs.

```
                    Coarse G-Cell Partition Grid
                    
                    ┌──────────┬──────────┬──────────┐
                    │  G-Cell  │  G-Cell  │  G-Cell  │
                    │  (1,1)   │  (1,2)   │  (1,3)   │
                    ├──────────┼──────────┼──────────┤
                    │  G-Cell  │  G-Cell  │  G-Cell  │
                    │  (2,1)   │  (2,2)   │  (2,3)   │
                    └──────────┴──────────┴──────────┘
```

### 2. Algorithmic Engine
The partitioner runs a coarse routing pass over the G-cell grid:
1.  **G-Cell Slicing:** It divides the board bounding box into uniform coarse tiles (e.g., $10\text{mm} \times 10\text{mm}$ regions).
2.  **Global Track Allocation:** It plans the paths of all nets across G-cell boundaries first, treating each G-cell interface as a virtual port.
3.  **Boundary Track Reservation Table:** The global router pre-negotiates and locks the coordinate points where each net crosses G-cell boundaries as **Immutable Interface Ports**. These ports are registered in a reservation table with minimum clearance envelopes ($C_{\text{clearance}}$). Detailed parallel routers are strictly prohibited from placing any obstacles (vias, other nets) within the clearance envelope of another G-cell's reserved interface ports.
4.  **Localized Boundary Port Relocation:** If a detailed router inside G-cell $G_i$ cannot physically reach a reserved interface port $P$ due to local congestion, it first attempts a **local boundary port relocation** within a constrained **Boundary Search Window** of $\pm 3 \cdot \text{track\_pitch}$ along the shared boundary between $G_i$ and the adjacent G-cell $G_j$. Because the port is shifted only along the shared boundary, only $G_i$ and $G_j$ are invalidated in Salsa's query cache—the rest of the board retains memoized results. Only if local relocation fails within the boundary window does the router yield and invoke the Negotiated Congestion Engine (Subsystem 4) for a full global renegotiation.
5.  **Rayon Parallelization:** Once global tracks are assigned and interface ports are locked, the compiler spawns independent detailed routers inside each G-cell concurrently using Rayon. Since the detailed routers are bounded by their local G-cell tiles and respect locked interface ports, they run in parallel without requiring global memory locks, achieving linear performance scaling.

### 3. Data Flow & Pipeline Integration
*   **Input:** Reads component bboxes and logical net connections from the Entity Graph.
*   **Output:** Generates local G-cell boundaries and passes them to the Soft Corridor Planner.

---

## Subsystem 4: Negotiated Congestion Engine (PathFinder-style)

### 1. Core Abstraction
Sequential routers pathfind one net at a time. The first net takes the shortest path, blocking subsequent nets and forcing them to take chaotic, winding detours, or leading to routing deadlocks.

The **Negotiated Congestion Engine** resolves this by routing all nets simultaneously. It allows nets to share the same physical space (overlap) during the first routing iteration, and then forces them to gradually spread out and negotiate alternative paths over subsequent iterations.

```
       Iterative Congestion Negotiation (PathFinder Loop)
       
       [ Iteration 1 ] ──► All nets route (Overlaps allowed)
                                │
                                v
       [ Iteration 2 ] ──► Inflate cost of shared zones (Nets detour)
                                │
                                v
       [ Iteration N ] ──► Costs spike to infinity (Conflict resolved)
```

### 2. Algorithmic Engine
The engine utilizes the negotiated congestion formula to calculate the cost of sharing a routing resource (voxel or grid track) $r$ during iteration $i$:

$$c(r) = (b(r) + h(r)) \times p(r)$$

Where:
*   $b(r)$ is the base cost of the resource.
*   $h(r)$ is the historical congestion of the resource, which accumulates over consecutive iterations where the resource was shared.
*   $p(r)$ is the present congestion penalty, proportional to the number of nets currently overlapping on the resource.

Over multiple iterations, the cost of shared resources spikes to infinity, forcing competing traces to find unique, non-overlapping paths.

### 3. Data Flow & Pipeline Integration
*   **Input:** Receives coarse G-cell routes from the Partition Stage.
*   **Output:** Allocates cleared, conflict-free global pathways to the Topological Line-Search Router.

---

## Subsystem 5: Route Segment Decomposition

### 1. Core Abstraction
In many layout compilers, an electrical net connecting multiple pins (such as a 16-pad power rail or a daisy-chained memory bus) is treated as a single, flat netlist object. This causes "identity collapse." The router tries to compile the entire multi-pin net as a single, un-decomposed lump, resulting in traces taking shortcuts, parallel "double traces" running on top of each other, and redundant connections.

The **Route Segment Decomposition** subsystem resolves this by breaking down a single logical net into a set of discrete, point-to-point routing jobs before pathfinding begins.

```
                     Route Segment Decomposition
                     
     [ Logical Net: ALL_PADS ] ────────► Decompose via MST ──┐
                                                             │
     ┌───────────────────┬───────────────────┬───────────────┘
     ▼                   ▼                   ▼
  Segment 1           Segment 2           Segment 3
  (J0[0] to J0[1])    (J0[1] to J0[2])    (J0[2] to J0[3])
```

### 2. Algorithmic Engine
The decomposition engine uses a **Minimum Spanning Tree (MST)** algorithm to break down a multi-pin net:
1.  **Pin Node Collection:** Extract the global coordinates of all pins bound to the active `NetId`.
2.  **Distance Matrix:** Construct a complete graph where edges represent the Euclidean distance between pin pairs.
3.  **Prim's / Kruskal's MST:** Solve for the Minimum Spanning Tree to identify the shortest, most efficient set of point-to-point connections that connect all pins without forming closed loops.
4.  **Virtual Junction Node Insertion:** When a trace taps into another trace (forming a T-junction), the compiler inserts a **Virtual Junction Node**—a three-port entity in the EntityGraph with a stable `JunctionId`. This node is registered as a valid same-net overlap point for the Topological Verifier and carries parasitic metadata ($C_{\text{junc}}$, $L_{\text{junc}}$) extracted by the Sakurai BEM Parasitic Extractor (Subsystem 21).
5.  **Route Job Registration:** Each edge in the resolved MST is registered as an independent `RouteSegment` with its own stable `RouteId`.

### 3. Data Flow & Pipeline Integration
*   **Input:** Reads the logical netlist and component pin positions from the Entity Graph.
*   **Output:** Generates a flat vector of pairwise `RouteSegment` jobs and feeds them to the Topological Line-Search Router.

---

## Subsystem 6: Topological Line-Search Router

### 1. Core Abstraction
Standard grid-crawling A* pathfinders must evaluate space cell-by-cell. At advanced silicon scales, this requires massive memory arrays and causes significant "quantization noise" (jagged traces and vertical gaps).

The **Topological Line-Search Router** resolves this by operating entirely on continuous coordinates using ray-casting. It ignores grids completely during the search phase, calculating mathematically straight, smooth trace segments.

```
              Topological Ray-Casting Pathfinder
              
                     North Ray
                         ▲
                         │
     West Ray ◄──────────┼──────────► East Ray
                         │
                         ▼
                     South Ray
```

### 2. Algorithmic Engine
The line-search router is based on the Mikami-Tabuchi and Hightower algorithms, adapted for 2.5D layered designs. It uses the **Axis-Aligned Slab Method** to minimize intersection tests:
1.  **Ray Projection:** From the start port and target port, the router projects orthogonal search rays (horizontal and vertical lines) along continuous coordinates.
2.  **Slab Intersection:** For each obstacle AABB, the ray direction defines two parallel slabs (X and Y axes). The ray enters the obstacle only if $t_{\text{enter}} < t_{\text{exit}}$ for both slabs simultaneously. This is a single $O(\log N)$ query per obstacle over the flat-packed `geo-index`—no iterative stepping required.
3.  **Obstacle Intersection:** When the Slab Method confirms an intersection within the current G-cell, the router computes the exact entry point and switches to fine-grained checks against the `geo-index`.
4.  **Orthogonal Bending:** If a ray hits an obstacle, the router steps back slightly and projects perpendicular rays (making clean $90^\circ$ or $135^\circ$ bends) to navigate around it.
5.  **Diagonal Grid-Snapping:** When routing at 45° (Octilinear), the router enforces that any diagonal segment has its length $L$ dynamically rounded so endpoints land exactly on orthogonal routing grid track intersections:
    $$L_{\text{snapped}} = \text{round}\left(\frac{N \cdot \text{track\_pitch}}{\sin(45°)}\right)$$
    This guarantees that when the diagonal segment terminates and the router returns to orthogonal routing (90°), the trace is perfectly snapped to the primary manufacturing tracks, preventing grid-drift propagation and cascading DRC violations.
6.  **Path Resolution:** When a ray from the start and a ray from the target intersect in open space, a path is found. 

### 3. Data Flow & Pipeline Integration
*   **Input:** Receives starting and ending port coordinates from the Route Segment Decomposition subsystem.
*   **Output:** Generates clean, straight coordinate vectors and hands them to the Localized Legalization Engine.

---

## Subsystem 7: Localized Legalization Engine (QP Window Solver)

### 1. Core Abstraction
When traces or pads violate clearance rules, traditional layout tools rely on slow "rip-up-and-reroute" loops that tear down paths and start over. 

The **Localized Legalization Engine** resolves this using continuous convex optimization. It represents trace clearances as mathematical inequalities and smoothly "nudges" adjacent trace vectors out of the way, avoiding costly complete reroutes.

```
          Localized Legalization Window (Bounded Solver)
          
                 ┌──────────────────────────┐
                 │  Legalization Window     │
                 │  (Bbox of collision)     │
                 │    ┌──────┐              │
                 │    │ Nudge│◄─── Vector   │
                 │    └──────┘    moved     │
                 │                locally   │
                 └──────────────────────────┘
```

### 2. Algorithmic Engine
To prevent a small edit from triggering a global re-routing avalanche across the entire design, the engine implements **Localized Legalization Windows** with a two-tier solver strategy:
1.  **Window Bounding:** When a collision is detected, the engine defines a small bounding box (the window) strictly around the affected area.
2.  **Convex Formulation:** All trace vectors inside this window are converted into Quadratic Programming (QP) variables. All surrounding geometry outside the window is locked as a hard obstacle.
3.  **Hybrid Solving:**
    *   **For macro-scale constraints** (large windows with many variables, e.g., block floorplanning), the engine uses the `clarabel` interior-point solver:
        $$\min: \alpha(\text{displacement})^2 + \beta(\text{via\_count}) + \gamma(\text{length})^2 + \delta(\text{crosstalk})^2$$
    *   **For local micro-adjustments** (small windows with $N < 20$ variables, e.g., nudging 2–3 parallel traces when a new via is dropped), the engine uses a lightweight **active-set solver** (like PIQP) or a dedicated **1D DAG graph compaction solver**. These solve in microseconds with zero heap allocations and no $O(N^3)$ matrix factorization overhead, preventing the pathfinder from hitting a numerical solver bottleneck during detailed route adjustments.
4.  **Nudge Application:** Traces are smoothly shifted sideways within the window boundaries, maintaining absolute layout integrity.

### 3. Data Flow & Pipeline Integration
*   **Input:** Receives raw, colliding coordinate vectors from the Topological Line-Search Router.
*   **Output:** Generates legalized, non-overlapping vector coordinates and passes them to the Compaction Engine.

---

## Subsystem 8: Constraint-Aware Compaction Engine

### 1. Core Abstraction
Simple geometric compaction algorithms slide parallel traces together to minimize layout area. However, compressing traces blindly without evaluating electrical and physical constraints degrades high-speed signal integrity.

The **Constraint-Aware Compaction Engine** resolves this by analyzing signal and fabrication rules during the compaction pass, ensuring layout density never compromises performance.

```
                 Constraint-Aware Compaction
                 
       Trace A ───────────────────────────────►
                               ▲
                               │ Squeezed safely up to 
                               │ minimum clearance limit
                               ▼
       Trace B ───────────────────────────────►
```

### 2. Algorithmic Engine
The compaction engine operates on the legalized vector paths:
1.  **Impedance-Constraint Evaluation:** It queries the profile to find the characteristic impedance targets (e.g., $50\Omega$ single-ended or $100\Omega$ differential pairs) of the active traces.
2.  **Crosstalk Spacing Check:** It calculates the maximum parallel run length of the adjacent traces and determines the minimum spacing required to prevent crosstalk.
3.  **The Squeeze:** It slides the traces closer together, but caps the movement strictly at the point where signal integrity rules are satisfied, maximizing routing density without violating clearances.

### 3. Data Flow & Pipeline Integration
*   **Input:** Reads optimized coordinates from the Localized Legalization Engine.
*   **Output:** Feeds compacted, performance-verified vector coordinates to the Verify Stage.

---

## Subsystem 9: First-Class Verify Stage (DRC Gate + Unified Sweep + Sakurai BEM Extraction)

### 1. Core Abstraction
In typical CAD layouts, physical verification (DRC and LVS) is treated as a separate, manual post-processing step. Designers often export their files to external simulators, find a violation, and manually patch the layout, creating a slow development loop.

The **First-Class Verify Stage** integrates verification directly into the compiler pipeline as a mandatory gate. It combines the **G-Cell-Local Unified Sweep Verification** (Subsystem 22)—which executes both different-net clearance checks and same-net topological checks in a single $O(N \log N + K)$ memory pass—with **Sakurai's empirical microstrip parasitic extraction** for ground-plane-aware capacitance and resistance. If any design rule or physical continuity check fails, compilation halts immediately, blocking mesh extrusion and file export. Crucially, **LVS, DRC, and BEM all run on raw, distinct vector segments**—`clipper2-rust` welding and `earcut` triangulation occur strictly at the final export boundary.

```
                   First-Class Verification Gate

  Topological Route ──► [ G-Cell-Local Unified Sweep (Subst. 22) ] ──► [ Sakurai BEM Extract ] ──► Extrude & Export
                                    │                                              │
                                    ▼ (If failed)                                  ▼ (Flag warnings)
                               Halt Build                                           Embed R/C in SPICE
```

### 2. Algorithmic Engine
The verification stage is divided into four discrete, highly optimized verification systems:
*   **Connectivity Verification:** Runs graph reachability analysis on the Entity Graph to prove all pins are connected and no nets are broken or shorted.
*   **Unified G-Cell Sweep Verification (Subsystem 22):** A single sweep-line pass within each G-cell handles both:
    *   **Different-Net DRC:** SIMD-accelerated clearance checks between segments with different `NetId` values.
    *   **Same-Net Topological Verification:** Overlaps between segments with the same `NetId` are asserted to land on a registered `VirtualJunctionNode` or within a component port bounding box. Unintentional same-net overlaps in open space are flagged as spurious loops or dead antennas.
*   **Sakurai BEM Parasitic Extraction:** Uses Wheeler's effective permittivity and Sakurai's empirical microstrip formulas to compute ground-plane-aware parasitic coupling capacitance ($C_{12}$), ground capacitance ($C_{1g}$), series resistance ($R_s$), and via self-inductance ($L_{\text{via}}$). Parasitic values are embedded directly into the SPICE netlist export.
*   **Current-Density & Thermal-Rise Verification (AC-Aware, G-Cell-Thermal-Coupled):** Validates minimum trace cross-sectional area against physical safety limits using **G-cell-local thermal maps** and **separate AC current declarations** ($I_{\text{RMS}}$ and $I_{\text{peak}}$):
    *   **Silicon (Electromigration Limit):** $A_{\text{min}} = I_{\text{peak}} / J_{\text{limit}}$ where $I_{\text{peak}}$ is the peak current from the pin's `current_limit: [peak: ...]` declaration and $J_{\text{limit}}$ is the electromigration threshold from the stackup profile (e.g., $10\text{ mA}/\mu\text{m}^2$).
    *   **PCBs (IPC-2152 Temperature Rise Limit):** $A_{\text{min}} = (I_{\text{RMS}} / (k \cdot \Delta T_{\text{local}}^b))^{1/c}$ where $I_{\text{RMS}}$ is the RMS current from the pin's `current_limit: [rms: ...]` declaration, $\Delta T_{\text{local}} = T_{\text{max\_allowed}} - T_{\text{local}}$ is the local temperature headroom from the G-cell thermal map, and $k$, $b$, $c$ are IPC-2152 constants.
    *   Current requirements are read from module pin declarations: `current_limit: [rms: 10mA, peak: 50mA]`. For DC-only pins, a single `current_limit: 20mA` value is interpreted as both $I_{\text{RMS}}$ and $I_{\text{peak}}$. Traces passing through thermal hotspots are automatically scaled wider. Violations are flagged as P21 (EM) or P22 (thermal) and halt the build.
*   **Manufacturing Verification:** Validates that via drill aspect ratios and stackup lamination limits match the fabrication rules. Technology-specific via constraints (stacked vs. staggered) are enforced based on the profile's `technology` tag.

### 3. Data Flow & Pipeline Integration
*   **Input:** Receives compacted, exact vector coordinates from the Compaction Engine.
*   **Output:** If successful, grants permission to the Geometry Refinement Engine to extrude and export the design.

---

## Subsystem 10: Semantic Lockfile System

### 1. Core Abstraction
Re-running routing and physical synthesis on every compilation run introduces significant compile-time latency. 

The **Semantic Lockfile System** resolves this by implementing a highly optimized, deterministic caching mechanism with a **single-source binary storage strategy**. It records resolved routing paths in `project.routes.lock` in the binary `rkyv` format only, allowing the compiler to bypass the pathfinding engine completely on subsequent builds. No secondary JSON file is generated during builds—if a human needs to inspect or audit the lockfile, a dedicated CLI tool (`hwc lock inspect project.routes.lock`) decodes the binary to stdout on demand, keeping the workspace and version control history clean.

```
                    Semantic Lockfile Invalidation
                    
          placement_hash = hash(components + rules + stackup)
          
          Current Hash == Stored Hash?
          ├── Yes ──► Load rkyv binary (zero-copy mmap, bypass A*)
          └── No  ──► Discard lock, execute pathfinder
```

### 2. Algorithmic Engine
On every build, the compiler calculates a comprehensive **Semantic Fingerprint** of the design space:

$$\text{Fingerprint} = \text{hash}(\text{Component Bounds} + \text{Rules} + \text{Stackup Layers} + \text{Router Version})$$

*   If this calculated fingerprint matches the `placement_hash` stored in the lockfile, the compiler knows the routing environment is identical.
    *   The compiler memory-maps the `project.routes.lock` file via `memmap2` and casts the raw bytes directly into Rust structs using `rkyv`'s zero-copy deserialization in microseconds with zero parsing overhead and zero heap allocations.
*   If any physical parameter (moving a component, altering stackup thickness, or updating the compiler) changes, the hashes mismatch. The compiler invalidates the lock and re-runs pathfinding.

### 3. Data Flow & Pipeline Integration
*   **Input:** Reads the existing `project.routes.lock` file on startup via `rkyv` + `memmap2` zero-copy.
*   **Output:** On a successful build, writes the updated routing data in the binary `rkyv` format. Human inspection available on demand via `hwc lock inspect` CLI tool.

---

## Subsystem 11: Incremental Dependency DAG

### 1. Core Abstraction
When a single component is shifted in a large layout, traditional layout tools often invalidate the entire design and re-route the entire board, which is computationally expensive.

The **Incremental Dependency DAG (Directed Acyclic Graph)** resolves this by tracking the exact dependencies between components, pins, nets, and physical traces, allowing the compiler to invalidate and re-route *only* the affected local regions.

```
                     Incremental Dependency DAG
                     
       Component ──► Pin ──► Net ──► RoutePlan ──► Route ──► Geometry
```

### 2. Algorithmic Engine
The dependency graph is managed as a directed acyclic graph in memory:
1.  **Dependency Mapping:** Every route segment registers its dependencies on the components and pins it connects to, and the spatial corridors it intersects.
2.  **Granular Invalidation:** When a component is moved:
    *   The compiler traverses the DAG upward from the modified component node.
    *   It identifies only the directly connected nets and any intersecting G-cell corridors.
    *   These specific nodes are marked as dirty and cleared from the cache.
3.  **Incremental Re-Route:** The rest of the board's routing paths remain locked. The router only runs pathfinding on the few dirty, unlocked nets, achieving fast incremental compilation.

### 3. Data Flow & Pipeline Integration
*   **Input:** Receives layout modification events from the parser or IDE viewport.
*   **Output:** Directs the AutoRouter on which specific nets must be re-routed.

---

## Subsystem 12: Vector Route Persistence

### 1. Core Abstraction
Storing raw 3D meshes or dense, uncompressed coordinate arrays in a cache file is inefficient, resulting in large file sizes and slow load times.

The **Vector Route Persistence** subsystem resolves this by strictly storing only the continuous 3D bend-points and relative direction vectors of the traces using the Case-Boundary Base-36 RLC format.

```
       Verbose JSON (v0.1.7):  "[[2500000, 10500000, 0], [5000000, 10500000, 0]]"
                                                     │
                                                     ▼ (AVS Compression)
       Vector AVS (v0.1.8):    "RkU46Dk"  (7 bytes total)
```

### 2. Algorithmic Engine
The subsystem utilizes the **Case-Boundary Base-36 Run-Length Compression (RLC)** format to serialize the vectors:
*   **Symmetrical Winding:** Traces are converted to direction-magnitude vectors.
*   **Case Separation:** Direction commands are strictly uppercase (`R`, `L`, `U`, `D`). Magnitude values are strictly lowercase alphanumeric (`0-9`, `a-z`) [HWS-SYNTAX-SPEC-V018.md].
*   **Zero-Delimiters:** Because the characters do not overlap, no spaces or commas are written. The parser splits commands from values on-the-fly, compressing a 50-line list of absolute coordinates down to a single, tiny, 10-character string (e.g., `"RkU46Dk"`).

### 3. Data Flow & Pipeline Integration
*   **Input:** Receives finalized vector coordinates from the Compaction and Verify stages.
*   **Output:** Serializes the compressed AVS strings directly into the `arcs` array of the `project.routes.lock` file.

---

## Subsystem 13: Geometry Refinement Engine

### 1. Core Abstraction
Exporting overlapping, raw 3D geometric shapes directly to CAD tools or simulators causes major defects. Coplanar surfaces share the same Z-coordinate, resulting in visual Z-fighting (flickering), and intersecting volumes confuse electromagnetic and thermal solvers.

The **Geometry Refinement Engine** resolves this by performing 2D polygon clipping and boundary canonicalization on the raw routing vectors before mesh generation.

```
   Overlapping Polygons (Raw) ──► [ Clipper2 (Union) ] ──► Unified Manifold Mesh
```

### 2. Algorithmic Engine
The refinement engine operates on the validated routing vectors:
1.  **2D Clipper Union:** It groups all copper traces, pours, and via landing pads belonging to the same Net ID and the same Z-layer. It welds them in 2D using `clipper2-rust` under the **Non-Zero Winding Rule** to dissolve all overlapping boundaries.
2.  **Boundary Canonicalization:** To remove jagged joints and simplify the geometry, the engine runs a canonicalization pass:
    *   It merges collinear edges (if points $A$, $B$, and $C$ lie on a straight line, it discards $B$).
    *   It removes microscopic self-intersecting slivers.
    *   It normalizes the polygon winding orders (outer contours wound CCW, inner hole contours wound CW).
3.  **Tessellation:** The clean, canonical 2D contours are passed to `earcut` (a zero-allocation, flat-array ear-clipping triangulator) to generate watertight 3D solid meshes. Because `clipper2-rust` already outputs perfectly clean, non-self-intersecting contours, `earcut` performs 3×–5× faster than heavier tessellators on this clean input.

### 3. Data Flow & Pipeline Integration
*   **Input:** Reads validated, compacted vector coordinates from the Verify Stage.
*   **Output:** Generates clean, non-overlapping 3D solid meshes for the GLB, DXF, and GDSII exporters.

---

## Subsystem 14: Export Isolation Layer

### 1. Core Abstraction
In many CAD tools, the database contains rendering-specific assets (like triangles, normals, and textures) mixed with the physical layout data. This bloats the database, slows down physical verification, and can introduce numerical rounding errors during compilation.

The **Export Isolation Layer** resolves this by completely separating the rendering and export representations from the compiler's internal logical database. Crucially, **LVS validation, DRC checking, and parasitic extraction all run on raw, distinct vector segments** to preserve geometric identity—`clipper2-rust` welding and `earcut` triangulation occur strictly at the final export boundary (GLB, DXF, GDSII).

```
    Compiler Database (Entity Graph)  ◄─── completely isolated ───►  Export Layer
    (Pristine i64 vectors, NO mesh data)    (DRC/LVS/BEM run here) (GLB / DXF / SPICE)
```

### 2. Algorithmic Engine
The isolation layer maintains a strict boundary:
*   The compiler’s Entity Graph stores **only pristine $i64$ continuous vector coordinates and logical relationships**. It contains absolutely no mesh data, triangles, or rendering assets.
*   Triangulation (using `earcut`) and face normal generation are treated strictly as **final export-layer concerns**.
*   This geometry generation is executed on-the-fly only when writing the GLB mesh or launching the viewport. Once exported, the mesh data is immediately discarded from memory, preventing database bloat.

### 3. Data Flow & Pipeline Integration
*   **Input:** Reads the refined, canonical 2D contours from the Geometry Refinement Engine.
*   **Output:** Stream-serializes the data into standard manufacturing and exchange formats: `.glb` (3D visual), `.dxf` (2D layout), `.sp` (electrical netlist), and `.csv` (BOM).

---

## Subsystem 15: Acceleration Index Layer (Hybrid R*-Tree)

### 1. Core Abstraction
While storing design state as a directed Entity Graph is excellent for structure and caching, searching a graph sequentially to perform spatial proximity queries (such as checking trace clearances or detecting component collisions) is too slow for large-scale layouts.

The **Acceleration Index Layer** resolves this by implementing a volatile, high-speed spatial lookup index that is completely decoupled from the master data. It uses a **hybrid indexing model** to balance dynamic updates against query throughput.

```
    Entity Graph (Master DB)  ──► Build on-the-fly ──►  Hybrid Spatial Index
   ( top_copper: [Trace, Pad] )
                                                        Dynamic: rstar (macro floorplanning)
                                                        Static:  geo-index (per-layer obstacles)
```

### 2. Algorithmic Engine
The acceleration layer uses two complementary index types:
1.  **Dynamic Index (`rstar`):** Used for top-level macro-placement of components during floorplanning, where dynamic insertion and movement occur frequently. The R\*-tree is constructed on-the-fly and discarded after use, providing $O(\log N)$ queries for dynamic workloads.
2.  **Static Index (`geo-index`):** After the partition stage allocates soft corridors, the compiler compiles static obstacle geometry (component boundaries, keepouts, and committed traces) per layer into flat, packed `geo-index` structures. These yield 5×–10× faster query performance on static obstacles due to contiguous memory layout, near-zero allocation overhead, and excellent L1/L2 cache locality. `geo-index` is ABI-stable and memory-map (`memmap2`) ready for persistent caching.
3.  **Logarithmic Queries:** Spatial queries (such as finding all copper objects within a $200\text{nm}$ radius of a trace coordinate during ray marching) are executed with **$O(\log N)$ logarithmic complexity** instead of flat $O(N)$ sequential scans.
4.  **Zero Memory Bloat:** Both indices are kept in volatile RAM during the active compilation run and are completely discarded once the build completes, avoiding database size overhead on disk.

### 3. Data Flow & Pipeline Integration
*   **Input:** Loads vector coordinate bounding boxes from the Entity Graph on compilation startup.
*   **Output:** Provides fast spatial proximity and collision queries to the AutoRouter (via `rstar` for dynamic queries, `geo-index` for static obstacle queries) and the Verify (DRC) Stage.

---

## Subsystem 16: Incremental DRC (Planned)

### 1. Core Abstraction
Running a complete design rule check (DRC) across an entire board with 100,000 components on every minor change is computationally expensive.

The **Incremental DRC** subsystem resolves this by checking design rules strictly inside the spatial bounding boxes of modified regions, bypassing the rest of the board.

```
          Incremental DRC (Local Validation Window)
          
                 ┌──────────────────────────┐
                 │  DRC Validation Window   │
                 │  (Bbox of moved component)
                 │    ┌──────┐              │
                 │    │Check │              │
                 │    └──────┘              │
                 │                          │
                 └──────────────────────────┘
```

### 2. Algorithmic Engine
The incremental DRC engine is tightly integrated with the lockfile's dependency DAG:
1.  **Local Windowing:** When a component or trace is modified, the compiler defines a localized bounding box around the edit.
2.  **Targeted Queries:** The DRC engine uses the R\*-Tree index to query only the physical geometries that intersect this local bounding box.
3.  **Local Rule Validation:** It runs clearance, width, and keepout verification strictly on these retrieved local geometries. Since unchanged regions outside the window are guaranteed to be valid from the previous cached compile, their validation is skipped, reducing DRC verification times by over 90%.

### 3. Data Flow & Pipeline Integration
*   **Input:** Receives local invalidation windows from the Incremental Dependency DAG.
*   **Output:** Passes local verification reports to the First-Class Verify Stage.

---

## Subsystem 17: Multi-Net Routing Manager

### 1. Core Abstraction
If the routing planner treats the entire board as one giant connected netlist, traces belonging to different nets can easily overlap, and the router will struggle to maintain identity separation, leading to short circuits.

The **Multi-Net Routing Manager** resolves this by keeping the routing identities and parameters of separate nets strictly isolated during spatial planning.

```
                     Multi-Net Routing Manager
                     
     [ Net A (signal) ]  ─────► Isolated Channel A ──► Pathfinder
     [ Net B (power)  ]  ─────► Isolated Channel B ──► Pathfinder
```

### 2. Algorithmic Engine
The multi-net manager handles the routing state of all nets in the design:
1.  **Net Isolation:** It extracts the logical nets and ensures each net is allocated its own distinct set of G-cells and soft corridor cost fields during the Partition stage.
2.  **Constraint Propagation:** It retrieves specific physical rules (such as minimum trace width, preferred layers, and target impedances) from the profile and attaches them as metadata directly to each net's route segments.
3.  **Same-Net Collision Bypass:** During A* pathfinding, the manager allows traces from the same net to share the same starting/ending pin bboxes, while enforcing strict clearance penalties against all different-net copper obstacles.

### 3. Data Flow & Pipeline Integration
*   **Input:** Reads logical net connections from the Entity Graph.
*   **Output:** Directs the detailed Topological Line-Search Router on which nets must be routed and what constraints apply to them.

---

## Subsystem 18: Deterministic Build Pipeline

### 1. Core Abstraction
In many CAD and layout compilers, minor variations in multi-threaded execution order, float-to-int rounding differences, or uninitialized memory leaks can cause subsequent compilations of the exact same source code to produce slightly different binary outputs. This makes version control and lockfile caching unstable.

The **Deterministic Build Pipeline** guarantees that compiling identical input always produces bit-identical output files across different runs and systems.

```
       Input File (.hw)
              │
              ▼
     [ Topological Sort ] ────► Establishes stable placement order
              │
              ▼
   [ Deterministic Router ] ──► Resolves pathfinding with static cost ties
              │
              ▼
     [ Byte-Identical Output ] (GDSII, DXF, GLB, SPICE)
```

### 2. Step-by-Step Implementation
1.  **Deterministic Topological Sorting:** Before placing any components or routing any traces, the compiler performs a topological sort on the dependency graph, ensuring that elements are always compiled in the exact same order regardless of their textual position in the source file.
2.  **Fixed-Point Coordinate Transforms (i128 Intermediate Promotion):** All coordinate transformations (rotation, translation) use pure 128-bit integer arithmetic for intermediate products via `FixedTransform2D` (scaled by $10^9$) instead of floating-point `glam::DAffine3`. On a standard 200 mm PCB, coordinate values reach $2 \times 10^{11}\text{ pm}$. When multiplied by a trigonometric scale factor ($\cos \approx 707{,}106{,}781$), the intermediate product reaches $\approx 1.414 \times 10^{20}$, exceeding i64's maximum of $\approx 9.22 \times 10^{18}$. All intermediate multiplications are therefore promoted to **i128** (max $\approx 3.40 \times 10^{38}$) before division, casting back to i64 only after the scaling factor is removed. This eliminates representation drift, rounding jitter, and overflow panics, guaranteeing identical picometer coordinates across AMD x86_64 and Apple Silicon ARM64.
3.  **Stable Hash Map Iteration:** The compiler uses the `indexmap` or `rustc_hash` crates with deterministic seed initializers instead of standard `HashMap` structures, ensuring that iterating over collections always yields elements in the exact same sequence.
4.  **Tie-Breaking Pathfinder Heuristics:** When the A* line-search pathfinder encounters multiple neighbor nodes with identical movement costs, it uses a strict, deterministic tie-breaking heuristic (e.g., always preferring horizontal steps before vertical steps, or sorting by coordinate values) instead of relying on arbitrary execution paths.
5.  **Bit-Identical Serialization:** All export files (including GLB meshes and DXF geometries) are sorted by coordinate and Net ID before writing, ensuring that their output bytes match identically on every run.

---

## Subsystem 19: Zero-Stamping Scene Graph (Analytic Component Stamps)

### 1. Core Abstraction
The legacy "Bit-Blit Stamping Engine" (from v0.1.6/v0.1.7) rasterized component geometries and physically copied (stamped) millions of standard-cell polygons into a 3D voxel grid, causing $O(\text{Atoms})$ memory scaling and a massive compilation bottleneck [BIT-BLIT-UNROLLER-IMPLEMENTATION.md].

The **Zero-Stamping Scene Graph** resolves this by eliminating all voxel writes during unrolling. Instead, component types are parsed once and stored as analytical vector definitions with an **Oriented Bounding Volume Hierarchy (OBVH)** in local coordinate space.

```
  ComponentStamp (Local Space)          ComponentInstance (World Space)
  ┌──────────────────────┐             ┌──────────────────────────────┐
  │ OBVH at origin       │             │ FixedTransform2D T           │
  │ [0, 0, 0]            │  ──────►    │ Global pre-transformed BVH   │
  │ Local OBB            │  instance   │ Net bindings: [NET_A, NET_B] │
  └──────────────────────┘             └──────────────────────────────┘
```

### 2. Algorithmic Engine
1.  **Stamp Parsing:** Each component definition is parsed exactly once into a `ComponentStamp`—an **Oriented Bounding Volume Hierarchy (OBVH)** of oriented bounding boxes (OBBs) for rotated components and AABBs for Manhattan shapes, plus port coordinates in local space (origin `[0, 0, 0]`).
2.  **Instance Registration with Global Pre-Transform:** Each placed component instance is stored in the Entity Graph as a lightweight `ComponentInstance` containing:
    *   A `stamp_id` reference to the shared `ComponentStamp`.
    *   A **FixedTransform2D** matrix $T$ (pure i128 intermediate arithmetic, scaled by $10^9$, no floating-point in core path).
    *   Logical net bindings.
    *   **Pre-transformed global bounding boxes** (`global_bbox`, `global_obb_children`, `global_aabb_children`)—the local stamp's bounding volumes are transformed **forward** into global world-coordinate space once at placement time. This eliminates lossy on-demand inverse transforms during pathfinding and DRC hot-paths. Since placements are memoized via Salsa (Subsystem 20), this forward transform cost is paid only once per placement edit.
3.  **Global-Space Collision Query:** All detailed routing, collision checks, and sweep-line DRC checks execute strictly in global coordinates using the pre-calculated global bounding boxes:
    *   Query the global bounding-box index (`geo-index` or `rstar`) to find intersected component bounding boxes.
    *   Execute **SAT-based OBB collision tests** against the pre-transformed `global_obb_children` for rotated components—still $O(1)$ per test, no integer truncation asymmetry. For Manhattan shapes, execute fast AABB box-bounds checks against `global_aabb_children`. Only components explicitly declared with non-standard custom shapes fall back to Jordan curve theorem polygon checks.
4.  **Memory Efficiency:** A 100M-transistor design stores only ~500 unique `ComponentStamp` templates (a few KB each) plus ~100K lightweight `ComponentInstance` structs with pre-transformed global bounds, reducing memory from $1.6\text{ GB}$ (voxel grid) to under $80\text{ MB}$.

### 3. Data Flow & Pipeline Integration
*   **Input:** Receives parsed component definitions from the AST during semantic translation.
*   **Output:** Provides on-demand collision queries to the Topological Line-Search Router and the DRC Verify Stage via the hybrid spatial index.

---

## Subsystem 20: Salsa-Style Memoized Query Engine

### 1. Core Abstraction
Traditional compiler pipelines execute in a strict, linear pass order—every change triggers a full re-evaluation of all downstream phases, even when most sub-graphs haven't changed. This introduces significant compile-time latency during active development.

The **Salsa-Style Memoized Query Engine** resolves this by modeling every compiler phase as a pure, demand-driven query function. When an entity is modified, the dependency DAG traces upward, invalidating only the specific nodes in the query cache. Critically, the AST is never fed as a single monolithic input—instead, it is lowered into a map of independent, hashed entity inputs to prevent AST-wide invalidation cascades.

```
  Demand-Driven Query Evaluation (Fine-Grained Inputs + Localized Port Relocation)
  
  component_input(file, comp_42) ──► place_component(file, comp_42)
                                          │
  route_input(file, route_42) ──► route_gcell(file, 42) ◄── partition_gcells(file)
                                          │
                                     (port blocked?)
                                     ├── Yes: shift port ±3·track_pitch along boundary
                                     │        invalidates only G-cell 42 + neighbor
                                     └── No:  proceed normally
  cache hit: 0ms for all other cells
```

### 2. Algorithmic Engine
1.  **Fine-Grained Query Inputs:** The compiler does not feed the entire raw AST as a single input to Salsa. Instead, the AST is lowered into independent, hashed entity inputs:
    *   `component_placement_input(file_id, component_id)` → `Arc<ComponentPlacement>`
    *   `route_statement_input(file_id, route_id)` → `Arc<RouteStatement>`
    *   `stackup_input(file_id)` → `Arc<StackupProfile>`
2.  **Query Functions:** Each compiler phase wraps these granular inputs in `#[salsa::query]` functions:
    *   `place_component(file_id, component_id)` → places a single component (memoized per component).
    *   `route_gcell(file_id, gcell_id)` → routes a single G-cell (memoized per cell, depends on affected `route_statement_input` nodes only).
    *   `verify_gcell(file_id, gcell_id)` → DRC checks a single G-cell (memoized per cell).
3.  **Incremental Invalidation:** When a developer modifies route Net_A:
    *   Only `route_statement_input(file_id, route_a)` is invalidated.
    *   Only `route_gcell(file_id, gcell_id)` and `verify_gcell(file_id, gcell_id)` for the affected G-cell are re-evaluated.
    *   When boundary port relocation occurs (shifting port along shared boundary), only the two adjacent G-cells sharing that boundary are invalidated—memoized results for all other G-cells remain cached.
    *   All other component placements, routes, and verification queries remain completely untouched in the cache, ensuring true sub-millisecond incremental compile times.
    *   Cyclic dependencies are mathematically impossible because queries are pure functions.
3.  **Integration with Lockfile:** If a G-cell's input boundaries and internal components haven't changed, its routing mesh is loaded instantly from the memory-mapped `rkyv` binary cache, bypassing the pathfinder entirely.

### 3. Data Flow & Pipeline Integration
*   **Input:** Receives source code and layout modification events from the parser or IDE viewport.
*   **Output:** Provides memoized, on-demand results for every compiler phase, enabling $<10\text{ ms}$ incremental compilation for minor edits.

---

## Subsystem 21: Analytic 2.5D BEM Parasitic Extraction (Sakurai + Wheeler + Via Inductance + Mutual Inductance)

### 1. Core Abstraction
Capturing "Physical Truth" (parasitics, thermal, signal integrity) requires accurate parasitic extraction. However, naive flat lookup tables are highly inaccurate for sub-micron silicon, while full 3D Finite Element Method (FEM) field solvers take hours to solve. Using raw substrate permittivity ($\epsilon_r$) without accounting for air/gas above the board overestimates coupling capacitance. Additionally, vertical via transitions introduce critical localized self-inductance at high frequencies. For high-speed lines (DDR5, PCIe Gen 6), capacitive coupling is only half of the crosstalk problem—mutual inductance ($M_{12}$) between parallel runs induces magnetic coupling that creates inductive crosstalk.

The **Analytic 2.5D BEM Parasitic Extractor** resolves this by computing:
1.  **Effective Relative Permittivity** ($\epsilon_{\text{eff}}$) using Wheeler's closed-form equation before executing Sakurai's formulas
2.  **Ground-plane-aware coupling capacitance** ($C_{12}$) and **ground capacitance** ($C_{1g}$) using Sakurai's empirical microstrip formulas with $\epsilon_{\text{eff}}$
3.  **Via self-inductance** ($L_{\text{via}}$) for vertical transitions using the analytical cylinder inductance formula
4.  **Mutual inductance** ($M_{12}$) between parallel trace runs using the Greenhouse approximation for full RLC/transmission line simulation fidelity

```
  Trace A (Conductor) ────────┐
                               │ Wheeler ε_eff + Sakurai Formulas
                               ▼
  Ground Plane (H below) ─────┘
  
  ε_eff = (εᵣ+1)/2 + (εᵣ-1)/2 · (1 + 12H/W)^(-1/2)
  C₁₂ = ε₀ε_effL [0.03(W/H) + 0.08(T/H) + 0.07(W/H)^0.25(T/H)^0.5(H/D)^1.34]
  L_via = 2·10⁻⁷ · h · [ln(4h/d) + 1]
  M₁₂ = μ₀μᵣL/(2π) · [ln(2L/D) - 1 + D/L]
```

### 2. Algorithmic Engine
The parasitic extraction is integrated directly into the post-routing validation pass:
1.  **Parallel Run Detection:** For each pair of traces on the same layer with different `NetId` values, compute their parallel run length (intersection of coordinate spans).
2.  **Effective Permittivity (Wheeler):** Before applying Sakurai's formulas, compute the effective relative permittivity of the microstrip configuration using Wheeler's closed-form equation:
    $$\epsilon_{\text{eff}} = \frac{\epsilon_r + 1}{2} + \frac{\epsilon_r - 1}{2} \left( 1 + 12\frac{H}{W} \right)^{-1/2}$$
    This accounts for electric field lines passing through both the substrate ($\epsilon_r$) and the air above ($\epsilon_r \approx 1.0$), preventing systematic overestimation of coupling capacitance.
3.  **Coupling Capacitance (Sakurai):** Using Sakurai's empirical microstrip formula with $\epsilon_{\text{eff}}$:
    $$C_{12} = \epsilon_0 \epsilon_{\text{eff}} L \left[ 0.03\left(\frac{W}{H}\right) + 0.08\left(\frac{T}{H}\right) + 0.07\left(\frac{W}{H}\right)^{0.25}\left(\frac{T}{H}\right)^{0.5}\left(\frac{H}{D}\right)^{1.34} \right]$$
    Where $L$ is the parallel run length, $W$ is the trace width, $D$ is the edge-to-edge spacing, $T$ is the trace thickness, and $H$ is the substrate height above the ground plane.
4.  **Ground Capacitance (Sakurai):** Compute the ground-plane capacitance per unit length:
    $$C_{1g} = \epsilon_0 \epsilon_{\text{eff}} L \left[ 1.15\left(\frac{W}{H}\right) + 2.80\left(\frac{T}{H}\right)^{0.222} \right]$$
5.  **Resistance Calculation:** Compute series resistance using the trace geometry and material resistivity:
    $$R_s = \rho \frac{L}{W \cdot t}$$
    Where $\rho$ is the copper resistivity, $L$ is the trace length, $W$ is the width, and $t$ is the layer thickness.
6.  **Via Self-Inductance:** For each vertical via transition, compute the analytical self-inductance:
    $$L_{\text{via}} \approx 2 \times 10^{-7} \cdot h \cdot \left[ \ln\left(\frac{4h}{d}\right) + 1 \right]$$
    Where $h$ is the via height and $d$ is the via diameter. This captures the critical localized inductive discontinuity at high frequencies that the 2D parasitic model misses.
7.  **Mutual Inductance (Greenhouse):** For each pair of parallel traces, compute the mutual inductance using the Greenhouse approximation:
    $$M_{12} = \frac{\mu_0 \mu_r L}{2\pi} \left[ \ln\left(\frac{2L}{D}\right) - 1 + \frac{D}{L} \right]$$
    Where $L$ is the parallel run length, $D$ is the center-to-edge spacing, and $\mu_0 \mu_r$ is the magnetic permeability of the medium (typically $1.2566 \times 10^{-6}\text{ H/m}$ for FR4/Silicon). This captures inductive crosstalk that the purely capacitive RC model misses.
8.  **SPICE Embedding:** The extracted $R_s$, $C_{12}$, $C_{1g}$, $L_{\text{via}}$, and $M_{12}$ values are embedded directly into the SPICE netlist export. Mutual inductances are emitted as coupled inductors (`K_coupling L_trace_a L_trace_b k_val`) to ensure full RLC/transmission line simulation fidelity.

### 3. Data Flow & Pipeline Integration
*   **Input:** Reads legalized, compacted trace coordinates and stackup material properties from the Entity Graph.
*   **Output:** Generates parasitic-aware SPICE netlists with embedded R/C values; flags critical crosstalk violations to the Verify Stage.

---

## Subsystem 22: G-Cell-Local Unified Sweep Verification (DRC + Same-Net Topology)

### 1. Core Abstraction
While the hybrid spatial index (geo-index + rstar) provides $O(\log N)$ queries, checking millions of continuous trace segments against each other via individual query calls still causes CPU thread stalls during the hot-path DRC gate. Furthermore, a global Bentley-Ottmann sweep is inherently sequential and cache-unfriendly due to pointer-chasing in its active interval tree and thread contention during concurrent access. Running two separate spatial-sweep passes (one for same-net topology, one for different-net clearances) is a computational waste.

The **G-Cell-Local Unified Sweep Verification** resolves this by replacing the global sweep with a G-cell-local flat array approach, welding both same-net topological checks and different-net clearance checks into a single memory pass.

```
  G-Cell Partition Grid (Each cell processed independently)
  ┌──────────┬──────────┬──────────┐
  │ Cell 0   │ Cell 1   │ Cell 2   │   Each cell:
  │ Rayon    │ Rayon    │ Rayon    │   - Morton-ordered flat array
  │ Thread 0 │ Thread 1 │ Thread 2 │   - 8-wide AVX-512 AABB checks
  └──────────┴──────────┴──────────┘   - Unified: same-net + different-net
```

### 2. Algorithmic Engine
1.  **G-Cell Partitioning with Boundary-Halo Expansion:** Trace segments are assigned to their containing G-cell based on coordinate bounding boxes. Any segment that lies within the G-cell boundary plus a **Boundary-Halo** distance equal to $C_{\text{max\_clearance}}$ (the maximum clearance limit of the active profile) is registered as a **Ghost Segment** in both adjacent G-cells. This ensures that near-boundary and boundary-crossing interactions are verified.
2.  **Morton Ordering:** Within each G-cell, native and ghost segments are merged into a single contiguous array sorted by Morton code (Z-order curve) to maximize spatial locality in memory and enable SIMD-friendly access patterns.
3.  **Local Sweep:** Within each G-cell, a vertical sweep-line moves left to right, maintaining a flat "Active Interval" array (not a balanced BST) of segments currently intersecting the sweep line.
4.  **Unified Overlap Dispatch:** When an overlap is detected between segment A and segment B:
    *   **Different-Net Case** (`net_id_A != net_id_B`): Evaluate different-net clearance rules using the SIMD overlap checker. Any overlapping bounding boxes that violate clearance rules are flagged as DRC violations with exact coordinate locations.
    *   **Same-Net Case** (`net_id_A == net_id_B`): Pass the intersection coordinates directly to the Topological Verifier to assert that the overlap lands exactly on a registered `VirtualJunctionNode` or within a component port bounding box. Unintentional same-net overlaps in open space are flagged as spurious loops (potential parasitic inductance sources) or dead antennas (floating trace stubs).
5.  **Rayon Parallelism:** Each G-cell is evaluated concurrently on separate threads using Rayon. Since G-cells are independent (with ghost segments providing boundary context), no global memory locks are required, achieving true linear scaling with CPU core count.
6.  **Complexity:** The unified pass completes both checks in a single memory pass with total complexity $O(N \log N + K)$ where $K$ is the number of overlapping pairs.

### 3. Data Flow & Pipeline Integration
*   **Input:** Reads compacted, legalized trace coordinates from the Compaction Engine.
*   **Output:** Passes flagged DRC clearance violations to the Localized Legalization Engine for micro-adjustment; passes same-net topology violations to the Verify Stage error handler; grants passage to the Refine Stage if all checks pass.

---

## Subsystem 23: Pattern-Guided Meander Injection (Two-Phase Post-Route)

### 1. Core Abstraction
Traditional pattern-guided routers embed meander geometry into the A* heuristic during pathfinding. This creates an $O(B^d)$ state-space explosion where $B$ is the branching factor and $d$ is the path depth—the router must evaluate every possible meander placement at every expansion step, causing compile times to scale exponentially on dense boards. Furthermore, only ~5% of nets in a typical design require length tuning, so routing 100% of nets through a pattern-aware A* is wasteful.

The **Pattern-Guided Meander Injection** subsystem resolves this by implementing a two-phase approach: Phase 1 routes all nets straight (fastest possible A*). Phase 2 scans the routed paths, selects the longest segment per net, and analytically injects meander waypoints using closed-form polar decomposition.

```
  Phase 1: Straight Routing              Phase 2: Meander Injection
  ┌──────────────────────────┐           ┌──────────────────────────────┐
  │ A* routes all nets       │           │ MeanderInjector scans paths  │
  │ with zero pattern intent │ ────────► │ Selects longest segment      │
  │ Fastest possible routing │           │ Decomposes pattern into XY   │
  └──────────────────────────┘           │ Injects waypoints at midpoint│
                                         │ Expands 4 pts → 21 pts      │
                                         └──────────────────────────────┘
```

### 2. Algorithmic Engine
1. **Policy Collection:** The compiler collects `RouteNetPolicy` declarations from the space AST, resolves net names to `NetId`s, and instantiates patterns by evaluating step expressions with the provided arguments (e.g., `Zigzag(gap: 0.5mm)` → 4 steps of 500,000 nm at angles [90°, 0°, -90°, 0°]).

2. **Longest Segment Selection:** For each net with a policy, scan all path segments (from MST decomposition) and select the one with the highest Manhattan length as the meander insertion target. The midpoint of this segment becomes the meander center.

3. **Closed-Form Meander Height:** Given the total forward distance of the pattern:
   $$\text{total\_forward} = \sum_{i} d_i \cdot \cos(\theta_i)$$
   And the number of repetitions (computed from the segment length divided by total forward distance), the meander half-span is:
   $$\text{half\_span} = \frac{\text{total\_forward}}{2}$$
   No iterative trial-and-error is required.

4. **Polar-to-Cartesian Decomposition:** Starting from `center - half_span`, each pattern step is decomposed:
   - **Forward component:** $d_x = d \cdot \cos(\theta)$ (along segment direction)
   - **Perpendicular component:** $d_y = d \cdot \sin(\theta)$ (perpendicular to segment)
   - For **horizontal segments**: forward = X, perpendicular = Y
   - For **vertical segments**: forward = Y, perpendicular = X
   This creates the characteristic back-and-forth meander geometry.

5. **Collision Detection:** Before injection, the engine constructs a bounding box around the meander path (inflated by `trace_width × 2`) and checks for intersections with adjacent traces using Minkowski-inflated AABB intersection. If a collision is detected, injection is skipped for that segment—no DRC violations are introduced.

6. **EntityGraph State-Sync:** After injection, the engine clears old route segments for the affected net from the EntityGraph and re-registers the expanded meandered paths canonically. The expanded paths are then processed by the AnalyticTrace converter and persisted to the lockfile.

### 3. Data Flow & Pipeline Integration
*   **Input:** Receives straight-routed paths from the AutoRouter (Phase 1) and resolved `RoutingPattern` objects from pattern/strategy instantiation.
*   **Output:** Generates meandered paths with expanded waypoint counts; syncs to EntityGraph; feeds into the 45° Miter Pass and AnalyticTrace conversion.

---

## Subsystem 24: 45° Mitered Chamfer Pass (Impedance-Stable Corner Geometry)

### 1. Core Abstraction
After meander injection, trace paths contain 90° orthogonal corners. At high frequencies (>5 GHz), 90° corners create impedance discontinuities due to excess capacitance at the corner vertex. The capacitance scales with the corner area, causing signal reflections and timing jitter on high-speed serial links.

The **45° Mitered Chamfer Pass** resolves this by scanning every routed path for 90° corners and inserting diagonal chamfer points. The chamfer distance is proportional to the trace width (`1.5 × W`), maintaining constant characteristic impedance ($Z_0$) through the bend per IPC-2152 guidelines.

```
  Before Miter: 90° Corner          After Miter: 45° Chamfer
  
       ┌──────                          ┌──────
       │                                │╲
       │                                │ ╲  ← 45° diagonal
       │                                │  ╲    (miter_dist = 1.5 × W)
       └──────                          └──────
  
  Excess capacitance at vertex    Smooth current flow, constant Z₀
```

### 2. Algorithmic Engine
1. **Corner Detection:** For each consecutive triple of waypoints $(P_{i-1}, P_i, P_{i+1})$, compute the direction vectors $\vec{d_1} = P_i - P_{i-1}$ and $\vec{d_2} = P_{i+1} - P_i$ in the XY plane. The dot product $\vec{d_1} \cdot \vec{d_2} = 0$ indicates a 90° corner. Z-axis differences are ignored because the miter is a 2D copper geometry operation on the routing layer.

2. **Miter Distance:** The chamfer insertion distance from the corner vertex is:
   $$\text{miter\_dist} = \text{trace\_width} \times 1.5$$
   This ratio preserves $Z_0$ continuity for microstrip and stripline geometries.

3. **Chamfer Point Insertion:** For a 90° corner at $P_i$:
   - $P_a = P_i - \text{miter\_dist} \times \hat{d_1}$ (point along incoming direction)
   - $P_b = P_i - \text{miter\_dist} \times \hat{d_2}$ (point along outgoing direction)
   The single corner vertex $P_i$ is replaced with the sequence $[P_a, P_b]$, creating a 45° diagonal chamfer.

4. **Path Expansion:** Each 90° corner adds 2 points to the path. A meander with 16 corners (4 Zigzag repetitions × 4 corners each) expands from 21 waypoints to 53 waypoints, all with smooth 45° transitions.

### 3. Data Flow & Pipeline Integration
*   **Input:** Receives meandered (or straight) path waypoint arrays from the MeanderInjector or AutoRouter.
*   **Output:** Produces mitered paths with 90° corners replaced by 45° chamfer pairs; feeds into AnalyticTrace conversion for lockfile persistence and GLB/DXF export.

---

## 5. Summary of the Unified Compilation Pipeline

| Subsystem | Stage | Primary Responsibility | Core Performance |
| :--- | :--- | :--- | :--- |
| **1. Entity Graph** | Database | Single, pristine, directed graph source of truth. | Memory-safe, pointer-free topological node map; OBVH collision; same-net verification. |
| **2. Soft Corridor Planner** | Partition | Generates preferred routing cost fields around G-cells. | Prevents routing deadlocks and via overhead. |
| **3. Partition Stage** | Partition | Spatial decomposition with Boundary Track Reservation Table + Localized Port Relocation. | Shifted ports ±3·track_pitch invalidate only 2 G-cells; parallel local routing via Rayon. |
| **4. Negotiated Congestion** | Partition | Resolves multi-net resource sharing iteratively. | No routing deadlocks on dense layouts. |
| **5. Route Segment Decomp** | Route | Decomposes logical nets into pairwise route segments with Virtual Junction Nodes. | Resolves same-net shortcutting; models T-junction parasitics for SPICE. |
| **6. Topological Router** | Route | Ray-casting pathfinder with Axis-Aligned Slab Method + Diagonal Grid-Snapping. | No voxel grid crawling; 45° segments snap to orthogonal tracks; O(log N) ray-AABB queries. |
| **7. Localized Legalization** | Legalize | Hybrid QP: clarabel (global IPM) + active-set / DAG solver (local). | Continuous trace nudging; no global rip-up loops; microsecond local solves. |
| **8. Constraint Compaction** | Compact | Slides parallel traces together up to clearance limits. | Maximizes layout density safely. |
| **9. Verify Stage** | Verify | Mandatory gate: Same-net topo + G-Cell SIMD DRC + Sakurai BEM + EM/Thermal (G-cell-coupled). | 8-wide AVX-512; ground-plane R/C; floating antenna detection; IPC-2152 compliance with local thermal maps. |
| **10. Semantic Lockfile** | Cache | Single-source binary cache: rkyv only (zero-copy mmap). Human inspection via CLI. | Near-instant load (<1ms); zero parsing overhead; clean VCS history. |
| **11. Dependency DAG** | Cache | Directed acyclic graph tracking of local dependencies. | Localized cache invalidation and re-routing. |
| **12. Vector Persistence** | Cache | Serializes paths as Base-36 RLC delta-turn strings. | Minimal disk footprint; lossless precision. |
| **13. Geometry Refinement** | Refine | Performs 2D polygon unioning and canonicalization. | Eliminates coplanar boundaries and Z-fighting. |
| **14. Export Isolation** | Refine | Decouples mesh triangulation from internal database. | Triangulation occurs strictly at the export boundary. |
| **15. Acceleration Index** | Database | Hybrid volatile index: rstar (dynamic) + geo-index (static layer slices). | Logarithmic spatial lookup; 5–10x faster on static obstacles. |
| **16. Incremental DRC** | Verify | Localized DRC validation within edit windows. | Bypasses full-board checks, accelerating rebuilds. |
| **17. Multi-Net Router** | Route | Manages and isolates individual net constraints. | Prevents different-net overlapping and shorts. |
| **18. Deterministic Build** | Pipeline | Guarantees identical input produces byte-identical output. | Fixed-point i64 transforms; stable sorting; deterministic tie-breaking. |
| **19. Zero-Stamping Scene Graph** | Database | Analytic ComponentStamps with OBVH + pre-transformed global bounding boxes. | 95%+ memory reduction; SAT-based OBB collision in global coords; no truncation drift. |
| **20. Salsa Query Engine** | Pipeline | Fine-grained memoized queries per component/route entity. | Sub-ms incremental compiles; no AST-wide invalidation cascades. |
| **21. BEM Parasitic Extract** | Verify | Sakurai empirical microstrip R/C + Virtual Junction L/C extraction. | Millisecond extraction; 5–10% accuracy vs 3D FEM; T-junction subcircuits in SPICE. |
| **22. G-Cell-Local Unified Sweep** | Verify | Unified sweep: different-net DRC (8-wide AVX-512) + same-net topology in single $O(N \log N + K)$ pass. | True linear Rayon scaling; eliminates redundant spatial sweeps. |
| **23. Pattern-Guided Meander Injection** | Route | Post-route analytical meander injection using polar decomposition. | O(N log N + K); closed-form height calc; no A* state-space explosion. |
| **24. 45° Mitered Chamfer Pass** | Refine | Scans 90° corners, inserts 45° diagonal chamfers at 1.5× trace width. | Impedance-stable bends; XY-plane-only dot product check. |