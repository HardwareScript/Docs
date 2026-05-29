# Hardware Script v0.1.2 - Physics to Constraints Translation

**Document Type**: Compiler Architecture Guide  
**Status**: External Research Integration  
**Last Updated**: March 2026

---

## The Ultimate "Aha!" Moment

You have realized the difference between **Topology** (drawing lines connecting dots) and **Physics** (designing a system that survives the real world).

If your compiler only knows geometry, the board might look perfect on screen, but the moment you plug it in:
- The copper melts
- Sparks fly across the air gaps
- The FR4 fiberglass catches fire

Your material database holds the exact keys to how your synthesizer must operate.

---

## The Core Problem

**You cannot run a live physics simulation at every single step.**

That would be computationally impossible. Instead, you must translate those physical material properties into **mathematical constraints** for your 3D voxel auto-router.

This document explains exactly how to do that.

---

## Translation #1: "Electron Jumping" = Dielectric Breakdown & Clearance Rules

### The Physical Phenomenon

When you mentioned electrons jumping between traces, that is called **Arcing** or **Dielectric Breakdown**.

**What happens**: If two copper traces carrying different voltages get too close, electrons literally jump through the air (or through the FR4 substrate) creating a spark.

### The Material Properties

Look at your data for Air and FR4:

```yaml
air:
  dielectric_strength_kv_mm: 3.0
  # It takes 3,000 Volts to jump across 1 millimeter of air

fr4:
  dielectric_strength_kv_mm: 20
  # It takes 20,000 Volts to punch through 1 millimeter of fiberglass
```

### How to Code This Into the Compiler

**You don't run a live physics simulation.**

Instead, your compiler does a **pre-calculation** based on the voltage of the net.

#### The Algorithm

**Step 1**: Identify the voltage difference between two nets

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

**Step 4**: Enforce in the router

When the "water" algorithm routes Net A, its "hitbox" or "collision box" isn't just the 1 voxel of copper. It projects a **"forcefield"** of empty voxels around it.

```
Net A occupies: [1, 50, 50]
Net A's forcefield: [1, 49, 49] to [1, 51, 51] (3×3 area)

The algorithm is strictly forbidden from routing Net B inside that forcefield.
```

### Implementation in Rust

```rust
struct Net {
    voltage: f32,
    occupied_voxels: Vec<(usize, usize, usize)>,
    clearance_voxels: Vec<(usize, usize, usize)>,
}

fn calculate_clearance(voltage_diff: f32, material: &Material) -> f32 {
    let safety_factor = 2.0;
    let min_clearance = voltage_diff / material.dielectric_strength_kv_mm;
    min_clearance * safety_factor
}

fn expand_clearance_zone(net: &mut Net, clearance_mm: f32, voxel_size: f32) {
    let clearance_voxels = (clearance_mm / voxel_size).ceil() as usize;
    
    for &(z, x, y) in &net.occupied_voxels {
        for dz in 0..=clearance_voxels {
            for dx in 0..=clearance_voxels {
                for dy in 0..=clearance_voxels {
                    net.clearance_voxels.push((z + dz, x + dx, y + dy));
                }
            }
        }
    }
}
```

---

## Translation #2: Heat and Conductivity = Trace Width & Ampacity

### The Physical Phenomenon

If you push 10 Amps through a single microscopic voxel of copper, it will act like a resistor, generate massive heat (**Joule heating**), and instantly vaporize.

### The Material Properties

Look at your data for Copper and FR4:

```yaml
copper:
  max_current_density_a_mm2: 35
  # Copper can only handle 35 Amps per square millimeter before failing

fr4:
  max_operating_temp_c: 130
  # If the copper gets hotter than 130°C, your board delaminates and burns
```

### How to Code This Into the Compiler

**Step 1**: Identify the current requirement

```
Motor connection: 10 Amps
```

**Step 2**: Calculate required cross-sectional area

```
Formula: area = current / max_current_density

10A / 35 A/mm² = 0.286 mm²
```

**Step 3**: Calculate required trace width

```
Copper thickness: 0.035mm (1oz copper, standard)
Required width: 0.286 mm² / 0.035mm = 8.17mm

Round up: 9mm trace width
```

**Step 4**: Enforce in the router

The router is no longer drawing a single-voxel thin wire. It must draw a **"thick river"**.

```
If voxel size = 1mm:
  Required width = 9mm
  Trace must be 9 voxels wide

If the space between components is only 5 voxels wide:
  ❌ ERROR: Insufficient space for required current width
  Suggestion: Move components apart or reduce current
```

### The IPC-2221 Formula (Industry Standard)

For more accurate calculations, use the IPC-2221 standard:

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
A = (10 / 0.127)^1.379
A = 78.74^1.379
A = 1.89 mm²

Width = 1.89 mm² / 0.035mm = 54mm

This is a THICK trace!
```

### Implementation in Rust

```rust
fn calculate_trace_width(current: f32, temp_rise: f32, is_external: bool) -> f32 {
    let k = if is_external { 0.048 } else { 0.024 };
    let copper_thickness = 0.035; // 1oz copper in mm
    
    let area = (current / (k * temp_rise.powf(0.44))).powf(1.0 / 0.725);
    let width = area / copper_thickness;
    
    width
}

fn enforce_trace_width(route: &Route, required_width_mm: f32, voxel_size: f32) -> Result<(), String> {
    let required_voxels = (required_width_mm / voxel_size).ceil() as usize;
    
    if route.available_width < required_voxels {
        return Err(format!(
            "Insufficient space: Need {}mm ({} voxels), have {}mm ({} voxels)",
            required_width_mm, required_voxels,
            route.available_width as f32 * voxel_size, route.available_width
        ));
    }
    
    Ok(())
}
```

---

## Translation #3: "Radioactivity" = EMI & Crosstalk

### The Physical Phenomenon

When you say "radioactivity," in PCB design, we refer to this as **EMI (Electromagnetic Interference)** or **RF (Radio Frequency) leakage**.

Every wire carrying a pulsing electrical signal acts like a tiny radio antenna broadcasting magnetic waves into the surrounding materials.

If another wire runs exactly parallel to it for too long, it acts as a receiving antenna, and the electrons "jump" over via **magnetic coupling** (called **Crosstalk**).

### The Material Properties

This is less about material properties and more about **geometry**:

```yaml
# Not in materials.yaml, but in routing rules
crosstalk:
  max_parallel_length_mm: 10
  min_spacing_for_high_speed_mm: 1.0
  preferred_crossing_angle_deg: 90
```

### How to Code This Into the Compiler

**The Algorithm Rule**: Add a **"Parallel Coupling Penalty"**.

If the algorithm routes Net B exactly parallel to a high-speed Net A for more than X voxels, the path cost exponentially increases.

This forces the router to either:
1. Cross paths at 90-degree angles (which cancels out the magnetic interference)
2. Move them further apart

### Implementation in Rust

```rust
fn calculate_crosstalk_penalty(
    net_a: &Route,
    net_b: &Route,
    max_parallel_length: f32,
    voxel_size: f32
) -> f32 {
    let parallel_length = calculate_parallel_length(net_a, net_b);
    let parallel_length_mm = parallel_length as f32 * voxel_size;
    
    if parallel_length_mm > max_parallel_length {
        // Exponential penalty
        let excess = parallel_length_mm - max_parallel_length;
        return (excess / max_parallel_length).exp();
    }
    
    0.0 // No penalty
}

fn calculate_parallel_length(net_a: &Route, net_b: &Route) -> usize {
    let mut parallel_count = 0;
    
    for segment_a in &net_a.segments {
        for segment_b in &net_b.segments {
            if segments_are_parallel(segment_a, segment_b) {
                parallel_count += segment_a.length.min(segment_b.length);
            }
        }
    }
    
    parallel_count
}
```

---

## The Architecture You Need: "Constraint-Driven Routing"

You are correct that you cannot blindly test this by watching the 3D water flow. It's too complex.

You need a **pipeline**. This is exactly how tools like Altium Designer or Cadence work.

### The 3-Phase Pipeline

#### Phase 1: The Constraint Manager (The Synthesizer)

**Before any routing happens**, the compiler takes the user's logic and your Material Database, and generates a **strict rulebook** for every single wire.

**Example Output**:

```
Net 1 (Power - 12V, 10A):
  - Must be 9mm wide (Current constraint)
  - Must have 0.5mm clearance from all other nets (Voltage constraint)
  - Must use external layer (Thermal constraint)

Net 2 (USB Data - 5V, 0.5A):
  - Can be 0.5mm wide (Current constraint)
  - Must have 0.1mm clearance (Voltage constraint)
  - Must maintain 90Ω impedance (Signal integrity constraint)
  - Maximum parallel length to other signals: 10mm (Crosstalk constraint)

Net 3 (Ground - 0V):
  - Use copper plane on Layer 2 (Power distribution strategy)
  - No routing required (connects via vias)
```

#### Phase 2: The Geometry Router (The Sandbox)

Your 3D voxel engine runs. It only cares about **geometry**, but the "size" of the objects it's moving is dictated by Phase 1.

It obeys:
- 45-degree angles
- Manhattan layer rules
- Collision boxes (from Phase 1)
- Trace widths (from Phase 1)

**It does NOT understand physics.** It just follows geometric rules.

#### Phase 3: The DRC (Design Rule Check)

Once the routing is finished, the compiler sweeps the entire 3D grid and does the **math checks** against your YAML file:

**Checks**:
1. "Did any copper voxels get too close to a high-voltage line?"
2. "Are there too many heat-generating components clustered together without an Aluminum heat sink?" (aluminum.thermal_conductivity_w_mk: 237)
3. "Are any traces too thin for their current?"
4. "Are any high-speed signals running parallel for too long?"

**If any check fails**:
```
❌ COMPILATION FAILED

Error at [1, 50, 60]: Clearance violation
  Net "Power_12V" (12V) is 0.3mm from Net "USB_Data" (5V)
  Required clearance: 0.5mm
  Actual clearance: 0.3mm
  
Suggestion: Reroute or move components apart
```

---

## Implementation Checklist

### Phase 1: Constraint Manager (High Priority)

- [ ] Parse material properties from YAML
- [ ] Calculate clearance requirements per net
- [ ] Calculate trace width requirements per net
- [ ] Generate constraint rulebook before routing
- [ ] Store constraints in AST

### Phase 2: Geometry Router (Current Work)

- [ ] Respect trace width constraints
- [ ] Respect clearance zones
- [ ] Implement collision detection with forcefields
- [ ] Add crosstalk penalty to pathfinding cost

### Phase 3: DRC (Next Sprint)

- [ ] Sweep entire grid after routing
- [ ] Check clearance violations
- [ ] Check trace width violations
- [ ] Check thermal clustering
- [ ] Generate detailed error reports with suggestions

---

## Key Takeaways

1. **Don't simulate physics** - Translate material properties into geometric constraints

2. **Three-phase pipeline** - Constraints → Routing → Validation

3. **Clearance = Voltage / Dielectric Strength** - Create "forcefields" around high-voltage nets

4. **Trace Width = Current / Current Density** - Make the router draw "thick rivers" for high current

5. **Crosstalk = Parallel Length Penalty** - Force perpendicular crossings or increased spacing

6. **Fail compilation on violations** - Treat physics like type checking, not warnings

---

## Summary: Your Next Steps

### Right Now
- ✅ Understand the three-phase pipeline
- ✅ Recognize that geometry and physics are separate concerns

### Next Sprint
- 🔄 Implement Phase 1: Constraint Manager
- 🔄 Calculate clearance and trace width before routing
- 🔄 Store constraints in the AST

### Next Month
- 📋 Implement Phase 3: DRC
- 📋 Add comprehensive validation checks
- 📋 Generate helpful error messages

**You don't need to build a literal quantum-physics simulator. You just need to translate the physics from your YAML file into geometric boundaries before you let the auto-router find the path.**

---

**Document Status**: Compiler Architecture Guide  
**Last Updated**: March 2026  
**Next Document**: Netlist and Routing Philosophy

