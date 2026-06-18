# Alphanumeric Vector Stream (AVS) Lock System

**Document Type:** Core Architectural Specification & Lockfile Schema
**Status:** Canonical Reference (v0.1.7)
**Focus:** High-Performance, Low-Overhead Spatial Path Cache System

---

## 1. Architectural Intent

The Alphanumeric Vector Stream (AVS) Lock System is a high-density, deterministic routing storage format designed specifically for the Hardware Script compiler.

During active development of SoC-scale layouts or dense multi-layer PCBs, recalculating A\* pathfinding routes on every compile introduces a significant bottleneck. The AVS Lock System addresses this by saving resolved paths into `project.routes.lock`.

### Design Goals

- **Sub-Millisecond Loading:** Skip A\* pathfinding on subsequent compiles, loading the cached paths in <1ms.
- **O(1) Memory Allocation:** Deserialize routing paths into a single contiguous heap allocation, preventing memory fragmentation.
- **ASCII Auditability:** Maintain human-readable text output for visual debugging without resorting to an unreadable binary format.
- **Extreme Size Reduction:** Shrink the coordinate database footprint by up to 80% compared to standard row-oriented JSON.

---

## 2. Core Mathematical Pillars

The AVS Lock System compresses coordinate paths down to their absolute minimum data footprint using three key mathematical techniques:

```
    ┌─────────────────────────────────────────┐
    │       High-Level Route Coordinates      │
    └────────────────────┬────────────────────┘
                         │
    ┌────────────────────▼────────────────────┐
    │   Pillar 1: Topology-Sharing (Arcs)     │  ← Deduplicate parallel buses
    └────────────────────┬────────────────────┘
                         │
    ┌────────────────────▼────────────────────┐
    │   Pillar 2: Base-36 Command-Value RLC   │  ← Eliminate coordinates and spaces
    └────────────────────┬────────────────────┘
                         │
    ┌────────────────────▼────────────────────┐
    │   Pillar 3: Columnar Flat Allocation    │  ← Single-allocation heap buffer
    └─────────────────────────────────────────┘
```

### Pillar 1: Topology-Sharing (Shared Arcs)

In professional layouts, the majority of routes travel in parallel bus configurations. The AVS engine extracts the relative directional sequence of a path only once as an "Arc".

Every parallel net in the bus simply references this Arc index, specifying only its unique absolute starting coordinate. This eliminates redundant coordinate blocks, dividing the storage requirements of bus structures by N (where N is the bus width).

### Pillar 2: Base-36 Command-Value Run-Length Compression (RLC)

Instead of storing absolute 64-bit coordinates (X, Y, Z) in nanometers, paths are represented as relative directional delta vectors.

**Directional Command Suffixes:**

| Command | Direction | Axis |
|---------|-----------|------|
| `R` | East / Right | +X |
| `L` | West / Left | -X |
| `U` | North / Up-Y (+Y) or Via Up (+Z) | +Y or +Z |
| `D` | South / Down-Y (-Y) or Via Down (-Z) | -Y or -Z |

**Base-36 Value Compression:** Length metrics are converted from Base-10 to Base-36, compressing multi-digit numbers into 1 or 2 alphanumeric characters.

**Zero-Delimiters:** All space delimiters are omitted. The parser splits commands from values on-the-fly because letters serve as natural boundaries.

**Example:** The verbose path `"R20 U R150 D R20"` condenses losslessly to `"RkU46Dk"`.

### Base-36 Conversion Reference

| Decimal | Base-36 | Decimal | Base-36 | Decimal | Base-36 |
|---------|---------|---------|---------|---------|---------|
| 10 | a | 100 | 2s | 500 | 18 |
| 15 | f | 150 | 46 | 1000 | rs |
| 20 | k | 200 | 5k | 5000 | 3ps |
| 34 | i | 365 | a5 | 10000 | 7sk |
| 50 | 14 | 400 | b4 | 100000 | 1d2s |

### Pillar 3: Columnar Flat Memory Allocation

Instead of serializing a deeply nested array of individual route structs (which requires allocating thousands of separate small heap buffers in Rust), the layout database flattens the routing instances into a single, contiguous `Vec<i32>`:

```rust
pub struct CompactLockfile {
    pub version: String,
    pub board: String,
    pub placement_hash: String,
    pub arcs: Vec<String>,
    pub instances: Vec<i32>,  // Flattened, contiguous array
}
```

This layout allows SIMD-accelerated JSON parsers to deserialize the entire routing database in a single memory allocation.

---

## 3. The JSON Schema Specification

The resulting `project.routes.lock` file is structured as follows:

```json
{
  "version": "0.1.7",
  "board": "PCB_Complex_Space",
  "placement_hash": "ea0ceb0f62439c4c",
  "arcs": [
    "RkU46Dk",
    "L4UfR34"
  ],
  "instances": [
    0, 0, 2000000, 10000000, 0,
    1, 0, 2000000, 11000000, 0,
    2, 1, 4000000, 12000000, 1235000
  ]
}
```

### Field Definitions

| Field | Type | Description |
|-------|------|-------------|
| `version` | `string` | The compiler version that generated this lockfile. |
| `board` | `string` | The identifier of the space being compiled. |
| `placement_hash` | `string` | A hash of component bounding boxes, grid dimensions, voxel size, and net count. |
| `arcs` | `string[]` | An array of pre-calculated Base-36 RLC path templates. Each string encodes a directional sequence shared by one or more nets. |
| `instances` | `int32[]` | A flat list of routing instances. Each instance is 5 consecutive elements: `[net_id, arc_idx, start_x, start_y, start_z]`. |

### Instance Encoding Layout

Each routing instance occupies exactly 5 `int32` slots in the `instances` array:

```
┌──────────┬──────────┬──────────┬──────────┬──────────┐
│ net_id   │ arc_idx  │ start_x  │ start_y  │ start_z  │
│ (i32)    │ (i32)    │ (i32 nm) │ (i32 nm) │ (i32 nm) │
└──────────┴──────────┴──────────┴──────────┴──────────┘
  [0]        [1]        [2]        [3]        [4]
```

- **`net_id`**: The logical net this route belongs to. Must match the netlist.
- **`arc_idx`**: Index into the `arcs` array. References the directional template.
- **`start_x`, `start_y`, `start_z`**: Absolute starting coordinate in nanometers.

---

## 4. Arc Decoding Algorithm

During compilation, the lockfile is resolved through the following pipeline:

```
1. PARSE
   Read lockfile JSON into CompactLockfile struct.
   Load `instances` into contiguous Vec<i32>.
          │
          ▼
2. VALIDATE
   Compute placement_hash from current AST.
   Compare against stored placement_hash.
   ├── Match   → Proceed to decode.
   └── Mismatch → Discard lock, execute A* pathfinder.
          │
          ▼
3. DECODE
   Iterate over `instances` chunked by 5 elements:
   a. Read starting coordinate from indices [2], [3], [4].
   b. Read arc template: arcs[arc_idx].
   c. For each character in the arc string:
      ├── Letter (R/L/U/D) → Direction command.
      └── Digit (0-9, a-z) → Base-36 magnitude (accumulated).
      Apply delta to current position.
      Commit resolved coordinate to Voxel Grid.
```

### Base-36 Arc Decoding Pseudocode

```rust
fn decode_arc(arc: &str, start: Point3D) -> Vec<Point3D> {
    let mut points = vec![start];
    let mut pos = start;
    let mut magnitude: i64 = 0;
    let mut has_magnitude = false;
    let mut prev_dir = 'R';

    for ch in arc.chars() {
        match ch {
            'R' | 'L' | 'U' | 'D' => {
                if has_magnitude {
                    pos = apply_direction(pos, prev_dir, magnitude);
                    points.push(pos);
                    magnitude = 0;
                    has_magnitude = false;
                }
                prev_dir = ch;
            }
            '0'..='9' => {
                magnitude = magnitude * 36 + (ch as i64 - '0' as i64);
                has_magnitude = true;
            }
            'a'..='z' => {
                magnitude = magnitude * 36 + (ch as i64 - 'a' as i64 + 10);
                has_magnitude = true;
            }
            _ => {}
        }
    }
    if has_magnitude {
        pos = apply_direction(pos, prev_dir, magnitude);
        points.push(pos);
    }
    points
}

fn apply_direction(p: Point3D, dir: char, mag: i64) -> Point3D {
    match dir {
        'R' => Point3D::new(p.x + mag, p.y, p.z),
        'L' => Point3D::new(p.x - mag, p.y, p.z),
        'U' => Point3D::new(p.x, p.y + mag, p.z),
        'D' => Point3D::new(p.x, p.y - mag, p.z),
        _   => p,
    }
}
```

---

## 5. Invalidation Rules

The lockfile is automatically invalidated and re-evaluated by the A\* pathfinder under the following conditions:

### 5.1 Placement Shift (Invalidates Lock)

Any component shifting, rotation, or footprint modification changes the `placement_hash`, invalidating the lockfile. This includes:

- Moving a component's X/Y/Z position
- Rotating a component
- Changing a component's footprint or pad layout
- Adding or removing a component

### 5.2 Netlist Alteration (Invalidates Lock)

Any additions, deletions, or structural re-bindings of logical nets invalidate the file:

- Adding a new `route` declaration
- Removing an existing `route` declaration
- Changing the `net:` binding of a route
- Modifying pin connections

### 5.3 Physical Boundary Changes (Invalidates Lock)

Modifying the space board dimensions or grid cell resolution invalidates the file:

- Changing `dimensions:` in the space definition
- Changing `grid:` resolution
- Modifying `profile:` stackup or layer definitions

### 5.4 Non-Geometric Changes (Preserved Lock)

The following changes do **NOT** invalidate the lockfile:

- Editing `description` fields in profiles or metadata
- Changing part numbers or comments
- Modifying `nets:` classification labels (e.g., `signal` → `power`)
- Any change that does not affect component positions or routing topology

---

## 6. Integration with Existing Systems

### 6.1 Analytic Trace Pipeline

The AVS decoder outputs `AnalyticTrace` primitives directly into `HardwareSpace.analytic_routes`. This bypasses the voxel grid entirely during the decode phase, maintaining the "Primitives Over Pixels" architecture.

### 6.2 A\* Pathfinder Bypass

When the lockfile is valid, the `AutoRouter::route_all_nets()` method skips nets that already have entries in `space.analytic_routes`. The existing collinearity check in the auto-router naturally avoids redundant routing.

### 6.3 Export Pipeline

The decoded `AnalyticTrace` objects flow into the export pipeline unchanged. The GLB, DXF, and netlist exporters consume them identically to freshly-routed traces.

---

## 7. Lockfile Versioning

The AVS format uses strict version checking. Because the routing lockfile is an automatically generated derivative of the AST, the compiler does not maintain backward compatibility with legacy JSON lockfiles (v0.1.6 and earlier).

### Invalidation Behavior

If the compiler detects a lockfile with a version string prior to `"0.1.7"`, or if JSON deserialization fails due to an obsolete schema, the compiler rejects the lockfile with a clear error message instructing the user to delete it and rebuild. The compiler does not automatically delete or migrate legacy files.
