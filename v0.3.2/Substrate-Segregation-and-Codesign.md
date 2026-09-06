# HardwareScript v0.3.2: Substrate Segregation & Chip-Package-Board Co-Design Specification

**Document Type:** Physical Domain Architecture & Multi-Substrate Integration Standard  
**Target Version:** v0.3.2 (Production-Locked Standard)  
**Status:** Authoritative & Canonical  
**Target Crates:** `hwc-engine`, `hwc-physics`, `hwc-export`, `hwc-compiler`  
**Downstream Dependents:** `hwc-cli`, `hwc-router`  
**Date:** September 2026  

---

## 1. Executive Summary: Substrate Segregation

Prior EDA attempts at unified hardware compilers failed because they commingled physical domains, applying PCB concepts to integrated circuits or treating silicon wafers like macro-boards.

HardwareScript v0.3.2 resolves this via **Strict Substrate Segregation with Frontend Neutrality**:

1. **Clean Frontend Neutrality:** The language grammar contains zero IC-specific (`transistor`, `gate_oxide`, `well_tap`) or PCB-specific (`drill_hole`, `copper_pour`, `prepreg`) keywords.
2. **Substrate Trait Dispatch:** Physical domains are isolated into backend traits:
   - `SemiconductorSubstrate`: Photolithographic integrated circuits (CMOS, FinFET, GaN, SiC).
   - `LaminateSubstrate`: Laminated printed circuit boards (FR-4, Rogers, Polyimide flex).
3. **Universal Flat Coordinate Arena:** Both backends consume the unified signed picometer IR (`FlatGeometryBuffer`), guaranteeing clean data exchange and single-source-of-truth export.
4. **Chip-Package-Board Co-Design:** Integrated circuit dies and multi-layer boards co-exist within the same project. Packaging contracts bridge microscopic on-chip pads to macroscopic solder lands, enabling full-system SPICE co-simulation.

```
                            SUBSTRATE TRAIT DISPATCH
                                       │
                         [ FlatGeometryBuffer (Picometers) ]
                                       │
        ┌──────────────────────────────┴──────────────────────────────┐
        ▼                                                             ▼
  [ SemiconductorSubstrate ]                                    [ LaminateSubstrate ]
  • Boolean mask derivation (Clipper2)                          • 3D dielectric/copper slab extrusion
  • P-N junction diode barrier tracking                         • Automated antipad plane clearance
  • 5nm DBU Manhattan constructive routing                     • Continuous topological vector routing
  • G-Cell SIMD lithography DRC sweep                           • IPC-2221 annular ring & drill rules
  • GDSII / OASIS mask stream commitment                        • Gerber RS-274X & NC Drill stream
```

---

## 2. Semiconductor Engine: Silicon Wafer Physics

When a `space` block binds to a profile implementing `SemiconductorSubstrate`, the compiler dispatches the layout to the semiconductor engine.

### 2.1 Boolean Mask Derivation via Clipper2

Rather than relying on human annotations or layer guessing, physical devices are extracted mathematically via 2D constructive solid geometry on `FlatGeometryBuffer`:

$$\text{Active Gate Channels} = \text{diff} \cap \text{poly}$$
$$\text{NMOS Source/Drain} = (\text{diff} \cap \text{nsdm}) \setminus \text{poly}$$
$$\text{PMOS Source/Drain} = (\text{diff} \cap \text{psdm} \cap \text{nwell}) \setminus \text{poly}$$

```rust
// hwc-substrate-cmos/src/masks.rs

use clipper2_rust::{Clipper, FillRule, PathsD};
use hwc_engine::FlatGeometryBuffer;

pub struct ExtractedChannels {
    pub nmos_channels: PathsD,
    pub pmos_channels: PathsD,
    pub nmos_source_drain: PathsD,
    pub pmos_source_drain: PathsD,
}

pub fn derive_silicon_masks(buffer: &FlatGeometryBuffer) -> ExtractedChannels {
    let diff = buffer.extract_layer_polygons("diff");
    let poly = buffer.extract_layer_polygons("poly");
    let nsdm = buffer.extract_layer_polygons("nsdm");
    let psdm = buffer.extract_layer_polygons("psdm");
    let nwell = buffer.extract_layer_polygons("nwell");

    // Intersect diff and poly for active gates
    let gates = Clipper::intersect(&diff, &poly, FillRule::NonZero);

    // Derive NMOS source/drain: (diff ∩ nsdm) \ poly
    let n_diff = Clipper::intersect(&diff, &nsdm, FillRule::NonZero);
    let nmos_sd = Clipper::difference(&n_diff, &poly, FillRule::NonZero);

    // Derive PMOS source/drain: (diff ∩ psdm ∩ nwell) \ poly
    let p_diff_well = Clipper::intersect(&Clipper::intersect(&diff, &psdm, FillRule::NonZero), &nwell, FillRule::NonZero);
    let pmos_sd = Clipper::difference(&p_diff_well, &poly, FillRule::NonZero);

    ExtractedChannels {
        nmos_channels: gates,
        pmos_channels: gates,
        nmos_source_drain: nmos_sd,
        pmos_source_drain: pmos_sd,
    }
}
```

### 2.2 Junction-Aware Interface Conductivity Matrix

To prevent false short circuits during electrical connectivity extraction, the compiler evaluates semiconductor junctions:
- **Ohmic Transitions:** Metal-to-diff contacts and well/substrate taps carry linear conductivity ($I = V/R$). Net connectivity propagates freely.
- **Rectifying P-N Junctions:** The boundary between $\text{P}^+\text{ Diffusion}$ and an $\text{N-Well}$ constitutes a diode barrier. Net propagation halts at the depletion boundary unless explicitly bridged by a metallization layer or tap.
- **Zero VDD-to-OUT Shorts:** In a CMOS inverter, the output diffusion touches the N-well boundary. The junction-aware conductivity matrix correctly isolates net `OUT` from net `VDD`.

### 2.3 Constructive Manhattan Line-Search Routing

Silicon interconnect routes strictly on the foundry Database Unit (DBU) grid (e.g. 5nm):
- 100% orthogonal Manhattan vectors ($90^\circ$ bends, optional $45^\circ$ chamfers).
- Strict preferred directionality per metal layer (e.g. `li1` Horizontal, `metal1` Vertical).
- Layer pitch quantization: trace centers snap to integer multiples of track pitch.

---

## 3. Board Engine: Laminated PCB Physics

When a `space` block binds to a profile implementing `LaminateSubstrate`, the compiler dispatches the layout to the laminate engine.

### 3.1 3D Dielectric & Conductor Slab Extrusion

The board engine reads stackup declarations and extrudes physical geometry into 3D space:
- Copper foils (`top_cu`, `bot_cu`) are modeled as physical conductor boundaries with designated thicknesses (e.g. 35µm / 1 oz copper).
- Dielectric cores (e.g. 1.5mm FR-4) are extruded between copper foils.
- Solder mask openings expose macroscopic solder lands.

### 3.2 Automated Antipad Plane Clearance

When traces or through-hole vias penetrate continuous copper planes (e.g. ground or power pours), the laminate engine computes subtractive antipad cutouts:

$$\text{Antipad Diameter} = \text{Drill Diameter} + 2 \times \text{Clearance}$$

Inner hole loops are added directly to the `PolygonAuxRecord` of the enclosing copper plane, preventing electrical shorts across drill barrels.

### 3.3 Continuous Topological Vector Routing

PCB traces utilize continuous topological routing:
- $45^\circ$ or curved vector paths snapped to macroscopic manufacturing grids (e.g. 0.05mm).
- Automated teardrop filleting at pad interfaces to prevent drill breakout.
- Annular ring verification according to IPC-2221 Class 2/3 tolerances.

---

## 4. Independent Layout-Versus-Schematic (LVS) Verification

HardwareScript decouples design intent from physical manifestation:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       INDEPENDENT LVS VERIFICATION                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ Golden Reference Schematic: G_S ]                                        │
│  • Extracted from explicit `module.netlist` declarations.                   │
│  • Contains ideal logical components and node-to-terminal connectivity.     │
│                                                                             │
│                                      │                                      │
│                                      ▼ 1-WL Bipartite Graph Isomorphism     │
│                             [ Mathematical Equivalence Gate ]                │
│                                      ▲ (Pass / Fail)                        │
│                                      │                                      │
│  [ Physical Silicon Wafer: G_L ]     │                                      │
│  • Derived via 2D Boolean mask operations on FlatGeometryBuffer:            │
│    - Active Gates = diff ∩ poly                                             │
│    - NMOS Source/Drain = (diff ∩ nsdm) \ poly                               │
│    - PMOS Source/Drain = (diff ∩ psdm ∩ nwell) \ poly                       │
│  • Evaluates Interface Conductivity Matrix (diodes halt net propagation).   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.1 The 1-Dimensional Weisfeiler-Lehman (1-WL) Algorithm

1. **Bipartite Graph Formulation:** Both graphs contain two node types: Devices ($D$) and Nets ($N$). Edges connect device terminals to nets.
2. **Initial Color Assignment:** Nodes are colored by type (MOSFET type, W/L dimensions, Net classification).
3. **Iterative Color Refinement:** At each step $k$, each node's color updates based on the multiset of its neighbors' colors:
   $$c_v^{(k+1)} = \text{Hash}\left(c_v^{(k)}, \{\!\!\{ c_u^{(k)} \mid u \in \mathcal{N}(v) \}\!\!\}\right)$$
4. **Automorphism Pin Permutation:** Symmetrical terminals (such as the Source and Drain of a symmetric MOSFET, or the inputs of a NAND gate) belong to the same automorphism permutation orbit. 1-WL recognizes these topological symmetries without triggering false mismatch diagnostics.

---

## 5. Chip-Package-Board Co-Design Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 MULTI-SUBSTRATE CO-DESIGN HIERARCHY                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  LEVEL 1: SILICON DIE (SemiconductorSubstrate)                              │
│  • Coordinates: Integer Picometers (5nm DBU Grid)                           │
│  • Output: Photolithography GDSII / OASIS masks + Subcircuit SPICE          │
│  • Exposes on-chip wire-bond pads (~40um pitch)                             │
│                                                                             │
│                                      │ Encapsulated by                      │
│                                      ▼ Packaging Contract                   │
│  LEVEL 2: COMPONENT PACKAGE FOOTPRINT (Package Contract)                    │
│  • Defines leadframe, bond wires, and solder ball geometry                  │
│  • Embeds RLC packaging parasitics: bond-wire inductance, pad capacitance   │
│  • Translates 40um die pads to 0.5mm macroscopic surface-mount landing pads │
│                                                                             │
│                                      │ Instantiated onto                    │
│                                      ▼ Board Space                          │
│  LEVEL 3: EVALUATION BOARD (LaminateSubstrate)                              │
│  • Coordinates: Integer Picometers (Snapped to 0.05mm PCB Grid)             │
│  • Output: Gerber RS-274X, IPC-2581, Excellon NC Drill                      │
│  • Routes macroscopic microstrip transmission lines to connectors and power │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.1 Packaging Contracts (`footprint ... for ...`)

The bridge between silicon and board is the **Packaging Contract**:

```hardware
footprint QFN16_Package for OperationalAmplifier {
    body_size: [4.0mm, 4.0mm]
    pitch: 0.5mm

    # Electrical package bond-wire parasitic contract
    parasitics {
        bond_wire_inductance: 1.2nH
        lead_resistance: 35mOhm
        pad_capacitance: 0.35pF
    }

    # Map microscopic silicon pads to macroscopic solder lands
    pins {
        pin 1:  OperationalAmplifier.IN_P { at: [-1.9mm,  0.5mm], size: [0.8mm, 0.28mm] }
        pin 2:  OperationalAmplifier.IN_N { at: [-1.9mm,  0.0mm], size: [0.8mm, 0.28mm] }
        pin 3:  OperationalAmplifier.VSS  { at: [-1.9mm, -0.5mm], size: [0.8mm, 0.28mm] }
        pin 9:  OperationalAmplifier.OUT  { at: [ 1.9mm,  0.0mm], size: [0.8mm, 0.28mm] }
        pin 16: OperationalAmplifier.VDD  { at: [ 0.0mm,  1.9mm], size: [0.28mm, 0.8mm] }
    }
}
```

### 5.2 Unified Electro-Thermal SPICE Co-Simulation

During full-system SPICE extraction (`hwc-export`), the compiler emits an integrated simulation deck:

```spice
* HardwareScript v0.3.2 Full-System Co-Simulation Deck
.include "models/sky130_fd_pr.spice"

* --- TIER 3: BOARD TRANSMISSION LINES ---
W_TL1 SIG_IN_P BOARD_GND PIN_1 BOARD_GND L=12.5m C=110p R=0.12 G=0

* --- TIER 2: QFN16 BOND WIRE PARASITICS ---
L_BOND_1 PIN_1 DIE_PAD_IN_P 1.2nH
R_LEAD_1 PIN_1 DIE_PAD_IN_P 35mOhm
C_PAD_1  DIE_PAD_IN_P BOARD_GND 0.35pF

* --- TIER 1: MONOLITHIC SILICON DIE SUBCIRCUIT ---
X_OPAMP DIE_PAD_IN_P DIE_PAD_IN_N DIE_PAD_OUT DIE_PAD_VDD DIE_PAD_VSS OperationalAmplifier_Die

.subckt OperationalAmplifier_Die IN_P IN_N OUT VDD VSS
M1 NODE_D1 IN_P TAIL VSS sky130_fd_pr__nfet_01v8 W=3.000u L=0.150u
M2 OUT IN_N TAIL VSS sky130_fd_pr__nfet_01v8 W=3.000u L=0.150u
M3 NODE_D1 NODE_D1 VDD VDD sky130_fd_pr__pfet_01v8 W=6.000u L=0.150u
M4 OUT NODE_D1 VDD VDD sky130_fd_pr__pfet_01v8 W=6.000u L=0.150u
M5 TAIL BIAS VSS VSS sky130_fd_pr__nfet_01v8 W=4.000u L=0.150u
.ends
```

Board traces, packaging parasitics, and internal on-chip MOSFETs simulate simultaneously in a single, closed-loop electro-thermal analysis.
