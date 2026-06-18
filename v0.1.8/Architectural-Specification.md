# Architectural Specification: Hardware Script v0.1.8
**Document Type:** Core Architectural Specification  
**Status:** Feature Freeze (v0.1.8-alpha)  
**Focus:** Continuous Vector Database, Topological Line-Search, and Convex Optimization

---

## 1. The Brutal Audit: Why we must destroy the v0.1.7 Core

While version 0.1.7 successfully validated logical netlist determinism and basic routing paths, its architectural foundation is fundamentally limited. It was built as a "Voxel-First" Painter—treating physical space as a discrete 3D grid [Z-AXIS-ABSTRACTION-IMPLEMENTATION.md, Volumetric-Solid-Modeling-via-Boundary-Representation.md].

As we transition to production-scale boards and sub-micron silicon chiplets, the voxel-based A* routing engine has hit its physical limits [TSV-STATUS.md, The-SoC-Engine.md].

### The Core Limitations of the v0.1.7 Architecture

```
   THE VOXEL ROUTER (v0.1.7)                   THE VECTOR-FIRST PIPELINE (v0.1.8)
  ┌─────────────────────────────┐             ┌─────────────────────────────────┐
  │ - 3D A* Voxel-Crawling      │             │ - Zero-Stamping Scene Graph     │
  │ - Voxel-Center Snapping     │   ──────►   │ - Affine-Transform Queries      │
  │ - O(N³) memory scaling      │             │ - O(log N) Hybrid Spatial Index │
  │ - "Phantom Via" generation  │             │ - Salsa-Style Memoized Queries  │
  │ - Full-board DRC sweeps     │             │ - SIMD Sweep-Line DRC + BEM    │
  └─────────────────────────────┘             └─────────────────────────────────┘
   Grid Locked / Fragile / Slow                 Infinitely Scalable / Physical-Aware
```

1.  **The Voxel Density Bomb:** To route a $2\text{nm}$ silicon gate or a $200\text{mm}$ PCB at sub-micron precision, a voxel grid requires billions of discrete cells [The-SoC-Engine.md]. Even with sparse chunk hash-maps, the memory footprint and CPU hashing overhead during active development cause compilation times to scale exponentially ($O(N^3)$), choking the compiler.
2.  **Quantization Noise (Staircasing & Jitter):** Because a voxel grid is made of discrete blocks, the pathfinder is forced to crawl cell-by-cell. This introduces jagged "staircase" traces, horizontal-plane "jitter," and vertical "roof" gaps [Z-AXIS-ABSTRACTION-IMPLEMENTATION.md, The-SoC-Engine.md, Compiler-Bug-Tracker.md].
3.  **Phantom Vias:** When a trace snaps to a voxel center (e.g., $Z=0.95\text{mm}$) when the pad sits on the layer boundary ($Z=1.27\text{mm}$), it creates an artificial vertical gap [The-SoC-Engine.md]. The via unroller detects this minor $Z$ drift as an implicit layer transition, placing redundant, vertical "phantom vias" that block adjacent traces [Via-Engine-Implementation.md, The-SoC-Engine.md].
4.  **Inefficient Rip-up Repair:** When traces collide, v0.1.7 is forced to tear down the entire trace and re-evaluate pathfinding from scratch. It has no concept of "sliding" or "nudging" existing traces out of the way.
5.  **Data Mismatch (Identity Leakage):** The compiler conflated Logical Nets, Route Segments, and Physical Geometries [LOGICAL-BEHAVIOR-GUIDE.md]. This caused the unroller to generate 52 separate, redundant physical layer objects for 52 trace segments, bloating the GDSII/GLB exporters and creating un-welded, lumpy visual joints [2D-POLYGON-UNIONING-IMPLEMENTATION.md].

---

## 2. The v0.1.8 Paradigm: Continuous Physical Synthesis

Version 0.1.8 rebuilds the entire middle-end of the compiler around a **Continuous Vector Database** and an **R\*-Tree Spatial Index**, completely removing voxel-grid dependencies during compilation [Z-AXIS-ABSTRACTION-IMPLEMENTATION.md, Unified-2.5D-3D-Routing-and-Placement.md, Advanced-Routing-Implementation.md].

The compiler transitions from a "descriptive painter" to a **Prescriptive Physical Synthesis Compiler** [ARCHITECTURAL-AUDIT-AND-ROADMAP.md].

### The 7-Stage Physical Synthesis Pipeline

```
0. UNROLL (Zero-Stamping Scene Graph)
   - Parse ComponentStamps once, store as analytical local-coordinate polygons
   - Instance placement = FixedTransform2D (T, pure i64) + net bindings
          │
          ▼
1. PARTITION (Global Route Policy)
   - Run PathFinder negotiated congestion solver to allocate soft corridors
   - Demand-driven via Salsa-style memoized query engine
          │
          ▼
2. ROUTE (Topological Line-Search)
   - Project orthogonal search rays along preferred-direction layers
   - Axis-Aligned Ray-AABB Slab Method over static geo-index obstacles
   - Affine-inverse collision tests against local-coordinate OBB/AABB stamps
          │
          ▼
3. LEGALIZE (Hybrid QP Solver)
   - Macro: clarabel IPM for complex multi-variable constraints
   - Local: Active-set / DAG solver for micro-sliding
          │
          ▼
4. COMPACT (Signal-Aware Squeezing)
   - Slide traces together, capped by high-speed clearance constraints
          │
          ▼
5. VERIFY (G-Cell-Local Unified Sweep + Wheeler-Sakurai BEM Parasitic Extraction)
   - G-Cell-local flat Morton-ordered interval sweep with 8-wide AVX-512 checks
   - Sakurai empirical microstrip parasitic R/C extraction (ground-plane aware)
          │
          ▼
6. REFINE (Mesh Generation & Cache)
   - Weld overlapping copper in 2D (Clipper2) and triangulate via earcut
   - Write binary rkyv lockfile (zero-copy mmap); CLI inspect on demand
```

*   **Unroll:** Each component type is parsed once into a `ComponentStamp`—an **Oriented Bounding Volume Hierarchy (OBVH)** in local coordinate space (origin `[0, 0, 0]`). Component instances are stored as lightweight **FixedTransform2D** matrices ($T$, pure i128 intermediate arithmetic) with pre-transformed global bounding boxes cached at placement time, completely eliminating voxel stamping and reducing memory by 95%+ [BIT-BLIT-UNROLLER-IMPLEMENTATION.md].
*   **Partition:** We divide the routing space into coarse, logical "G-cell corridors" [Adaptive-Heuristic.md]. Instead of global search, we use a hybrid of heuristic net ordering and the **PathFinder negotiated congestion algorithm**. This solves global track allocation, ensuring nets negotiate paths before any physical traces are drawn. All partition queries are modeled as **Salsa-style memoized functions**—if a G-cell's inputs haven't changed, its routing result is fetched from cache in $<1\text{ms}$ [Core-System-Architecture.md, LAZY-REALIZATION-ARCHITECTURE.md].
*   **Route:** The **Topological Line-Search Router** executes within these soft, pre-allocated corridors [Adaptive-Heuristic.md]. It projects horizontal and vertical search rays along continuous coordinates using the **Axis-Aligned Slab Method**—performing single $O(\log N)$ ray-AABB intersection queries directly over the flat-packed `geo-index` to resolve the next obstacle distance instantly without iterative stepping—generating straight, orthogonal trace vectors with perfect $90^\circ$ or $135^\circ$ bends, avoiding voxel quantization entirely [Unified-2.5D-3D-Routing-and-Placement.md]. Collision checks against component instances use pre-calculated global bounding boxes—SAT-based OBB collision for rotated shapes, fast AABB for Manhattan (see Critique 4).
*   **Legalize:** If two traces collide or violate clearance rules, we formulate their spatial boundaries as continuous quadratic inequalities [ROUTING-AND-MANUFACTURING-ARCHITECTURE.md, Unified-2.5D-3D-Routing-and-Placement.md]. For macro-scale constraints, a **`clarabel` interior-point solver** evaluates the vectors and "nudges" them smoothly out of the way. For local micro-adjustments inside small legalization windows, a lightweight **active-set solver or DAG graph solver** handles the sliding in microseconds, preventing cascading global reroutes.
*   **Compact:** Slides adjacent traces together, capped by signal integrity constraints (crosstalk limits, target impedance, and continuous return-path shielding) to maximize routing density.
*   **Verify:** Runs a **G-Cell-local unified sweep** across the continuous 2D coordinate plane. Within each G-cell, trace segments are flattened into contiguous Morton-ordered arrays. A single $O(N \log N + K)$ sweep handles both **different-net clearance checks** (SIMD-accelerated 8-wide AVX-512) and **same-net topological verification** (asserting overlaps land on `VirtualJunctionNode`s or port bounding boxes)—eliminating the need for two separate spatial passes [Core-System-Architecture.md]. Simultaneously executes **Wheeler's effective permittivity** and **Sakurai's empirical microstrip parasitic formulas** to compute ground-plane-aware coupling capacitance ($C_{12}$), ground capacitance ($C_{1g}$), series resistance ($R_s$), and via self-inductance ($L_{\text{via}}$) [Maturity-and-Expansion-Vision.md].
*   **Refine:** Groups copper by layer/net, welds overlapping regions in 2D using the Non-Zero Winding Rule, performs earcut triangulation only during GLB viewport export, and caches the clean, delta-encoded bend-points in the binary `rkyv` lockfile. **Crucially, geometric welding (Clipper2) and triangulation (earcut) occur strictly at the final export boundary**—LVS validation, DRC checking, and parasitic extraction all run on the raw, distinct vector segments to preserve geometric identity throughout compilation [Core-System-Architecture.md].

---

## 3. The State-of-the-Art Rust Library Stack

To achieve sub-millisecond, SoC-scale execution without relying on bloated, non-deterministic machine learning models, version 0.1.8 relies entirely on a curated, high-performance, native Rust library stack [COMPILER-CHANGES.md].

```
           HWS COMPILER CORE (Rust-Native)
                         │
        ┌────┬────┬──────┼──────┬────┬────────┐
        ▼    ▼    ▼      ▼      ▼    ▼        ▼
   rstar  geo-  clarabel earcut rkyv glam    std::
   (Dyn   index (Global  (Map  (Zero salsa  simd
    R*-   (Stat IPM)    Ear-  Copy (Query (SIMD
    Tree) ic)           Clip) Bin)  Engine) Sweep)
```

### A. Spatial Indexing: Hybrid `rstar` + `geo-index`
*   **The Libraries:**
    *   `rstar`: A highly optimized, dynamic R\*-tree spatial index written in pure, safe Rust.
    *   `geo-index`: A static, packed R\*-tree built on Flatbush/Hilton sorting with a flat, contiguous `Vec<u8>` buffer layout.
*   **The Role:** The compiler uses a **hybrid indexing model** to balance dynamic updates against query throughput:
    *   **`rstar`** is reserved for the top-level macro-placement of components during floorplanning, where dynamic movement and insertion occur frequently.
    *   **`geo-index`** is used for detailed routing layers, where the vast majority of spatial queries target static obstacles (component boundaries, keepouts, and committed traces). The compiler compiles obstacle geometry into flat, packed `geo-index` structures per layer after the partition stage.
*   **The Performance:** `rstar` provides $O(\log N)$ logarithmic queries for dynamic workloads. `geo-index` delivers 5×–10× faster query performance on static obstacles due to near-zero allocation overhead, contiguous memory layout, and excellent L1/L2 cache locality. It is also ABI-stable and memory-map (`memmap2`) ready for persistent caching.

### B. Convex Legalization: Hybrid `clarabel` + Active-Set / DAG Solver
*   **The Libraries:**
    *   `clarabel`: A state-of-the-art, high-performance conic interior-point optimization solver written in pure Rust.
    *   `PIQP` (or equivalent active-set solver): A lightweight proximal active-set method for small, sparse Quadratic Programs with near-zero startup cost.
    *   DAG Graph Solver: A dedicated longest-path constraint solver for 1D trace compaction, modeled on standard graph-based simplex compaction from VLSI routing.
*   **The Role:** The legalization subsystem uses a **two-tier solver strategy**:
    *   **`clarabel`** is used for **macro-scale physical synthesis** (e.g., global block floorplanning and macro-corridor legalization) where complex, multi-variable convex constraints exist and the variable count is large.
    *   **The active-set solver or DAG graph solver** is used for **local detailed trace sliding and compaction** inside small legalization windows (e.g., nudging 2–3 parallel trace segments when a new via is dropped). For small, sparse QP problems ($N < 20$), active-set solvers solve in microseconds with zero heap allocations, avoiding the $O(N^3)$ matrix factorization overhead that `clarabel` incurs on every call.
*   **The Performance:** Macro-scale optimization benefits from `clarabel`'s robustness on large constraint sets. Local micro-adjustments achieve near-instant solve times without numerical bottlenecks, preventing the pathfinder from stalling during detailed route adjustments.

### C. Polygon Unioning & Tessellation: `clipper2-rust` + `earcut`
*   **The Libraries:**
    *   `clipper2-rust`: A pure Rust port of the standard Clipper2 library, executing 2D polygon boolean clipping with absolute mathematical robustness.
    *   `earcut` (Mapbox/GeoRust): A zero-allocation, flat-array ear-clipping triangulation library optimized for clean polygons with holes.
*   **The Role:** `clipper2-rust` performs the 2D boolean union (welding) of overlapping traces and annular rings using the Non-Zero Winding Rule [2D-POLYGON-UNIONING-IMPLEMENTATION.md, MICROKERNEL-ARCHITECTURE.md]. Because `clipper2-rust` already outputs perfectly clean, non-self-intersecting outer contours with properly oriented nested holes (PolyTree), the resulting contours are passed directly to `earcut` for zero-allocation cap triangulation during the final GLB export pass [2D-POLYGON-UNIONING-IMPLEMENTATION.md, Volumetric-Solid-Modeling-via-Boundary-Representation.md].
*   **The Performance:** `earcut` is 3×–5× faster than heavier tessellators (like `tess2-rust`) on clean, unioned contours due to its flat-array architecture and zero heap allocations, significantly accelerating the GLB export pass on large-scale boards.

### D. Sub-Millisecond Serialization: Binary `rkyv` + `memmap2` (AlignedVec)
*   **The Library:** `rkyv` (zero-copy binary serialization) + `memmap2` (memory-mapped I/O) + `AlignedVec` (16-byte alignment guarantee).
    *   `rkyv`: A zero-copy binary serialization framework that constructs ABI-stable byte representations on disk, enabling memory-mapped (`memmap2`) deserialization with zero parsing overhead.
    *   `AlignedVec`: Guarantees serialized payloads are perfectly aligned to CPU native word boundaries, preventing unaligned memory access panics during zero-copy casts.
*   **The Role:** The compiler enforces a **single-source-of-truth lockfile** (`project.routes.lock`) stored exclusively in the binary `rkyv` format. The serialization path uses `AlignedVec` (16-byte alignment) to guarantee safe zero-copy deserialization. The compiler memory-maps the lock file and casts the raw bytes directly into Rust structs in microseconds with zero CPU cycles spent parsing and zero heap allocations. No secondary JSON file is generated during builds. If a human needs to inspect or audit the lockfile, a dedicated CLI tool (`hwc lock inspect project.routes.lock`) decodes the binary to stdout on demand, keeping the workspace and version control history clean.
*   **The Performance:** Binary lockfile loads achieve sub-millisecond, near-instantaneous deserialization even for SoC-scale designs (100-million transistor layouts), keeping development loops instantaneous.

### E. Zero-Stamping Scene Graph: FixedTransform2D (Deterministic Integer Transforms)
*   **The Library:** `glam` is forbidden in the core pathfinder, collision, and coordinate transform engines. All coordinate transformations use **`FixedTransform2D`** (pure i128 intermediate arithmetic, scaled by $10^9$) to guarantee bit-identical determinism across CPU platforms. On a standard 200 mm PCB, coordinate values reach $2 \times 10^{11}\text{ pm}$; intermediate products with trigonometric scale factors reach $\approx 1.414 \times 10^{20}$, exceeding i64's max ($\approx 9.22 \times 10^{18}$). All intermediate multiplications are therefore promoted to **i128** (max $\approx 3.40 \times 10^{38}$) before division, casting back to i64 only after the scaling factor is removed. `glam` may only be used in the GLB viewport renderer (non-deterministic visualization only).
*   **The Role:** Instead of rasterizing millions of standard-cell geometries into a physical voxel grid during unrolling, the compiler stores each component type exactly once as a `ComponentStamp`—an **Oriented Bounding Volume Hierarchy (OBVH)** in local coordinate space (origin `[0, 0, 0]`). Each placed instance is represented as a lightweight **FixedTransform2D** ($T$, pure i128 intermediate arithmetic) with logical net bindings. At placement time, local bounding volumes are transformed **forward** into global world-coordinate space and cached on the `ComponentInstance`. All collision checks execute in global coordinates against pre-calculated global bounding boxes—eliminating lossy on-demand inverse transforms and integer truncation asymmetry:
    1.  The local stamp bounding boxes are transformed **forward** into global space once at placement time via $P_{\text{global}} = T \cdot P_{\text{local}}$ using i128 intermediate products.
    2.  The collision test is resolved via fast register-friendly AABB/OBB comparisons against pre-calculated global bounds. Only components explicitly declared with non-standard custom shapes fall back to Jordan curve theorem polygon checks.
*   **The Performance:** Memory consumption for a 100M-transistor design drops from $1.6\text{ GB}$ to under $80\text{ MB}$ because the physical voxel grid is completely eliminated. AABB/OBB collision tests are $O(1)$ per box with near-zero branch misprediction, keeping the CPU cache footprint minimal.

### F. Salsa-Style Memoized Query Engine: `salsa`
*   **The Library:** `salsa`—a demand-driven, incremental computation framework originally developed for the Rust compiler (`rustc`) and `rust-analyzer`.
*   **The Role:** Every compiler phase (parsing, macro-placement, G-cell partition, detailed routing, DRC verification, and mesh extrusion) is modeled as a set of pure, memoized query functions. When an entity is modified, the dependency DAG traces upward, invalidating only the specific nodes in the query cache. If a G-cell's input boundary and internal components haven't changed, its detailed routing mesh is loaded instantly from the memory-mapped `rkyv` cache, bypassing the pathfinder entirely. Cyclic dependencies are mathematically impossible because queries are pure functions with no side effects.
*   **The Performance:** Incremental compilation times for minor edits on major designs drop to $<10\text{ ms}$. The compiler avoids re-evaluating unchanged sub-graphs, achieving near-zero overhead on cached query paths.

### G. Analytic 2.5D BEM Parasitic Solver (Wheeler + Sakurai + Via Inductance + Mutual Inductance)
*   **The Method:** Boundary Element Method (BEM) / Method of Moments (MoM) operating on continuous coordinate vectors, using **Wheeler's effective permittivity** to compute $\epsilon_{\text{eff}}$ before applying **Sakurai's empirical microstrip formulas** for ground-plane-aware parasitic capacitance ($C_{12}$, $C_{1g}$) and resistance ($R_s$), analytical **via self-inductance** ($L_{\text{via}}$), and **mutual inductance** ($M_{12}$) using the Greenhouse approximation for parallel trace runs.
*   **The Role:** Instead of meshing the entire 3D space (as in slow FEM solvers) or using inaccurate free-space formulas, the compiler first computes the effective relative permittivity accounting for field lines passing through both substrate and air:
    $$\epsilon_{\text{eff}} = \frac{\epsilon_r + 1}{2} + \frac{\epsilon_r - 1}{2} \left( 1 + 12\frac{H}{W} \right)^{-1/2}$$
    Then applies Sakurai's empirical formulation with $\epsilon_{\text{eff}}$:
    $$C_{12} = \epsilon_0 \epsilon_{\text{eff}} L \left[ 0.03\left(\frac{W}{H}\right) + 0.08\left(\frac{T}{H}\right) + 0.07\left(\frac{W}{H}\right)^{0.25}\left(\frac{T}{H}\right)^{0.5}\left(\frac{H}{D}\right)^{1.34} \right]$$
    $$C_{1g} = \epsilon_0 \epsilon_{\text{eff}} L \left[ 1.15\left(\frac{W}{H}\right) + 2.80\left(\frac{T}{H}\right)^{0.222} \right]$$
    For vertical via transitions: $L_{\text{via}} \approx 2 \times 10^{-7} \cdot h \cdot [\ln(4h/d) + 1]$.
    For parallel trace runs, mutual inductance via Greenhouse: $M_{12} = \frac{\mu_0 \mu_r L}{2\pi} \left[ \ln\left(\frac{2L}{D}\right) - 1 + \frac{D}{L} \right]$.
    This calculation is integrated directly into the compiler's post-routing validation pass, generating highly accurate parasitic-aware SPICE netlists with embedded $R_{\text{series}}$, $C_{\text{coupling}}$, $C_{\text{ground}}$, $L_{\text{via}}$, and $M_{12}$ (coupled inductor `K` cards) values in microseconds.
*   **The Performance:** Eliminates the need for expensive third-party extraction tools. Parasitic extraction on a full SoC design completes in $<50\text{ ms}$ with 5–10% accuracy versus 3D numerical field solvers.

### H. G-Cell-Local Unified Sweep Verification (DRC + Same-Net Topology): `std::simd` (Portable SIMD)
*   **The Method:** G-Cell-local flat Morton-ordered interval sweep accelerated with portable SIMD vector registers (AVX-512 / NEON) for 8-way parallel bounding box overlap checks, executing both different-net DRC and same-net topological verification in a single $O(N \log N + K)$ pass. **Boundary-Halo Expansion (Ghost Zones)** ensures near-boundary violations are not missed—any segment within $C_{\text{max\_clearance}}$ of a G-cell boundary is registered as a Ghost Segment in both adjacent G-cells.
*   **The Role:** Instead of running two separate spatial-sweep passes (one for same-net topology, one for different-net clearances), the compiler welds both checks into the same G-cell-local sweep-line event-loop. When an overlap is detected: if `net_id_A != net_id_B`, evaluate clearance rules via SIMD; if `net_id_A == net_id_B`, assert the overlap lands on a `VirtualJunctionNode` or component port bounding box. Each G-cell is evaluated concurrently on separate threads using Rayon, achieving true linear scaling with CPU core count.
*   **The Performance:** Physical verification times on complex ASIC layers drop from seconds to milliseconds. The unified pass eliminates redundant spatial sweeps, while Rayon parallelism across G-cells provides linear multi-core scaling.

---

## 4. Summary of the Architectural Blueprint

By moving to this vector-first paradigm with production-hardened architectural corrections, version 0.1.8 positions Hardware Script to effortlessly handle picometer-scale silicon and complex multilayer PCBs [Z-AXIS-ABSTRACTION-IMPLEMENTATION.md, Unified-2.5D-3D-Routing-and-Placement.md]:

*   **Picometer-Precision Database:** All internal coordinates are stored as absolute 64-bit integer picometers (pm), where $1\text{ pm} = 10^{-12}\text{ m}$. Using i64, the maximum addressable coordinate space is $\pm 9.22 \times 10^{18}\text{ pm} = \pm 9,220\text{ km}$, spanning from planetary-scale PCBs to sub-atomic quantum silicon layouts without floating-point jitter or rounding errors. The user-facing `resolution:` attribute remains customizable (e.g., `resolution: 1nm` or `resolution: 1pm`), acting as a snapping constraint on the parser, while the engine always executes in picometers [Syntax-&-Definition.md].

*   **Zero-Stamping Scene Graph with OBB Collision:** Component types are parsed once as analytical `ComponentStamp` templates with Oriented Bounding Volume Hierarchies (OBVH). At placement time, local bounding volumes are transformed **forward** into global world-coordinate space and cached on the `ComponentInstance`—eliminating lossy on-demand inverse transforms during pathfinding and DRC. All collision checks execute in global coordinates against pre-calculated global bounding boxes, preventing integer truncation asymmetry and grid-drift [BIT-BLIT-UNROLLER-IMPLEMENTATION.md].
*   **Same-Net Topological Verifier:** Same-net overlaps are only permitted at registered junction nodes or within component port bounding boxes. Unintentional same-net overlaps in open space are flagged as spurious loops or dead antennas, eliminating floating trace stubs and unanticipated parasitic inductances [Core-System-Architecture.md].
*   **Technology-Specific Via Rules:** The Stackup Profile encodes `technology: PCB` or `technology: ASIC` tags with per-technology via constraints (`allow_stacked_vias`, `min_stagger_offset`, `aspect_ratio_limit`). PCB vias are treated as column-obstacles across all intermediate layers; silicon vias are layer-local. The pathfinder penalizes stacked microvias on PCB designs per IPC Class 3 rules [Unified-2.5D-3D-Routing-and-Placement.md].
*   **Virtual Junction Nodes for SPICE:** T-junction taps are promoted from coordinate intersections to first-class three-port virtual junction nodes in the EntityGraph. The Sakurai BEM Parasitic Extractor calculates $C_{\text{junc}}$ and $L_{\text{junc}}$ of the physical junction shape and emits lumped three-port subcircuits in the SPICE output [Core-System-Architecture.md].
*   **Subtractive Mask Keepouts:** Solder mask openings are decoupled from component landing pads. Mask keepouts are first-class subtractive boundary polygons stored in the database, merged dynamically during Gerber Top-Mask layer generation. This enables arbitrary trace-level mask openings for high-power thermal dissipation [Core-System-Architecture.md].
*   **Salsa-Style Memoized Queries:** Every compiler phase is modeled as a pure, demand-driven query function. The dependency DAG traces invalidation at the G-cell level, keeping incremental compile times under $10\text{ ms}$ for minor edits [Core-System-Architecture.md, LAZY-REALIZATION-ARCHITECTURE.md].
*   **Hybrid Spatial Index:** `rstar` for dynamic floorplanning, `geo-index` for static detailed routing. The **Topological Line-Search Router** calculates straight, orthogonal trace paths using **Axis-Aligned Slab Method** ray-AABB intersection queries over the flat-packed `geo-index`, eliminating iterative SDF stepping and achieving single-pass $O(\log N)$ obstacle resolution [Unified-2.5D-3D-Routing-and-Placement.md, ADVANCED-ROUTING-IMPLEMENTATION.md].
*   **Boundary Track Buffering:** The Partition Stage pre-negotiates and locks coordinate points where each net crosses G-cell boundaries as Immutable Interface Ports. Detailed parallel routers are prohibited from placing obstacles within a clearance envelope of reserved interface ports. If a detailed router cannot reach a reserved port, it first attempts **localized boundary port relocation** within $\pm 3 \cdot \text{track\_pitch}$ along the shared boundary—invalidating only the two adjacent G-cells in Salsa's cache. Only if local relocation fails does it yield to the Negotiated Congestion Engine for global renegotiation, preventing G-cell boundary deadlocks while preserving sub-10ms incremental compile times [Core-System-Architecture.md].
*   **Diagonal Grid-Snapping Law:** The Topological Line-Search Router enforces that any 45° diagonal segment has its length $L$ dynamically rounded so endpoints land exactly on orthogonal routing grid track intersections: $L_{\text{snapped}} = \text{round}(N \cdot \text{track\_pitch} / \sin(45°))$, preventing grid-drift propagation during octilinear routing [Core-System-Architecture.md].
*   **Two-Tier Legalizer:** `clarabel` for global floorplanning, active-set / DAG solver for local micro-adjustments. Resolves clearance violations by smoothly sliding and nudging vectors, avoiding costly complete rip-up-and-reroute loops and $O(N^3)$ factorization bottlenecks [ROUTING-AND-MANUFACTURING-ARCHITECTURE.md].
*   **G-Cell-Local Unified Sweep Verification:** Flat Morton-ordered interval sweep within each G-cell with Boundary-Halo Expansion (Ghost Zones), executing both different-net DRC (8-wide AVX-512) and same-net topological verification in a single $O(N \log N + K)$ pass. Rayon-parallel across G-cells. Eliminates redundant spatial sweeps and global Bentley-Ottmann pointer-chasing [Core-System-Architecture.md].
*   **Wheeler + Sakurai + Greenhouse BEM Parasitic Extraction:** Wheeler's effective permittivity ($\epsilon_{\text{eff}}$) accounts for fringing fields through air; Sakurai's empirical microstrip formulas compute ground-plane-aware $C_{12}$, $C_{1g}$, $R_s$, and via self-inductance $L_{\text{via}}$; Greenhouse approximation computes mutual inductance $M_{12}$ for full RLC/transmission line SPICE fidelity—all in microseconds with 5–10% accuracy versus 3D field solvers [Maturity-and-Expansion-Vision.md].
*   **Strict Export Isolation:** LVS validation, DRC checking, and parasitic extraction run on **raw, distinct vector segments** to preserve geometric identity. `clipper2-rust` welding and `earcut` triangulation occur strictly at the final export boundary (GLB, DXF, GDSII) [Core-System-Architecture.md].
*   **Current-Density & Thermal-Rise Verification (AC-Aware, G-Cell-Thermal-Coupled):** The Verify Stage validates minimum trace cross-sectional area against electromigration limits ($A_{\text{min}} = I_{\text{peak}} / J_{\text{limit}}$) for silicon and IPC-2152 temperature rise limits for PCBs using **separate AC current declarations**: EM limits use $I_{\text{peak}}$ and thermal-rise limits use $I_{\text{RMS}}$ from `current_limit: [rms: ..., peak: ...]`. For PCBs, the maximum allowable temperature rise ($\Delta T$) is scaled dynamically based on G-cell-local thermal maps—traces passing through thermal hotspots (e.g., near voltage regulators at $125°\text{C}$) are automatically designed wider to prevent localized delamination or electromigration failures. Violations halt the build as P21 (EM) or P22 (thermal) errors [Core-System-Architecture.md].
*   **Semantic Plane Separation:** `substrate` is reserved strictly for non-conductive dielectric materials (FR4, Silicon, Glass). `plane` is introduced as a first-class keyword for large conductive sheets (Copper, Aluminum) with subtractive cutouts (antipads, plane voids). This eliminates semantic conflation between dielectric mounting holes and conductive antipads [Syntax-&-Definition.md].
*   **Single-Source Binary Lockfile:** `rkyv` + `memmap2` zero-copy loads only. No secondary JSON file is generated during builds. Human inspection via `hwc lock inspect` CLI tool on demand, keeping workspace and VCS history clean [ROUTING-LOCK-SYSTEM-SPEC.md].