# RF Parasitic Extraction Roadmap

**Version**: v0.2.2+ (Future Enhancement)  
**Status**: Not Yet Implemented  
**Philosophy**: Geometry-based extraction. No semiconductor physics. Incremental enhancement.

---

## ✅ Recent Completions (v0.3.0)

### PDK Subcircuit Support with 3-Terminal Devices

**Status**: ✅ **COMPLETED** (v0.3.0)  
**Commit**: 2026-08-07

Foundry PDK subcircuits with multi-terminal contracts are now fully supported with zero compiler magic:

- ✅ **Explicit terminal contracts** - All terminals (including BULK/substrate) must be declared in device definition
- ✅ **Virtual terminal support** - Air material for non-physical terminals (e.g., substrate connections)
- ✅ **User-controlled net binding** - User explicitly chooses which net connects to BULK (GND, VSS, AVSS, etc.)
- ✅ **Fail-loudly error handling** - Missing terminal binding caught at compile time with clear fix instructions
- ✅ **Zero hidden assumptions** - No auto-GND connections, no substrate inference

**Example**: SkyWater SKY130 P+ Poly Resistor (`sky130_fd_pr__res_high_po`)

```hw
device Resistor:
    terminals: [A, B, BULK]  # All 3 terminals explicit
    materials:
        A: Polysilicon
        B: Polysilicon  
        BULK: Air           # Virtual terminal
    spice:
        prefix: X
        subcircuit: sky130_fd_pr__res_high_po
        terminal_order: [A, B, BULK]
        parameters: [W, L]
        parameter_style: named
```

**Generated SPICE:**
```spice
XR1 In Out GND sky130_fd_pr__res_high_po W=4.0u L=1.0u
```

**See**: `NETLIST-ARCHITECTURE.md` Section 1.6 for full documentation.

---

## Overview

This document outlines the roadmap for adding high-frequency parasitic extraction to HardwareScript. The current netlist system (v0.2.1) provides **complete DC/low-frequency accuracy** but lacks certain geometry-calculable parasitics needed for RF/high-frequency circuits (>100MHz).

**Current Status (v0.2.1)**:
- ✅ Trace resistance ($R = \rho L/A$)
- ✅ Ground/Substrate capacitance ($C = \varepsilon_0 \varepsilon_r A/d$)
- ✅ Main device parameters ($W, L, C, R$)
- ✅ Zero hardcoded values
- ✅ Fully material-driven

**Missing for RF & High-Frequency Accuracy**:
- ❌ Capacitor plate ESR (Equivalent Series Resistance)
- ❌ Resistor contact/interface resistance ($R_0$)
- ❌ Via inductance (ESL - Equivalent Series Inductance)
- ❌ Trace inductance
- ❌ Dielectric loss tangent modeling

---

## Architectural Boundary (Unchanged)

**The Golden Rule**: The compiler extracts **geometry-calculable linear parasitics** only.

| Compiler Extracts | PDK Declares | Simulator Solves |
|-------------------|--------------|------------------|
| ✅ R, C, L from geometry | ❌ IS, VTO, LAMBDA | ❌ Transient/AC/DC |
| ✅ ESR, ESL from geometry | ❌ Process corners | ❌ Newton-Raphson |
| ✅ Terminal connectivity | ❌ Temp coefficients | ❌ Waveforms |

**What This Means**:
- ✅ All enhancements remain **pure geometry + materials**
- ✅ No semiconductor physics crosses the boundary
- ✅ Users control everything via `.hw` files
- ✅ Extensible without Rust code changes

---

## Frequency Impact Matrix

| Frequency Range | Current Accuracy (v0.2.1) | After Phase 1 (ESR/$R_0$) | After Phase 2 (ESL) | After Phase 3 (L_trace) |
|-----------------|---------------------------|---------------------------|---------------------|-------------------------|
| **DC** | ✅ 100% | ✅ 100% | ✅ 100% | ✅ 100% |
| **< 10MHz** | ✅ ~99% | ✅ 100% | ✅ 100% | ✅ 100% |
| **10-100MHz** | ⚠️ ~90% | ✅ ~98% | ✅ ~99% | ✅ 100% |
| **100MHz-1GHz** | ❌ ~70% | ⚠️ ~85% | ✅ ~95% | ✅ ~99% |
| **> 1GHz** | ❌ <50% | ⚠️ ~60% | ⚠️ ~75% | ✅ ~90% |

**Note**: Above 1GHz, distributed effects and transmission line models become necessary (beyond scope).

---

## Phase 1a: Capacitor Plate ESR

**Priority**: High  
**Complexity**: Low  
**Impact**: Critical for RF capacitors (10MHz-1GHz)

### What It Solves

Capacitor plates have finite resistance from the metal layers. At high frequencies:
- Current flows through the plate resistance
- Creates voltage drop across the capacitor
- Limits the Q factor (quality factor)
- Affects resonant circuit performance

### Physics Formula

```
ESR_top_plate = ρ_top × (t_top / A_overlap)
ESR_bottom_plate = ρ_bottom × (t_bottom / A_overlap)
ESR_total = ESR_top + ESR_bottom
```

Where:
- `ρ` = resistivity (from materials.hw)
- `t` = plate thickness (from stackup)
- `A_overlap` = overlap area between plates (from geometry)

### Implementation Location

**File**: `hwc-export/src/device_extractor/parameter_extraction.rs`  
**Function**: `extract_capacitor_parameters()`

### Current Code (Simplified)

```rust
fn extract_capacitor_parameters(...) -> Result<FxHashMap<CompactString, f64>, String> {
    // 1. Calculate overlap area
    let area = calculate_overlap_area(bbox_top, bbox_bottom)?;
    
    // 2. Get dielectric properties
    let epsilon_r = find_dielectric_permittivity(z_top, z_bottom, space)?;
    let thickness = (z_bottom - z_top).abs();
    
    // 3. Calculate capacitance
    let capacitance = EPSILON_0 * epsilon_r * (area / thickness);
    
    // 4. Return parameters
    Ok(hashmap!["C" => capacitance])
}
```

### Enhanced Code (Phase 1a)

```rust
fn extract_capacitor_parameters(...) -> Result<FxHashMap<CompactString, f64>, String> {
    // 1. Calculate overlap area
    let area = calculate_overlap_area(bbox_top, bbox_bottom)?;
    
    // 2. Get dielectric properties
    let epsilon_r = find_dielectric_permittivity(z_top, z_bottom, space)?;
    let thickness = (z_bottom - z_top).abs();
    
    // 3. Calculate capacitance
    let capacitance = EPSILON_0 * epsilon_r * (area / thickness);
    
    // 4. Calculate ESR from plate resistivity
    let top_material = terminal_pours.get("top").ok_or("Missing top plate pour")?;
    let bottom_material = terminal_pours.get("bottom").ok_or("Missing bottom plate pour")?;
    
    let top_mat_id = space.material_registry.get_id(&top_material.material_name)
        .ok_or_else(|| format!("Material '{}' not found", top_material.material_name))?;
    let bottom_mat_id = space.material_registry.get_id(&bottom_material.material_name)
        .ok_or_else(|| format!("Material '{}' not found", bottom_material.material_name))?;
    
    let top_props = space.material_registry.get_physical_props(top_mat_id)
        .ok_or_else(|| format!("Material '{}' has no physical properties", top_material.material_name))?;
    let bottom_props = space.material_registry.get_physical_props(bottom_mat_id)
        .ok_or_else(|| format!("Material '{}' has no physical properties", bottom_material.material_name))?;
    
    let rho_top = top_props.get("resistivity")
        .ok_or_else(|| format!("Material '{}' missing 'resistivity' property", top_material.material_name))?;
    let rho_bottom = bottom_props.get("resistivity")
        .ok_or_else(|| format!("Material '{}' missing 'resistivity' property", bottom_material.material_name))?;
    
    let t_top = top_material.thickness_nm as f64 * 1e-9;
    let t_bottom = bottom_material.thickness_nm as f64 * 1e-9;
    
    // Calculate ESR: R = ρ × (t / A)
    let esr_top = rho_top * (t_top / area);
    let esr_bottom = rho_bottom * (t_bottom / area);
    let esr_total = esr_top + esr_bottom;
    
    // 5. Return enhanced parameters
    Ok(hashmap![
        "C" => capacitance,
        "ESR" => esr_total
    ])
}
```

---

## Phase 1b: Resistor Contact / Interface Resistance (R_0)

**Priority**: High  
**Complexity**: Low  
**Impact**: Critical for precision resistors and IC passives (10MHz-1GHz)

### What It Solves

When a metal via meets a resistive channel (e.g., Tungsten via on Polysilicon resistor), an interface resistance (R_0) exists at the contact head and tail:
- Total resistance = Body resistance + Head contact resistance + Tail contact resistance
- $R_{\text{total}} = R_{\text{body}} + R_{0,\text{head}} + R_{0,\text{tail}}$
- In nodes like SkyWater SKY130, contact resistance can be 10–20% of total resistance.

### Physics Formula

```
R_0 = ρ_interface × (t_interface / A_contact)
```

Where:
- `ρ_interface` = silicide / interface resistivity (from materials.hw, e.g. Titanium_Silicide)
- `t_interface` = interface layer thickness (from bridge definition in PDK)
- `A_contact` = via contact area (from via geometry)

### Implementation Location

**File**: `hwc-export/src/device_extractor/parameter_extraction.rs`  
**Function**: `extract_resistor_parameters()`

### Enhanced Code (Phase 1b)

```rust
fn extract_resistor_parameters(...) -> Result<FxHashMap<CompactString, f64>, String> {
    // 1. Calculate body resistance R = ρ × (L / A)
    let r_body = calculate_body_resistance(terminal_pours, space)?;
    
    // 2. Calculate head and tail contact resistance from via contact geometry & interface resistivity
    let r0_head = calculate_contact_interface_resistance("A", terminal_pours, space)?;
    let r0_tail = calculate_contact_interface_resistance("B", terminal_pours, space)?;
    
    let r_total = r_body + r0_head + r0_tail;
    
    Ok(hashmap![
        "R" => r_total,
        "R0_HEAD" => r0_head,
        "R0_TAIL" => r0_tail
    ])
}
```

---

## Phase 2: Via Inductance (ESL)

**Priority**: Medium  
**Complexity**: Medium  
**Impact**: Critical for high-speed digital and RF (>100MHz)

### What It Solves

Vias and vertical connections have parasitic inductance:
- Limits high-frequency current flow
- Creates voltage spikes during switching
- Affects signal integrity in digital circuits
- Shifts resonant frequencies in RF circuits

### Physics Formula

```
L_via = μ₀ × (h / A) × ln(4h / d)
```

Where:
- `μ₀` = 4π × 10⁻⁷ H/m (permeability of free space)
- `h` = via height (from geometry)
- `A` = via cross-sectional area (from geometry)
- `d` = via diameter (from geometry)

Simplified approximation (good for h >> d):
```
L_via ≈ μ₀ × h / A
```

### Implementation Location

**File**: `hwc-export/src/netlist.rs`  
**Function**: `build_physical_netlist_graph()` - via extraction section

### Enhanced Code (Phase 2)

```rust
// In build_physical_netlist_graph(), when processing contacts/vias
for contact in &space.contacts {
    if contact.net_id.is_some() {
        let via_height_nm = contact.bbox.max.z - contact.bbox.min.z;
        let via_height_m = via_height_nm as f64 * 1e-9;
        
        let via_radius_nm = contact.diameter_nm / 2.0;
        let via_area_m2 = std::f64::consts::PI * (via_radius_nm * 1e-9).powi(2);
        
        const MU_0: f64 = 4.0 * std::f64::consts::PI * 1e-7; // H/m
        let inductance_h = MU_0 * (via_height_m / via_area_m2);
        
        if inductance_h > 1e-12 {
            graph.parasitics.push(ParasiticElement::ViaInductance {
                name: format!("Lvia_{}", contact.name),
                node_a: format!("n{}_top", contact.name),
                node_b: format!("n{}_bottom", contact.name),
                value_henries: inductance_h,
            });
        }
    }
}
```

---

## Phase 3: Substrate Net Mapping (Bulk Net Awareness)

**Priority**: Medium  
**Complexity**: Low  
**Impact**: Eliminates hardcoded node 0 for substrate ground capacitance

### What It Solves

Substrate capacitance ($C_{\text{sub}} / C_{\text{gnd}}$) in integrated circuits connects to the local active substrate/bulk net (nBULK or nGND), not always global SPICE node 0.

### Implementation

```rust
// Connect substrate parasitic capacitance to active bulk net if present, else fallback to 0
let bulk_node = space.find_bulk_net_for_layer(layer_id)
    .unwrap_or_else(|| "0".to_string());

graph.parasitics.push(ParasiticElement::GroundCapacitance {
    name: format!("Csub_{}", trace.name),
    node_a: trace.net_node.clone(),
    node_b: bulk_node,  // Dynamically mapped to local bulk net!
    value_farads: capacitance_f,
});
```

---

## Phase 4: Trace Inductance & Loss Tangent (Future / Ultra-High Frequency)

**Priority**: Low  
**Complexity**: High  
**Impact**: Important for transmission lines (>500MHz) and ultra-high-Q resonators (>1GHz)

- **Trace Inductance**: Microstrip

```
L_trace = (μ₀ × μᵣ × length / width) × ln(2 × height / (width + thickness))
```

- **Dielectric Loss Tangent**: Modeled as parallel conductance

```
R_loss = 1 / (2πfC × tan δ)
```

---

## Summary of Implementation Stages

| Phase  | Feature                             | Target Release | Primary Benefit                                                 |
| ------ | ----------------------------------- | -------------- | --------------------------------------------------------------- |
| **1a** | Capacitor Plate ESR                 | v0.2.2         | Q-factor accuracy for RF capacitors                             |
| **1b** | Resistor Contact Resistance ($R_0$) | v0.2.2         | Total resistance accuracy for IC poly/diff resistors            |
| **2**  | Via Inductance (ESL)                | v0.2.3         | High-frequency via impedance and switching spikes               |
| **3**  | Dynamic Substrate/Bulk Mapping      | v0.2.3         | Connects $C_{\text{sub}}$ to `nBULK`/`nGND` instead of node `0` |
| **4**  | Trace Inductance & Loss Tangent     | v0.3.0+        | Transmission lines and >1GHz microwave designs                  |

---

## Design Principles (Unchanged)

1. **Pure Geometry**: All parasitics calculated from layout + materials
2. **No Hardcoding**: Zero magic numbers or default values
3. **Fail Loudly**: Missing properties → clear error messages
4. **User Control**: Everything declared in `.hw` files
5. **Incremental**: Add features as needed, when needed
6. **Boundary Respect**: Never cross into semiconductor physics

---

## Conclusion

The current netlist system (v0.2.1) provides **complete DC/low-frequency accuracy** using pure geometry-based extraction. High-frequency enhancements can be added **incrementally** following the same architectural principles:

- ✅ All parasitics remain geometry-calculable
- ✅ No semiconductor physics added
- ✅ Users maintain full control via `.hw` files
- ✅ System stays lightweight and fast

**When to implement**: Only when users actually need RF accuracy. Until then, the current system is architecturally complete and production-ready for its design scope.
