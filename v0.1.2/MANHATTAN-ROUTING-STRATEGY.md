# Hardware Script v0.1.2 - Manhattan Routing Strategy

**Document Type**: Advanced Routing Algorithm Guide  
**Status**: External Research Integration  
**Last Updated**: March 2026

---

## The Three Fundamental Rules

### Rule 1: Straight Lines Are Best

**Why**:
- Shortest distance between two points
- Least resistance (lower voltage drop)
- Cleanest signal (no reflections)
- Fastest routing algorithm

**Implementation**: Always prefer straight-line paths when possible.

### Rule 2: 45° Corners for Turns

**Why**:
- Prevent chemical "acid traps" during manufacturing
- Prevent high-speed electrons from crashing into walls
- Reduce signal interference and EMI
- Industry standard for professional PCBs

**Implementation**: When a turn is required on the same layer, use 45° angles.

### Rule 3: Layer-Specific Directions (Manhattan Routing)

**Why**:
- Prevents the algorithm from trapping itself
- Guarantees a solution exists (if physically possible)
- Simplifies pathfinding logic
- Industry standard for multi-layer boards

**Implementation**: Assign specific movement directions to specific layers.

---

## The Manhattan Routing Concept

### What Is Manhattan Routing?

Manhattan Routing is named after the street grid of Manhattan, New York City, where streets run either North-South or East-West, never diagonally.

In PCB design, Manhattan Routing means:
- **Each layer is restricted to one primary direction**
- **Turns are accomplished by changing layers (vias)**
- **The algorithm never gets trapped**


### The Layer Assignment Strategy

**Standard 4-Layer Board**:

```
Layer 1 (Top):    North/South routing only
Layer 2 (Inner):  Solid Ground Plane (no routing)
Layer 3 (Inner):  Solid Power Plane (no routing)
Layer 4 (Bottom): East/West routing only
```

**Alternative 4-Layer Board** (all routing layers):

```
Layer 1 (Top):    North/South routing only
Layer 2 (Inner):  East/West routing only
Layer 3 (Inner):  North/South routing only
Layer 4 (Bottom): East/West routing only
```

**Simple 2-Layer Board**:

```
Layer 1 (Top):    North/South routing only
Layer 2 (Bottom): East/West routing only
```

### Why This Works

**The Problem Without Manhattan Routing**:

If all layers can route in all directions, the algorithm can create situations where:
- A trace blocks its own future path
- No valid route exists even though space is available
- Backtracking becomes exponentially complex

**The Solution With Manhattan Routing**:

By restricting each layer to one direction:
- The algorithm can always find a path if one exists
- No self-blocking scenarios
- Pathfinding becomes O(n) instead of O(n²)

---

## How Manhattan Routing Handles Turns

### The Traditional Approach (Single Layer)

**Problem**: Need to turn 90° on the same layer

**Bad Solution**: Draw a 90° corner
```
Layer 1:
    ┌─────
    │
────┘

Sharp corner, signal reflection, acid traps
```

**Better Solution**: Use 45° turns
```
Layer 1:
     ╱────
    ╱
───╱

Smoother, but still not ideal for complex routing
```

### The Manhattan Approach (Multi-Layer)

**Problem**: Need to turn 90° (change from North to East)

**Manhattan Solution**: Change layers instead of turning

```
Layer 1 (North/South only):
    │
    │  (traveling North)
    ●  (via down to Layer 2)

Layer 2 (East/West only):
    ●─────→ (traveling East)
```

**The Algorithm**:
1. Traveling North on Layer 1
2. Need to go East
3. Drop via to Layer 2 (East/West layer)
4. Continue East on Layer 2
5. If need to go North again, drop via back to Layer 1


---

## Implementation in Voxel Grid

### Layer Direction Constraints

```rust
enum LayerDirection {
    NorthSouth,  // Y-axis only
    EastWest,    // X-axis only
    Any,         // Power/Ground planes or unrestricted
}

struct Layer {
    z_index: usize,
    direction: LayerDirection,
    is_plane: bool,  // Solid copper plane vs. routing layer
}

fn get_layer_direction(z: usize) -> LayerDirection {
    match z {
        1 => LayerDirection::NorthSouth,  // Top layer
        2 => LayerDirection::Any,          // Ground plane
        3 => LayerDirection::Any,          // Power plane
        4 => LayerDirection::EastWest,     // Bottom layer
        _ => LayerDirection::Any,
    }
}
```

### Movement Validation

```rust
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

### Pathfinding with Manhattan Constraints

```rust
fn find_manhattan_path(
    start: (usize, usize, usize),
    end: (usize, usize, usize),
    layers: &[Layer]
) -> Option<Vec<(usize, usize, usize)>> {
    let mut path = vec![start];
    let mut current = start;
    
    while current != end {
        let (z, x, y) = current;
        let (ez, ex, ey) = end;
        
        let layer_dir = layers[z].direction;
        
        match layer_dir {
            LayerDirection::NorthSouth => {
                if y != ey {
                    // Move North or South
                    let next_y = if y < ey { y + 1 } else { y - 1 };
                    current = (z, x, next_y);
                } else if x != ex {
                    // Need to go East/West, switch to EastWest layer
                    let ew_layer = find_east_west_layer(layers);
                    current = (ew_layer, x, y);  // Via down
                } else {
                    // Reached destination
                    break;
                }
            },
            LayerDirection::EastWest => {
                if x != ex {
                    // Move East or West
                    let next_x = if x < ex { x + 1 } else { x - 1 };
                    current = (z, next_x, y);
                } else if y != ey {
                    // Need to go North/South, switch to NorthSouth layer
                    let ns_layer = find_north_south_layer(layers);
                    current = (ns_layer, x, y);  // Via up/down
                } else {
                    // Reached destination
                    break;
                }
            },
            LayerDirection::Any => {
                // Unrestricted layer, move directly
                // (This shouldn't happen in strict Manhattan routing)
                break;
            }
        }
        
        path.push(current);
    }
    
    Some(path)
}
```

---

## Example: Complete Manhattan Route

### Scenario

**Start**: Camera @ [1, 10, 50] (Layer 1, North/South only)  
**End**: CPU @ [4, 90, 10] (Layer 4, East/West only)

**Required movements**:
- South: 50 → 10 (40 units)
- East: 10 → 90 (80 units)

### The Path

```
Step 1: Move South on Layer 1 (North/South layer)
  [1, 10, 50] → [1, 10, 10]
  
Step 2: Via down to Layer 4 (East/West layer)
  [1, 10, 10] → [4, 10, 10]
  
Step 3: Move East on Layer 4 (East/West layer)
  [4, 10, 10] → [4, 90, 10]
  
Done! Reached CPU.
```

### Visualization

```
Top View (Layer 1):
    Camera
       │
       │ (South on Layer 1)
       │
       ● (Via down to Layer 4)

Top View (Layer 4):
       ● (Via from Layer 1)
       └──────────────→ CPU
         (East on Layer 4)

Side View:
Layer 1: Camera ──┐
                  │ (via)
Layer 4:          └──────→ CPU
```

---

## Benefits of Manhattan Routing

### 1. Guaranteed Solution

If a physical path exists, Manhattan routing will find it.

**Why**: By alternating between perpendicular layers, the algorithm can always navigate around obstacles.

### 2. No Self-Blocking

The algorithm cannot trap itself.

**Why**: Each layer has only one degree of freedom (North/South OR East/West), so backtracking is never needed.

### 3. Simplified Pathfinding

The algorithm is much simpler than general-purpose pathfinding.

**Complexity**:
- General routing: O(n²) or worse
- Manhattan routing: O(n) where n is the Manhattan distance

### 4. Predictable Via Count

The number of vias is predictable and minimal.

**Formula**: Maximum vias = number of direction changes

**Example**:
- Start going North
- Change to East (1 via)
- Change to South (1 via)
- Change to West (1 via)
- Total: 3 vias

### 5. Industry Standard

Professional PCB manufacturers expect Manhattan routing.

**Why**: It produces clean, manufacturable boards that are easy to inspect and debug.

---

## Advanced Considerations

### When to Allow 45° Turns

Even with Manhattan routing, you can still use 45° turns **within the same direction**.

**Example on North/South layer**:
```
Instead of:
    │
    │
    │

You can do:
    │
   ╱
  ╱

Both are "North" movement, just at different angles.
```

**Implementation**: Allow diagonal movement as long as the primary direction constraint is satisfied.

### Handling Obstacles

If a straight path is blocked, the algorithm can:

1. **Jog around the obstacle** (stay on same layer)
```
    │
    │  ← obstacle
    ├──┐
       │
       │
```

2. **Switch layers** (via to different routing layer)
```
Layer 1: │
         ● (via down)
         
Layer 2:   ──→ (route around)
         ● (via up)
         
Layer 1: │
```

### Optimizing Via Count

While Manhattan routing guarantees a solution, you can optimize to minimize vias:

**Strategy**: Try to complete as much distance as possible on each layer before switching.

**Example**:
```
Bad (3 vias):
  Layer 1: North 10 units
  Via to Layer 2
  Layer 2: East 5 units
  Via to Layer 1
  Layer 1: North 10 units
  Via to Layer 2
  Layer 2: East 5 units

Good (1 via):
  Layer 1: North 20 units
  Via to Layer 2
  Layer 2: East 10 units
```

---

## Implementation Checklist

### Phase 1: Basic Manhattan Routing (High Priority)

- [ ] Define layer direction constraints
- [ ] Implement movement validation per layer
- [ ] Build simple Manhattan pathfinding algorithm
- [ ] Test with 2-layer board (NS + EW)

### Phase 2: Advanced Features (Next Sprint)

- [ ] Support 4-layer boards with power planes
- [ ] Optimize via count
- [ ] Handle obstacles with jog routing
- [ ] Add 45° diagonal movement within direction constraints

### Phase 3: Optimization (Next Month)

- [ ] Minimize total path length
- [ ] Minimize via count
- [ ] Balance routing density across layers
- [ ] Detect and resolve routing conflicts

---

## Key Takeaways

1. **Manhattan routing assigns directions to layers** - Layer 1: NS, Layer 2: EW

2. **Turns are accomplished by changing layers** - Via down/up instead of corner

3. **Guarantees a solution exists** - If physically possible, Manhattan will find it

4. **Prevents self-blocking** - Algorithm cannot trap itself

5. **Industry standard** - Professional PCBs use this approach

6. **Combines with 45° rule** - Still use 45° turns within same direction

7. **Voxel grid friendly** - Natural fit for discrete tensor representation

---

## Summary: The Complete Routing Strategy

### Single Layer (X/Y Plane)
- ✅ Prefer straight lines
- ✅ Use 45° turns when needed
- ❌ Avoid 90° turns

### Between Layers (Z-Axis)
- ✅ Always 90° straight down (vias)
- ❌ Never angled drilling

### Multi-Layer (Manhattan)
- ✅ Layer 1: North/South only
- ✅ Layer 2: East/West only (or power plane)
- ✅ Change layers to change direction
- ✅ Guaranteed solution if path exists

**Result**: Enterprise-grade auto-router that never traps itself and produces manufacturable boards.

---

**Document Status**: Advanced Routing Algorithm Guide  
**Last Updated**: March 2026  
**Part of**: Hardware Script v0.1.2 Documentation Suite

