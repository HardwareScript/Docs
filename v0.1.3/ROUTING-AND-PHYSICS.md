# Book 5: The Physics & Routing Engine

**Hardware Script v0.1.3**  
**Target Audience**: Algorithm developers and routing contributors  
**Last Updated**: March 2026

---

## Foundation

This document consolidates the following source materials:

**Source Files**:
- `Docs/v0.1.2/PHYSICS-TO-CONSTRAINTS.md` — Translation of material properties into geometric constraints
- `Docs/v0.1.2/NETLIST-AND-ROUTING-PHILOSOPHY.md` — Division of labor between user and auto-router
- `Docs/v0.1.2/MANHATTAN-ROUTING-STRATEGY.md` — Layer-specific directional routing rules
- `Docs/v0.1.2/ROUTING-GEOMETRY-RULES.md` — 45° turns on planes, 90° vias on Z-axis
- `Docs/v0.1.2/DETERMINISTIC-ROUTING-IMPLEMENTATION.md` — VecDeque-based deterministic pathfinding
- `Docs/v0.1.2/FIRST-PRINCIPLES-IMPLEMENTATION.md` — Discrete Manhattan Geometry with i64 fixed-point math
- `Docs/v0.1.2/HYPER-LEAN-ARCHITECTURE.md` — Rayon parallel validation strategy

**Note**: Material database architecture (3-tiered YAML system, macro-properties approach) is now documented in Book 3 (Ecosystem). This book focuses on how those material properties are used in routing algorithms.

---

## Introduction

This document is the blueprint for Hardware Script's physics validation and routing engine. If you're an algorithm developer who wants to understand or contribute to the routing system, this is your guide.

Hardware Script's routing engine is fundamentally different from traditional EDA auto-routers. Instead of treating physics as post-routing validation, we translate material properties into geometric constraints before routing begins. The router then operates as a constraint-driven pathfinding algorithm on a discrete 3D voxel grid.

**Relationship to the Compiler Pipeline**: This routing engine operates within the broader 5-Layer MLIR Pipeline described in Book 4 (Compiler Internals). Specifically, the routing and physics validation described in this book execute during Layers 3 and 4 of that pipeline:

- **Layer 3 (Physical IR)**: The voxel engine places components and routes nets using the geometry router
- **Layer 4 (Physics IR)**: The physics engine validates the routed design against material constraints

The 3-Phase Routing Pipeline described in this book is a focused sub-pipeline that executes within these two layers.

---

## The Core Philosophy

### Physics as Constraints, Not Simulation

You cannot run a live physics simulation at every routing step — that would be computationally impossible. Instead, Hardware Script translates material properties into mathematical constraints that guide the routing algorithm.

**Book 5 Scope**: This book covers **Automatic Routing** algorithms - the pathfinding, physics constraints, and validation systems used when users do NOT provide explicit waypoints. For **Manual Waypoint Routing** (when users provide explicit `path:` waypoints), see Book 2 (Language Spec), Section 4: Manual Waypoint Routing.

**The 3-Phase Routing Sub-Pipeline**:

This routing pipeline executes within the compiler's Physical IR (Layer 3) and Physics IR (Layer 4) stages. It transforms material properties into geometric constraints, performs pathfinding, and validates the result:

```
Phase 1: Constraint Manager (pre-routing)
    ↓ (translates physics to geometry)
Phase 2: Geometry Router (routing)
    ↓ (pathfinding with constraints)
Phase 3: Design Rule Check (post-routing)
    ↓ (validates final result)
```

This separation allows the router to remain fast and deterministic while still respecting the laws of physics.

**Context in the 5-Layer MLIR Pipeline**:

```
Layer 1: Intent / Behavioral Layer (user writes .hw code)
Layer 2: Logical IR (compiler resolves imports, builds netlist)
Layer 3: Physical IR (hwc-engine) ← Phase 1 & 2 execute here
Layer 4: Physics IR (hwc-physics) ← Phase 3 executes here
Layer 5: Manufacturing Layer (hwc-export generates output files)
```

---

## Discrete Manhattan Geometry with Fixed-Point Math

### Why Integer Math Is Critical

Hardware Script uses **i64 fixed-point math** (nanometers) instead of f32/f64 floating-point for all spatial calculations. This guarantees 100% determinism across all CPU architectures, no rounding errors, perfect Git diffs, and cache-friendly integer operations.

**Data structures**: The `Point3D`, `BoundingBox`, and `TraceSegment` data structures are defined in Book 4 (Compiler Internals) under the Layer 3: Physical IR section. This book focuses on how these structures are used in routing algorithms.

---

## Material Properties and Constraints

### Material Properties for Routing

Hardware Script's routing engine uses material properties from `.hwmat` files to calculate geometric constraints. Book 3 (Ecosystem) defines the `.hwmat` schema. The routing engine ingests these macro-properties to calculate constraints as follows:

---

## Translation 1: Dielectric Breakdown to Clearance Rules

### The Physical Phenomenon

When two copper traces carrying different voltages get too close, electrons can jump through the air or substrate, creating a spark. This is called arcing or dielectric breakdown.

### The Material Properties

Material databases define dielectric strength values that determine minimum clearances (Air: 3.0 kV/mm, FR4: 20 kV/mm). For complete material property definitions, see Book 3 (Ecosystem).

### The Algorithm

**Step 1**: Identify voltage difference between nets

```
Net A: 120V (AC power line)
Net B: 0V (Ground)
Voltage difference: 120V
```

**Step 2**: Calculate minimum clearance

```
Formula: clearance = voltage / dielectric_strength

For air:
  120V / 3000V/mm = 0.04mm minimum air gap

For FR4:
  120V / 20000V/mm = 0.006mm minimum substrate gap
```

**Step 3**: Add safety margin

```
Safety factor: 2× (industry standard)
Required clearance: 0.04mm × 2 = 0.08mm
```

**Step 4**: Enforce in router

The router creates a "forcefield" or "clearance zone" around high-voltage nets. Other nets are forbidden from entering this zone.

### Implementation

```rust
struct Net {
    voltage_mv: i64,  // Millivolts (fixed-point)
    occupied_voxels: Vec<Point3D>,
    clearance_voxels: Vec<Point3D>,
}

fn calculate_clearance_nm(voltage_diff_mv: i64, dielectric_strength_kv_mm: i64) -> i64 {
    let safety_factor = 2;
    
    // Convert voltage to volts: mv / 1000
    // Convert dielectric strength to V/nm: (kv_mm * 1000) / 1_000_000
    // clearance_nm = (voltage_v / dielectric_v_nm) * safety_factor
    
    let voltage_v = voltage_diff_mv / 1000;
    let dielectric_v_nm = (dielectric_strength_kv_mm * 1000) / 1_000_000;
    
    let min_clearance_nm = voltage_v / dielectric_v_nm;
    min_clearance_nm * safety_factor
}

fn expand_clearance_zone(net: &mut Net, clearance_nm: i64, voxel_size_nm: i64) {
    let clearance_voxels = (clearance_nm + voxel_size_nm - 1) / voxel_size_nm;  // Ceiling division
    
    for occupied in &net.occupied_voxels {
        for dz in 0..=clearance_voxels {
            for dx in 0..=clearance_voxels {
                for dy in 0..=clearance_voxels {
                    let clearance_point = Point3D::new(
                        occupied.z + dz * voxel_size_nm,
                        occupied.x + dx * voxel_size_nm,
                        occupied.y + dy * voxel_size_nm,
                    );
                    net.clearance_voxels.push(clearance_point);
                }
            }
        }
    }
}
```

---

## Translation 2: Current Capacity to Trace Width

### The Physical Phenomenon

If you push too much current through a thin trace, it acts like a resistor, generates heat (Joule heating), and can vaporize or melt.

### The Material Properties

Material databases define current capacity and thermal limits (Copper: 35 A/mm² max current density, FR4: 130°C max operating temperature). For complete material property definitions, see Book 3 (Ecosystem).

### The IPC-2221 Formula

For accurate trace width calculations, use the industry-standard IPC-2221 formula:

```
Formula: A = (I / (k × ΔT^0.44))^(1/0.725)

Where:
  A = cross-sectional area (mm²)
  I = current (Amps)
  k = 0.048 for external layers, 0.024 for internal layers
  ΔT = temperature rise (°C)
```

**Example**:
```
Current: 10A
Temperature rise: 10°C (safe limit)
Layer: External

A = (10 / (0.048 × 10^0.44))^(1/0.725)
A = 1.89 mm²

Width = 1.89 mm² / 0.035mm = 54mm

This is a THICK trace!
```

### Implementation

```rust
fn calculate_trace_width_nm(current_ma: i64, temp_rise_c: i64, is_external: bool) -> i64 {
    // IPC-2221 formula adapted for fixed-point math
    // A = (I / (k × ΔT^0.44))^(1/0.725)
    
    let k = if is_external { 48 } else { 24 };  // × 1000 for fixed-point
    let copper_thickness_nm = 35_000;  // 1oz copper = 35 micrometers
    
    // Convert to floating point only for the complex calculation
    let current_a = current_ma as f64 / 1000.0;
    let temp_rise = temp_rise_c as f64;
    let k_f = k as f64 / 1000.0;
    
    let area_mm2 = (current_a / (k_f * temp_rise.powf(0.44))).powf(1.0 / 0.725);
    let width_mm = area_mm2 / 0.035;
    
    // Convert back to nanometers
    (width_mm * 1_000_000.0) as i64
}

fn enforce_trace_width(
    route: &Route,
    required_width_nm: i64,
    voxel_size_nm: i64
) -> Result<(), String> {
    let required_voxels = (required_width_nm + voxel_size_nm - 1) / voxel_size_nm;
    
    if route.available_width < required_voxels {
        return Err(format!(
            "Insufficient space: Need {}mm ({} voxels), have {}mm ({} voxels)",
            required_width_nm as f64 / 1_000_000.0,
            required_voxels,
            (route.available_width * voxel_size_nm) as f64 / 1_000_000.0,
            route.available_width
        ));
    }
    
    Ok(())
}
```

---

## Translation 3: EMI and Crosstalk

### The Physical Phenomenon

Every wire carrying a pulsing electrical signal acts like a tiny radio antenna, broadcasting electromagnetic waves. If another wire runs parallel to it for too long, it acts as a receiving antenna, and signals couple via magnetic induction (crosstalk).

### The Routing Rules

Crosstalk constraints from material database (Maximum parallel length: 10mm, Minimum spacing for high-speed signals: 1.0mm, Preferred crossing angle: 90°). For complete constraint definitions, see Book 3 (Ecosystem).

### The Algorithm

Add a "parallel coupling penalty" to the pathfinding cost function. If the router tries to run Net B parallel to high-speed Net A for more than X voxels, the path cost increases exponentially.

This forces the router to either:
1. Cross paths at 90-degree angles (which cancels magnetic interference)
2. Move traces further apart

### Implementation

```rust
fn calculate_crosstalk_penalty(
    net_a: &Route,
    net_b: &Route,
    max_parallel_length_nm: i64,
    voxel_size_nm: i64
) -> i64 {
    let parallel_length_voxels = calculate_parallel_length(net_a, net_b);
    let parallel_length_nm = parallel_length_voxels * voxel_size_nm;
    
    if parallel_length_nm > max_parallel_length_nm {
        // Exponential penalty (scaled to integer)
        let excess = parallel_length_nm - max_parallel_length_nm;
        let ratio = (excess * 1000) / max_parallel_length_nm;  // Fixed-point ratio
        
        // Approximate exp(ratio/1000) using integer math
        // For small values: exp(x) ≈ 1 + x + x²/2
        let penalty = 1000 + ratio + (ratio * ratio) / 2000;
        penalty
    } else {
        0  // No penalty
    }
}
```

---

## The Netlist: User Defines WHAT, Auto-Router Defines HOW

### The Division of Labor

The automatic routing system should never decide what component connects to what. That is 100% determined by the user (or the logic schematic). The auto-router's only job is to figure out the physical geometry of how to connect them.

**Book 5 Scope**: This section focuses on **Automatic Routing** - the algorithms used when users specify connections without explicit waypoints. For **Manual Waypoint Routing** syntax (when users provide explicit `path:` waypoints), see Book 2 (Language Spec), Section 4: Manual Waypoint Routing.

### Understanding Nets

In electronics, connections are grouped into "Nets" (short for Networks). A net is simply a list of pins that must all share the same electrical signal.

**Two main types**:

**1. Data Nets** (point-to-point or buses)
- Usually 1-to-1 or 1-to-few connections
- Specific signal paths
- Order matters

**2. Power & Ground Nets** (global web)
- Connect to EVERY component
- Order doesn't matter
- Massive fanout

**Automatic vs Manual Routing**:
```hw
# Manual routing (user provides explicit path)
route PowerSource.out to LED.in:
    path:
        - [10, 10, 1]
        - [50, 10, 1]

# Automatic routing (compiler calculates path)
route PowerSource.out to LED.in
# No path: section - auto-router takes over
```

For complete manual waypoint routing syntax and examples, see Book 2 (Language Spec), Section 4: Manual Waypoint Routing.

### The Copper Plane Strategy

For power and ground nets, don't route individual wires. Instead, dedicate entire Z-layers as solid copper planes.

**Layer strategy**:
```
Layer 1: X-axis routing (horizontal traces)
Layer 2: Solid copper sheet (Ground plane)
Layer 3: Solid copper sheet (Power plane)
Layer 4: Y-axis routing (vertical traces)
```

**How components connect to planes**:

If a component needs ground, the router doesn't draw a line to the battery. It just drops a single vertical via straight down into the ground plane.

```
Camera @ [1, 50, 50]
Camera.GND needs ground

Algorithm:
  1. Drill via from [1, 50, 50] to [2, 50, 50]
  2. Layer 2 is solid copper (Ground plane)
  3. Ground plane connects to Battery.Minus
  4. Done. Camera is grounded.
```

### When the Auto-Router Should Stop

The auto-router should be a "slave" to the user's constraints. It should stop and ask for intervention when:

**1. Trapped scenario** — All paths blocked, no valid route exists

**2. Critical high-speed signals** — User wants manual control for timing-critical paths

**3. Waypoints needed** — User should provide manual waypoints to guide the router

For complete manual waypoint routing syntax and examples, see Book 2 (Language Spec), Section 4: Manual Waypoint Routing.

---

## Manhattan Routing Strategy (Automatic Routing Only)

**Book 5 Scope**: This section describes the **Automatic Routing** pathfinding algorithm used when users do NOT provide explicit waypoints. For **Manual Waypoint Routing** (when users provide explicit `path:` waypoints using Bresenham interpolation), see Book 2 (Language Spec), Section 4: Manual Waypoint Routing.

### Manual vs Automatic Routing

Hardware Script supports two routing modes:

**1. Manual Waypoint Routing (User-Defined Paths)**:
- User provides explicit waypoints in `path:` section
- Compiler uses Bresenham's line algorithm to draw straight lines between waypoints
- No pathfinding required - user has complete control
- Covered in Book 2 (Language Spec), Section 4: Manual Waypoint Routing

**2. Automatic Routing (Auto-Router Pathfinding)**:
- User specifies only source and destination (no `path:` section)
- Compiler uses Manhattan routing with A* pathfinding
- Respects layer-specific directions (North/South or East/West)
- Follows 45° turn rules and 90° via rules
- Applies physics constraints (clearance, trace width)
- Covered in this book (Book 5)

**Example Syntax Comparison**:
```hw
# Manual Waypoint Routing (Book 2 scope)
route A to B:
    path:
        - [10, 10, 1]
        - [20, 15, 1]

# Automatic Routing (Book 5 scope)
route A to B
# No path: section - auto-router calculates path
```

The following sections describe the automatic routing algorithm used when no waypoints are provided.

### The Three Fundamental Rules

**Rule 1: Straight lines are best**
- Shortest distance
- Least resistance
- Cleanest signal
- Fastest algorithm

**Rule 2: 45° corners for turns**
- Prevents acid traps during manufacturing
- Reduces signal reflection
- Industry standard

**Rule 3: Layer-specific directions**
- Prevents algorithm from trapping itself
- Guarantees solution exists (if physically possible)
- Simplifies pathfinding logic

### What Is Manhattan Routing?

Manhattan Routing is named after the street grid of Manhattan, New York, where streets run either North-South or East-West, never diagonally.

In PCB design:
- Each layer is restricted to one primary direction
- Turns are accomplished by changing layers (vias)
- The algorithm never gets trapped

### Layer Assignment Strategy

**Standard 4-layer board**:
```
Layer 1 (Top):    North/South routing only
Layer 2 (Inner):  Solid Ground Plane (no routing)
Layer 3 (Inner):  Solid Power Plane (no routing)
Layer 4 (Bottom): East/West routing only
```

**Simple 2-layer board**:
```
Layer 1 (Top):    North/South routing only
Layer 2 (Bottom): East/West routing only
```

### Why This Works

By restricting each layer to one direction:
- The algorithm can always find a path if one exists
- No self-blocking scenarios
- Pathfinding becomes O(n) instead of O(n²)

### How Manhattan Routing Handles Turns

**Traditional approach** (single layer):
- Draw 45° turns on the same layer
- Can create complex routing problems

**Manhattan approach** (multi-layer):
- Change layers instead of turning
- Via down/up to switch directions

**Example**:
```
Layer 1 (North/South only):
    │
    │  (traveling North)
    ●  (via down to Layer 2)

Layer 2 (East/West only):
    ●─────→ (traveling East)
```

### Implementation

```rust
enum LayerDirection {
    NorthSouth,  // Y-axis only
    EastWest,    // X-axis only
    Any,         // Power/Ground planes or unrestricted
}

fn is_valid_move(
    from: (usize, usize, usize),
    to: (usize, usize, usize),
    layer_direction: LayerDirection
) -> bool {
    let (z1, x1, y1) = from;
    let (z2, x2, y2) = to;
    
    // Via (Z-axis change) - always allowed
    if z1 != z2 {
        // Must be straight down (X,Y unchanged)
        return x1 == x2 && y1 == y2;
    }
    
    // Same layer movement
    match layer_direction {
        LayerDirection::NorthSouth => {
            // Only Y can change (North/South)
            x1 == x2 && y1 != y2
        },
        LayerDirection::EastWest => {
            // Only X can change (East/West)
            x1 != x2 && y1 == y2
        },
        LayerDirection::Any => {
            // Any direction allowed
            true
        }
    }
}
```

---

## Routing Geometry Rules

### The Critical Distinction: X/Y Plane vs. Z-Axis

You must separate the X/Y plane (moving flat along a layer) from the Z-axis (moving up and down between layers). They are manufactured using completely different physical processes.

### Rule 1: The X/Y Plane (Flat Routing) = Etching

**Manufacturing process**:
1. Start with flat copper sheet on fiberglass
2. Apply photoresist
3. Shine UV light through mask
4. Wash away exposed photoresist
5. Acid bath eats away unprotected copper
6. Result: Copper traces remain

**Why 90° corners are bad**:

**Problem 1: Acid traps** — Acid pools at sharp corners, creating rounded edges and weakened traces

**Problem 2: Signal reflection** — High-speed signals bounce back at sharp corners, causing EMI and data errors

**The rule**: On the X/Y plane, turns must be 45°

**Good (45° turns)**:
```
    ╱────
   ╱
──╱

Smooth transition, no acid traps, minimal signal reflection
```

**Bad (90° turns)**:
```
    ┌────
    │
────┘

Sharp corner, acid pools, signal reflects
```

### Rule 2: The Z-Axis (Moving Up/Down) = Drilling

**Manufacturing process**:
1. Stack all layers of PCB
2. Use CNC drill to bore vertical holes
3. Electroplate hole walls with copper
4. Result: Conductive tube (via) connecting layers

**Why 90° is required**:

**Reason 1: Physical drilling constraints** — Drills go straight down. Standard PCB manufacturers do not drill holes at angles.

**Reason 2: No acid trap problem** — A via is a vertical cylinder, not a flat trace, so etching concerns don't apply.

**The rule**: On the Z-axis, transitions must be 90° straight down

**Correct (90° via)**:
```
Layer 1: ──●──  (trace on top layer)
           │
           │    (via drilled straight down)
           │
Layer 2: ──●──  (trace on bottom layer)
```

**Impossible (45° via)**:
```
Layer 1: ──●
           ╲
            ╲   (cannot drill at angle)
             ╲
Layer 2:     ●──

This is not manufacturable!
```

### Implementation Rules

**Moving North/South/East/West (X/Y plane)**:
- Preferred: Straight lines
- If you must turn: Use 45° angles
- Avoid: 90° turns on the same layer

**Moving Up/Down (Z-axis)**:
- Always: Drop straight down at exactly 90°
- Never: Change X or Y when changing Z

**Example valid via**:
```rust
[1, 20, 15] → [2, 20, 15]  // Via down (X,Y unchanged)
```

**Example invalid via**:
```rust
[1, 20, 15] → [2, 21, 16]  // ❌ ERROR: X and Y changed!
```

---

## Deterministic Routing Implementation

### The Critical Problem

In Rust, standard `HashMap` and `HashSet` iterate in randomized order for security reasons. If your routing algorithm loops over a `HashSet` to decide where to flow next, the route will change every time you compile.

**Same input, different output every time!**

This breaks the fundamental promise of Hardware Script: deterministic compilation.

### The Solution: VecDeque

**VecDeque** is a First-In-First-Out (FIFO) queue that guarantees:
- Elements are processed in the order they were added
- Same input always produces same output
- Perfect for breadth-first search (BFS) algorithms

### Implementation

```rust
use std::collections::{VecDeque, HashSet};

fn route_deterministic(
    start: Point3D,
    end: Point3D,
    grid: &Grid
) -> Option<Vec<Point3D>> {
    // VecDeque for processing order (deterministic)
    let mut frontier = VecDeque::new();
    frontier.push_back(start);
    
    // HashSet for visited tracking (order doesn't matter here)
    let mut visited = HashSet::new();
    visited.insert(start);
    
    // HashMap for path reconstruction
    let mut came_from = HashMap::new();
    
    while let Some(current) = frontier.pop_front() {  // FIFO order!
        if current == end {
            return Some(reconstruct_path(came_from, start, end));
        }
        
        // Get neighbors in STABLE order
        let neighbors = get_neighbors_stable(current, grid);
        
        for neighbor in neighbors {
            if !visited.contains(&neighbor) {
                visited.insert(neighbor);
                came_from.insert(neighbor, current);
                frontier.push_back(neighbor);  // Add to back of queue
            }
        }
    }
    
    None  // No path found
}
```

### Ensuring Neighbor Order Stability

Even with VecDeque, you must generate neighbors in a fixed, predictable order.

```rust
fn get_neighbors_stable(
    cell: Point3D,
    grid: &Grid
) -> Vec<Point3D> {
    let mut neighbors = Vec::new();
    
    // FIXED ORDER: North, South, East, West, Up, Down
    
    // North (Y+1)
    if cell.y + grid.voxel_size_nm < grid.height_nm {
        neighbors.push(cell.move_direction(Direction::North, grid.voxel_size_nm));
    }
    
    // South (Y-1)
    if cell.y > 0 {
        neighbors.push(cell.move_direction(Direction::South, grid.voxel_size_nm));
    }
    
    // East (X+1)
    if cell.x + grid.voxel_size_nm < grid.width_nm {
        neighbors.push(cell.move_direction(Direction::East, grid.voxel_size_nm));
    }
    
    // West (X-1)
    if cell.x > 0 {
        neighbors.push(cell.move_direction(Direction::West, grid.voxel_size_nm));
    }
    
    // Up (Z+1)
    if cell.z + grid.voxel_size_nm < grid.depth_nm {
        neighbors.push(cell.move_direction(Direction::Up, grid.voxel_size_nm));
    }
    
    // Down (Z-1)
    if cell.z > 0 {
        neighbors.push(cell.move_direction(Direction::Down, grid.voxel_size_nm));
    }
    
    neighbors  // Always in same order!
}
```

### Why This Matters

**With deterministic routing**:
- Same .hw file always produces same Gerber output
- Version control diffs are meaningful
- Debugging is possible (reproducible bugs)
- CI/CD works reliably
- Users can trust the compiler

**Without deterministic routing**:
- Every compile produces different board
- Can't reproduce bugs
- Can't verify changes
- Users lose trust in the tool

---

## Parallel Validation with Rayon

### The Deterministic Parallelism Strategy

**Critical principle**: Don't parallelize the routing. Parallelize the validation.

**Why this matters**:
- **Single-threaded routing**: Deterministic, reproducible, no race conditions
- **Multi-threaded validation**: Fast, safe (read-only), perfectly parallel

### The Problem with Parallel Routing

If Thread A routes the Power Net and Thread B routes the Data Net simultaneously, and they race to claim the same voxel, whoever gets there first wins. This breaks determinism.

```rust
// ❌ BAD: Non-deterministic parallel routing
rayon::scope(|s| {
    for net in &nets {
        s.spawn(|_| {
            route_net(net, &mut board);  // Race condition!
        });
    }
});
// Different thread scheduling = different routing = non-deterministic
```

### The Solution: Parallel Validation

Route all nets single-threaded (fast enough with optimized algorithms), then validate in parallel with read-only access.

```rust
use rayon::prelude::*;

// ✅ GOOD: Single-threaded routing (deterministic)
for net in &nets {
    let route = route_net_deterministic(net, &board)?;
    board.add_route(route);
}

// ✅ GOOD: Multi-threaded validation (parallel, read-only)
let validators = vec![
    validate_thermal,
    validate_clearances,
    validate_signal_integrity,
    validate_power_delivery,
    validate_impedance,
];

let violations: Vec<Violation> = validators
    .par_iter()  // Parallel iterator
    .flat_map(|validator| validator(&board))
    .collect();
```

### Implementation: Parallel Design Rule Check

```rust
use rayon::prelude::*;

pub fn validate_physics_parallel(
    board: &Board,
    constraints: &ConstraintSet
) -> Result<ValidationReport, PhysicsError> {
    let mut report = ValidationReport::new();
    
    // Define all validation functions
    let validators: Vec<Box<dyn Fn(&Board) -> Vec<Violation> + Sync>> = vec![
        Box::new(|b| validate_thermal(b, constraints)),
        Box::new(|b| validate_clearances(b, constraints)),
        Box::new(|b| validate_signal_integrity(b, constraints)),
        Box::new(|b| validate_power_delivery(b, constraints)),
        Box::new(|b| validate_trace_widths(b, constraints)),
    ];
    
    // Run all validators in parallel (read-only access to board)
    let all_violations: Vec<Violation> = validators
        .par_iter()
        .flat_map(|validator| validator(board))
        .collect();
    
    // Aggregate results
    for violation in all_violations {
        report.add_violation(violation);
    }
    
    Ok(report)
}

// Each validator only reads the board (no mutations)
fn validate_thermal(board: &Board, constraints: &ConstraintSet) -> Vec<Violation> {
    board.traces
        .iter()
        .filter_map(|trace| {
            let temp = calculate_temperature(trace, board);
            if temp > constraints.max_temp_c {
                Some(Violation::Thermal {
                    trace_id: trace.id,
                    temperature_c: temp,
                    max_allowed_c: constraints.max_temp_c,
                })
            } else {
                None
            }
        })
        .collect()
}

fn validate_clearances(board: &Board, constraints: &ConstraintSet) -> Vec<Violation> {
    let mut violations = Vec::new();
    
    // Check all pairs of nets
    for i in 0..board.nets.len() {
        for j in i+1..board.nets.len() {
            let net_a = &board.nets[i];
            let net_b = &board.nets[j];
            
            let clearance_nm = calculate_clearance_nm(net_a, net_b, board);
            let required_nm = calculate_required_clearance_nm(
                net_a.voltage_mv,
                net_b.voltage_mv,
                constraints
            );
            
            if clearance_nm < required_nm {
                violations.push(Violation::Clearance {
                    net_a: net_a.name.clone(),
                    net_b: net_b.name.clone(),
                    actual_nm: clearance_nm,
                    required_nm,
                });
            }
        }
    }
    
    violations
}
```

### Performance Impact

**Single-threaded validation**:
```
5 validators × 200ms each = 1000ms total
```

**Parallel validation (8 cores)**:
```
5 validators / 8 cores ≈ 200ms total
```

**5× speedup with perfect determinism.**

### Why This Works

**Routing (single-threaded)**:
```
Thread 1: Routes all nets sequentially
No race conditions
Perfectly deterministic
Fast enough (optimized algorithms)
```

**Validation (multi-threaded)**:
```
Thread 1: Checks thermal
Thread 2: Checks clearances
Thread 3: Checks signal integrity
Thread 4: Checks power delivery
Thread 5: Checks trace widths

All threads read the same board
No mutations
No race conditions
Perfectly deterministic
4-8× faster
```

### The Guarantee

Because all validation threads only read the board (no writes), there are no race conditions, and the validation results are mathematically guaranteed to be deterministic regardless of thread scheduling.

---

## The Complete Routing Algorithm

### The Three-Phase Routing Sub-Pipeline

**Context**: This 3-phase pipeline executes within the compiler's Physical IR (Layer 3) and Physics IR (Layer 4) stages of the broader 5-Layer MLIR Pipeline. See Book 4 (Compiler Internals) for the complete compiler architecture.

**Phase 1: Constraint Manager** (pre-routing, Layer 3)
- Parse material properties from YAML
- Calculate clearance requirements per net
- Calculate trace width requirements per net
- Generate constraint rulebook
- Store constraints in IR

**Phase 2: Geometry Router** (routing, Layer 3)
- Respect trace width constraints
- Respect clearance zones
- Implement collision detection with forcefields
- Add crosstalk penalty to pathfinding cost
- Use Manhattan routing with layer directions
- Use VecDeque for deterministic ordering

**Phase 3: Design Rule Check** (post-routing validation, Layer 4)
- Sweep entire grid
- Check clearance violations
- Check trace width violations
- Check thermal clustering
- Generate detailed error reports with suggestions

### Example Complete Flow

```rust
pub fn route_board(hw_source: &str) -> Result<Board, Error> {
    // Parse to AST
    let ast = parse(hw_source)?;
    
    // Compile to IR
    let ir = compile_to_ir(ast)?;
    
    // Phase 1: Generate constraints
    let constraints = generate_constraints(&ir)?;
    
    // Phase 2: Route with constraints
    let mut board = Board::new(ir.dimensions, ir.grid);
    
    for net in &ir.nets {
        let route = route_net_deterministic(
            net,
            &constraints,
            &board,
            &ir.materials
        )?;
        
        board.add_route(route);
    }
    
    // Phase 3: Validate
    let validation = validate_drc(&board, &constraints)?;
    if !validation.is_valid() {
        return Err(Error::DRCViolation(validation));
    }
    
    Ok(board)
}
```

---

## Key Takeaways

1. **Physics as constraints** — Translate material properties into geometric rules before routing

2. **3-Phase Routing Sub-Pipeline** — Executes within Layers 3 & 4 of the 5-Layer MLIR Pipeline:
   - Phase 1: Constraint Manager (pre-routing, Layer 3)
   - Phase 2: Geometry Router (routing, Layer 3)
   - Phase 3: Design Rule Check (validation, Layer 4)

3. **Clearance from voltage** — voltage / dielectric_strength with safety margin

4. **Trace width from current** — IPC-2221 formula for ampacity

5. **Crosstalk from geometry** — Parallel length penalty in pathfinding

6. **User defines WHAT** — Netlist specifies connections

7. **Router defines HOW** — Algorithm handles geometry only

8. **Copper planes for power** — Don't route individual wires for power distribution

9. **Manhattan routing** — Layer-specific directions prevent self-blocking

10. **45° on X/Y, 90° on Z** — Different manufacturing processes require different rules

11. **VecDeque for determinism** — FIFO queue guarantees reproducible routing

12. **Stable neighbor order** — Fixed sequence ensures same output every time

---

## Book Separation Reference

**This Book (Book 5: Routing & Physics) Covers**:
- Automatic routing algorithms (A* pathfinding, Manhattan routing)
- Physics constraint generation (clearance, trace width, crosstalk)
- Material property translation to geometric constraints
- Design rule checking and validation
- Deterministic routing implementation
- Parallel validation strategies

**Book 2 (Language Spec) Covers**:
- Manual waypoint routing syntax (`path:` section)
- Bresenham line interpolation between user-defined waypoints
- Complete routing syntax and examples
- User-controlled trace geometry

**The Strict Division**:
- **Book 2 owns**: Manual Waypoint Routing (what the user types: `path: -[1,1,1]`)
- **Book 5 owns**: Automatic Routing (what the A* algorithm does when user doesn't provide waypoints)

For manual waypoint routing syntax, examples, and user-controlled trace paths, see Book 2 (Language Spec), Section 4: Manual Waypoint Routing.

---

## Contributing to the Routing Engine

### Getting Started

1. Clone the repository
2. Navigate to `hwc/crates/hwc-engine/`
3. Run tests with `cargo test`
4. Read the routing module documentation

### Areas for Contribution

**Constraint generation**:
- More sophisticated clearance calculations
- Temperature-dependent properties
- Frequency-dependent impedance

**Routing algorithms**:
- A* pathfinding with better heuristics
- Multi-net routing optimization
- Via count minimization
- Length matching for differential pairs

**Validation**:
- More comprehensive DRC checks
- Thermal simulation
- Signal integrity analysis
- Power distribution validation

**Performance**:
- Parallel routing of independent nets
- GPU acceleration for pathfinding
- Incremental routing for design changes

---

## Conclusion

Hardware Script's routing engine transforms physics into geometry through a constraint-driven approach. By translating material properties into clearance zones, trace widths, and pathfinding costs before routing begins, we achieve both correctness and performance.

The Manhattan routing strategy with deterministic pathfinding ensures that the same input always produces the same output, making Hardware Script truly Git-friendly and trustworthy.

If you're an algorithm developer who wants to help build the future of hardware routing, we'd love your contributions.

Welcome to the routing engine. Let's build something amazing.
