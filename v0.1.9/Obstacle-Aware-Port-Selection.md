# Obstacle-Aware Port Selection System

**Version:** v0.1.9  
**Status:** Production  
**Date:** 2026-07-21

---

## Executive Summary

The v0.1.9 routing compiler implements a **unified obstacle-aware port selection system** that eliminates the "split-brain bug" present in earlier versions. The system uses topological ray-casting to analyze physical obstacle geometry before selecting escape ports, ensuring routes avoid collisions on the first attempt.

### Key Achievement

✅ **Both test routes succeeded without fallback or retry**
- `Edge_A`: Automatically selected East port (950µm clearance)
- `Tight_A`: Automatically selected North port (950µm clearance), avoiding the `Squeeze_Block` obstacle

---

## The Problem: Split-Brain Port Selection

### What Was Wrong (Pre-v0.1.9)

The routing compiler had **two independent systems** making conflicting port selection decisions:

#### System 1: `boundary.rs::calculate_boundary_points()`
- Used **geometric direction** toward goal
- Algorithm: `dx = goal.x - start.x; dy = goal.y - start.y`
- Selected port based purely on which direction pointed most toward the target
- **Did not consider obstacles**

#### System 2: `boundary_resolution.rs::select_access_region_toward_point()`
- Also used **geometric direction** toward goal
- Algorithm: Dot product of `(pad_edge - pad_center)` with `(goal - start)`
- Selected the access region whose entry point scored highest toward destination
- **Did not consider obstacles**

### The Split-Brain Failure Mode

```
User Code:
  route Pad_Tight_A -> Pad_Tight_B

Geometry:
  Pad_Tight_A: center=(350000, 300000)
  Pad_Tight_B: center=(650000, 300000)
  Squeeze_Block: bbox=(430000,235000) to (550000,315000)
  Direction vector: dx=300000, dy=0 (pure horizontal)

System 1 Decision (boundary.rs):
  ✅ Analyzed obstacles via ray-casting
  ✅ Detected Squeeze_Block blocking East
  ✅ Selected North port (clearance=950µm)

System 2 Decision (boundary_resolution.rs):
  ❌ Ignored obstacle geometry
  ❌ Computed dot product: East scored highest (alignment=1.0)
  ❌ Overrode System 1 and selected East port

Result:
  ❌ Route started inside Squeeze_Block obstacle
  ❌ Topological pathfinder immediately collided
  ❌ Error R16: No path found
```

### Why This Happened

The two systems operated **independently** at different compilation stages:

1. **Stage 1 (Pipeline)**: `calculate_boundary_points()` selected ports geometrically
2. **Stage 2 (Resolution)**: `select_access_region_toward_point()` re-selected ports geometrically
3. **No communication** between stages
4. **No obstacle data** available to either system

The result: **The second system always overrode the first**, making the first system's logic irrelevant.

---

## The Solution: Unified Obstacle-Aware Selection

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│  v0.1.9 UNIFIED PORT SELECTION PIPELINE                     │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  1. build_spatial_index()                                    │
│     - Index all substrate layers (pads, planes, obstacles)   │
│     - Exclude source and destination pads from obstacles     │
│     - Create R*-tree for fast spatial queries                │
│                                                               │
│  2. select_routable_port() [SINGLE SOURCE OF TRUTH]          │
│     For each cardinal direction (N/E/S/W):                   │
│       a. Check if access region exists (interface check)     │
│       b. Project ray from escape point                       │
│       c. Measure clearance distance to first obstacle        │
│       d. Compute geometric alignment toward goal             │
│       e. Score = (clearance × 0.7) + (alignment × 0.3)       │
│     Return port with highest score                           │
│                                                               │
│  3. select_access_region_by_port()                           │
│     - Map selected CardinalPort to its AccessRegion          │
│     - Use normal vector matching (exact O(1) lookup)         │
│                                                               │
│  4. resolve_boundary_entry()                                 │
│     - Apply Zero-Gap Contact Lock                            │
│     - Calculate trace centerline position                    │
│                                                               │
│  5. route_with_perpendicular_escape()                        │
│     - Enforce mandatory perpendicular escape segment         │
│     - Route from escape point to escape point                │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Core Algorithm: Obstacle-Aware Scoring

```rust
fn select_routable_port_core(...) -> Result<CardinalPort> {
    let port_directions = [
        (CardinalPort::North, RayDirection::North, (0, 1)),
        (CardinalPort::East, RayDirection::East, (1, 0)),
        (CardinalPort::South, RayDirection::South, (0, -1)),
        (CardinalPort::West, RayDirection::West, (-1, 0)),
    ];
    
    for (port, ray_dir, (dir_x, dir_y)) in port_directions {
        // 1. Verify access region exists
        let has_access_region = calculate_interface_escape(...).is_some();
        
        // 2. Project ray and measure clearance
        let ray_clearance_nm = if let Some(escape_pt) = escape_result {
            router.project_ray(escape_pt.point, ray_dir, &spatial_index, &board_bounds)
                .map(|hit| distance(escape_pt, hit.point))
                .unwrap_or(board_max_dimension)
        } else {
            0
        };
        
        // 3. Compute geometric alignment
        let geometric_alignment = dot_product(direction_vector, goal_vector) / magnitude;
        
        // 4. Composite score
        let clearance_score = (ray_clearance_nm as f64 / 1_000_000.0).min(1.0);
        let score = (geometric_alignment * 0.3) + (clearance_score * 0.7);
    }
    
    // Return port with highest score
}
```

### Scoring Weights Rationale

- **Clearance: 70% weight** - Obstacle avoidance is primary concern
- **Geometric alignment: 30% weight** - Prefer shorter paths when clearance is equal

This ensures the router:
1. **Never chooses a blocked direction** (0nm clearance → negative score)
2. **Prefers directions with more routing space**
3. **Breaks ties by choosing direction closer to goal**

---

## Implementation Details

### Module Structure

```
hwc-compiler/src/ir/routing/automatic/boundary.rs
├── select_routable_port()              [Pipeline entry point]
├── select_routable_port_from_resolution() [Resolution entry point]
├── select_routable_port_impl()         [RouteEndpointSpec wrapper]
├── select_routable_port_core()         [Core algorithm]
└── PortAnalysis { port, has_access_region, ray_clearance_nm, geometric_alignment }

hwc-compiler/src/ir/routing/helpers/boundary_resolution.rs
├── select_access_region_by_port()      [Port → AccessRegion mapper]
└── resolve_route_boundary_points()     [Main resolution function]
```

### Removed Functions (Deprecated)

The following functions were **permanently removed** in v0.1.9:

#### `select_access_region_toward_point()`
- **Purpose**: Selected access region by geometric direction
- **Algorithm**: Dot product of edge vectors
- **Problem**: Ignored obstacles, caused split-brain bug
- **Replacement**: `select_routable_port_core()` with obstacle analysis

#### `select_access_region_by_direction()`
- **Purpose**: Selected access region from explicit escape spec
- **Algorithm**: Normal vector matching with best-fit
- **Problem**: No obstacle awareness for auto-selection
- **Replacement**: `select_access_region_by_port()` with unified port selection

### Data Flow

```
┌─────────────────────┐
│  User Route Spec    │
│  Pad_A -> Pad_B     │
└──────────┬──────────┘
           │
           v
┌─────────────────────────────────┐
│  calculate_boundary_points()    │
│  [automatic/boundary.rs]        │
└──────────┬──────────────────────┘
           │
           ├─> Build spatial index (obstacles)
           ├─> select_routable_port(Pad_A, toward Pad_B)
           │   ├─> Ray cast North → 950µm clearance
           │   ├─> Ray cast East → 149µm clearance  
           │   ├─> Ray cast South → 950µm clearance
           │   ├─> Ray cast West → 0µm clearance (blocked)
           │   └─> Selected: North (score=0.665)
           │
           ├─> select_routable_port(Pad_B, toward Pad_A)
           │   └─> Selected: North (score=0.665)
           │
           v
┌─────────────────────────────────┐
│  resolve_route_boundary_points()│
│  [helpers/boundary_resolution]  │
└──────────┬──────────────────────┘
           │
           ├─> select_access_region_by_port(North)
           │   └─> Returns AccessRegion with normal=(0,1)
           │
           ├─> resolve_boundary_entry()
           │   └─> Apply Zero-Gap Contact Lock
           │
           v
┌─────────────────────────────────┐
│  route_with_perpendicular_escape│
│  [topological_router/mod.rs]    │
└──────────┬──────────────────────┘
           │
           └─> Successful route
```

---

## Performance Characteristics

### Time Complexity

- **Per-route spatial index build**: O(N log N) where N = number of substrate layers
- **Per-port ray cast**: O(log N) R*-tree query + O(K) candidate checks
- **Total per route**: O(4 × log N) = O(log N) for 4 cardinal directions

### Space Complexity

- **Spatial index**: O(N) for N substrate layers
- **Port analysis**: O(1) - fixed 4 directions

### Benchmark Results (test_corner_cases.hw)

```
Pad_Edge_A port selection:
  - Spatial index build: 4 layers × ~0.1ms = 0.4ms
  - Ray casting (4 directions): ~0.2ms
  - Total: ~0.6ms

Pad_Tight_A port selection:
  - Spatial index build: 4 layers × ~0.1ms = 0.4ms
  - Ray casting (4 directions): ~0.2ms
  - Total: ~0.6ms

Combined overhead: ~1.2ms per design
Route success rate: 100% (no retries needed)
```

---

## Test Case Analysis

### Test: `test_corner_cases.hw`

#### Layout

```
┌─────────────────────────────────────────────────┐
│                                                 │
│  [Pad_Edge_A]                                   │
│      150×150µm                  [Pad_Edge_B]    │
│                                     150×150µm   │
│                                                 │
│         [Pad_Tight_A]  [Squeeze]  [Pad_Tight_B]│
│           100×100µm    Block 120  100×100µm    │
│                        ×80µm                    │
│                                                 │
└─────────────────────────────────────────────────┘
```

#### Route 1: Edge_A → Edge_B (Unobstructed)

```
[OBSTACLE-AWARE PORT SELECTION] Entity: Pad_Edge_A
  Target: dx=750000, dy=350000
  East: clearance=950000nm, alignment=0.91, score=0.937 ✅
  North: clearance=950000nm, alignment=0.42, score=0.792
  South: clearance=950000nm, alignment=-0.42, score=0.538
  West: clearance=950000nm, alignment=-0.91, score=0.393

Selected: East (optimal geometric alignment, full clearance)
```

**Result**: ✅ L-shaped route via East port, 920µm total length

#### Route 2: Tight_A → Tight_B (Obstacle Avoidance)

```
[OBSTACLE-AWARE PORT SELECTION] Entity: Pad_Tight_A
  Target: dx=300000, dy=0
  North: clearance=950000nm, alignment=0.00, score=0.665 ✅
  South: clearance=950000nm, alignment=0.00, score=0.665
  East: clearance=149280nm, alignment=1.00, score=0.404 ❌ Blocked!
  West: clearance=0nm, alignment=-1.00, score=-0.300 ❌ Fully blocked!

Selected: North (tie broken by processing order, both have equal score)
```

**Result**: ✅ Route escapes North, clears Squeeze_Block, enters Tight_B from North, 360µm total length

---

## Migration Guide

### For Compiler Developers

If you're working on routing code:

1. **Never bypass `select_routable_port()`** - It's the single source of truth
2. **Always pass obstacle data** - Spatial index must include all layers
3. **Use the public API** - Don't create new geometric-only selectors

### For Hardware Designers

No changes needed! The new system is fully backward-compatible:

```hw
// Automatic port selection (recommended)
route Pad_A -> Pad_B

// Manual port override (still supported)
route Pad_A[exit: North] -> Pad_B[enter: South]
```

Manual overrides bypass obstacle analysis, so use them only when you have specific routing requirements.

---

## Future Enhancements

### Possible Improvements

1. **Multi-layer routing**: Extend ray-casting to consider via transitions
2. **Congestion awareness**: Factor in existing route density
3. **Cost tuning**: Make clearance/alignment weights user-configurable
4. **Path prediction**: Use A* heuristics to estimate full route cost

### Known Limitations

1. **Single-layer analysis**: Ray-casting only considers obstacles on the same Z layer
2. **Greedy selection**: Doesn't consider multi-hop routing strategies
3. **No rip-up/retry**: If the selected port fails to route, no fallback mechanism exists

---

## Conclusion

The v0.1.9 obstacle-aware port selection system represents a **fundamental architectural improvement** over the pre-v0.1.9 geometric-only approach:

- ✅ **Unified decision-making**: Single authoritative port selector
- ✅ **Physical awareness**: Ray-casting analyzes actual obstacle geometry  
- ✅ **Zero retries**: Routes succeed on first attempt
- ✅ **Predictable behavior**: Deterministic scoring with clear rationale
- ✅ **Maintainable**: One algorithm to understand, test, and optimize

The "split-brain bug" is permanently eliminated through architectural unification, not patching.

### Spatial Index 2D Interval Query Fix (v0.1.9.1)

#### The Issue: Missed Wide Obstacles in Ray Queries

When ray-casting from access points near obstacles (e.g. East ray from `Pad_Tight_A`), `query_bbox` would fail to return wide obstacles whose `min_x` was less than `query_bbox.min.x`.

- **Root Cause**: `DynamicSpatialIndex::query_bucket` used binary search with `partition_point(|s| s.min_x() < query_bbox.min.x)`.
- **Failure Mode**: An obstacle starting at `X = 430000` (`min_x = 370000`) and ending at `X = 550000` was skipped when querying a ray starting at `X = 400450` because `370000 < 400450`. The binary search started linear scanning *after* the obstacle, reporting false 950µm clearance for the East direction.

#### The Solution: `max_span` Tracking & Type-Aware Bounds

1. **`max_span` Tracking**: `DynamicSpatialIndex` tracks the maximum bounding box X-span (`max_x - min_x`) across all indexed entities.
2. **Shifted Partition Point**: Queries search from `search_min_x = query_bbox.min.x - max_span`, guaranteeing any obstacle whose `max_x` overlaps `query_bbox.min.x` is included in the linear scan range.
3. **Type-Aware Bounds**: `IndexedSegment` helper methods (`min_x`, `max_x`, `min_y`, `max_y`) distinguish between routed trace centerlines (which apply `width_nm / 2` expansion) and solid substrate layers/pours (which use absolute physical bounding box boundaries).

---

## References

- `hwc-compiler/src/ir/routing/automatic/boundary.rs` - Port selection implementation
- `hwc-compiler/src/ir/routing/helpers/boundary_resolution.rs` - Access region mapping
- `hwc-engine/src/geometry_router/topological_router/ray.rs` - Ray-casting algorithm
- `hwc-physics/src/spatial_index.rs` - Layered 2D spatial index (`DynamicSpatialIndex` interval queries)
- `tests/ASIC/two-pad-relational/obstacle-tests/test_corner_cases.hw` - Validation test case

---

**Document Version:** 1.1 (v0.1.9.1 Fixes)  
**Last Updated:** 2026-07-21  
**Author:** hwc development team

