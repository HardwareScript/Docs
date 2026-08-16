# Hierarchical Legalization Engine

**Version:** v0.2.1
**Status:** Active  
**Implementation:** `hwc-engine/src/geometry_router/legalizer.rs`, `hwc-compiler/src/ir/routing/global/post_process.rs`

## Overview

Post-routing legalization is a critical compiler phase that resolves minor clearance violations by nudging traces apart. This document explains why legalization was temporarily disabled, the problems that caused, how hierarchical legalization restores this capability, and the  engine enhancements for obstacle awareness, Manhattan elbow continuity, spatial index rebuilding, and dynamic via/contact sliding.

## Background: Why Was Legalization Disabled?

### The Original Problem (v0.2.0)

Before the HierarchicalRoutingDatabase refactoring, the legalization engine treated all route segments equally—it would attempt to nudge any segment that violated clearance rules, regardless of its origin.

This created a critical bug in hierarchical designs:

```
Parent Space (CMOS Inverter)
├─ PMOS Instance (pre-routed, tape-out verified)
│  └─ Internal routing: FROZEN
├─ NMOS Instance (pre-routed, tape-out verified)
│  └─ Internal routing: FROZEN
└─ Parent interconnects (new routes connecting PMOS/NMOS)
   └─ Routing: SHOULD BE MUTABLE
```

**The Bug:** When the legalizer detected a clearance violation between a parent interconnect and a child cell's internal routing, it would:

1. Attempt to nudge **both** the parent route and the child cell's internal traces
2. Corrupt the child cell's pre-verified geometry
3. Break LVS (Layout vs. Schematic) checks
4. Cause the QP solver to panic when trying to move fixed pins

### The Temporary Fix

To prevent hierarchical designs from crashing, a hard-skip was added:

```rust
// DISABLED to prevent clearing child routes
eprintln!("[LEGALIZATION] Skipping post-routing legalization (hierarchical design - needs refactor)");
```

### The Cost of Skipping Legalization

Disabling legalization violated the core compiler contract:

- **No Self-Healing:** Minor sub-micron clearance violations (e.g., 20nm spacing error) could not be automatically fixed
- **Brittle Routing:** Any routing imperfection immediately failed DRC, with no recovery mechanism
- **Lost Single-Pass Pipeline:** The architecture was designed to avoid rip-up-and-reroute (RRR) loops by using QP/DAG legalization as Stage 4 of routing

---

## The Solution: Hierarchical Legalization & Obstacle Awareness

### Core Principle: Immutability Boundaries

The solution enforces a clear distinction between **frozen** (immutable) and **mutable** (movable) route segments:

```
┌──────────────────────────────────────────────────────────┐
│           HIERARCHICAL LEGALIZATION ENGINE               │
└──────────────────────────────────────────────────────────┘
         │
         ├─► [Frozen Obstacles (Immutable)]
         │   • Child instance routes (is_frozen = true)
         │   • Substrate pours, pads, & diffusions
         │   • Pre-placed contacts & vias
         │   • DRC Keepout zones
         │
         └─► [Mutable Segments]
             • Parent interconnects (can be nudged)
             • Top-level routes (can be nudged)
```

---

### Implementation & Architectural Upgrades

#### 1. Comprehensive Obstacle Population (`post_process.rs`)

In Stage 2 of post-processing (`run_legalization`), all physical structures are fetched and converted into frozen obstacle segments:

- **Child Routes**: Immutable internal cell routes from `routing_database.get_child_segments_for_legalization()`
- **Substrate Layers**: Pours, pads, bulk taps, diffusions converted via `bbox_to_frozen_segment()`
- **Contacts**: Standard licon/via contacts from `self.space.contacts`
- **Vias**: Pre-placed vertical interconnects from `self.space.vias`
- **Keepout Zones**: Explicit obstacle bounding boxes from `data.obstacle_bboxes`

All frozen obstacles are tagged with `is_frozen = true`.

#### 2. Asymmetric Constraint Formulation (`legalizer.rs`)

The legalizer applies asymmetric push rules based on segment mutability:

| Segment A | Segment B | Behavior |
|-----------|-----------|----------|
| Mutable   | Mutable   | Symmetric nudge (each takes 50% shift) |
| Frozen    | Mutable   | B moves 100%, A stays fixed (0%) |
| Mutable   | Frozen    | A moves 100%, B stays fixed (0%) |
| Frozen    | Frozen    | Skip (pre-verified cell spacing) |

Implementation in `legalizer.rs`:

```rust
match (seg_a.is_frozen, seg_b.is_frozen) {
    (false, false) => (total_shift / 2, total_shift / 2),
    (true, false)  => (0, total_shift),
    (false, true)  => (total_shift, 0),
    (true, true)   => (0, 0),  // Skip - pre-verified cell spacing
}
```

#### 3. Iterative Spatial Index Rebuilding (Flaw 1 Fix)

During multi-iteration legalization, parent routes change position. To prevent stale spatial queries and false 3D cross-layer shorts, `legalize_hierarchical` takes ownership of `DynamicSpatialIndex` and rebuilds it at the end of every iteration using `rebuild_spatial_index`:

```rust
spatial_index = rebuild_spatial_index(&all_segments, &all_net_ids, z_ranges.as_deref());
```

This preserves layer Z-ranges and stackup layer thickness while keeping spatial bounds 100% accurate across all iterations.

#### 4. Manhattan Orthogonal Elbow Continuity (Flaw 2 & Fatal Bug 4 Fix)

To prevent orthogonal right-angle corners (elbows) from snapping open or slanting diagonally when a trace is nudged:
- `propagate_elbow_continuity(&all_segments, &raw_displacements)` calculates endpoint-specific deltas `(dx_start, dy_start, dx_end, dy_end)` for adjacent connecting segments.
- `apply_nudges_with_elbow(&all_segments, &elbow_deltas)` applies these per-endpoint displacements, keeping corner joints perfectly intact.

#### 5. Dynamic Via & Contact Sliding with EntityGraph Synchronization

When parent route segment endpoints shift by `(dx, dy)` during legalization:
- Connected **Vias** (`self.space.vias`) in capture radius automatically slide by `(dx, dy)`.
- Connected **Contacts** (`self.space.contacts`) bounding boxes automatically slide by `(dx, dy)`.
- **EntityGraph** substrate layer bounding boxes for contacts/vias are synchronized (`[ENTITY_GRAPH SYNC]`), ensuring BOM, SPICE, DXF, and 3D GLB exports remain physically aligned.

#### 6. Pipeline Sequencing & Post-Legalization Mitering

Post-routing operates in a strictly ordered pipeline:

```
Topological Route -> Hierarchical Legalization & Sliding -> 45° Miter Pass -> Spatial Commit & Export
```

1. `register_analytic_route_from_segments()` registers un-mitered parent routes.
2. `run_legalization()` performs obstacle-aware QP/nudge legalization and via/contact sliding.
3. `apply_post_legalization_mitering()` applies 45° miter chamfering to the legalized parent traces.
4. `rebuild_analytic_routes()` commits final geometry for SPICE, BOM, and GLB export.

---

## Build Log Evidence

From `simple_resistor_test.log`:

```text
[LEGALIZATION] Running hierarchical post-routing legalization
[OBSTACLE-DEBUG]   10 substrate layers
[OBSTACLE-DEBUG]   13 contacts
[OBSTACLE-DEBUG]   12 vias
[OBSTACLE-DEBUG]   2 keepout zones
[LEGALIZATION] Fetched 2 parent segments, 54 child/obstacle segments (frozen)
[LEGALIZATION] Built spatial index with 56 segments
[HIERARCHICAL LEGALIZER] Starting legalization: 2 parent segments, 54 child segments (frozen)
[HIERARCHICAL LEGALIZER] No parent violations found - legalization complete at iteration 0
[LEGALIZATION] Legalization complete: 2 parent segments processed
[ROUTING DB] Updated net NetId(2) with 1 legalized segments
[ROUTING DB] Updated net NetId(1) with 1 legalized segments
[LEGALIZATION] Parent routes updated in routing database
```

---

## Benefits

1. **Self-Healing Routing:** Automatically resolves sub-micron clearance violations without re-routing
2. **Child Cell Protection:** Pre-verified child cell geometry remains 100% frozen and immutable
3. **Via-Trace Landing Integrity:** Dynamic via/contact sliding prevents via detachment when traces are nudged
4. **No Open Corners:** Manhattan elbow propagation prevents trace disconnects or off-grid diagonal slanting
5. **No False 3D Shorts:** Iterative spatial index rebuilding ensures layer-aware Z-range accuracy
6. **LVS & DRC Guarantee:** Layout remains schematic-accurate and DRC clean

---

## Physical Interpretation

The hierarchical legalizer mirrors physical chip design principles:

- **Standard Cells & IP Blocks** are pre-characterized, tape-out verified blocks with fixed internal routing.
- **Parent Interconnects** are flexible top-level wiring between cells that can be adjusted during physical design.
- **Via Landing Pads** move in lockstep with interconnect endpoints to prevent open circuits.
- **LVS Preservation** requires cell internals to remain bit-exact with their schematic representation.

By treating child routes and pre-placed physical structures as immutable obstacles, the compiler respects physical constraints while maintaining routing flexibility where needed.

---

## References

- **Implementation:** `hwc-engine/src/geometry_router/legalizer.rs`
- **Pipeline Integration:** `hwc-compiler/src/ir/routing/global/post_process.rs`
- **Database Integration:** `hwc-engine/src/routing_database/export.rs`
- **Type Definition:** `hwc-physics/src/geometry.rs` (`TraceSegment`)
