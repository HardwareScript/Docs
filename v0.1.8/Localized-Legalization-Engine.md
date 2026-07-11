# Architectural Specification: Localized Legalization Engine

**Target Version:** v0.1.8-alpha  
**Document Type:** Subsystem Specification  
**Status:** Approved for Implementation  
**Focus:** Legalization-Only Routing Pipeline, Continuous Convex Optimization, Via Sliding, Capacity-Aware Partitioning, and Profile-Driven Net Priority

---

## 1. Design Rationale: Why Rip-Up and Reroute Was Removed

Traditional EDA tools solve routing congestion through iterative **rip-up and reroute (RRR)** loops: when a net cannot find a path, previously routed segments from other nets are ripped up, and both the new net and the ripped-up nets are re-routed. This creates an $O(N \cdot K)$ convergence loop where $N$ is the number of nets and $K$ is the number of rip-up iterations. In the worst case, the algorithm oscillates between conflicting routing solutions without converging.

Hardware Script v0.1.8 replaces this with a fundamentally different insight: **a system natively built on continuous geometry can bypass the rip-up-and-reroute loop entirely**. Three structural rules make this possible:

1.  **Capacity-Aware Global Planning:** The Partition stage reserves boundary ports with explicit capacity limits. If a boundary is full, the router receives a `None` allocation and must find an alternative corridor — no congestion accumulates silently.
2.  **Flexible Local Compaction with Via/Pin Sliding:** After routing, the Legalizer nudges overlapping traces apart using continuous convex optimization. Vias belonging to displaced nets are slid by the same average displacement, preserving connectivity without re-routing.
3.  **Strict Layer-Aware Keepouts & Port Exemptions:** Substrate pours are registered as routing obstacles. Same-net pours are exempt. Start/goal components of the active route are exempted from the obstacle map, allowing clean docking.

The result: routing is a single-pass process. Topological conflicts are resolved by the Legalizer, not by ripping up and re-routing.

---

## 2. Pipeline Overview

The v0.1.8 routing pipeline is a four-stage sequential process with no feedback loops:

```
    ┌─────────────────────────────────────────────────────────────────┐
    │                  LEGALIZATION-ONLY ROUTING PIPELINE              │
    ├─────────────────────────────────────────────────────────────────┤
    │                                                                 │
    │  Stage 1: Global Partition                                      │
    │  ┌───────────────────────────────────────────────────────────┐  │
    │  │  PartitionGrid divides board into G-cells                 │  │
    │  │  Capacity-aware boundary port allocation                  │  │
    │  │  Refuses allocation if boundary at max capacity           │  │
    │  └───────────────────────────────────────────────────────────┘  │
    │                           │                                     │
    │                           ▼                                     │
    │  Stage 2: Topological Routing                                  │
    │  ┌───────────────────────────────────────────────────────────┐  │
    │  │  TopologicalRouter / SDF-accelerated A* pathfinding       │  │
    │  │  Profile-driven heuristics (base_cost, via_penalty, etc.) │  │
    │  │  Pour-aware obstacle map (substrate layers as obstacles)  │  │
    │  │  Same-net pour exemption, same-net route tapping          │  │
    │  └───────────────────────────────────────────────────────────┘  │
    │                           │                                     │
    │                           ▼                                     │
    │  Stage 3: Via Sliding                                          │
    │  ┌───────────────────────────────────────────────────────────┐  │
    │  │  Per-net average displacement computed from legalization   │  │
    │  │  All vias belonging to displaced nets slide by same delta  │  │
    │  └───────────────────────────────────────────────────────────┘  │
    │                           │                                     │
    │                           ▼                                     │
    │  Stage 4: Localized Legalization                               │
    │  ┌───────────────────────────────────────────────────────────┐  │
    │  │  Legalizer detects clearance violations                   │  │
    │  │  Creates merged legalization windows                       │  │
    │  │  Computes directional nudges (QP-based)                   │  │
    │  │  Applies full-shift displacements (not half-shift)        │  │
    │  │  Rebuilds entity_graph and spatial index                   │  │
    │  │  Rebuilds analytic_routes for DRC correctness              │  │
    │  └───────────────────────────────────────────────────────────┘  │
    │                                                                 │
    └─────────────────────────────────────────────────────────────────┘
```

---

## 3. Subsystem: Global Partition Grid

### 3.1 Core Abstraction

The **Partition Grid** divides the continuous routing space into a uniform grid of coarse **G-cells**. Each G-cell tracks which nets pass through it and reserves **boundary ports** at shared edges where nets cross from one cell to the next.

```
    ┌──────────────┬──────────────┬──────────────┐
    │   G-Cell A   │   G-Cell B   │   G-Cell C   │
    │              │              │              │
    │  [Net VDD]   │──BoundaryPort──[Net VDD]    │
    │              │  (capacity:  │              │
    │              │   max=12)    │              │
    ├──────────────┼──────────────┼──────────────┤
    │   G-Cell D   │   G-Cell E   │   G-Cell F   │
    │              │              │              │
    └──────────────┴──────────────┴──────────────┘
```

### 3.2 Capacity-Aware Boundary Allocation

The critical innovation in v0.1.8 is **capacity enforcement** at boundary interfaces. Each boundary between two adjacent G-cells has a maximum capacity computed from the physical boundary length divided by the track pitch:

```rust
fn boundary_max_capacity(&self, from: GCellId, to: GCellId) -> usize {
    let boundary_length = (boundary.max.x - boundary.min.x)
        .max(boundary.max.y - boundary.min.y);
    (boundary_length / self.track_pitch_nm) as usize
}
```

When `allocate_boundary_port()` is called, it checks the current utilization against this maximum. If the boundary is full, it returns `None`, forcing the router to find an alternative corridor:

```rust
if entry.0 >= entry.1 {
    return None; // Boundary at capacity — caller must find alternative route
}
```

This prevents congestion accumulation that would otherwise require rip-up-and-reroute to resolve.

### 3.3 Boundary Port Relocation

When a boundary port is congested, the partition grid supports localized relocation — shifting the port by $\pm 3 \times \text{track\_pitch}$ along the shared boundary:

```rust
let shift = 3 * self.track_pitch_nm;
// Vertical boundary → shift along Y
// Horizontal boundary → shift along X
```

This provides a bounded search space for congestion relief without global re-planning.

### 3.4 Data Flow

*   **Input:** Board bounding box, cell dimensions, track pitch from PDK profile.
*   **Output:** `PartitionGrid` with registered nets, boundary ports, and capacity counters.

---

## 4. Subsystem: Topological Routing

### 4.1 Core Abstraction

After partition, each net is routed through the G-cell grid using the **TopologicalRouter** (Manhattan A\*) or the **SDF-accelerated A\*** when a Signed Distance Field generator is available. The router operates on a spatial index built from component obstacles, substrate pours, and previously committed route segments.

### 4.2 Pour-Aware Obstacle Map

The spatial index is built per-route in `build_routing_spatial_index()`. Three categories of obstacles are inserted:

1.  **Component boundaries:** All components except the start/goal of the active route are hard obstacles. Even same-net components that are not the active endpoints are obstacles.
2.  **Substrate layers (pours):** Pours from the entity graph are inserted as obstacles. **Same-net pours are exempt** — a route may pass over its own net's pour. Different-net pours are hard obstacles.
3.  **Committed route segments:** All previously routed segments from different nets are obstacles. Same-net segments are exempt (enabling T-junction tapping). Segments touching the active route's start/goal are exempt (enabling clean docking).

```
    build_routing_spatial_index() obstacle insertion order:
    
    1. Component boundaries    →  (Net ID 0, unconnected)
    2. Substrate pours         →  (respective net IDs)
    3. Committed route segments → (respective net IDs)
    
    Exemptions:
    - Start/goal components of active route → SKIPPED
    - Same-net routes → SKIPPED
    - Routes touching start/goal endpoints → SKIPPED
    - Same-net pours → SKIPPED
```

### 4.3 Profile-Driven Routing Heuristics

All routing cost weights are sourced from the PDK profile's `routing:` block. The compiler fails immediately if any heuristic is missing:

*   `base_cost` — base movement cost per step
*   `via_penalty` — additional cost for via insertion
*   `direction_penalty` — cost for changing routing direction
*   `tight_clearance_penalty` — cost for routing in tight-clearance zones
*   `crosstalk_penalty` — cost for parallel runs near aggressors
*   `impedance_penalty` — cost for impedance mismatches
*   `reference_void_penalty` — cost for routing over reference plane voids

No `Default` implementation exists. Missing values produce a `RoutingError::MissingFabricationConstraints` terminal error.

### 4.4 Data Flow

*   **Input:** `PartitionGrid` with boundary ports, component metadata, substrate layers, net priority map.
*   **Output:** Routed paths registered in the EntityGraph as `TraceSegment` vectors.

---

## 5. Subsystem: Localized Legalization Engine

### 5.1 Core Abstraction

The **Localized Legalization Engine** resolves clearance violations between routed traces without global re-routing. It operates on a continuous coordinate space using convex optimization (QP) for small, local problems.

```
    Clearance Violation Detection:
    
    Segment A (Net VDD)     Segment B (Net GND)
    ┌──────────────┐        ┌──────────────┐
    │              │◄─gap──►│              │
    │   width=250  │  <min  │   width=250  │
    │              │clearance│              │
    └──────────────┘        └──────────────┘
           │                        │
           └──── Violation! ────────┘
                     │
                     ▼
            Legalization Window
            (merged bounding boxes)
                     │
                     ▼
            Compute Nudge (shift violator)
                     │
                     ▼
            Apply Full Displacement
```

### 5.2 Violation Detection

The legalizer detects clearance violations by querying the spatial index for nearby segments and computing the required shift to separate overlapping traces:

```rust
fn required_shift(seg_a: &TraceSegment, seg_b: &TraceSegment, min_clearance: i64) -> i64 {
    // Overlap case: overlap + clearance
    if overlap_x > 0 && overlap_y > 0 {
        overlap_x.max(overlap_y) + min_clearance
    }
    // Gap case: clearance - gap
    else if gap < min_clearance {
        min_clearance - gap
    }
    // Sufficient gap: no shift needed
    else {
        0
    }
}
```

Same-net overlaps are skipped — these represent legal T-junctions or taps.

### 5.3 Legalization Window Construction

For each violation, a legalization window is constructed by:
1.  Taking the overlap bounding box of the two violating segments.
2.  Expanding by `window_margin_nm` (= `min_clearance_nm`).
3.  Unioning with the bounding boxes of both violator and victim.
4.  Expanding again by `window_margin_nm`.
5.  Collecting all segments whose bounding boxes overlap the window.

Overlapping windows are then **merged** using `merge_windows()` — a union-find-style pass that iteratively combines windows whose bounding boxes overlap, deduplicating segment IDs.

### 5.4 Nudge Computation

The nudge direction is determined by the segment orientation and the relative positions of the violator and victim:

```rust
if violator.is_horizontal() {
    // Horizontal trace → nudge perpendicular (Y axis)
    if dir_y >= 0 { (0, -shift) } else { (0, shift) }
} else if violator.is_vertical() {
    // Vertical trace → nudge perpendicular (X axis)
    if dir_x >= 0 { (-shift, 0) } else { (shift, 0) }
} else {
    // Diagonal trace → nudge along perpendicular direction
    let nx = -dir_y; let ny = dir_x;
    // Scale to required shift
}
```

**Critical fix (v0.1.8):** The original implementation used `half_shift = required_shift / 2` and moved only the violator. This left clearance at half the required amount. The corrected implementation applies the **full** `required_shift` to the violator, since only one side moves:

```rust
let shift = violation.required_shift_nm;  // FULL shift, not half
```

### 5.5 Iterative Application

The legalizer runs iteratively (up to `max_iterations`):

```
    for iter in 0..max_iterations:
        violations = detect_violations(segments, net_ids, spatial_index)
        if violations.is_empty() → BREAK (converged)
        
        windows = merge_windows(violations)
        displacements = [compute_nudge(v, segments) for v in windows]
        if displacements.is_empty() → BREAK (stalled)
        
        segments = apply_nudges(segments, displacements)
        rebuild spatial_index from updated segments
```

### 5.6 Material Registry Integration

The legalizer requires the `MaterialRegistry` to look up trace thickness for spatial index construction. If a material has zero thickness (not declared in PDK), the legalizer panics with a fail-fast error:

```rust
let thickness_nm = material_registry
    .get_material(seg.material_id)
    .map(|m| m.thickness_nm)
    .unwrap_or_else(|| panic!(
        "FATAL: Material id={} has zero thickness — must be declared in PDK",
        seg.material_id
    ));
```

### 5.7 Data Flow

*   **Input:** All routed `TraceSegment`s, their `NetId`s, the `MaterialRegistry`, and a `DynamicSpatialIndex`.
*   **Output:** Legalized segments with clearance violations resolved.

---

## 6. Subsystem: Via Sliding

### 6.1 Core Abstraction

When the Legalizer nudges traces apart, vias that belong to the displaced nets must also move to maintain connectivity. The **Via Sliding** mechanism computes the average displacement of all segments belonging to a net and applies the same displacement to all vias of that net.

### 6.2 Displacement Computation

After legalization, the per-net average displacement is computed:

```rust
// For each net, accumulate center-point deltas
for (idx, seg) in legalized_segments.iter().enumerate() {
    let net_id = legalized_net_ids[idx];
    let new_cx = (seg.start.x + seg.end.x) / 2;
    let new_cy = (seg.start.y + seg.end.y) / 2;
    let entry = net_displacements.entry(net_id).or_insert((0, 0));
    entry.0 += new_cx - orig_cx;
    entry.1 += new_cy - orig_cy;
    *net_counts.entry(net_id).or_insert(0) += 1;
}

// Average
for (net_id, count) in &net_counts {
    disp.0 /= count as i64;
    disp.1 /= count as i64;
}
```

### 6.3 Via Position Update

All vias belonging to a displaced net are shifted by the average displacement:

```rust
for via in &mut self.vias {
    if let Some(&(dx, dy)) = net_displacements.get(&via.net_id) {
        if dx != 0 || dy != 0 {
            via.position.0 += dx;
            via.position.1 += dy;
        }
    }
}
```

This preserves the relative positioning of vias within their net's routing topology while adapting to the legalization nudge.

### 6.4 Data Flow

*   **Input:** Original and legalized segment positions, via positions.
*   **Output:** Updated via positions matching legalized trace geometry.

---

## 7. Subsystem: Constraint-Aware Compaction Engine

### 7.1 Core Abstraction

After legalization resolves clearance violations, the **Compactor** slides adjacent parallel traces closer together to minimize layout area, subject to signal integrity constraints. It caps movement at impedance, crosstalk, and clearance limits.

### 7.2 Signal Integrity Constraints

Each net can declare signal integrity constraints:

```rust
pub struct SignalConstraints {
    pub net_id: NetId,
    pub target_impedance_ohms: Option<f64>,
    pub max_crosstalk: Option<f64>,
    pub max_parallel_run_nm: i64,
    pub min_spacing_impedance_nm: i64,
    pub min_spacing_crosstalk_nm: i64,
}
```

The minimum spacing between two parallel traces is computed as:

```
    min_spacing = max(
        default_clearance,
        impedance_spacing,          // if either net has impedance target
        crosstalk_spacing * scale   // scales with parallel run length
    )
```

### 7.3 Net-Id Correct Lookup (v0.1.8 Fix)

The original `compact()` method always looked up constraints for `NetId(0)`, ignoring the actual net identity. The v0.1.8 fix changes the signature to accept `net_ids: &[NetId]` and looks up constraints per-net:

```rust
pub fn compact(
    &self,
    segments: &[TraceSegment],
    net_ids: &[NetId],
    constraints: &FxHashMap<NetId, SignalConstraints>,
) -> Vec<CompactionMove> {
    // ...
    let net_a = net_ids.get(i).copied().unwrap_or(NetId(0));
    let net_b = net_ids.get(j).copied().unwrap_or(NetId(0));
    let constraints_a = constraints.get(&net_a);
    let constraints_b = constraints.get(&net_b);
    // ...
}
```

### 7.4 Compaction Moves

When the current spacing between two parallel traces exceeds the minimum required spacing, the compactor shifts one trace by `(current_spacing - min_spacing) / 2` toward the other:

```
    Before compaction:
    
    Trace A ─────────────────────
              ↕ current_spacing = 800nm
    Trace B ─────────────────────
    
    min_spacing = 300nm (from signal integrity constraints)
    shift = (800 - 300) / 2 = 250nm
    
    After compaction:
    
    Trace A ─────────────────────
              ↕ 300nm (= min_spacing)
    Trace B ─────────────────────
```

### 7.5 Data Flow

*   **Input:** Routed `TraceSegment`s, `NetId`s, `SignalConstraints` per net.
*   **Output:** `Vec<CompactionMove>` (segment_id, dx, dy) applied to segments.

---

## 8. Subsystem: Analytic Route Rebuild

### 8.1 Core Abstraction

DRC checks operate on `space.analytic_routes`, which are generated during routing **before** legalization. After legalization nudges traces apart, the analytic routes are stale and must be rebuilt from the legalized entity_graph.

### 8.2 Rebuild Process

After Stage 4 legalization:

```rust
// 1. Clear all routes from entity_graph
for net_id in net_ids_to_clear {
    entity_graph.clear_routes_for_net(net_id);
}

// 2. Re-register legalized segments
for (idx, seg) in legalized_segments.iter().enumerate() {
    entity_graph.register_trace_segments(net_id, vec![seg.clone()]);
}

// 3. Rebuild spatial index
entity_graph.rebuild_spatial_index(&material_registry);

// 4. Rebuild analytic_routes from legalized entity_graph
let all_routes = entity_graph.get_all_routes();
for (net_id, segments) in all_routes {
    let line_segments: Vec<LineSegment> = segments
        .iter()
        .map(|seg| LineSegment { start: seg.start, end: seg.end })
        .collect();
    new_analytic_routes.push(AnalyticTrace { segments: line_segments, ... });
}
```

This ensures DRC sees the post-legalization geometry, not the pre-legalization geometry.

### 8.3 Data Flow

*   **Input:** Legalized entity_graph routes.
*   **Output:** Rebuilt `analytic_routes` for DRC verification.

---

## 9. Subsystem: Profile-Driven Net Priority

### 9.1 Core Abstraction

Net priority determines routing order — higher priority nets are routed first. In v0.1.8, net priority is **entirely profile-driven**. The compiler does NOT guess priority from net names.

### 9.2 ZERO-MAGIC Enforcement

The legacy `from_net_name()` function was hardcoded magic — it always returned `LowSpeed` regardless of input. This violates the ZERO-MAGIC principle. It was replaced with a flat lookup table:

```rust
pub type NetPriorityMap = FxHashMap<NetId, u8>;

pub fn get_net_priority(net_id: NetId, priorities: &NetPriorityMap) -> u8 {
    priorities.get(&net_id).copied().unwrap_or(0)
}
```

### 9.3 Profile Declaration

Priorities are declared in the `.hw` file using flat syntax (the parser does not support nested blocks):

```
routing {
    net_priority_VDD: 5
    net_priority_GND: 5
    net_priority_CLK: 3
    base_cost: 1.0
    via_penalty: 5.0
}
```

The parser matches fields starting with `net_priority_` and extracts the net name suffix and priority level:

```rust
if field_name.name.starts_with("net_priority_") {
    let net_name = field_name.name.strip_prefix("net_priority_").unwrap();
    let priority = parse_u8_value(value)?;
    constraints.net_priorities.insert(net_name.to_string(), priority);
}
```

### 9.4 Data Flow

*   **Input:** Profile declaration (`routing.net_priority_*` fields).
*   **Output:** `NetPriorityMap` used by `route_all_nets_with_priority()` to sort nets by priority before routing.

---

## 10. Subsystem: Convex Solvers (QP and DAG)

### 10.1 QP Solver (Quadratic Programming)

The QP solver is used for macro-scale legalization problems (N ≥ 20 variables). It minimizes total trace displacement while maintaining clearance constraints:

```
    Minimize: Σ (displacement_i)²
    Subject to: dist(variable_i, variable_j) ≥ min_clearance
                variable_i ∈ window_bounds
```

The solver uses iterative gradient descent:

```rust
for iteration in 0..max_iterations {
    for &(i, j, min_dist) in clearance_constraints {
        let overlap = min_dist - current_distance;
        if overlap > 0 {
            // Push variables apart along the separation axis
            positions[i] -= push;
            positions[j] += push;
        }
    }
    // Clamp to window bounds
    // Check convergence
}
```

### 10.2 DAG Solver (1D Compaction)

The DAG solver is used for micro-scale problems (N < 20) where QP overhead is not justified. It solves 1D compaction using longest-path on a constraint DAG:

```
    Constraints: from_position + min_gap ≤ to_position
    
    DAG:  A ──(gap=5)──► B ──(gap=5)──► C
    
    Longest path:
    A = 0
    B = max(B, A + 5) = 5
    C = max(C, B + 5) = 10
```

The solver uses Kahn's algorithm for topological sort, then a forward pass for longest-path computation.

### 10.3 Solver Selection

The routing pipeline selects the solver based on problem size:

```
    N ≥ 20 variables → QP Solver (clarabel or iterative gradient)
    N < 20 variables → DAG Solver (longest-path compaction)
```

---

## 11. v0.1.8 Architectural Rules

All subsystems enforce the following v0.1.8 architectural rules:

| Rule | Implementation |
|------|---------------|
| **No hardcoded fallbacks** | All routing heuristics from PDK profile; `RoutingHeuristics` has no `Default` impl |
| **Fail-Fast** | Missing profile values → `IrError::MissingPdkBridge` or `RoutingError::MissingFabricationConstraints` |
| **PDK Violation = Terminal Error** | `Legalizer` panics on zero-thickness materials; router fails on missing fabrication constraints |
| **ZERO-MAGIC** | `NetPriority` removed; `from_net_name()` deleted; priorities declared in profile |
| **No Rip-Up** | `ripup.rs`, `negotiated_congestion.rs`, `incremental_dag.rs` all deleted; `NoPathFound` states "terminal error — no rip-up or retry mechanism" |
| **Convexity** | Legalizer uses QP/DAG — convex optimization; topological conflicts resolved before legalization |
| **Analytic Route Rebuild** | DRC checks `space.analytic_routes`, not `entity_graph.routed_segments`; rebuilt after legalization |
| **Pour-Aware Routing** | Substrate layers inserted into obstacle index; same-net pours exempt |

---

## 12. File Reference

| File | Purpose |
|------|---------|
| `legalizer.rs` | Core legalization engine — violation detection, window creation, nudge computation, iterative application |
| `partition.rs` | Partition grid — G-cell management, capacity-aware boundary port allocation, boundary relocation |
| `compaction.rs` | Constraint-aware compactor — parallel trace spacing, signal integrity limits, net-Id correct lookup |
| `priority.rs` | Net priority — profile-driven `NetPriorityMap`, `get_net_priority()` |
| `solvers/qp_solver.rs` | QP solver — iterative gradient descent for macro-scale legalization |
| `solvers/dag_solver.rs` | DAG solver — longest-path 1D compaction for micro-scale problems |
| `router/routing_methods/single_net.rs` | Per-net routing — `build_routing_spatial_index()`, `route_net()`, `legalize_local_window()` |
| `router/routing_methods/batch_routing.rs` | Multi-net routing — `route_all_nets_with_priority()` |
| `router/mod.rs` | Module declarations — removed ripup, negotiated_congestion, incremental_dag |
| `types.rs` | Error types — updated `NoPathFound` terminal error message |
| `global.rs` | Pipeline integration — Stage 4 legalization, analytic route rebuild |
