# Hardware Script v0.1.2 - Routing Geometry Rules

**Document Type**: Manufacturing Constraints Guide  
**Status**: External Research Integration  
**Last Updated**: March 2026

---

## The Fundamental Question

Can traces move straight down (90 degrees) into the board, or do they need to follow the 45-degree rule in all dimensions?

**Answer**: Moving straight down (or up) into the board is perfectly fine, and it **must** be 90 degrees.

---

## The Critical Distinction: X/Y Plane vs. Z-Axis

You have to separate the **X/Y plane** (moving flat along a layer) from the **Z-axis** (moving up and down between layers).

They are manufactured using two **completely different physical processes**.

---

## Rule 1: The X/Y Plane (Flat Routing) = Etching

### The Manufacturing Process

When a trace is moving horizontally along a flat copper layer (like driving on a highway), it is created using the **chemical etching process**.

**How it works**:
1. Start with a flat sheet of copper laminated to fiberglass (FR4)
2. Apply photoresist (light-sensitive chemical)
3. Shine UV light through a mask (your circuit pattern)
4. Wash away exposed photoresist
5. Submerge in acid bath
6. Acid eats away unprotected copper
7. Remove remaining photoresist
8. Result: Copper traces remain where you wanted them

### Why 90° Corners Are Bad on X/Y Plane

Because it is **flat liquid acid eating away flat copper**, sharp 90° corners create two problems:

#### Problem 1: Acid Traps

At a sharp 90° corner, acid can pool and over-etch, creating:
- Rounded corners (loss of precision)
- Weakened traces
- Inconsistent trace width


#### Problem 2: Signal Reflection

Because the electrons are zooming along a flat plane, a 90° flat wall causes a microscopic "crash" (**signal reflection**).

**What happens**:
- High-speed signals bounce back at sharp corners
- Creates electromagnetic interference (EMI)
- Degrades signal quality
- Can cause data errors in high-speed circuits

### The Rule for X/Y Plane

**On the X/Y plane, turns must be 45°.**

**Good (45° turns)**:
```
    ┌─────
    │
────┘

Smooth transition, no acid traps, minimal signal reflection
```

**Bad (90° turns)**:
```
    ┌─────
    │
────┘

Sharp corner, acid pools, signal reflects
```

### Implementation in Voxel Grid

When routing on a single layer (Z constant), the algorithm should:

1. **Prefer straight lines** (North, South, East, West)
2. **Use 45° angles for turns** (Northeast, Northwest, Southeast, Southwest)
3. **Avoid 90° turns** (direct North→East, South→West, etc.)

**Example path on Layer 1**:
```
Start: [1, 10, 10]
  → [1, 15, 10]  # Move East (straight)
  → [1, 20, 15]  # Move Northeast (45° turn)
  → [1, 20, 25]  # Move North (straight)
End: [1, 20, 25]
```

---

## Rule 2: The Z-Axis (Moving Up/Down) = Drilling

### The Manufacturing Process

When your path needs to go from the component down into the board (or move from Layer 1 down to Layer 2), it is no longer a flat etched line.

It is a **microscopic hole drilled straight down** through the fiberglass board, which is then **electroplated with copper**.

This vertical pipe is called a **Via**.

**How it works**:
1. Stack all layers of the PCB
2. Use CNC drill to bore vertical holes
3. Electroplate the hole walls with copper
4. Result: Conductive tube connecting layers

### Why 90° Is Required on Z-Axis

#### Reason 1: Physical Drilling Constraints

**Drills go straight down.**

Standard PCB manufacturers do not drill holes at 45-degree angles. The drill bit is perpendicular to the board surface.

**Attempting angled drilling would**:
- Require specialized equipment (expensive)
- Create weak, unreliable connections
- Not be supported by any standard PCB manufacturer

#### Reason 2: No Acid Trap Problem

Because a via is a **vertical cylinder** (a tube of copper), liquid acid doesn't pool on the outside corners of it in the same way it does on a flat trace.

The via is a 3D structure, not a 2D trace, so the etching concerns don't apply.

### The Rule for Z-Axis

**On the Z-axis, transitions must be 90° straight down.**

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

### Implementation in Voxel Grid

When changing layers (Z changes), the algorithm must:

1. **Keep X and Y coordinates constant**
2. **Change only Z coordinate**
3. **Move exactly one layer at a time** (or multiple layers for through-hole vias)

**Example via**:
```
Layer 1: [1, 20, 15]  # Trace on top layer
Via:     [2, 20, 15]  # Drill straight down (X,Y unchanged)
Layer 2: [2, 20, 15]  # Continue trace on bottom layer
```

**Invalid via** (X or Y changed):
```
Layer 1: [1, 20, 15]
Via:     [2, 21, 16]  # ❌ ERROR: X and Y changed!
Layer 2: [2, 21, 16]

This would require angled drilling - not possible!
```

---

## Visual Example: Side Profile View

### The Correct Routing Pattern

```
Side view of PCB:

Component Pin
     │
     │ (90° straight down - via or component lead)
     ●─────────────────● (horizontal trace on Layer 1)
     │                 │
     │                 │ (90° straight down - via)
     ●─────────────────● (horizontal trace on Layer 2)
                       │
                       │ (90° straight down - via)
                       ●── (connects to another component)

Nodes 1 and 6: Vertical drops (90° to board)
Nodes 1-6: Horizontal path (can use 45° turns when viewed from top)
```

### Key Observations

**The Vertical Drops (Nodes 1 and 6)**:
- Represent physical metal pins of components
- Or vias drilled into the board
- Going straight up/down at a perfect 90° angle to the board is exactly right

**The Horizontal Path (Nodes 1 through 6)**:
- Represents the etched copper trace on a specific flat layer
- As long as this specific flat line doesn't take any sharp 90-degree left/right turns when looking at it from the top down, you are perfectly safe

---

## Summary: Algorithm Movement Rules

### Moving North/South/East/West (X/Y Plane)

**Preferred**: Straight lines
```
[1, 10, 10] → [1, 20, 10]  # Move East (straight)
[1, 20, 10] → [1, 20, 20]  # Move North (straight)
```

**If you must turn**: Use 45° angles
```
[1, 10, 10] → [1, 15, 15]  # Move Northeast (45°)
[1, 15, 15] → [1, 20, 20]  # Move Northeast (45°)
```

**Avoid**: 90° turns on the same layer
```
[1, 10, 10] → [1, 20, 10]  # Move East
[1, 20, 10] → [1, 20, 20]  # Move North (90° turn - avoid!)
```

### Moving Up/Down (Z-Axis)

**Always**: Drop straight down at exactly 90°
```
[1, 20, 15] → [2, 20, 15]  # Via down (X,Y unchanged)
[2, 20, 15] → [3, 20, 15]  # Continue down (X,Y unchanged)
```

**Never**: Change X or Y when changing Z
```
[1, 20, 15] → [2, 21, 16]  # ❌ INVALID: X and Y changed!
```

---

## Implementation Checklist

### Phase 1: Basic Routing Rules (High Priority)

- [ ] Implement straight-line routing on X/Y plane
- [ ] Implement 45° turn support on X/Y plane
- [ ] Detect and warn about 90° turns on X/Y plane
- [ ] Implement 90° via generation (Z-axis only)
- [ ] Validate that vias keep X,Y constant

### Phase 2: Advanced Routing (Next Sprint)

- [ ] Add pathfinding cost penalties for 90° turns
- [ ] Prefer 45° turns over 90° turns in algorithm
- [ ] Optimize via placement (minimize via count)
- [ ] Support blind vias (don't go through all layers)
- [ ] Support buried vias (internal layers only)

### Phase 3: Validation (Next Month)

- [ ] DRC check for 90° turns on X/Y plane
- [ ] DRC check for angled vias (X,Y changed with Z)
- [ ] Generate warnings for suboptimal routing
- [ ] Suggest alternative paths with better geometry

---

## Key Takeaways

1. **X/Y plane (horizontal)**: Prefer straight lines, use 45° turns, avoid 90° turns

2. **Z-axis (vertical)**: Always 90° straight down, never angled

3. **Different manufacturing processes**: Etching (X/Y) vs. Drilling (Z)

4. **Physical constraints**: Drills can't bore at angles

5. **Signal integrity**: 45° turns reduce reflection and EMI

6. **Voxel grid rule**: When Z changes, X and Y must stay constant

---

## Compiler Error Examples

### Error: 90° Turn on X/Y Plane

```
⚠️ WARNING: Sharp 90° turn detected
   Location: [1, 20, 10] → [1, 20, 20]
   Layer: 1 (Top)
   
   Sharp corners can cause:
   - Signal reflection (high-speed signals)
   - Acid traps during manufacturing
   - EMI issues
   
   Suggestion: Use 45° turn instead
   Alternative path: [1, 20, 10] → [1, 25, 15] → [1, 20, 20]
```

### Error: Angled Via

```
❌ ERROR: Invalid via geometry
   From: [1, 20, 15]
   To:   [2, 21, 16]
   
   Via must be drilled straight down (90°)
   X and Y coordinates must remain constant
   
   Current: X changed from 20 to 21, Y changed from 15 to 16
   Required: X=20, Y=15 (unchanged)
   
   Correct via: [1, 20, 15] → [2, 20, 15]
```

---

**Document Status**: Manufacturing Constraints Guide  
**Last Updated**: March 2026  
**Part of**: Hardware Script v0.1.2 Documentation Suite

