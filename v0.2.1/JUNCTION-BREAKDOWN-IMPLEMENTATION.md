# Junction Breakdown Validation (P46)

## Overview

The junction breakdown checker validates that p-n junctions in semiconductor layouts are not operated beyond their safe voltage ratings. This prevents avalanche breakdown and permanent device damage.

## How It Works

### Detection
The DRC engine scans all semiconductor pours (materials with `category: semiconductor`) and calculates the voltage difference between each pour and the ground reference net:

```
Junction Voltage = |V_pour - V_ground|
```

If this exceeds the material's `max_junction_voltage` property, a P46 violation is triggered.

### Ground Reference
The checker requires a net with `classification: ground` to establish the substrate reference potential (typically 0V). Without this, junction voltages cannot be calculated.

### Material Ratings
Junction breakdown limits are defined in the material database:

```hw
export material N_Diffusion:
    category: semiconductor
    properties:
        max_junction_voltage: 1.98V      # Standard 1.8V domain + 10% margin

export material HV_N_Well:
    category: semiconductor
    properties:
        max_junction_voltage: 11.0V      # High voltage domain
```

## Example Violation

```hw
space JunctionTest:
    nets:
        Vin:  { classification: signal, potential: 3.3V }
        GND:  { classification: ground, potential: 0.0V }
    
    # Standard diffusion at 3.3V exceeds 1.98V rating
    add pour(N_Diffusion) named HotNode on layer: nwell:
        net: Vin
        dimensions: 2.0um by 2.0um
```

**Output:**
```
error[P46]: Junction breakdown violation for net Vin at [3000nm, 2000nm, 1000nm]
    N_Diffusion-to-GND junction biased at 3.30V (max: 1.98V)
= help: Use HV_N_Well for voltages above 2.0V or reduce operating voltage
```

## Compiler Enhancement: Multi-Material Layer Support

### Problem
The via library builder originally assumed a strict 1-to-1 mapping between layers and materials. It only generated vias for materials explicitly listed in the profile stackup:

```rust
// Old logic - only matched exact stackup materials
if stack_mat_name == from_material { /* generate via */ }
```

This failed for semiconductor layouts where multiple materials (N_Diffusion, P_Diffusion, N_Well) are placed on the same physical layer (e.g., `nwell`).

### Solution
Enhanced the via library generator to recognize material compatibility beyond exact matches:

**1. Exact Match** - Stackup material matches bridge material  
**2. Category Match** - Semiconductor materials can share semiconductor layers  
**3. Profile Allowlist** - Materials in `allowed_conductors` can use conductive layers

```rust
let is_material_compatible_with_layer = |mat_name: &str, layer_idx: usize| -> bool {
    if let Some(stack_mat) = stackup_manager.get_material_for_layer_index(layer_idx) {
        // Exact match
        if stack_mat == mat_name { return true; }
        
        // Semiconductor category match
        if let (Ok(m1), Ok(m2)) = (st.get_material(mat_name), st.get_material(&stack_mat)) {
            if m1.category == m2.category 
                && m1.category == MaterialCategory::Semiconductor {
                return true;
            }
        }
        
        // Profile allowlist check
        if profile.layer.allowed_conductors.contains(mat_name) {
            if material_is_conductive(stack_mat) { return true; }
        }
    }
    false
};
```

### Impact
Via generation increased from 2 vias (only Polysilicon, N_Well) to 30 vias (all semiconductor materials on all compatible layers), enabling proper connectivity for complex semiconductor layouts.

**Before:**
```
Via 0: Polysilicon [poly] -> Aluminum_Met1 [metal1]
Via 1: N_Well [nwell] -> Aluminum_Met1 [metal1]
Total: 2 vias
```

**After:**
```
Via 0: N_Diffusion [nwell] -> Aluminum_Met1 [metal1]
Via 1: P_Diffusion [nwell] -> Aluminum_Met1 [metal1]
Via 2: HV_N_Well [nwell] -> Aluminum_Met1 [metal1]
...
Total: 30 vias (all material combinations)
```

This allows the compiler to properly synthesize vias for layouts where different semiconductor materials are placed on the same layer, which is fundamental to ASIC design.

## Related
- **Error Code:** P46
- **Category:** Parasitic / DRC
- **Files:** `hwc-engine/src/design_rule_check/junction.rs`
- **Via Library:** `hwc-compiler/src/via_resolver/library/mod.rs`
