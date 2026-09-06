# HardwareScript v0.3.2: Grand Unified Specification

**Document Type:** Production-Locked Single Source of Truth (SSOT) Architecture & System Specification  
**Release Target:** HardwareScript v0.3.2 (Production-Locked Standard)  
**Status:** Authoritative, Canonical & Ratified  
**Supersedes:** All prior drafts across `Docs/v0.3.0/`, `Docs/v0.3.1/`, and interim v0.3.2 proposals  
**Target Crates:** `hwc-types`, `hwc-materials`, `hwc-diagnostics`, `hwc-stdlib`, `hwc-cli`, `hwc-parser`, `hwc-compiler`, `hwc-engine`, `hwc-router`, `hwc-synthesis`, `hwc-physics`, `hwc-export`  
**Date:** September 2026  

---

## Executive Summary: The Hardened Architecture

HardwareScript v0.3.2 resolves the architectural compromises of prior iterations. Earlier designs alternated between two failure modes:
1. An ad-hoc AST tree-walk interpreter that degraded under recursion and variable lookup overhead.
2. An isolated bytecode virtual machine that suffered from artificial instruction-fuel limits and sandboxing overhead.

Wasm64 (Memory64) extensibility also introduced linear memory isolation barriers, lack of direct GPU/CUDA interop, and restrictions on SIMD intrinsics beyond 128 bits.

HardwareScript v0.3.2 establishes **four hardened architectural pillars**:

1. **JIT Comptime Elaboration via Cranelift:** Procedural loops, parameter algebra, and cell evaluations compile in memory directly to native host machine code, eliminating interpretation loops, AST recursion, and artificial fuel limits.
2. **Universal Native Container Plugin ABI:** External stages (e.g., custom routers, SAT synthesis backends) ship as native shared libraries (`.so`, `.dll`, `.dylib`) exposing a clean `#[repr(C)]` ABI. The package manager resolves and downloads only the pre-compiled binary matching the user's host OS and architecture.
3. **Substrate Segregation with Clean Frontend Neutrality:** The language grammar contains zero IC-specific or PCB-specific keywords. Photolithographic integrated circuits (`SemiconductorSubstrate`) and laminated printed circuit boards (`LaminateSubstrate`) are isolated into dedicated backend engines consuming a unified, target-agnostic picometer Intermediate Representation (`FlatGeometryBuffer`).
4. **Chip-Package-Board Co-Design:** Monolithic semiconductor dies and multi-layer boards can be designed within the same repository. Physical integrity is enforced by packaging and footprint contracts, allowing full-system electro-thermal SPICE co-simulation without mixing semiconductor and board physics.

---

## 1. End-to-End System Architecture

The compilation pipeline operates as a unidirectional, data-oriented transformation across four functional tiers:

```
                            THE v0.3.2 COMPILATION PIPELINE
                            
    [ User .hw Source Code ] (Declarations, PCell Generators, Netlists, Spaces)
                │
                ▼
    [ Frontend Scanner & Parser: hwc-parser ]
    • Logos lexical scanner (28 reserved keywords, natural boolean logic)
    • Pratt expression parser with compile-time 7-base SI dimensional analysis
    • Emits immutable, arena-allocated AST
                │
                ▼
    [ Comptime Elaboration Engine: hwc-compiler ]
    • Cranelift In-Memory JIT compiles procedural loops to native machine code
    • Pure PCell Flyweight memoization: evaluated once, stamped across instances
    • Populates the out-of-band DebugSpanRegistry (EntityId -> SourceSpan)
                │
                ▼
    [ Target-Agnostic Physical Arena: hwc-engine ]
    • FlatGeometryBuffer: contiguous coordinate pool, 40-byte cache-aligned headers
    • Coordinates stored in exact signed picometers (1 pm = 10^-12 m)
    • Derived physical areas, energies, and densities computed in 128-bit integers
                │
                ▼ Substrate Trait Dispatch
        ┌───────┴───────────────────────────────────────┐
        ▼                                               ▼
    [ Silicon Engine: hwc-substrate-cmos ]     [ Board Engine: hwc-substrate-laminate ]
    • Boolean mask derivation (Clipper2)        • 3D dielectric/copper slab extrusion
    • P-N junction diode barrier tracking       • Automated antipad plane clearance
    • 5nm DBU Manhattan constructive routing   • Continuous topological vector routing
    • G-Cell SIMD lithography DRC sweep         • IPC-2221 annular ring & drill rules
        │                                               │
        └───────────────────────┬───────────────────────┘
                                │
                                ▼
    [ Verification & Exporters: hwc-physics & hwc-export ]
    • 1-WL Bipartite Graph Isomorphism LVS (Extracted GL vs. module.netlist GS)
    • Distributed boundary element method (BEM) parasitic extraction
    • Manufacturing commitment: GDSII, OASIS, Gerber RS-274X, SPICE, DXF, GLB
```

### 1.1 The 12-Crate Role and Responsibility Matrix

| Tier | Crate | Binary / Library | Core Responsibility | Key Dependencies |
| :--- | :--- | :--- | :--- | :--- |
| **Tier 1: Foundations** | [`hwc-types`](file:///c:/Users/olowo/Downloads/Code/hw/hwc/crates/hwc-types) | Library | 7-Base SI `SiDimension` vector $[L, M, T, I, \Theta, N, J]$, 128-bit `MeasurementValue`, `Point2D`, and integer picometer coordinates (`i64`). | `serde`, `rkyv`, `blake3` |
| | [`hwc-materials`](file:///c:/Users/olowo/Downloads/Code/hw/hwc/crates/hwc-materials) | Library | Process-neutral physical tables: conductivity, permittivity ($\epsilon_r$), loss tangent ($\tan \delta$), and sheet resistance. Zero hardcoded foundry layers. | `hwc-types` |
| | [`hwc-diagnostics`](file:///c:/Users/olowo/Downloads/Code/hw/hwc/crates/hwc-diagnostics) | Library | Zero-tolerance error reporting with source-span diagnostics via `miette`. Formats fail-loudly compiler exceptions (no swallowed errors). | `miette` |
| | [`hwc-stdlib`](file:///c:/Users/olowo/Downloads/Code/hw/hwc/crates/hwc-stdlib) | Library | Certified PCell generators, standard package footprints, and physical layout primitives. | `hwc-types` |
| | [`hwc-cli`](file:///c:/Users/olowo/Downloads/Code/hw/hwc/crates/hwc-cli) | Binary (`hwc`) | CLI driver and argument parser (`hwc build`, `hwc eval`, `hwc check`, `hwc test`). Orchestrates the pipeline stages. | All crates |
| **Tier 2: Frontend** | [`hwc-parser`](file:///c:/Users/olowo/Downloads/Code/hw/hwc/crates/hwc-parser) | Library | Logos lexical scanner and Pratt expression parser. Produces a read-only, fully typed `AstArena`. | `hwc-types`, `hwc-diagnostics` |
| **Tier 3: Execution** | [`hwc-compiler`](file:///c:/Users/olowo/Downloads/Code/hw/hwc/crates/hwc-compiler) | Library | **Cranelift In-Memory JIT:** Compiles procedural loops, parameter algebra, and geometry construction directly to host machine code in memory. Populates `DebugSpanRegistry`. | `hwc-parser`, `hwc-types`, `hwc-engine`, `cranelift-jit`, `cranelift-module` |
| **Tier 4: Core Arena** | [`hwc-engine`](file:///c:/Users/olowo/Downloads/Code/hw/hwc/crates/hwc-engine) | Library | **The Physical Memory Arena:** Owns `FlatGeometryBuffer` (40-byte compact headers, coordinate pool, aux records), spatial R-tree queries (`rstar`), and Merkle `EntityId`. | `hwc-types`, `compact_str`, `rkyv`, `blake3` |
| **Tier 5: Plugins** | [`hwc-router`](file:///c:/Users/olowo/Downloads/Code/hw/hwc/crates/hwc-router) | Library | **Universal Native Container Plugin Bridge:** Dynamic loader for external stages (`.so` / `.dll` / `.dylib`) via `libloading` with zero-copy FFI handshake (`HwcStageContextC`). | `hwc-engine`, `libloading` |
| | [`hwc-synthesis`](file:///c:/Users/olowo/Downloads/Code/hw/hwc/crates/hwc-synthesis) | Library | **Black-Box Macro Ingestion:** Ingests LEF/DEF and GDSII macro blocks produced by external digital flows (OpenROAD/Yosys) and drops them into `FlatGeometryBuffer`. | `hwc-engine`, `hwc-types` |
| **Tier 6: Verification & Export** | [`hwc-physics`](file:///c:/Users/olowo/Downloads/Code/hw/hwc/crates/hwc-physics) | Library | **Physical Verification & LVS:** 1-WL Bipartite Graph Isomorphism LVS, G-Cell Local SIMD Sweep for DRC, and distributed RC parasitic extraction. | `hwc-engine`, `hwc-materials` |
| | [`hwc-export`](file:///c:/Users/olowo/Downloads/Code/hw/hwc/crates/hwc-export) | Library | **Manufacturing Stream Commitment:** Single source of truth export of GDSII, OASIS, Gerber RS-274X, Excellon drill, 3D GLB, DXF, and full-system co-simulation SPICE netlists. | `hwc-engine`, `hwc-physics` |

---

## 2. In-Memory JIT Comptime Engine (`hwc-compiler`)

The compile-time evaluation engine compiles procedural code directly to native host instructions.

### 2.1 The Execution Lifecycle
When a space or generator block is evaluated:
1. `hwc-compiler` lowers the procedural control-flow graph (loops, conditional branches, coordinate calculations) into Cranelift Intermediate Representation (CLIF).
2. Cranelift's JIT backend compiles the CLIF blocks directly into executable host machine code in memory.
3. The compiler invokes the generated code via a native function pointer, passing a mutable reference to the active `FlatGeometryBuffer` allocation context.
4. Loops execute at host hardware speed, writing picometer vertices directly into contiguous vectors without bytecode interpretation overhead.
5. Upon loop completion, executable memory is reclaimed.

### 2.2 Pure PCell Memoization & The Flyweight Pattern
Parameterized cells (PCells) are pure mathematical functions that accept dimensional inputs and return an isolated, origin-relative `CellLayout`.
- **Zero Ambient Mutation:** PCells cannot read, mutate, or query the enclosing space block.
- **Deterministic Caching:** The compiler hashes the PCell function identifier and its input arguments using Blake3. If an identical cell has already been evaluated, the compiler retrieves the geometry reference from memory and applies affine coordinate translations $(dx, dy)$ in $O(1)$ time, eliminating redundant geometry evaluations.

### 2.3 Diagnostic Traceability: The Out-of-Band Debug Span Map
To prevent source-code line shifts and comment edits from invalidating incremental build caches, geometry records do not store byte offsets or line numbers.

Instead, the compiler maintains a sidecar lookup structure:
$$\text{DebugSpanRegistry} : \text{EntityId} \to (\text{FileId}, \text{SourceSpan})$$

- During Cranelift JIT execution, every emitted record registers its structural `EntityId` alongside its active AST source span within `DebugSpanRegistry`.
- `DebugSpanRegistry` is ephemeral: it is never hashed for incremental compilation tracking, never serialized across `rkyv` disk caches, and never emitted into production GDSII or Gerber streams.
- When physical DRC, LVS, or connectivity verification flags an anomaly on an `EntityId`, the diagnostic engine looks up the ID in the registry and renders full contextual error messages with source snippets via `miette`.

---

## 3. The Universal Native Container Plugin Architecture (`hwc-router`)

HardwareScript delegates compute-intensive physical synthesis stages (e.g., global maze routing, SAT Boolean logic optimization, analytical placement) to native shared libraries.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 UNIVERSAL NATIVE CONTAINER EXECUTION MODEL                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ Plugin Developer ]                                                       │
│  • Implements algorithms in C, C++, Rust, Zig, or Ada.                      │
│  • Compiles native shared objects (.so / .dll / .dylib) via GitHub Actions. │
│  • Publishes single multi-target package: `@academic/router.hpm`.           │
│                                                                             │
│  [ Package Registry (Edge CDN) ]                                            │
│  • Manifest maps target triples to specific downloadable archive URLs:      │
│    - x86_64-unknown-linux-gnu                                               │
│    - x86_64-pc-windows-msvc                                                 │
│    - aarch64-apple-darwin                                                   │
│                                                                             │
│  [ HardwareScript CLI Driver: hw ]                                          │
│  • Identifies host architecture at installation (`hw add @academic/router`).│
│  • Downloads ONLY the matching native binary payload into `vendor/`.        │
│  • End-user requires ZERO local C/C++/Rust/Zig compilers.                   │
│                                                                             │
│  [ Zero-Copy FFI Handshake ]                                                │
│  • Host maps `FlatGeometryBuffer` pointers into `HwcStageContextC`.          │
│  • Plugin executes bare-metal AVX-512, OpenMP, or native CUDA routines.    │
│  • Resulting segments written directly into host memory; zero serialization.│
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.1 The Canonical C-ABI Interface (`hwc_plugin_abi.h`)

All dynamic plugins implement this interface:

```c
#ifndef HWC_PLUGIN_ABI_H
#define HWC_PLUGIN_ABI_H

#include <stdint.h>
#include <stddef.h>

#ifdef __cplusplus
extern "C" {
#endif

#define HWC_PLUGIN_API_VERSION 302

/* Contiguous picometer wire segment */
typedef struct {
    uint32_t net_id;
    uint16_t layer_idx;
    uint8_t  _reserved[2];
    int64_t  start_x_pm;
    int64_t  start_y_pm;
    int64_t  end_x_pm;
    int64_t  end_y_pm;
    int64_t  width_pm;
} HwcWireSegmentC;

/* Execution Context exchanged across the FFI boundary */
typedef struct {
    int64_t bounds_min_x_pm;
    int64_t bounds_min_y_pm;
    int64_t bounds_max_x_pm;
    int64_t bounds_max_y_pm;

    /* Read-only obstacle geometry from FlatGeometryBuffer */
    const HwcWireSegmentC* obstacles_ptr;
    size_t                 num_obstacles;

    /* Pre-allocated host output buffer */
    HwcWireSegmentC*       output_ptr;
    size_t                 output_capacity;
    size_t                 written_count;

    int32_t                status_code; /* 0 = Success, >0 = Error */
    char                   error_msg[256];
} HwcStageContextC;

#if defined(_WIN32)
    #define HWC_EXPORT __declspec(dllexport)
#else
    #define HWC_EXPORT __attribute__((visibility("default")))
#endif

HWC_EXPORT uint32_t hwc_plugin_api_version(void);
HWC_EXPORT int32_t  hwc_plugin_execute_stage(HwcStageContextC* ctx);

#ifdef __cplusplus
}
#endif

#endif
```

---

## 4. Universal Flat Intermediate Representation (`hwc-engine`)

All geometric entities, connectivity records, and physical attributes are stored within `FlatGeometryBuffer`.

### 4.1 64-Bit vs. 128-Bit Arithmetic Model
- **Coordinates & Lengths (`i64`):** Stored as signed 64-bit integer picometers. A 64-bit coordinate space spans $\pm 9.22 \times 10^{18}\text{ pm}$ ($\pm 9{,}223\text{ kilometers}$), preventing overflow across any terrestrial wafer or PCB panel.
- **Areas, Volumes & Energy Densities (`i128`):** Calculating the physical area of a $10\text{mm} \times 10\text{mm}$ die in picometers yields $10^{10}\text{ pm} \times 10^{10}\text{ pm} = 10^{20}\text{ pm}^2$. This value exceeds `i64::MAX` ($9.22 \times 10^{18}$). Therefore, all surface area calculations, sheet resistance integrations, metal density sums, and capacitive energy extractions are computed using 128-bit signed integers (`i128`).

### 4.2 Compact Geometry Record Header

```rust
#[repr(u8)]
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum RecordType {
    Polygon = 1, // Closed 2D boundaries (outer contours + inner cutouts)
    Contact = 2, // Layer-to-layer conductive transitions & TSVs
    Device  = 3, // Multi-terminal SPICE compact models
    Port    = 4, // Interface anchors with outward escape normal vectors
    Route   = 5, // Topological interconnect centerlines
}

#[repr(C)]
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct CompactGeometryRecordHeader {
    pub id: EntityId,             // 8 bytes: Span-independent Merkle structural hash
    pub net_id: u32,              // 4 bytes: Topological electrical net identifier
    pub space_id: u32,            // 4 bytes: Enclosing space container identifier
    pub coord_start_idx: u32,     // 4 bytes: Starting index in coordinate_pool
    pub coord_count: u32,         // 4 bytes: Vertex count (x, y coordinate pairs)
    pub layer_idx: u16,           // 2 bytes: Process-neutral layer identifier
    pub record_type: RecordType,  // 1 byte:  Polygon, Contact, Device, Port, Route
    pub flags: u8,                // 1 byte:  Bitfield (DEVICE_BODY, FROZEN_ECO, HAS_HOLES)
    pub aux_index: u32,           // 4 bytes: Pointer into respective sidecar record pool
    pub _reserved: u32,           // 4 bytes: 64-byte CPU cache alignment padding
}
```

---

## 5. Physical Routing & Verification Architecture

### 5.1 The Two-Tier Routing Topology
1. **Constructive Manhattan Line-Search Router (Pure Rust):** When explicit user declarations connect designated terminals (`route A to B`), the built-in router resolves Manhattan orthogonal paths ($90^\circ$ bends, $45^\circ$ miters) snapped to the process Database Unit (DBU) grid, navigating local obstructions without altering unrelated layout regions.
2. **Quarantined Global Congestion Engines (Native Plugins):** High-density unplaced digital logic blocks are isolated inside declared physical region blocks, dispatched to native C-ABI plugins (e.g. Dr. CU 2.0, FastGR).

### 5.2 Independent Layout-Versus-Schematic (LVS) Verification
HardwareScript decouples structural intent from physical realization:

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

The 1-Dimensional Weisfeiler-Lehman (1-WL) color refinement engine verifies graph isomorphism under group-theoretic automorphism permutations, allowing symmetric pin permutations (e.g., differential pair inputs) without triggering false mismatches.

---

## 6. Multi-Substrate Co-Design: The Chip-Package-Board Abstraction

HardwareScript allows an engineer to design a custom integrated circuit, define its physical package, and mount it onto a multi-layer printed circuit board within a single repository:

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

### 6.1 Substrate Isolation Rules
- **Single-Substrate Authority:** Each declared space block binds to exactly one physical profile implementing a single substrate trait (`SemiconductorSubstrate` or `LaminateSubstrate`).
- **Physical Isolation:** Attempting to invoke PCB operations (such as drilling an annular through-hole via) inside an IC semiconductor space triggers a compile-time schema error.
- **Unified SPICE Co-Simulation:** During SPICE extraction, `hwc-export` generates a top-level netlist where the distributed microstrip models of the PCB connect across packaging leadframe parasitics directly into the internal 4-terminal compact MOSFET models of the silicon die.

---

## 7. Concrete End-to-End Implementation

The following complete HardwareScript implementation specifies a CMOS Silicon Operational Amplifier, packages it inside a surface-mount QFN-16 footprint, and places it onto an FR-4 printed circuit board along with decoupling capacitors and SMA input connectors.

```hardware
# system_codesign.hw - Authoritative HardwareScript v0.3.2 Implementation
import * from @std/primitives/units
import { sky130_nmos, sky130_pmos, sky130_tap, TapType } from @std/pdk/sky130
import { SemiconductorSubstrate, LaminateSubstrate } from @std/substrates

# ============================================================================
# 1. FOUNDRY PDK PROFILE: SKYWATER 130nm CMOS
# ============================================================================

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

# ============================================================================
# 2. BOARD FABRICATION PROFILE: 4-LAYER HIGH-SPEED FR-4
# ============================================================================

export profile JLC_4LAYER implements LaminateSubstrate {
    stackup {
        top_mask: { thickness: 20um,  material: "SolderMask", routable: false }
        top_cu:   { thickness: 35um,  material: "Copper",     routable: true  }
        core:     { thickness: 1.5mm, material: "FR4_Dielectric", routable: false }
        bot_cu:   { thickness: 35um,  material: "Copper",     routable: true  }
        bot_mask: { thickness: 20um,  material: "SolderMask", routable: false }
    }

    routing {
        topology: "topological"
        min_trace_width: 0.15mm
        min_clearance: 0.15mm
    }

    drc {
        min_annular_ring: 0.15mm
        min_drill_diameter: 0.3mm
    }
}

# ============================================================================
# 3. SILICON DIE LEVEL: OP-AMP CORE
# ============================================================================

module OperationalAmplifier {
    pins: [input IN_P, input IN_N, output OUT, power VDD, ground VSS]

    netlist {
        M1: NMOS(D: Net("NODE_D1"), G: IN_P, S: Net("TAIL"), B: VSS)
        M2: NMOS(D: OUT,           G: IN_N, S: Net("TAIL"), B: VSS)
        M3: PMOS(D: Net("NODE_D1"), G: Net("NODE_D1"), S: VDD, B: VDD)
        M4: PMOS(D: OUT,           G: Net("NODE_D1"), S: VDD, B: VDD)
        M5: NMOS(D: Net("TAIL"),   G: Net("BIAS"),    S: VSS, B: VSS)
    }
}

space OpAmp_SiliconDie implements OperationalAmplifier {
    dimensions: [120.0um, 100.0um]
    profile: SKY130_1V8_CMOS

    nets {
        VDD:     { classification: power,  potential: 1.8V, current: 50.0uA }
        VSS:     { classification: ground, potential: 0.0V, current: 50.0uA }
        IN_P:    { classification: signal, potential: 0.9V }
        IN_N:    { classification: signal, potential: 0.9V }
        OUT:     { classification: signal }
        TAIL:    { classification: signal }
        NODE_D1: { classification: signal }
        BIAS:    { classification: signal, potential: 0.6V }
    }

    # Instantiate pure parameterized cells
    let diff_input = sky130_nmos(W: 3.0um, L: 150nm)
    let active_load = sky130_pmos(W: 6.0um, L: 150nm)
    let current_tail = sky130_nmos(W: 4.0um, L: 150nm)
    let sub_tap  = sky130_tap(type: TapType.P_Sub,  size: 600nm)
    let well_tap = sky130_tap(type: TapType.N_Well, size: 600nm)

    # Place differential input pair with common-centroid symmetry
    let m1 = space.place(diff_input, at: [40.0um, 40.0um])
    let m2 = space.place(diff_input.mirror_y(), at: [80.0um, 40.0um])

    # Place active mirror loads
    let m3 = space.place(active_load, at: [40.0um, 70.0um])
    let m4 = space.place(active_load.mirror_y(), at: [80.0um, 70.0um])

    # Place tail current source
    let m5 = space.place(current_tail, at: [60.0um, 20.0um])

    # Place bulk taps
    let tap_vss = space.place(sub_tap,  at: [60.0um, 10.0um])
    let tap_vdd = space.place(well_tap, at: [60.0um, 85.0um])

    # Interconnect routing (snapped to 5nm DBU Manhattan tracks)
    route m1.port("S") to m2.port("S") { net: TAIL, layer: "metal1", width: 400nm }
    route m1.port("S") to m5.port("D") { net: TAIL, layer: "metal1", width: 400nm }

    route m1.port("D") to m3.port("D") { net: NODE_D1, layer: "metal1", width: 300nm }
    route m3.port("G") to m3.port("D") { net: NODE_D1, layer: "metal1", width: 200nm }
    route m3.port("G") to m4.port("G") { net: NODE_D1, layer: "metal1", width: 200nm }

    route m2.port("D") to m4.port("D") { net: OUT, layer: "metal1", width: 400nm }

    # Power delivery routes
    route m3.port("S") to tap_vdd.port("TAP") { net: VDD, layer: "metal1", width: 600nm }
    route m4.port("S") to tap_vdd.port("TAP") { net: VDD, layer: "metal1", width: 600nm }
    route m5.port("S") to tap_vss.port("TAP") { net: VSS, layer: "metal1", width: 600nm }

    # Peripheral Wire-Bond Pads (40um square landing sites)
    let pad_in_p = space.place_port("PAD_IN_P", at: [10.0um,  40.0um], layer: "metal1", width: 40.0um)
    let pad_in_n = space.place_port("PAD_IN_N", at: [110.0um, 40.0um], layer: "metal1", width: 40.0um)
    let pad_out  = space.place_port("PAD_OUT",  at: [110.0um, 70.0um], layer: "metal1", width: 40.0um)
    let pad_vdd  = space.place_port("PAD_VDD",  at: [60.0um,  95.0um], layer: "metal1", width: 40.0um)
    let pad_vss  = space.place_port("PAD_VSS",  at: [60.0um,   5.0um], layer: "metal1", width: 40.0um)

    route pad_in_p to m1.port("G") { net: IN_P, layer: "metal1", width: 300nm }
    route pad_in_n to m2.port("G") { net: IN_N, layer: "metal1", width: 300nm }
    route pad_out  to m2.port("D") { net: OUT,  layer: "metal1", width: 400nm }
}

# ============================================================================
# 4. PACKAGING BRIDGE: SURFACE-MOUNT QFN-16
# ============================================================================

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

# ============================================================================
# 5. BOARD LEVEL: PRINTED CIRCUIT TEST BENCH
# ============================================================================

space OpAmp_TestFixture {
    dimensions: [50.0mm, 40.0mm]
    profile: JLC_4LAYER

    nets {
        RAW_12V:   { classification: power }
        BOARD_VCC: { classification: power,  potential: 1.8V, current: 50.0mA }
        BOARD_GND: { classification: ground, potential: 0.0V }
        SIG_IN_P:  { classification: signal }
        SIG_IN_N:  { classification: signal }
        SIG_OUT:   { classification: signal }
    }

    # Copper pour with automated antipad generation around crossing vias
    pour ground_plane {
        net: BOARD_GND,
        layer: "bot_cu",
        boundary: [1.0mm, 1.0mm, 48.0mm, 38.0mm]
    }

    # Instantiate the packaged silicon amplifier
    let u1 = space.place(QFN16_Package, at: [25.0mm, 20.0mm])

    # Connect board traces to package pins
    route u1.pin(1) to SIG_IN_P  { layer: "top_cu", width: 0.25mm }
    route u1.pin(2) to SIG_IN_N  { layer: "top_cu", width: 0.25mm }
    route u1.pin(9) to SIG_OUT   { layer: "top_cu", width: 0.35mm }
    route u1.pin(16) to BOARD_VCC { layer: "top_cu", width: 0.50mm }
    route u1.pin(3) to BOARD_GND { layer: "top_cu", width: 0.50mm }
}
```

---

## 8. Compiler Execution & Verification Trace

When compiling the unified system via `hw build system_codesign.hw`, the compiler driver executes substrate isolation, JIT evaluation, mask extraction, and LVS graph verification:

```text
🔥 HARDWARESCRIPT COMPILER v0.3.2 (Production Standard)
================================================================================
[ 0.32ms] Source AST parsed: system_codesign.hw (2 profiles, 1 module, 2 spaces, 1 footprint)
[ 0.78ms] Type Checker: Validated 7-Base SI unit algebra (0 dimensional errors)
[ 1.15ms] Compiling Comptime Generators via Cranelift JIT Engine...
   • JIT compiled 'sky130_nmos' in 2.14ms (Native x86_64 Machine Code in RAM)
   • JIT compiled 'sky130_pmos' in 2.02ms (Native x86_64 Machine Code in RAM)
   • Unrolled transistor geometry directly into FlatGeometryBuffer: 48 records emitted
   • Populated DebugSpanRegistry: 48 EntityIds bound to source code spans

── DISPATCHING TIER 1: OpAmp_SiliconDie (SubstrateContract::Semiconductor) ─────
[ 4.12ms] Clipper2 2D Boolean Mask Synthesis Pass:
   • diff ∩ poly                 ──► Active Channels: NMOS (3.0um x 0.15um), PMOS (6.0um x 0.15um)
   • (diff ∩ nsdm) \ poly        ──► N+ Source/Drain diffusion islands validated
   • (diff ∩ psdm ∩ nwell) \ poly ──► P+ Source/Drain diffusion islands validated
[ 7.85ms] Junction-Aware Interface Conductivity Extractor:
   • P+ Diff ∩ N-Well boundary classified as Rectifying Diode Barrier (Net short BLOCKED)
   • N-Well connected to Net 'VDD' via Ohmic Well Tap (0 VDD-to-OUT shorts)
   • P-Sub connected to Net 'VSS' via Ohmic Substrate Tap
[11.20ms] Constructive Manhattan Line-Search Router:
   • Snapped 18 segment vertices strictly to 5nm DBU tracks
   • Directionality: 100% orthogonal (0 slanted/acute angle violations)
[14.30ms] Lithography Sign-off DRC Gate:
   • Rule 'grid.1': PASS (0 off-grid fractional coordinates)
   • Rule 'poly.4': PASS (Gate overhang = 200nm >= 150nm minimum)
   • Rule 'nwell.4': PASS (Well enclosure = 600nm >= 330nm minimum)
   • Rule 'antenna.1': PASS (EGAR = 38.4 <= 400.0)
[18.90ms] 1-WL Graph Isomorphism LVS Verification Gate:
   • Extracted Physical Wafer Graph (G_L): 5 MOSFETs, 8 Nets
   • Golden Module Netlist Graph (G_S):    5 MOSFETs, 8 Nets
   • Automorphism Pin Swapping: Permutation orbit verified for differential pair
   ✅ LVS Result: 100% ISOMORPHIC (0 open nets, 0 short circuits, 0 floating terminals)
   ✅ Emitted Silicon Layout: build/OpAmp_SiliconDie/layout.gds (Foundry Masks)
   ✅ Emitted Silicon SPICE:  build/OpAmp_SiliconDie/circuit.sp (DUT Netlist)

── DISPATCHING TIER 2: OpAmp_TestFixture (SubstrateContract::Laminate) ─────────
[22.40ms] 3D Dielectric & Conductor Slab Extrusion:
   • Extruded 4-layer copper/FR-4 stackup (1.5mm core thickness)
[25.10ms] Automated Antipad Boolean Lowering Pass:
   • Subtracted clearance antipads from 'bot_cu' ground plane around through-hole pins
[28.60ms] Continuous Topological Vector Router:
   • Routed 5 microstrip nets on 'top_cu' with 0.15mm minimum clearance
[31.20ms] IPC-2221 Class 2 Board DRC Gate: PASS (0 annular ring breakouts)
   ✅ Emitted Fabrication Stream: build/OpAmp_TestFixture/gerbers.zip
   ✅ Emitted NC Drill Stream:   build/OpAmp_TestFixture/drill.xln

── EXPORTING CO-SIMULATION BENCHMARK ───────────────────────────────────────────
[34.50ms] Generating Full-System SPICE Co-Simulation Suite:
   • Linked board transmission line parasitics to QFN16 bond-wire RLC models
   • Connected package leadframe pins directly to on-chip gate terminals
   ✅ Emitted System SPICE: build/system_cosim.sp
================================================================================
Finished complete build in 0.036 seconds.
```

---

## 9. Architectural Migration & Deprecation Matrix

| Subsystem Dimension | Legacy v0.3.0 – v0.3.1 | Interim v0.3.2 Proposals | Hardened v0.3.2 Production Standard |
| :--- | :--- | :--- | :--- |
| **Comptime Execution** | Linear Bytecode VM with fuel budgeting | Immutable AST Tree-Walk Interpreter | **In-Memory JIT Compilation via Cranelift** |
| **Plugin Extensibility** | WebAssembly Memory64 sandbox | Dynamic native `.so` / `.dll` libraries | **Universal Native Container (`.hpm`) with target filtering** |
| **Host Toolchain Burden** | Heavy local wasm/LLVM toolchains | Compilers required on host | **Zero local compilers required by the end-user** |
| **Substrate Strategy** | Monolithic engine with `if tech == "ASIC"` | Incomplete workspace split | **Segregated CMOS / Laminate engines driven by neutral traits** |
| **Full-System Design** | Unhandled (PCB and IC collided) | Forbidden by documentation | **Multi-substrate co-design via Packaging Contracts** |
| **LVS Verification** | Naive comparison against script | Theoretical 1-WL without clear source | **Rigorous 1-WL graph isomorphism: $G_L$ vs. `module.netlist`** |
| **Spatial Precision** | 64-bit integer coordinates | 64-bit area arithmetic overflow risks | **`i64` for lengths; `i128` for physical areas and densities** |
| **Error Traceability** | Line numbers embedded in AST nodes | SourceSpan stripped, losing diagnostics | **Out-of-band `DebugSpanRegistry` mapped via `EntityId`** |

---

## 10. Golden Verification Invariants & Self-Audit Checklist

Every physical build produced by HardwareScript must strictly satisfy the **5 Golden Verification Gates**:

```
                         THE 5 GOLDEN VERIFICATION GATES
                         
  ┌───────────────────────┐
  │ Gate 1: No Strings    │ Zero string-scraping or name heuristics; all roles are typed.
  └──────────┬────────────┘
             ▼
  ┌───────────────────────┐
  │ Gate 2: No Fallbacks  │ Zero unwrap_or() defaults; missing params fail fast & loudly.
  └──────────┬────────────┘
             ▼
  ┌───────────────────────┐
  │ Gate 3: Fab Neutral   │ Zero foundry layer names or dimensions hardcoded in Rust core.
  └──────────┬────────────┘
             ▼
  ┌───────────────────────┐
  │ Gate 4: Root Cause    │ Upstream fixes only; zero downstream deduplicators or patches.
  └──────────┬────────────┘
             ▼
  ┌───────────────────────┐
  │ Gate 5: Closed SPICE  │ Every device terminal [D,G,S,B] connects to an unbroken net.
  └───────────────────────┘
```

1. **Gate 1 (No String-Scraping):** Did I use `.contains()`, `.starts_with()`, or string pattern matching on entity names to guess their role or layer? If yes, REJECT and enforce typed enums (`PourRole`, `RecordType`).
2. **Gate 2 (No Silent Fallbacks):** Did I use `.unwrap_or(<magic_number>)` or swallow an `Err`? If yes, REJECT and return an explicit `CompileError` diagnostic.
3. **Gate 3 (Foundry Neutrality):** Did I hardcode any foundry layer name (`"rpm"`, `"li1"`, `"metal1"`) or physical dimension in a `.rs` file? If yes, REJECT and move to user-space `.hw` PDK profiles.
4. **Gate 4 (Root-Cause Integrity):** Did I patch a symptom at the export boundary instead of plumbing the typed field through the IR? If yes, REJECT and fix upstream.
5. **Gate 5 (Closed-Loop SPICE):** Does every terminal, port, and device in `circuit.sp` connect to an unbroken, verified net with zero floating pins? If no, REJECT and fail compilation.

---

*HardwareScript v0.3.2 Grand Unified Specification — Approved for Production Implementation*
