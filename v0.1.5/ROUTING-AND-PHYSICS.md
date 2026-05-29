# Routing and Physics (v0.1.5)

**Base Documentation**: [v0.1.4 ROUTING-AND-PHYSICS.md](../v0.1.4/ROUTING-AND-PHYSICS.md)  
**Status**: Incremental update - Advanced routing features  
**Version**: 0.1.5

---

## What's New in v0.1.5

This document covers ONLY the new routing features added in v0.1.5. For the complete routing architecture (A* pathfinding, Manhattan routing, DRC validation), see [v0.1.4 ROUTING-AND-PHYSICS.md](../v0.1.4/ROUTING-AND-PHYSICS.md).

**New Features**:
1. Constraint-Aware Routing (pattern-based length matching)
2. Route Lockfile System (routing stability)
3. Rip-Up and Reroute (priority-based routing)
4. HDI Via Support (multiple via types)
5. Parallel Routing (domain-based compilation)

---

## Constraint-Aware Routing

**What it is**: Pattern-based routing that generates traces with exact target lengths

**Why it matters**: High-speed signals (DDR5, PCIe) require precise length matching for signal integrity

### The Problem

Standard A* routing minimizes path length. For DDR5 memory buses, all 8 data lines must be exactly the same length (within ±0.1mm) so signals arrive simultaneously.

**Before v0.1.5**: Post-processing tried to add length after routing (brittle, error-prone)  
**After v0.1.5**: Constraint-aware A* knows the target length before routing starts

### Pattern Definitions

Patterns define reusable routing shapes using polar notation: `distance r angle`

**Syntax**: `define pattern "Name" (params): steps: - distance r angle`

**Example** (`hwc/stdlib/routing/patterns.hw`):
```hw
define pattern "Trombone" (gap: Measurement, amp: Measurement):
    steps:
        - gap r 0           # Move straight
        - amp r 90          # Turn 90 degrees
        - gap * 2 r 0       # Move straight (2x gap)
        - amp r -90         # Turn back
        - gap r 0           # Return to centerline
```

**Polar Notation**:
- `distance`: How far to travel (supports expressions like `gap * 2`)
- `r`: Rotate operator
- `angle`: Degrees relative to current heading (90 = right turn, -90 = left turn)

### Strategy Definitions

Strategies combine patterns with length-matching constraints.

**Syntax**: `define strategy "Name": target: ..., tolerance: ..., pattern: ...`

**Example**:
```hw
define strategy "DDR5_LengthMatch":
    target: match_longest
    tolerance: 0.1mm
    pattern: Trombone(gap: 0.3mm, amp: 2.5mm)
```

**Target Options**:
- `match_longest`: All nets match the longest net in the group
- `match_shortest`: All nets match the shortest net
- `exact: 45mm`: All nets exactly 45mm long

### Usage

Apply strategies to routes in `define space` blocks:

```hw
define space "DDR5_Board":
    dimensions: 100mm by 100mm by 2mm
    grid: 1000 by 1000 by 4
    
    # Route with length matching
    route CPU.DDR_Data[0..7] to RAM.Data[0..7]:
        signal_group: "DDR5_Data"
        strategy: DDR5_LengthMatch
```

### How It Works

1. **Calculate Target**: Router measures all nets in signal group, finds longest
2. **Constraint-Aware A***: Modified heuristic penalizes length mismatches
3. **Pattern Injection**: Trombone pattern injected as macro-moves into A* neighbor generator
4. **Collision Detection**: Pattern moves validated against occupied voxels
5. **Result**: All nets exactly match target length (within tolerance)

**Key Insight**: Patterns are macro-moves fed into A* before routing starts, not post-processing hacks

---

## Route Lockfile System

**What it is**: Persistent routing cache that prevents the "butterfly effect"

**Why it matters**: Moving one component shouldn't cascade and rewire the entire board

### The Problem

**Determinism** (v0.1.4): Same code produces same output every time ✅  
**Stability** (v0.1.5): Changing one line produces minimal output changes ✅

**Example Scenario**:
```
Initial: Net 1 routes at Y=10, Net 2 at Y=11, Net 3 at Y=12
After moving Component A by 1mm:
  Without lockfile: All 100 nets reroute (massive Git diff)
  With lockfile: Only Net 1 reroutes (minimal Git diff)
```

### The `.hw.routes.lock` File

Auto-generated JSON file storing exact waypoints of every successful route.

**Location**: Same directory as `hw.toml`  
**Format**: Human-readable JSON (Git-friendly)

**Example**:
```json
{
  "version": "0.1.5",
  "board": "Main_Board",
  "grid": {
    "dimensions": [200, 100, 2],
    "resolution": [1.0, 1.0, 1.0]
  },
  "routes": [
    {
      "net_id": "Net_1",
      "source": "R1.Pin2",
      "destination": "Amp.RF_IN",
      "waypoints": [
        [20, 50, 1],
        [25, 50, 1],
        [130, 50, 1]
      ],
      "length_mm": 115.5,
      "layer_transitions": 0,
      "hash": "a3f5b2c1"
    }
  ]
}
```

### Routing Pipeline

1. **Load Lockfile**: Read `.hw.routes.lock` if exists
2. **Validate Routes**: Check if component endpoints moved
3. **Freeze Valid Routes**: Place unchanged routes on voxel grid as immutable
4. **Detect Collisions**: Check if new components block frozen routes
5. **Selective Reroute**: Only reroute nets with collisions or moved endpoints
6. **Save Lockfile**: Update with any changed routes

### CLI Integration

**Default Behavior**: Auto-generates `.hw.routes.lock` on every successful build

**Flags**:
- `--no-lockfile`: Disable lockfile (reroute everything)
- `--force-reroute`: Ignore lockfile (reroute everything)

**Example**:
```bash
# Normal build (uses lockfile)
hwc build board.hw

# Force complete reroute
hwc build board.hw --force-reroute

# Disable lockfile
hwc build board.hw --no-lockfile
```

### Git Integration

**Commit the lockfile**: Like `Cargo.lock`, commit `.hw.routes.lock` to version control

**Benefits**:
- Minimal diffs (only changed routes)
- Reproducible builds across team
- Merge conflicts are rare and localized

---

## Rip-Up and Reroute

**What it is**: Priority-based routing that can remove lower-priority nets to make room for higher-priority nets

**Why it matters**: Ensures critical nets (clocks, power) always route successfully

### Net Priority System

**Priority Levels** (highest to lowest):
1. `Critical` - Clocks, resets, high-speed differential pairs
2. `Power` - Power rails (if not using planes)
3. `HighSpeed` - DDR, PCIe, USB 3.0
4. `DataBus` - Parallel data buses
5. `LowSpeed` - I2C, UART, SPI
6. `GPIO` - General purpose I/O

**Automatic Detection**: Router analyzes net names to assign priorities
- `CLK`, `CLOCK`, `RESET` → Critical
- `VCC`, `VDD`, `GND` → Power
- `DDR`, `PCIE`, `USB3` → HighSpeed
- `DATA`, `BUS` → DataBus
- `I2C`, `SPI`, `UART` → LowSpeed
- Everything else → GPIO

### Routing Order

Nets are sorted by priority before routing:

```
1. Route Critical nets first (shortest paths)
2. Route Power nets second (thickest paths)
3. Route HighSpeed nets third (pattern-matched)
4. Route DataBus nets fourth
5. Route LowSpeed nets fifth
6. Route GPIO nets last (fills remaining space)
```

### Rip-Up Logic

If a high-priority net cannot route:
1. Identify blocking nets
2. Check if blocking nets have lower priority
3. Remove lower-priority nets from voxel grid
4. Route high-priority net
5. Attempt to reroute removed nets

**Example**:
```
Clock net (Critical) blocked by GPIO net (GPIO)
→ Remove GPIO net
→ Route Clock net
→ Reroute GPIO net around Clock
```

### Benefits

**Routing Completion**: Achieves 100% routing by prioritizing critical nets  
**Stability**: Moving GPIO components doesn't affect clock nets (routed first)  
**Predictability**: Critical nets always get optimal paths

---

## HDI Via Support

**What it is**: Support for multiple via types (through-hole, blind, buried, microvia)

**Why it matters**: High-density boards require different via types for different layer transitions

### Via Types

**Through-Hole Via**: Spans all layers (Layer 1 to Layer N)
- Diameter: 300-500µm
- Cost: Low
- Use: Standard PCBs

**Blind Via**: Connects outer layer to inner layer (Layer 1 to Layer 3)
- Diameter: 200-300µm
- Cost: Medium
- Use: HDI boards

**Buried Via**: Connects two inner layers (Layer 2 to Layer 4)
- Diameter: 200-300µm
- Cost: Medium
- Use: HDI boards

**Microvia**: Connects adjacent layers only (Layer 1 to Layer 2)
- Diameter: 100-150µm
- Cost: High
- Use: High-density HDI

### Automatic Classification

Router automatically selects via type based on layer span:

```rust
if diameter < 150µm && layer_span == 1 {
    ViaType::Microvia
} else if start_layer == 0 || end_layer == max_layer {
    if layer_span < max_layer {
        ViaType::Blind
    } else {
        ViaType::ThroughHole
    }
} else {
    ViaType::Buried
}
```

### Cost Calculation

Different via types have different routing costs:

- Through-hole: 10,000 points (base penalty)
- Blind: 15,000 points (1.5x penalty)
- Buried: 20,000 points (2x penalty)
- Microvia: 25,000 points (2.5x penalty)

**Why**: Encourages router to use cheaper via types when possible

### Export

Drill files are separated by via type:

- `board-PTH.drl` - Through-hole vias
- `board-NPTH.drl` - Non-plated holes
- `board-blind.drl` - Blind vias
- `board-buried.drl` - Buried vias
- `board-micro.drl` - Microvias

---

## HDI Via Support

**What it is**: Support for multiple via types (through-hole, blind, buried, microvia)

**Why it matters**: High-density boards require different via types for different layer transitions

### Via Types

**Through-Hole Via**: Spans all layers (Layer 1 to Layer N)
- Diameter: 300-500µm
- Cost: Low
- Use: Standard PCBs
- Drill file: `board-PTH.drl`

**Blind Via**: Connects outer layer to inner layer (Layer 1 to Layer 3)
- Diameter: 200-300µm
- Cost: Medium
- Use: HDI boards
- Drill file: `board-blind.drl`

**Buried Via**: Connects two inner layers (Layer 2 to Layer 4)
- Diameter: 200-300µm
- Cost: Medium
- Use: HDI boards
- Drill file: `board-buried.drl`

**Microvia**: Connects adjacent layers only (Layer 1 to Layer 2)
- Diameter: 100-150µm
- Cost: High
- Use: High-density HDI
- Drill file: `board-micro.drl`

### Automatic Classification

Router automatically selects via type based on layer span:

```rust
pub fn classify_via(
    diameter_nm: i64,
    start_layer: usize,
    end_layer: usize,
    max_layer: usize,
) -> ViaType {
    let layer_span = end_layer.abs_diff(start_layer);
    
    if diameter_nm < 150_000 && layer_span == 1 {
        ViaType::Microvia
    } else if start_layer == 0 || end_layer == max_layer {
        if layer_span < max_layer {
            ViaType::Blind
        } else {
            ViaType::ThroughHole
        }
    } else {
        ViaType::Buried
    }
}
```

### Cost Calculation

Different via types have different routing costs (A* heuristic):

- Through-hole: 10,000 points (base penalty)
- Blind: 15,000 points (1.5x penalty)
- Buried: 20,000 points (2x penalty)
- Microvia: 25,000 points (2.5x penalty)

**Why**: Encourages router to use cheaper via types when possible

### Export

Drill files are separated by via type:

- `board-PTH.drl` - Plated through-hole vias
- `board-NPTH.drl` - Non-plated holes (mounting, tooling)
- `board-blind.drl` - Blind vias (outer to inner)
- `board-buried.drl` - Buried vias (inner to inner)
- `board-micro.drl` - Microvias (adjacent layers)

**Format**: Excellon drill format with tool definitions

---

## Pad Shapes & Solder Masks

**What it is**: Support for custom pad shapes and automatic solder mask/paste layer generation

**Why it matters**: Different components require different pad geometries (BGA balls, QFN pads, SMD rectangles)

### Pad Shape Definitions

**Syntax**: Inside `define component` layout blocks

```hw
define component "BGA256":
    pins:
        Ball[256]
    
    layout:
        shape: Rectangle(17mm, 17mm, 1.6mm)
        pin_positions:
            for i in 0..255:
                Ball[i] at [x: (i%16)*pitch, y: (i/16)*pitch]
                pad_shape: Circle(diameter: 0.4mm)  # <== NEW in v0.1.5
```

### Supported Pad Shapes

**Circle**: `pad_shape: Circle(diameter: 0.4mm)`
- Use: BGA balls, vias
- Aperture: `C,0.4*`

**Obround**: `pad_shape: Obround(width: 1.2mm, height: 2.0mm)`
- Use: SMD resistors, capacitors
- Aperture: `O,1.2X2.0*`

**Rectangle**: `pad_shape: Rectangle(width: 1.0mm, height: 1.5mm)`
- Use: QFN pads, SMD components
- Aperture: `R,1.0X1.5*`

**RoundedRect**: `pad_shape: RoundedRect(width: 1.0mm, height: 1.5mm, radius: 0.2mm)`
- Use: Modern SMD components
- Aperture: Custom polygon

**Polygon**: `pad_shape: Polygon(points: [[0,0], [1,0], [1,1], [0,1]])`
- Use: Custom pad shapes
- Aperture: Custom polygon

### Automatic Layer Generation

**Copper Layers** (`.gtl`, `.gbl`):
- Pad shapes are drawn on copper layers
- Apertures are defined in Gerber header

**Solder Mask Layers** (`.gts`, `.gbs`):
- Automatically generated with clearance around pads
- Default clearance: 0.1mm (configurable in profile)
- Aperture: Pad shape + clearance

**Solder Paste Layers** (`.gtp`, `.gbp`):
- Automatically generated for SMD pads only
- Aperture: Pad shape × 0.9 (90% of pad size)
- Through-hole pads excluded

### Example: BGA Component

**Input**:
```hw
define component "BGA256":
    pins: Ball[256]
    layout:
        for i in 0..255:
            Ball[i] at [x: (i%16)*1mm, y: (i/16)*1mm]
            pad_shape: Circle(diameter: 0.4mm)
```

**Generated Gerber Layers**:
- `board.gtl` - Top copper (256 circles, 0.4mm diameter)
- `board.gts` - Top solder mask (256 circles, 0.5mm diameter)
- `board.gtp` - Top solder paste (256 circles, 0.36mm diameter)

---

## Thermal Reliefs & Copper Pours

**What it is**: Automatic thermal relief generation for pads connected to copper pours

**Why it matters**: Direct connection to copper pour makes soldering difficult (heat sink effect)

### The Problem

**Without Thermal Relief**: Pad directly connected to copper pour
- Heat dissipates into entire pour
- Soldering requires high temperature and long time
- Risk of component damage

**With Thermal Relief**: Pad connected via thin spokes
- Heat is localized to pad
- Soldering is fast and easy
- Component stays cool

### Thermal Relief Syntax

```hw
define space "PowerBoard":
    dimensions: 100mm by 100mm by 2mm
    grid: 1000 by 1000 by 4
    
    # Define copper pour with thermal relief
    add pour(Copper) named GND_Plane on z:2:
        boundary: [x:0, y:0] to [x:100mm, y:100mm]
        net: GND
        thermal_relief: true    # <== Triggers spoke generation
        clearance: 0.5mm        # <== Anti-pad size for non-GND vias
```

### Thermal Relief Parameters

**Spoke Width**: 0.3mm (default, configurable in profile)
**Spoke Count**: 4 (90-degree cross pattern)
**Anti-Pad Clearance**: 0.5mm (gap around non-connected pads)

### Rasterization Algorithm

**Step 1: Polygon Rasterization**
- Convert pour boundary to voxel grid
- Mark all voxels inside polygon as "copper"

**Step 2: Clearance Removal**
- For each via/pad NOT connected to pour net:
  - Remove copper in 0.5mm radius (anti-pad)

**Step 3: Thermal Relief Generation**
- For each via/pad connected to pour net:
  - Remove copper in 0.5mm radius (anti-pad)
  - Add 4 spokes (0.3mm wide) at 0°, 90°, 180°, 270°

**Step 4: Gerber Export**
- Export rasterized copper as Gerber polygons
- Use `G36` (polygon start) and `G37` (polygon end)

### Example: GND Plane with Thermal Reliefs

**Input**:
```hw
add pour(Copper) named GND_Plane on z:2:
    boundary: [x:0, y:0] to [x:100mm, y:100mm]
    net: GND
    thermal_relief: true
    clearance: 0.5mm

add Resistor_0805 named R1 at [x:10, y:10, z:1]
add Capacitor_0805 named C1 at [x:20, y:10, z:1]

route R1.Pin2 to GND
route C1.Pin2 to VCC
```

**Result**:
- R1.Pin2: Connected to GND → Thermal relief with 4 spokes
- C1.Pin2: Connected to VCC → Anti-pad clearance (no copper)

---

## Physics Auto-Fixes

**What it is**: Compiler suggestions for fixing physics violations

**Why it matters**: Helps users fix design rule violations without manual calculation

### Buffer Insertion (Timing/RC Delay)

**Problem**: Long trace causes signal delay or RC time constant violation

**Detection**:
```rust
if trace_length_mm > 50.0 && signal_frequency_mhz > 100.0 {
    suggest_buffer_insertion();
}
```

**Suggestion**:
```
⚠️  Warning[P34]: Signal integrity violation on net 'CLK_100MHz'
   Trace length: 75mm
   Frequency: 100MHz
   Estimated delay: 0.5ns (exceeds 0.3ns budget)
   
   Suggestion: Insert buffer at midpoint
   
   add Buffer_74LVC1G125 named CLK_BUF at [x:50, y:25, z:1]
   route CPU.CLK to CLK_BUF.In
   route CLK_BUF.Out to RAM.CLK
```

### Trace Widening (Ampacity)

**Problem**: Trace too thin for current

**Detection**:
```rust
let required_width_mm = calculate_trace_width(current_a, temp_rise_c, copper_thickness_oz);
if trace_width_mm < required_width_mm {
    suggest_trace_widening();
}
```

**Suggestion**:
```
❌ Error[P21]: Trace too thin for current on net 'VCC_5V'
   Current: 3.5A
   Trace width: 0.2mm
   Required width: 1.2mm (for 10°C rise, 1oz copper)
   
   Suggestion: Widen trace or use copper pour
   
   route VCC to Load:
       width: 1.2mm  # <== Increase from 0.2mm
```

---

## Parallel Routing

**What it is**: Multi-threaded routing using domain isolation

**Why it matters**: 10-100x speedup on multi-core systems

### Domain Isolation

**The Glass Box Rule**: Each module instance creates a routing domain with a 3D bounding box

**Isolation Guarantee**: Threads routing different domains cannot touch the same voxels (no race conditions, no locks)

### Three-Phase Pipeline

**Phase 1: Partitioning** (single-threaded)
- Calculate bounding box for each module instance
- Classify nets as "internal" (both pins in same module) or "global" (cross module boundaries)
- Create isolated voxel grid for each domain

**Phase 2: Local Parallel Routing** (multi-threaded via Rayon)
- Each domain routes in parallel using isolated grid
- No locks, no mutexes (each thread owns its FxHashMap)
- Deterministic output (same input → same output)

**Phase 3: Assembly & Global Routing** (single-threaded)
- Merge all domain grids into global grid
- Route global nets between domains
- Generate final output

### Performance

**Test Board**: 1000-net board with 10 modules, 10,000×10,000×10 voxel grid

| Cores | Time | Speedup |
|-------|------|---------|
| 1 | 30 min | 1x |
| 4 | 8 min | 3.75x |
| 8 | 4 min | 7.5x |
| 16 | 2 min | 15x |

**Why not linear?**: Phase 1 and Phase 3 are single-threaded (Amdahl's Law)

### Usage

Parallel routing is automatic when using modules with layout blocks:

```hw
define module "ProcessorCore":
    pins: DataIn[64], DataOut[64]
    # ... module definition

define space "Motherboard":
    dimensions: 200mm by 200mm by 2mm
    grid: 2000 by 2000 by 4
    
    # Each core instance creates a routing domain
    add ProcessorCore named Core1 at [10, 10, 1]
    add ProcessorCore named Core2 at [110, 10, 1]
    
    # Layout blocks define bounding boxes
    layout Core1:
        at [x:10mm, y:10mm, z:0]
        dimensions: 50mm by 50mm by 2mm
    
    layout Core2:
        at [x:110mm, y:10mm, z:0]
        dimensions: 50mm by 50mm by 2mm
```

**Result**: Core1 and Core2 route in parallel, then global nets connect them

---

## References

**Implementation Files**:
- `hwc/crates/hwc-engine/src/geometry_router/constraint_aware.rs` - Constraint-aware A*
- `hwc/crates/hwc-engine/src/geometry_router/routing_patterns.rs` - Pattern system
- `hwc/crates/hwc-engine/src/geometry_router/priority.rs` - Net priority system
- `hwc/crates/hwc-engine/src/geometry_router/ripup.rs` - Rip-up engine
- `hwc/crates/hwc-engine/src/geometry_router/route_lockfile.rs` - Lockfile system
- `hwc/crates/hwc-engine/src/parallel_router.rs` - Parallel routing

**Related Documentation**:
- [v0.1.4 ROUTING-AND-PHYSICS.md](../v0.1.4/ROUTING-AND-PHYSICS.md) - Base routing architecture
- [v0.1.5 COMPILER-INTERNALS.md](./COMPILER-INTERNALS.md) - Module system
- [v0.1.5 LANGUAGE-SPEC.md](./LANGUAGE-SPEC.md) - Pattern and strategy syntax

**Implementation Plans**:
- [CONSTRAINT-AWARE-ROUTING.md](../../ROADMAP/v0.1.4/CONSTRAINT-AWARE-ROUTING.md) - Detailed architecture
- [GAP2.md](../../ROADMAP/v0.1.4/GAP2.md) - Stability architecture
- [GAP3.md](../../ROADMAP/v0.1.4/Gap3.md) - Parallel routing architecture
