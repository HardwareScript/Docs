# HardwareScript v0.3.2: Standard Library & Certified SKY130 PCell Reference Implementation

**Document Type:** Standard Library Architecture & Reference Implementations  
**Target Version:** v0.3.2 (Production-Locked Standard)  
**Status:** Authoritative & Canonical  
**Package Root:** `@std/`  
**Foundry PDK Target:** SkyWater SKY130 (130nm 1.8V/3.3V CMOS)  
**Date:** September 2026  

---

## 1. The Pure PCell Architectural Contract

In HardwareScript v0.3.2, every parameterized cell (PCell) is a **pure mathematical function**:

1. **Zero Ambient Mutation:** A PCell cannot read or mutate the surrounding `space` block or global environment.
2. **Local Relative Coordinates:** All geometries inside a PCell are defined relative to the local origin `[0, 0]` in signed 64-bit picometers.
3. **Blake3 Flyweight Memoization:** The compiler hashes the PCell function identifier and parameters using Blake3. Identical calls return cached geometry references in $O(1)$ time.
4. **Explicit Placement:** Instantiation requires calling `space.place(cell_layout, at: [X, Y])`.
5. **Fluent Affine Transformations:** A `CellLayout` supports pure coordinate transformations before placement:  
   `space.place(nmos.rotate_90().mirror_y(), at: [10.0um, 20.0um])`

---

## 2. Standard PDK Imports & Type Definitions

```hardware
# stdlib/pdk/sky130/types.hw
import * from @std/primitives/units

export struct TransistorPort {
    center: Point2D,
    layer: String,
    net: Net,
}

export struct Sky130Rules {
    diff_min_width: Measurement,
    diff_min_spacing: Measurement,
    poly_min_width: Measurement,
    poly_min_spacing: Measurement,
    poly_overhang_diff: Measurement,
    licon_size: Measurement,
    licon_spacing: Measurement,
    nwell_margin_diff: Measurement,
    npc_enclosure_poly: Measurement,
}

export const SKY130_RULES = Sky130Rules {
    diff_min_width: 150nm,
    diff_min_spacing: 270nm,
    poly_min_width: 150nm,
    poly_min_spacing: 210nm,
    poly_overhang_diff: 200nm,
    licon_size: 170nm,
    licon_spacing: 170nm,
    nwell_margin_diff: 600nm,
    npc_enclosure_poly: 90nm,
}
```

---

## 3. Pure Transistor Generators (`nmos.hw` & `pmos.hw`)

### 3.1 Pure Parameterized NMOS Transistor (`@std/pdk/sky130/nmos.hw`)

```hardware
# stdlib/pdk/sky130/nmos.hw
import * from @std/primitives/units
import { SKY130_RULES } from "./types"

export fn sky130_nmos(
    W: Measurement = 1.0um,
    L: Measurement = 150nm,
    sd_len: Measurement = 750nm
) -> CellLayout {
    assert(L >= SKY130_RULES.poly_min_width, "NMOS gate length below 150nm foundry minimum!")
    assert(W >= SKY130_RULES.diff_min_width, "NMOS channel width below 150nm foundry minimum!")

    let mut cell = CellLayout::new(name: "sky130_nmos")
    let diff_len = (2 * sd_len) + L
    let poly_overhang = SKY130_RULES.poly_overhang_diff
    let gate_head_size = 400nm
    let via_pitch = 400nm

    # 1. Register SPICE 4-Terminal Compact Model Contract
    cell.add_device(
        device_type: "sky130_fd_pr__nfet_01v8",
        name: "M",
        terminals: ["D", "G", "S", "B"],
        params: { W: W, L: L }
    )

    # 2. Local Origin-Relative Mask Geometries (Origin [0, 0] at gate channel center)
    # Active Diffusion Mask
    cell.add_polygon(layer: "diff", rect: [-diff_len/2, -W/2, diff_len, W])

    # N-Diffusion Implant Mask (NSDM)
    cell.add_polygon(layer: "nsdm", rect: [-diff_len/2 - 130nm, -W/2 - 130nm, diff_len + 260nm, W + 260nm])

    # Polysilicon Gate Stripe
    cell.add_polygon(layer: "poly", rect: [-L/2, -W/2 - poly_overhang, L, W + (2 * poly_overhang)])

    # Off-Channel Gate Landing Head & NPC Mask
    let head_y = W/2 + poly_overhang
    cell.add_polygon(layer: "poly", rect: [-gate_head_size/2, head_y, gate_head_size, gate_head_size])
    cell.add_polygon(layer: "npc",  rect: [-gate_head_size/2 - 90nm, head_y - 90nm, gate_head_size + 180nm, gate_head_size + 180nm])

    # 3. Source & Drain Contact Arrays (Local Interconnect Li1 & Metal1)
    let src_x = -diff_len/2 + sd_len/2
    let drn_x =  diff_len/2 - sd_len/2

    cell.add_polygon(layer: "li1", rect: [src_x - 200nm, -W/2, 400nm, W])
    cell.add_polygon(layer: "li1", rect: [drn_x - 200nm, -W/2, 400nm, W])

    let num_vias = max(1, int((W - 200nm) / via_pitch))
    let via_offset = (num_vias - 1) * via_pitch / 2

    for i in 0..num_vias {
        let vy = -via_offset + (i * via_pitch)
        cell.add_contact(from: "diff", to: "li1",    at: [src_x, vy], diameter: 170nm)
        cell.add_contact(from: "li1",  to: "metal1", at: [src_x, vy], diameter: 170nm)
        cell.add_contact(from: "diff", to: "li1",    at: [drn_x, vy], diameter: 170nm)
        cell.add_contact(from: "li1",  to: "metal1", at: [drn_x, vy], diameter: 170nm)
    }

    # 4. Expose Relative Electrical Port Anchors
    cell.add_port(name: "S", at: [src_x, 0nm], layer: "metal1")
    cell.add_port(name: "D", at: [drn_x, 0nm], layer: "metal1")
    cell.add_port(name: "G", at: [0nm, head_y + gate_head_size/2], layer: "poly")

    return cell
}
```

### 3.2 Pure Parameterized PMOS Transistor (`@std/pdk/sky130/pmos.hw`)

```hardware
# stdlib/pdk/sky130/pmos.hw
import * from @std/primitives/units
import { SKY130_RULES } from "./types"

export fn sky130_pmos(
    W: Measurement = 2.0um,
    L: Measurement = 150nm,
    sd_len: Measurement = 750nm
) -> CellLayout {
    assert(L >= SKY130_RULES.poly_min_width, "PMOS gate length below 150nm foundry minimum!")
    assert(W >= SKY130_RULES.diff_min_width, "PMOS channel width below 150nm foundry minimum!")

    let mut cell = CellLayout::new(name: "sky130_pmos")
    let diff_len = (2 * sd_len) + L
    let poly_overhang = SKY130_RULES.poly_overhang_diff
    let nwell_margin = SKY130_RULES.nwell_margin_diff
    let gate_head_size = 400nm
    let via_pitch = 400nm

    # 1. Register Compact Model
    cell.add_device(
        device_type: "sky130_fd_pr__pfet_01v8",
        name: "M",
        terminals: ["D", "G", "S", "B"],
        params: { W: W, L: L }
    )

    # 2. Local Relative Geometries
    # N-Well Mask
    cell.add_polygon(layer: "nwell", rect: [-diff_len/2 - nwell_margin, -W/2 - nwell_margin, diff_len + 2*nwell_margin, W + 2*nwell_margin])

    # Active Diffusion & P-Diffusion Implant (PSDM)
    cell.add_polygon(layer: "diff", rect: [-diff_len/2, -W/2, diff_len, W])
    cell.add_polygon(layer: "psdm", rect: [-diff_len/2 - 130nm, -W/2 - 130nm, diff_len + 260nm, W + 260nm])

    # Polysilicon Gate & Off-Channel Safe Landing Head
    cell.add_polygon(layer: "poly", rect: [-L/2, -W/2 - poly_overhang, L, W + (2 * poly_overhang)])
    let head_y = -W/2 - poly_overhang - gate_head_size
    cell.add_polygon(layer: "poly", rect: [-gate_head_size/2, head_y, gate_head_size, gate_head_size])
    cell.add_polygon(layer: "npc",  rect: [-gate_head_size/2 - 90nm, head_y - 90nm, gate_head_size + 180nm, gate_head_size + 180nm])

    # Source & Drain Contacts
    let src_x = -diff_len/2 + sd_len/2
    let drn_x =  diff_len/2 - sd_len/2

    cell.add_polygon(layer: "li1", rect: [src_x - 200nm, -W/2, 400nm, W])
    cell.add_polygon(layer: "li1", rect: [drn_x - 200nm, -W/2, 400nm, W])

    let num_vias = max(1, int((W - 200nm) / via_pitch))
    let via_offset = (num_vias - 1) * via_pitch / 2

    for i in 0..num_vias {
        let vy = -via_offset + (i * via_pitch)
        cell.add_contact(from: "diff", to: "li1",    at: [src_x, vy], diameter: 170nm)
        cell.add_contact(from: "li1",  to: "metal1", at: [src_x, vy], diameter: 170nm)
        cell.add_contact(from: "diff", to: "li1",    at: [drn_x, vy], diameter: 170nm)
        cell.add_contact(from: "li1",  to: "metal1", at: [drn_x, vy], diameter: 170nm)
    }

    # Relative Ports
    cell.add_port(name: "S", at: [src_x, 0nm], layer: "metal1")
    cell.add_port(name: "D", at: [drn_x, 0nm], layer: "metal1")
    cell.add_port(name: "G", at: [0nm, head_y + gate_head_size/2], layer: "poly")

    return cell
}
```

---

## 4. Pure Substrate Taps (`tap.hw`) & Contact Arrays (`via.hw`)

### 4.1 Pure Substrate & Well Tap (`@std/pdk/sky130/tap.hw`)

```hardware
# stdlib/pdk/sky130/tap.hw
import * from @std/primitives/units

export enum TapType {
    P_Sub,
    N_Well
}

export fn sky130_tap(type: TapType, size: Measurement = 600nm) -> CellLayout {
    let mut cell = CellLayout::new(name: "sky130_tap")
    let implant_margin = 130nm

    cell.add_polygon(layer: "tap", rect: [-size/2, -size/2, size, size])

    match type {
        TapType.P_Sub => {
            cell.add_polygon(layer: "psdm", rect: [-size/2 - implant_margin, -size/2 - implant_margin, size + 2*implant_margin, size + 2*implant_margin])
            cell.add_polygon(layer: "diff", rect: [-size/2, -size/2, size, size])
        },
        TapType.N_Well => {
            cell.add_polygon(layer: "nsdm", rect: [-size/2 - implant_margin, -size/2 - implant_margin, size + 2*implant_margin, size + 2*implant_margin])
            cell.add_polygon(layer: "diff", rect: [-size/2, -size/2, size, size])
        }
    }

    cell.add_polygon(layer: "li1",    rect: [-size/2, -size/2, size, size])
    cell.add_polygon(layer: "metal1", rect: [-size/2, -size/2, size, size])

    cell.add_contact(from: "diff", to: "li1",    at: [0nm, 0nm], diameter: 170nm)
    cell.add_contact(from: "li1",  to: "metal1", at: [0nm, 0nm], diameter: 170nm)

    cell.add_port(name: "TAP", at: [0nm, 0nm], layer: "metal1")
    return cell
}
```

### 4.2 Contact Array Matrix (`@std/layout/via.hw`)

```hardware
# stdlib/layout/via.hw
import * from @std/primitives/units

export fn via_matrix(
    from_layer: String,
    to_layer: String,
    rows: Int,
    cols: Int,
    pitch: Measurement,
    diameter: Measurement
) -> CellLayout {
    let mut cell = CellLayout::new(name: "via_matrix")
    let offset_x = (cols - 1) * pitch / 2
    let offset_y = (rows - 1) * pitch / 2

    for r in 0..rows {
        for c in 0..cols {
            let vx = -offset_x + (c * pitch)
            let vy = -offset_y + (r * pitch)
            cell.add_contact(from: from_layer, to: to_layer, at: [vx, vy], diameter: diameter)
        }
    }

    cell.add_port(name: "CENTER", at: [0nm, 0nm], layer: to_layer)
    return cell
}
```

---

## 5. End-to-End Concrete Implementations

### 5.1 Canonical Silicon Inverter (`cmos_inverter.hw`)

```hardware
# cmos_inverter.hw - HardwareScript v0.3.2 Canonical Implementation
import * from @std/primitives/units
import { sky130_nmos } from @std/pdk/sky130/nmos
import { sky130_pmos } from @std/pdk/sky130/pmos
import { sky130_tap, TapType } from @std/pdk/sky130/tap
import { SemiconductorSubstrate } from @std/substrates

export profile SKY130_1V8_CMOS implements SemiconductorSubstrate {
    manufacturing_grid: 5nm
    lambda: 50nm

    masks {
        nwell:  { gds_layer: 64, datatype: 20 }
        diff:   { gds_layer: 65, datatype: 20 }
        tap:    { gds_layer: 65, datatype: 44 }
        nsdm:   { gds_layer: 93, datatype: 44 }
        psdm:   { gds_layer: 94, datatype: 20 }
        poly:   { gds_layer: 66, datatype: 20 }
        licon:  { gds_layer: 66, datatype: 44 }
        li1:    { gds_layer: 67, datatype: 20 }
        mcon:   { gds_layer: 67, datatype: 44 }
        metal1: { gds_layer: 68, datatype: 20 }
    }

    routing {
        topology: "manhattan"
        grid: 5nm
        preferred_directions {
            li1:    "horizontal"
            metal1: "vertical"
        }
    }

    drc {
        antenna_ratio_max: 400.0
        min_density metal1: 20%
    }
}

space InverterBlock {
    dimensions: [50.0um, 30.0um]
    profile: SKY130_1V8_CMOS

    nets {
        VDD: { classification: power,  potential: 1.8V, current: 20.0uA }
        VSS: { classification: ground, potential: 0.0V, current: 20.0uA }
        IN:  { classification: signal, potential: 1.8V }
        OUT: { classification: signal }
    }

    # 1. Pure PCell Invocation (Returns pure, detached CellLayouts)
    let nmos_cell = sky130_nmos(W: 1.0um, L: 150nm)
    let pmos_cell = sky130_pmos(W: 2.0um, L: 150nm)
    let sub_tap_cell  = sky130_tap(type: TapType.P_Sub,  size: 600nm)
    let well_tap_cell = sky130_tap(type: TapType.N_Well, size: 600nm)

    # 2. Explicit Space Placement (World Coordinates resolved automatically)
    let m1 = space.place(nmos_cell, at: [25.0um, 10.0um])
    let m2 = space.place(pmos_cell, at: [25.0um, 20.0um])
    let tap_sub  = space.place(sub_tap_cell,  at: [20.0um, 10.0um])
    let tap_well = space.place(well_tap_cell, at: [20.0um, 20.0um])

    # 3. Interconnect Routing (Snaps strictly to 5nm DBU Manhattan tracks)
    route m1.port("G") to m2.port("G") { net: IN,  layer: "metal1", width: 200nm }
    route m1.port("D") to m2.port("D") { net: OUT, layer: "metal1", width: 300nm }

    # 4. Power & Mandatory Bulk Connections (Satisfies 4-Terminal Foundry Contract)
    route m1.port("S") to tap_sub.port("TAP")  { net: VSS, layer: "metal1", width: 500nm }
    route m2.port("S") to tap_well.port("TAP") { net: VDD, layer: "metal1", width: 500nm }
}
```

### 5.2 Canonical Differential Pair with Struct Bundles (`diff_amp.hw`)

```hardware
# diff_amp.hw - HardwareScript v0.3.2 Canonical Implementation
import * from @std/primitives/units
import { sky130_nmos } from @std/pdk/sky130/nmos
import { sky130_tap, TapType } from @std/pdk/sky130/tap
import { SKY130_1V8_CMOS } from "./cmos_inverter"

bundle DifferentialPair {
    p: signal,
    n: signal,
}

space DifferentialStage {
    dimensions: [60.0um, 40.0um]
    profile: SKY130_1V8_CMOS

    nets {
        VDD:   { classification: power,  potential: 1.8V, current: 50.0uA }
        VSS:   { classification: ground, potential: 0.0V, current: 50.0uA }
        IN_P:  { classification: signal, potential: 0.9V }
        IN_N:  { classification: signal, potential: 0.9V }
        OUT_P: { classification: signal }
        OUT_N: { classification: signal }
        TAIL:  { classification: signal }
    }

    # Initialize Structural Interface Bundles
    let in_diff = DifferentialPair { p: IN_P, n: IN_N }

    # 1. Pure Cell Generation
    let nmos_core = sky130_nmos(W: 3.0um, L: 150nm)

    # 2. Symmetric Placement using Fluent Affine Transformations
    let m1 = space.place(nmos_core, at: [20.0um, 20.0um])
    let m2 = space.place(nmos_core.mirror_y(), at: [35.0um, 20.0um])

    # 3. Source Coupling
    route m1.port("S") to m2.port("S") { net: TAIL, layer: "metal1", width: 400nm }

    # 4. One-Line Bundle Route with Differential Constraints
    route in_diff to DifferentialPair { p: m1.port("G"), n: m2.port("G") } {
        layer: "metal1",
        width: 200nm,
        spacing: 300nm,
        skew_tolerance: 50fs
    }
}
```

### 5.3 Chip-Package-Board Co-Design System Reference (`system_codesign.hw`)

For the full chip-to-board co-design reference implementation containing the operational amplifier silicon die, QFN-16 packaging contract, and 4-layer FR-4 board test bench, refer to Section 7 of [`Grand-Unified-Specification.md`](file:///c:/Users/olowo/Downloads/Code/hw/Docs/v0.3.2/Grand-Unified-Specification.md).
