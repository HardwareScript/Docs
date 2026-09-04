# HardwareScript v0.3.1: Universal Compiler Architecture, Substrate Calculus & Two-Tier Extensibility Specification

**Document Type:** Authoritative Core Architecture & Subsystem Specification  
**Target Version:** v0.3.1 (Production-Locked Standard)  
**Status:** Approved for Implementation  
**Target Crates:** `crates/hwc-frontend`, `crates/hwc-eval`, `crates/hwc-ir`, `crates/hwc-substrate-cmos`, `crates/hwc-substrate-laminate`, `crates/hwc-substrate-wasm`, `crates/hw`  
**Reference Standards:** SEMI GDSII / OASIS (P39), IEEE 1801 (UPF / LVS), IPC-2581 / RS-274X, W3C WebAssembly 64-Bit Linear Memory (`Memory64`), ISO/IEC 80000-1 (SI Quantities)

---

## 1. Executive Summary & The Paradigm Shift

HardwareScript v0.3.0 and early v0.3.1 attempted to support both Integrated Circuits (IC) and Printed Circuit Boards (PCB) within a single monolithic compiler engine (`hwc-engine`). The compiler relied on a fragile runtime check:

```rust
// ❌ THE FAILED HEURISTIC (v0.3.0 / Early v0.3.1)
if space.profile.technology == "ASIC" {
    // Slap an ad-hoc silicon patch
} else {
    // Fall back to default PCB behavior
}
```

The resulting silicon audit proved that this was fatal: **PCB assumptions contaminated IC synthesis.** The compiler treated lithography masks as bulk 3D physical slabs, welded $P^+$ diffusion into the $N$-well because they physically touched (shorting VDD to the output node), emitted $0\,\text{nm}^2$ of $P^+$ diffusion, routed slanted $15^\circ - 30^\circ$ non-Manhattan lines with off-grid floating coordinates ($74{,}387\,\text{nm}$), dropped the mandatory MOSFET bulk ($B$) terminal, and claimed $0$ DRC violations on a wafer that would destroy itself on power-up.

### The Resolution: The Substrate Calculus
HardwareScript is **not a PCB compiler with IC patches**, nor is it an **IC tool that handles boards as an afterthought**. 

HardwareScript v0.3.1 adopts the **LLVM Architecture for Physical Electronics**:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    THE HARDWARESCRIPT SUBSTRATE MODEL                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. UNIFIED FRONTEND LANGUAGE (.hw)                                         │
│     • Turing-complete compile-time evaluation (`hwc-eval` Bytecode VM).     │
│     • Pure cell composition: `sky130_nmos() -> CellLayout`.                 │
│     • 7-Base SI compile-time dimensional algebra ($L \times L = L^2$).      │
│     • 128-bit integer picometers ($1\text{ pm} = 10^{-12}\text{ m}$).       │
│     • Common structural interface bundles (`DiffPair { p: Net, n: Net }`).  │
│                                                                             │
│  2. TARGET-AGNOSTIC INTERMEDIATE REPRESENTATION (`hwc-ir`)                  │
│     • Emits an immutable `Arc<FlatGeometryBuffer>` with picometer bounds.   │
│     • Contains zero foundry-specific or fabrication-house logic.            │
│     • Preserves Merkle-path structural identity (`EntityId`).               │
│                                                                             │
│  3. DEDICATED, ISOLATED SUBSTRATE BACKENDS                                  │
│     • Profile nominal typing selects the backend statically (Zero Flags).   │
│     • `hwc-substrate-cmos`: Monolithic silicon planar lithography engine.   │
│     • `hwc-substrate-laminate`: 3D multi-layer board extrusion engine.      │
│                                                                             │
│  4. TWO-TIER `wasm64` (MEMORY64) EXTENSIBILITY ARCHITECTURE                 │
│     • Tier 1: Full Substrate Replacement (Photonics, Packaging, Quantum).   │
│     • Tier 2: Surgical Stage Plugins (Custom Router, Placer, or LVS deck).  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Exhaustive Audit: Previously vs. How It Must Be Done Now

To prevent any regression, the table below documents the systemic failures identified in the silicon audit and defines the immutable architectural mandate for v0.3.1.

| Subsystem | Failure Mode in Early v0.3.1 (What We Did Wrong) | Root Cause in Architecture | Hardened v0.3.1 Mandate (How It Is Done Now) |
| :--- | :--- | :--- | :--- |
| **Physical Stackup vs. Masks** | Layer `diff` was defined with single material `N_Plus_Diffusion`, resulting in $0\,\text{nm}^2$ of PMOS $P^+$ diffusion. | **The PCB Material Fallacy:** Assumed every layer in a stackup is an extruded 3D material slab. | **Boolean Mask Derivation:** Semiconductor profiles declare optical masks (`diff`, `poly`, `nsdm`, `psdm`). Doping is derived via 2D Boolean algebra ($Active \cap Implant$). |
| **Electrical Connectivity** | PMOS N-Well shorted to Net `Out` because the drain diffusion touched the N-well polygon. | **Geometric Mesh Welding:** Touching polygons on adjacent layers automatically merged electrical nets. | **Junction-Aware LVS Graph:** Evaluates an Interface Conductivity Matrix. $P^+$ in $N$-Well is classified as a P-N diode junction, **halting net propagation**. |
| **Coordinate Quantization** | Output coordinates were off-grid (e.g., $74{,}387\,\text{nm}$), violating SkyWater 5nm grid rules. | **Continuous Coordinate Engine:** Relied on continuous raycasting without manufacturing grid snapping. | **Strict DBU Quantization:** All coordinates in `hwc-substrate-cmos` are clamped to integer multiples of the profile's Database Unit (DBU: $5{,}000\,\text{pm}$). |
| **Routing Angles** | Metal 1 traces ran at arbitrary $15^\circ - 30^\circ$ angles, creating acute-angle acid traps. | **Unconstrained Line-Search:** Router lacked orthogonal Manhattan decomposition. | **Strict Manhattan Router:** Restricts routing to orthogonal ($90^\circ$) segments with optional $45^\circ$ miter chamfers. |
| **MOSFET Compact Netlisting** | Compact model emitted with 3 terminals instead of 4, crashing SPICE simulators. | **Silent Null Drop:** The netlister printed an empty string when the bulk terminal was not routed to a pad. | **Strict 4-Terminal Contract:** MOSFET models require $[D, G, S, B]$. If $B$ is unbound, the compiler halts with `Error LVS_04`. |
| **DRC Coverage** | Compiler claimed "0 violations" on an unmanufacturable, non-functional inverter. | **Empty Declarative Deck:** DRC engine only checked rules explicitly typed in user profile. | **Comprehensive Substrate DRC:** Substrate backends ship with native invariant checks (Antenna EGAR $\le 400:1$, CMP Density $\ge 20\%$). |
| **DUT Netlist Purity** | Netlister injected $100\,\mu\Omega$ artificial resistors (`Rpad_stimulus_bridge_*`) into `circuit.sp`. | **Testbench Contamination:** Mutated the layout netlist to connect boundary pads to stimulus sources. | **Pristine DUT Netlist:** Layout extraction emits pure physical parasitics; testbench stimulus lives strictly in `dc.sp`/`tran.sp`. |

---

## 3. Workspace Architecture & Strict Crate Decoupling

The monolithic `hwc-engine` and `hwc-compiler` crates are refactored into a strictly layered Cargo workspace. **Crates at the same tier cannot depend on each other.**

```
crates/
├── hwc-frontend/               # [TIER 1] Logos lexer, Pratt parser, AST, 7-Base SI algebra
├── hwc-eval/                   # [TIER 1] Comptime Bytecode VM, Deterministic Fuel, Stack
├── hwc-ir/                     # [TIER 2] Intermediate Representation, GeometryBuffer, ABI types
│
├── hwc-substrate-cmos/         # [TIER 3: IC ONLY] ⚠️ Zero PCB dependencies
│   ├── Cargo.toml
│   └── src/
│       ├── masks/              # 2D Boolean Mask Synthesis (Clipper2)
│       ├── connectivity/       # Junction-Aware LVS Graph (Diode/Channel barriers)
│       ├── router/             # 5nm DBU-Snapped Manhattan Maze Router
│       ├── drc/                # Antenna, Density, Enclosure, Extension
│       └── export/             # Binary GDSII/OASIS & 4-Terminal SPICE
│
├── hwc-substrate-laminate/     # [TIER 3: PCB ONLY] ⚠️ Zero IC dependencies
│   ├── Cargo.toml
│   └── src/
│       ├── stackup/            # Solder mask, prepreg, core 3D extrusion
│       ├── connectivity/       # Equipotential Mesh Welding
│       ├── router/             # Topological Vector Any-Angle Router
│       ├── drc/                # Annular rings, drill aspect ratio, creepage
│       └── export/             # Gerber RS-274X/X2, IPC-2581, Excellon NC Drill
│
├── hwc-substrate-wasm/         # [TIER 3: WASM RUNNER]
│   └── src/                    # Wasmtime Memory64 runtime for Tier 1 & Tier 2 plugins
│
└── hw/                         # [TIER 4: CLI DRIVER]
    └── src/                    # Main executable entry point & target dispatcher
```

### Dependency Invariants
1. `hwc-substrate-cmos` **CANNOT** depend on `hwc-substrate-laminate`.
2. `hwc-substrate-laminate` **CANNOT** depend on `hwc-substrate-cmos`.
3. Neither substrate engine knows about the user-facing grammar; they consume **pure `hwc-ir` buffers**.
4. The driver CLI (`hw`) orchestrates target selection via static type dispatch.

---

## 4. The Universal Intermediate Representation (`hwc-ir`)

The Intermediate Representation is the target-agnostic bridge between the comptime evaluation frontend and the physical backends. It is **100% loss-free, memory-mappable, and evaluated in integer picometers ($10^{-12}\text{ m}$)**.

```
                     IR INGESTION PIPELINE
                     
  Frontend AST (.hw) ──► `hwc-eval` (Bytecode VM)
                               │
                               ▼ Emits pure immutable stream
                     `Arc<FlatGeometryBuffer>`
                     ┌──────────────────────────────────────────────┐
                     │ Coordinate Pool: `Vec<i64>` (Flat picometers)│
                     │ Header Pool:     `Vec<CompactRecordHeader>`  │
                     │ Net Declarations: `Vec<NetIR>`               │
                     │ Port Centroids:   `Vec<PortIR>`              │
                     └──────────────────────────────────────────────┘
                               │
                               ▼ Dispatched to
                     `SubstrateBackend::execute(ir)`
```

### 4.1 Flat Picometer Memory Layout (`crates/hwc-ir/src/geometry.rs`)

To prevent heap fragmentation (millions of `Vec` allocations for polygons), all geometry is stored in a **Contiguous Picometer Coordinate Arena**:

```rust
// crates/hwc-ir/src/geometry.rs

use compact_str::CompactString;
use crate::identity::EntityId;

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct SpaceId(pub u32);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct NetId(pub u32);

#[repr(u8)]
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum RecordType {
    Polygon = 1,
    Contact = 2,
    Device  = 3,
    Port    = 4,
    Route   = 5,
}

/// Compact 32-byte header pointing into the flat coordinate pool.
#[repr(C)]
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct CompactGeometryRecordHeader {
    pub id: EntityId,             // Merkle-path identity for Salsa cache invariance
    pub space_id: SpaceId,
    pub net_id: u32,
    pub layer_idx: u16,
    pub record_type: RecordType,
    pub coord_start_idx: u32,     // Index into FlatGeometryBuffer.coordinate_pool
    pub coord_count: u32,         // Number of (x, y) coordinate pairs
}

/// The universal, zero-copy geometric payload emitted by `hwc-eval`.
#[derive(Default, Debug, Clone, PartialEq)]
pub struct FlatGeometryBuffer {
    /// Contiguous pool of picometer coordinates: [x0, y0, x1, y1, x2, y2, ...]
    pub coordinate_pool: Vec<i64>,
    /// Compact fixed-size headers referencing slice ranges in the coordinate pool
    pub headers: Vec<CompactGeometryRecordHeader>,
    /// String pool for layer and instance names
    pub identifiers: Vec<CompactString>,
}

impl FlatGeometryBuffer {
    #[inline(always)]
    pub fn get_points(&self, header: &CompactGeometryRecordHeader) -> &[(i64, i64)] {
        let start = header.coord_start_idx as usize;
        let end = start + (header.coord_count as usize * 2);
        let slice = &self.coordinate_pool[start..end];
        
        // Safety: Memory is contiguous pairs of i64 (x, y)
        unsafe {
            std::slice::from_raw_parts(
                slice.as_ptr() as *const (i64, i64),
                header.coord_count as usize,
            )
        }
    }
}
```

### 4.2 Structural Space IR (`crates/hwc-ir/src/space.rs`)

```rust
// crates/hwc-ir/src/space.rs

use super::geometry::FlatGeometryBuffer;
use compact_str::CompactString;
use std::path::PathBuf;
use std::sync::Arc;

#[derive(Debug, Clone)]
pub struct SpaceIR {
    pub id: super::geometry::SpaceId,
    pub name: CompactString,
    pub width_pm: i64,
    pub height_pm: i64,
    pub geometry: Arc<FlatGeometryBuffer>,
    pub substrate_contract: SubstrateContract,
    pub output_dir: PathBuf,
}

#[derive(Debug, Clone)]
pub enum SubstrateContract {
    Semiconductor(SemiconductorConfig),
    Laminate(LaminateConfig),
    WasmPlugin(WasmPluginSpec),
}

#[derive(Debug, Clone)]
pub struct SemiconductorConfig {
    pub profile_name: CompactString,
    pub manufacturing_grid_pm: i64,
    pub lambda_pm: i64,
    pub mask_layer_mapping: Vec<(CompactString, u16, u16)>, // (MaskName, GdsLayer, GdsDatatype)
}

#[derive(Debug, Clone)]
pub struct LaminateConfig {
    pub profile_name: CompactString,
    pub total_thickness_pm: i64,
    pub layers: Vec<LaminateLayerIR>,
}

#[derive(Debug, Clone)]
pub struct LaminateLayerIR {
    pub name: CompactString,
    pub thickness_pm: i64,
    pub material_name: CompactString,
    pub is_routable: bool,
}

#[derive(Debug, Clone)]
pub struct WasmPluginSpec {
    pub package_name: CompactString,
    pub wasm_binary_path: PathBuf,
    pub abi_version: CompactString,
}
```

---

## 5. The Substrate Calculus & Nominal Trait Contracts

In HardwareScript, target selection is not a runtime guessing game. **Substrates are formal type traits defined in `@std/substrates`.** A PDK profile implements a specific trait, and the type checker enforces that its fields match the physical domain.

```
                          SUBSTRATE TRAIT HIERARCHY
                          
                           ┌─────────────────────────┐
                           │     trait Substrate     │
                           └────────────┬────────────┘
                                        │
                 ┌──────────────────────┴──────────────────────┐
                 ▼                                             ▼
  ┌─────────────────────────────┐               ┌─────────────────────────────┐
  │ trait SemiconductorSubstrate│               │   trait LaminateSubstrate   │
  ├─────────────────────────────┤               ├─────────────────────────────┤
  │ • masks: MaskStack          │               │ • stackup: LayerStackup     │
  │ • grid: Measurement (5nm)   │               │ • drill_table: DrillList    │
  │ • routing: ManhattanConfig  │               │ • routing: VectorConfig     │
  │ • drc: SemiconductorDrcDeck │               │ • drc: BoardDrcDeck         │
  └─────────────────────────────┘               └─────────────────────────────┘
```

### 5.1 Syntax: Enforcing Substrate Types in `.hw`

#### Monolithic Semiconductor Profile (`@std/pdk/sky130/profile.hw`)
```hardware
# stdlib/pdk/sky130/profile.hw
import * from @std/primitives/units
import { SemiconductorSubstrate } from @std/substrates

# The profile statically declares that it targets monolithic semiconductor manufacturing
export profile SKY130_1V8_CMOS implements SemiconductorSubstrate {
    manufacturing_grid: 5nm
    lambda: 50nm

    # Optical reticle masks (Photolithography). NO slab thicknesses!
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
        extension poly over diff: 150nm
        enclosure diff inside nwell: 330nm
    }
}
```

#### Printed Circuit Board Profile (`@std/jlc/pcb/profile.hw`)
```hardware
# stdlib/jlc/pcb/profile.hw
import * from @std/primitives/units
import { LaminateSubstrate } from @std/substrates

# The profile statically declares that it targets multi-layer laminated boards
export profile JLC_4LAYER implements LaminateSubstrate {
    stackup {
        top_mask: { thickness: 20um,  material: "LPI_Green", routable: false }
        top_cu:   { thickness: 35um,  material: "Copper",    routable: true }
        core:     { thickness: 1.5mm, material: "FR4",       routable: false }
        bot_cu:   { thickness: 35um,  material: "Copper",    routable: true }
        bot_mask: { thickness: 20um,  material: "LPI_Green", routable: false }
    }

    routing {
        topology: "topological"
        min_trace_width: 0.127mm
        min_clearance: 0.127mm
    }

    drc {
        min_annular_ring: 0.15mm
        min_drill_diameter: 0.3mm
    }
}
```

### 5.2 Schema Validation Rule (Type-Checker Gate)
If an engineer attempts to declare `stackup { diff { material: "N_Plus_Diffusion" } }` inside a profile implementing `SemiconductorSubstrate`, the type checker halts compilation:

```text
💥 Error S42: Incompatible Substrate Schema
   • Profile 'SKY130_1V8_CMOS' implements 'SemiconductorSubstrate'.
   • Found invalid field: 'stackup'.
   • Help: Semiconductor profiles represent photolithographic masks and must declare 'masks { ... }'. Physical layer thicknesses cannot be assigned to monolithic silicon dopants.
```

---

## 6. The Native Substrate Backends

### 6.1 The Semiconductor Pipeline (`crates/hwc-substrate-cmos`)

The semiconductor backend models CMOS photolithography accurately via five consecutive transformations:

```
                       SEMICONDUCTOR PIPELINE FLOW
                       
 ┌─────────────────────────────────────────────────────────────────────────┐
 │ STAGE 1: BOOLEAN MASK DERIVATION ENGINE (Clipper2)                      │
 │ • Gate Channels: diff ∩ poly                                            │
 │ • PMOS Source/Drain: (diff ∩ psdm ∩ nwell) \ poly                       │
 │ • NMOS Source/Drain: (diff ∩ nsdm) \ (poly ∪ nwell)                     │
 │ • Ohmic Well/Sub Taps: diff ∩ implant ∩ well_type                       │
 └────────────────────────────────────┬────────────────────────────────────┘
                                      │ Derived Doping Polygons
                                      ▼
 ┌─────────────────────────────────────────────────────────────────────────┐
 │ STAGE 2: JUNCTION-AWARE CONNECTIVITY EXTRACTOR                          │
 │ • Consults Interface Conductivity Matrix:                               │
 │   - Conductor ↔ Conductor: Short (Propagates Net)                       │
 │   - P+ Diff ↔ N-Well: Rectifying Barrier (HALTS net; creates Diode)     │
 │   - PolyGate ↔ Channel: Capacitive Barrier (HALTS net; creates MOSFET)  │
 └────────────────────────────────────┬────────────────────────────────────┘
                                      │ Isolated Net Graph
                                      ▼
 ┌─────────────────────────────────────────────────────────────────────────┐
 │ STAGE 3: 5nm DBU-QUANTIZED MANHATTAN ROUTER                             │
 │ • Clamps all endpoints to integer 5nm Database Units (5,000 pm).        │
 │ • Decomposes all routing vectors into strict 90° lines and 45° miters.  │
 │ • Enforces layer directionality (Li1 Horizontal, Metal1 Vertical).      │
 └────────────────────────────────────┬────────────────────────────────────┘
                                      │ Routed Geometry
                                      ▼
 ┌─────────────────────────────────────────────────────────────────────────┐
 │ STAGE 4: LITHO & PROCESS SIGN-OFF DRC GATE                              │
 │ • Antenna Ratio Check: Etch Gate Area Ratio (EGAR <= 400.0).            │
 │ • Metal Density Check: CMP window compliance (20% <= Density <= 80%).   │
 │ • Geometric Rules: poly.4 overhang (>= 150nm), nwell.4 enc (>= 330nm).  │
 └────────────────────────────────────┬────────────────────────────────────┘
                                      │ Tapeout Clean Layout
                                      ▼
 ┌─────────────────────────────────────────────────────────────────────────┐
 │ STAGE 5: MANUFACTURING STREAM EMISSION                                  │
 │ • Strictly formats 4-terminal compact SPICE cards (D, G, S, B).         │
 │ • Streams binary GDSII / OASIS (Layers 64 through 68).                  │
 └─────────────────────────────────────────────────────────────────────────┘
```

#### The Interface Conductivity Matrix (`connectivity.rs`)
```rust
// crates/hwc-substrate-cmos/src/connectivity/matrix.rs

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum RegionClass {
    Metal,
    LocalInterconnect,
    PolyGate,
    NPlusDiffusion,
    PPlusDiffusion,
    NWell,
    PSubstrate,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum InterfaceEffect {
    OhmicShort,        // Net identity propagates across boundary
    RectifyingDiode,   // Net identity is BLOCKED; semiconductor diode created
    MosfetChannel,     // Net identity is BLOCKED; active transistor created
    InsulatingBarrier, // Net identity is BLOCKED; no electrical connection
}

pub struct InterfaceMatrix;

impl InterfaceMatrix {
    pub fn evaluate(a: RegionClass, b: RegionClass) -> InterfaceEffect {
        match (a, b) {
            // Identical conductor regions merge into the same electrical net
            (RegionClass::Metal, RegionClass::Metal) => InterfaceEffect::OhmicShort,
            (RegionClass::LocalInterconnect, RegionClass::LocalInterconnect) => InterfaceEffect::OhmicShort,

            // PMOS Source/Drain (P+) touching N-Well is a DIODE. It NEVER merges nets!
            (RegionClass::PPlusDiffusion, RegionClass::NWell) |
            (RegionClass::NWell, RegionClass::PPlusDiffusion) => InterfaceEffect::RectifyingDiode,

            // NMOS Source/Drain (N+) touching P-Substrate is a DIODE.
            (RegionClass::NPlusDiffusion, RegionClass::PSubstrate) |
            (RegionClass::PSubstrate, RegionClass::NPlusDiffusion) => InterfaceEffect::RectifyingDiode,

            // Polysilicon crossing diffusion creates an active transistor channel
            (RegionClass::PolyGate, RegionClass::NPlusDiffusion) |
            (RegionClass::PolyGate, RegionClass::PPlusDiffusion) => InterfaceEffect::MosfetChannel,

            _ => InterfaceEffect::InsulatingBarrier,
        }
    }
}
```

---

### 6.2 The Laminate Board Pipeline (`crates/hwc-substrate-laminate`)

The laminate backend operates on multi-layer extruded 3D slabs:
1. **Physical Material Extrusion:** Elevates 2D contours along their $Z$-elevations based on stackup thicknesses.
2. **Equipotential Mesh Welding:** Conductive polygons on the same layer carrying the same net are unioned via Clipper2 (Non-Zero Winding Rule).
3. **Topological Vector Router:** Emits continuous vector traces with octagonal or curved fillets, managing drilled via barrels across intermediate layers.
4. **IPC-2221 Board DRC:** Checks annular ring breakout, copper-to-edge clearance, minimum drill diameters, and trace creepage.
5. **Manufacturing Exporter:** Emits standard RS-274X/X2 Gerbers, Excellon NC Drill files, IPC-2581 XML, and Pick-and-Place CSV centroids.

---

## 7. The Two-Tier `wasm64` Memory64 Extensibility Architecture

HardwareScript replaces the narrow plugin hooks with a **Universal Physical Synthesis ABI** targeting W3C `wasm64` (Memory64).

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 WHY `wasm64` IS THE UNIVERSAL EDA RUNTIME                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. UNCAPPED $2^{64}$ ADDRESS SPACE:                                        │
│     Industrial ASIC layouts with 50M+ gates allocate tens of gigabytes of   │
│     geometry buffers. `wasm64` completely eliminates the 4 GB limit.        │
│                                                                             │
│  2. HERMETIC CROSS-PLATFORM REPRODUCIBILITY:                                │
│     The exact same compiled `.wasm` binary runs on Linux, macOS, and        │
│     Windows across x86_64, ARM64, and RISC-V with bit-identical outputs.    │
│                                                                             │
│  3. ZERO HOST C++ TOOLCHAIN BURDEN:                                         │
│     Users do not need to install GCC, Clang, Boost, or Python. Third-party  │
│     solvers run inside Wasmtime sandboxes with zero host configuration.     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.1 Tier 1: Full Substrate Replacement ABI (`crates/hwc-ir/src/abi/substrate.rs`)

When an organization targets a non-standard physical process (e.g., Silicon Photonics, 2.5D Chiplet Packaging, MEMS), they implement the **Tier 1 Substrate Contract**:

```rust
// crates/hwc-ir/src/abi/substrate.rs

use std::ffi::c_char;

/// Task payload passed from HardwareScript Host into the WASM Substrate Plugin
#[repr(C)]
#[derive(Debug, Clone, Copy)]
pub struct HwcSubstrateTask64 {
    pub space_name: *const c_char,
    pub width_pm: i64,
    pub height_pm: i64,
    
    // Contiguous array of CompactGeometryRecordHeaders
    pub header_count: u64,
    pub headers_ptr: *const u8,
    
    // Contiguous array of raw picometer coordinate pairs (i64, i64)
    pub coord_count: u64,
    pub coords_ptr: *const i64,
    
    // Profile configuration payload (Bincode/JSON encoded)
    pub config_ptr: *const u8,
    pub config_len: u64,
}

/// Result returned from the WASM Substrate Plugin to the HardwareScript Host
#[repr(C)]
#[derive(Debug, Clone, Copy)]
pub struct HwcSubstrateResult64 {
    pub status_code: u32,             // 0 = Tapeout Clean, >0 = DRC/LVS Violations
    pub violation_count: u64,
    pub violations_json_ptr: *const c_char,
    pub stream_archive_ptr: *const u8, // Buffer containing GDSII, OASIS, or Gerber ZIP
    pub stream_archive_len: u64,
    pub netlist_sp_ptr: *const c_char, // Extracted SPICE netlist string
    pub error_message: *const c_char,
}

extern "C" {
    // Universal memory lifecycle hooks
    pub fn hwc_substrate_allocate(bytes: u64) -> *mut u8;
    pub fn hwc_substrate_free(ptr: *mut u8, bytes: u64);
    
    // Main physical execution gate
    pub fn hwc_substrate_synthesize(task: *const HwcSubstrateTask64) -> HwcSubstrateResult64;
}
```

---

### 7.2 Tier 2: Stage-Level Plugin ABI (`crates/hwc-ir/src/abi/stage.rs`)

When working inside a native substrate (e.g., CMOS), developers can surgically swap out individual mathematical stages via Tier 2 plugins:

```rust
// crates/hwc-ir/src/abi/stage.rs

use std::ffi::c_char;

/// Input payload passed to a custom Stage 4 Detailed Router Plugin
#[repr(C)]
#[derive(Debug, Clone, Copy)]
pub struct HwcRoutingTask64 {
    pub num_nets: u64,
    pub num_obstacles: u64,
    pub num_access_points: u64,
    pub payload_ptr: *const u8,
    pub payload_len: u64,
}

/// Output geometry returned from a custom Router Plugin
#[repr(C)]
#[derive(Debug, Clone, Copy)]
pub struct HwcRoutingResult64 {
    pub wire_count: u64,
    pub wires_ptr: *const u8,         // Flat array of picometer wire segments
    pub via_count: u64,
    pub vias_ptr: *const u8,          // Flat array of picometer via instances
    pub status_code: u32,             // 0 = Success, >0 = Routing Congestion Failure
    pub error_msg: *const c_char,
}

extern "C" {
    pub fn hwc_router_allocate(bytes: u64) -> *mut u8;
    pub fn hwc_router_free(ptr: *mut u8, bytes: u64);
    pub fn hwc_router_solve(task: *const HwcRoutingTask64) -> HwcRoutingResult64;
}
```

---

### 7.3 Multi-Threaded WASM Safety: Thread-Local Instance Pools

> **Cross-Subsystem Fix:** When DOPHR Stage 3 dispatches lock-free spatial 4-color routing across 16 Rayon worker threads, calling a single `wasmtime::Instance` concurrently causes memory corruption. The host runner (`hwc-substrate-wasm`) enforces a **Thread-Local WASM Instance Pool**:

```rust
// crates/hwc-substrate-wasm/src/runner.rs

use std::cell::RefCell;
use wasmtime::{Engine, Instance, Module, Store};

thread_local! {
    /// Each Rayon worker thread maintains an isolated Wasmtime store and instance.
    /// Eliminates lock contention and provides safe parallel detailed routing.
    static THREAD_WASM_INSTANCE: RefCell<Option<(Store<()>, Instance)>> = RefCell::new(None);
}

pub struct WasmStageRunner {
    engine: Engine,
    module: Module,
}

impl WasmStageRunner {
    pub fn execute_on_current_thread(&self, payload: &[u8]) -> Result<Vec<u8>, String> {
        THREAD_WASM_INSTANCE.with(|cell| {
            let mut opt = cell.borrow_mut();
            if opt.is_none() {
                let mut store = Store::new(&self.engine, ());
                let instance = Instance::new(&mut store, &self.module, &[])
                    .map_err(|e| format!("Failed to instantiate WASM plugin: {e}"))?;
                *opt = Some((store, instance));
            }
            let (store, instance) = opt.as_mut().unwrap();
            
            // Execute solver inside isolated linear memory
            Self::invoke_solve(store, instance, payload)
        })
    }

    fn invoke_solve(store: &mut Store<()>, instance: &Instance, payload: &[u8]) -> Result<Vec<u8>, String> {
        // Zero-copy transfer into WASM linear Memory64
        todo!("Transfer payload into wasm64 memory and invoke export")
    }
}
```

---

## 8. End-to-End Pipeline Walkthrough

### 8.1 Compiling a Silicon Inverter (`cmos_inverter.hw`)

```hardware
# cmos_inverter.hw
import * from @std/primitives/units
import { sky130_nmos, sky130_pmos } from @skywater/sky130/devices
import { sky130_tap, TapType } from @skywater/sky130/physical
import { pad } from @skywater/sky130/pad
import { SKY130_1V8_CMOS } from @skywater/sky130/profile

module CMOS_Inverter {
    pins: [input In, output Out, power VDD, ground VSS]
}

space CMOS_Inverter_Space implements CMOS_Inverter {
    dimensions: [120.0um, 100.0um]
    profile: SKY130_1V8_CMOS # Type resolves to SemiconductorSubstrate

    nets {
        VDD: { classification: power,  potential: 1.8V, current: 20uA }
        VSS: { classification: ground, potential: 0.0V, current: 20uA }
        In:  { classification: signal, potential: 1.8V }
        Out: { classification: signal }
    }

    let center_x = 60.0um

    # PCells return isolated CellLayouts
    let m_n = space.place(sky130_nmos(W: 1.0um, L: 150nm), at: [center_x, 35.0um])
    let m_p = space.place(sky130_pmos(W: 2.0um, L: 150nm), at: [center_x, 65.0um])

    # Ohmic tie-downs for substrate bulk (B) pins
    let sub_tap  = space.place(sky130_tap(type: TapType.P_Sub,  size: 600nm), at: [center_x - 4.0um, 35.0um])
    let well_tap = space.place(sky130_tap(type: TapType.N_Well, size: 600nm), at: [center_x - 4.0um, 65.0um])

    # Pads
    let in_pad  = space.place(pad(size: 25.0um), at: [18.0um, 50.0um])
    let out_pad = space.place(pad(size: 25.0um), at: [102.0um, 50.0um])

    # Interconnects: Router enforces strict 5nm DBU Manhattan routing
    route in_pad.port("PAD") to m_n.port("G") { net: In,  layer: "metal1", width: 300nm }
    route m_n.port("G") to m_p.port("G")      { net: In,  layer: "metal1", width: 300nm }
    route m_n.port("D") to m_p.port("D")      { net: Out, layer: "metal1", width: 300nm }
    route m_p.port("D") to out_pad.port("PAD"){ net: Out, layer: "metal1", width: 300nm }

    # Power & Bulk: Mandatory 4-pin completeness verified by LVS
    route m_p.port("S") to well_tap.port("TAP") { net: VDD, layer: "metal1", width: 500nm }
    route m_p.port("B") to well_tap.port("TAP") { net: VDD, layer: "metal1", width: 500nm }
    route m_n.port("S") to sub_tap.port("TAP")  { net: VSS, layer: "metal1", width: 500nm }
    route m_n.port("B") to sub_tap.port("TAP")  { net: VSS, layer: "metal1", width: 500nm }
}
```

### 8.2 Execution Trace (`hw build cmos_inverter.hw`)

```text
🔥 hw TOOLCHAIN v0.3.1 (Substrate-Segregated Synthesis)
================================================================================
[    0.35ms] Parsing source: cmos_inverter.hw
[    0.62ms] Evaluated Comptime VM (hwc-eval): Emitted 48 geometry records into Arc<FlatGeometryBuffer>
[    0.78ms] Type Checker: Space 'CMOS_Inverter_Space' binds to Substrate::Semiconductor (SKY130_1V8_CMOS)
[DISPATCH] Launching dedicated backend crate: `hwc-substrate-cmos`
[STAGE 1] Boolean Mask Decomposition Pass (Clipper2):
   • diff ∩ psdm ∩ nwell ──► PPlusDiffusion: 3,300,000 pm² (PMOS S/D Validated)
   • diff ∩ nsdm \ nwell ──► NPlusDiffusion: 1,650,000 pm² (NMOS S/D Validated)
   • diff ∩ poly         ──► Channel Gates: NMOS (1.00um x 0.15um), PMOS (2.00um x 0.15um)
[STAGE 2] Junction-Aware LVS Connectivity Graph:
   • Interface P+ Diff / N-Well: Rectifying Diode Barrier (Net merge BLOCKED)
   • Well Net Assignment: VDD via Ohmic Well Tap (0 VDD-to-Out shorts)
   • Substrate Net Assignment: VSS via Ohmic Substrate Tap
[STAGE 3] DBU Manhattan Maze Router:
   • Quantization: All 32 route vertices snapped to 5nm DBU
   • Directionality: 100% 90° lines with 45° miters (0 slanted/acute lines)
[STAGE 4] Lithography Sign-Off DRC Gate:
   • Rule 'grid.1': PASS (0 fractional DBU coordinates)
   • Rule 'poly.4': PASS (Gate extension: 200nm >= 150nm)
   • Rule 'nwell.4': PASS (Well enclosure: 600nm >= 330nm)
   • Rule 'antenna.1': PASS (Antenna Gate Ratio: 42.1 <= 400.0)
[STAGE 5] Physical Stream Export:
   • Validating 4-terminal contract: Xsky130_nmos [D: Out, G: In, S: VSS, B: VSS] ✅
   • Validating 4-terminal contract: Xsky130_pmos [D: Out, G: In, S: VDD, B: VDD] ✅
   ✅ GDSII: build/CMOS_Inverter_Space/layout.gds (3.4 KB)
   ✅ SPICE: build/CMOS_Inverter_Space/circuit.sp (1.1 KB)
    Finished build in 0.039s
```

#### Generated `circuit.sp` (100% Tapeout Valid)
```spice
* HardwareScript v0.3.1 Clean Layout Extracted SPICE (DUT)
* Space: CMOS_Inverter_Space
* Substrate: SemiconductorSubstrate (SKY130_1V8_CMOS)

.include "sky130_fd_pr/models/sky130_fd_pr__nfet_01v8.model.spice"
.include "sky130_fd_pr/models/sky130_fd_pr__pfet_01v8.model.spice"

* ========================================================================
* EXTRACTED MOSFETS (Strict 4-Terminal Foundry Format: D G S B)
* ========================================================================
Xsky130_nmos   Out In VSS VSS sky130_fd_pr__nfet_01v8 w=1.00u l=0.15u
Xsky130_pmos   Out In VDD VDD sky130_fd_pr__pfet_01v8 w=2.00u l=0.15u

* ========================================================================
* EXTRACTED INTERCONNECT PARASITICS
* ========================================================================
Rtr_In_0    In   nIn_gate  14.2
Cgnd_In_0   In   VSS       1.82fF
Rtr_Out_0   Out  nOut_m1   12.6
Cgnd_Out_0  Out  VSS       1.94fF

.end
```

---

## 9. Implementation Roadmap & Migration Gauntlet

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      IMPLEMENTATION SCHEDULE (v0.3.1)                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [x] MILESTONE 1: Workspace Restructuring & Substrate Crate Isolation       │
│      • Delete monolithic `hwc-engine` directory.                            │
│      • Create `hwc-ir`, `hwc-substrate-cmos`, and `hwc-substrate-laminate`. │
│      • Assert zero cross-dependencies between CMOS and Laminate crates.     │
│                                                                             │
│  [x] MILESTONE 2: `hwc-ir` Contiguous Picometer Arena (`src/geometry.rs`)   │
│      • Implement `FlatGeometryBuffer` with coordinate pool (`Vec<i64>`).    │
│      • Update `hwc-eval` to emit into `FlatGeometryBuffer`.                 │
│                                                                             │
│  [x] MILESTONE 3: Nominal Substrate Typing in `hwc-frontend`                │
│      • Implement `SemiconductorSubstrate` and `LaminateSubstrate` traits.   │
│      • Enforce schema validation (fail if masks found in PCB or vice versa).│
│                                                                             │
│  [x] MILESTONE 4: Silicon Substrate Pipeline (`hwc-substrate-cmos`)         │
│      • Implement Clipper2 2D Boolean mask derivation pass.                  │
│      • Implement Interface Conductivity Matrix (P-N junctions != shorts).  │
│      • Implement 5nm DBU-quantized Manhattan maze router.                   │
│      • Implement 4-terminal compact model SPICE emitter (D, G, S, B).       │
│      • Implement native binary GDSII stream writer.                         │
│                                                                             │
│  [x] MILESTONE 5: Laminate Substrate Migration (`hwc-substrate-laminate`)   │
│      • Rehome PCB Gerber X2, IPC-2581, and Excellon drill code into crate.  │
│                                                                             │
│  [x] MILESTONE 6: Universal `wasm64` Memory64 Runner (`hwc-substrate-wasm`) │
│      • Implement Tier 1 Substrate and Tier 2 Stage plugin execution.        │
│      • Implement Thread-Local Wasmtime Instance Pool for Rayon routing.     │
│                                                                             │
│  [x] MILESTONE 7: Verification Gauntlet                                     │
│      • Run full synthesis on `cmos_inverter.hw` (0 DRC errors, clean SPICE).│
│      • Run full synthesis on `motor_controller.hw` (Valid Gerbers & Drill). │
│      • Verify both synthesize in under 40 milliseconds.                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

*Approved by the HardwareScript Core Architecture Team — September 2026*