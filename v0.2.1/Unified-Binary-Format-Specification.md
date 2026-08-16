# HardwareScript `.hwx` Unified Binary Format Specification & Transpilation Architecture

**Document Type:** Core Format Specification & Ecosystem Integration Architecture  
**Version:** v0.2.1 (Architectural Reference)  
**Status:** Approved for Implementation (Deferred until Engine Stabilization)  
**Focus:** The `.hwx` (HardwareScript eXchange) Binary Standard, Zero-Copy Ingestion, and Multi-Target Transpilation  

---

## 1. Executive Summary & Philosophy

HardwareScript's physical synthesis pipeline generates complete geometric, topological, material, and parasitic truth in pure $i64$ picometer coordinates. 

To bridge the compiler core (`hwc`), visual runtime environments (HardwareScript Monitor / HardwareScript Studio), and industrial manufacturing factories without bloating the compiler core, we define the **`.hwx` (HardwareScript eXchange)** binary format.

```text
 ┌─────────────────────────────────────────────────────────────┐
 │                HardwareScript Source (.hw)                  │
 └──────────────────────────────┬──────────────────────────────┘
                                │
                                ▼ [hwc build]
 ┌─────────────────────────────────────────────────────────────┐
 │                The Unified .hwx Binary File                 │
 │           "The LLVM Bitcode of Physical Hardware"           │
 │  - Zero-Copy rkyv memory-mappable payload                   │
 │  - 100% loss-free representation of IC and PCB designs      │
 │  - Contains stackup, welded geometry, netlist, & parasitics │
 └──────────────────────────────┬──────────────────────────────┘
                                │
               [Zero-Copy mmap Load (< 1ms)]
                                │
                                ▼
 ┌─────────────────────────────────────────────────────────────┐
 │          HardwareScript Monitor / Studio (HSM / HSS)        │
 │  - Real-time 120 FPS Viewport (3D Babylon.js / 2D PixiJS)   │
 │  - Embedded WASM SPICE Simulation & Interactive Probing     │
 ├─────────────────────────────────────────────────────────────┤
 │               MODULAR TRANSPILATION BACKENDS                │
 │  ├── ASIC / Silicon Fab:   GDSII, OASIS, LEF/DEF            │
 │  ├── PCB Manufacturing:    Gerber (RS-274X/X2) + Drills,    │
 │  │                         IPC-2581, ODB++                  │
 │  ├── 3D / Mechanical CAD:  STEP, glTF/GLB, 2D DXF           │
 │  └── Third-Party EDA:      KiCad S-Expressions, Magic,      │
 │                            KLayout                          │
 └─────────────────────────────────────────────────────────────┘
```

---

## 2. Phased Rollout Strategy

To maintain maximum engineering velocity and ensure verification accuracy, the format rollout is split into two distinct phases:

### Phase 1: Current Proving Phase (Active)
During active language development and semiconductor verification, the compiler emits human-readable and standard third-party formats for immediate debugging:
* **`board.dxf`**: 2D cross-section and mask alignment inspection in AutoCAD, LibreCAD, and KLayout.
* **`board.glb`**: 3D manifold mesh inspection in Blender and Babylon.js.
* **`circuit.sp` / `ac.sp` / `dc.sp` / `tran.sp`**: Extracted parasitic netlists simulated in LTSpice and ngspice.
* **`bom.csv`**: Material volume and discrete component verification.
* **`*.routes.lock`**: Deterministic route cache for instant rebuilds.

### Phase 2: Production Unified `.hwx` Phase (Deferred)
Once the compiler core reaches absolute physical stability across IC and PCB test suites:
* `hwc build` emits **only `design.hwx`** (and the `.lock` file) in $<20\text{ms}$.
* The standalone Monitor/Studio loads the `.hwx` file instantly via memory-mapping (`memmap2`).
* All conversions to factory manufacturing standards (GDSII, Gerber ZIP, STEP, IPC-2581) are handled **on demand** by the Studio UI or headless CLI plugins.

---

## 3. The `.hwx` Binary Architecture (Single Source of Truth)

The `.hwx` file is structured as a contiguous, 16-byte aligned binary payload managed by `rkyv`. It uses **zero pointers and zero heap allocations on load**, casting raw bytes directly into typed Rust structs.

```text
┌──────────────────────────────────────────────────────────────────────────┐
│                         .hwx BINARY FILE LAYOUT                          │
├──────────────────────────────────────────────────────────────────────────┤
│ 1. MAGIC & HEADER (64 Bytes)                                             │
│    • Magic: [0x48, 0x57, 0x58, 0x00] ("HWX\0")                          │
│    • Technology Flag: ASIC (0x01) | PCB (0x02) | AdvancedPackaging (0x03)│
│    • Total Bounds: BoundingBox { min: (X,Y,Z), max: (X,Y,Z) } in pm      │
│    • Fingerprint / Build Hash: u128 Blake3 semantic checksum             │
├──────────────────────────────────────────────────────────────────────────┤
│ 2. STACKUP TABLE CHUNK                                                   │
│    • Ordered array of layer records (Layer ID, Name, Z-start, Z-end)     │
│    • Material attributes (Resistivity, Permittivity, Thermal K, Color)   │
│    • Layer type flags (Routable, Dielectric, 0nm Mask, Substrate)        │
├──────────────────────────────────────────────────────────────────────────┤
│ 3. 2D/2.5D GEOMETRY POOL (GPU Vertex-Ready)                              │
│    • Contiguous array of Clipper2-welded closed polygon contours         │
│    • Point arrays stored as raw integer picometers (i64, i64)            │
│    • Tags: NetId, LayerId, MaterialId, PourID (for interactive probing)  │
├──────────────────────────────────────────────────────────────────────────┤
│ 4. VERTICAL INTERCONNECT & VIA CHUNK                                     │
│    • Via coordinate list: (X, Y) center in picometers                    │
│    • Span: (from_layer_idx, to_layer_idx)                                │
│    • Drill diameter, pad geometry, and extracted contact resistance (Ω)  │
├──────────────────────────────────────────────────────────────────────────┤
│ 5. NETLIST & PARASITIC CHUNK                                             │
│    • Logical net declarations (Voltage budget, Peak/RMS current)         │
│    • Physical Device Instances (W, L, terminal-to-net map)               │
│    • Extracted RLC parasitics (Trace R, Ground C, Coupling C12, Via L)   │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Recommended Transpilation Targets & Export Ecosystem

Because `.hwx` retains complete geometric, topological, and electrical information, the HardwareScript Studio / CLI can transpile to any industry format without loss of fidelity.

```
                               .hwx Core File
                                      │
     ┌──────────────────┬─────────────┼─────────────┬──────────────────┐
     ▼                  ▼             ▼             ▼                  ▼
[ ASIC Fab ]       [ PCB Fab ]   [ 3D / CAD ]  [ Simulation ]    [ Open EDA ]
• GDSII Stream     • Gerber X2   • STEP AP242  • SPICE3 Netlist  • KiCad 8+
• OASIS (SEMI P39) • Excellon DRL• glTF / GLB  • Touchstone S2P  • KLayout
• LEF / DEF        • IPC-2581    • 2D DXF      • IBIS Models     • Magic VLSI
                   • ODB++ (.tgz)
```

### 4.1 Semiconductor & Silicon Tape-Out Targets
1. **GDSII Stream Format (`.gds`):**
   * *Target:* Primary tape-out format for commercial fabs (SkyWater, TSMC, GlobalFoundries).
   * *Transpilation Logic:* Maps each conductive and mask layer in `.hwx` to `[layer: X, datatype: Y]` tuples. Writes boundary polygons and instanced cell references (`SREF`/`AREF`).
2. **OASIS (`.oas` / SEMI P39):**
   * *Target:* High-density nanoscale nodes (sub-28nm).
   * *Transpilation Logic:* 10x–50x smaller than GDSII using variable-length integer byte streams. Essential for designs with millions of standard cells.
3. **LEF / DEF (Library/Design Exchange Format):**
   * *Target:* Standard-cell ASIC place-and-route integration (OpenROAD, Cadence Innovus, Synopsys IC Compiler).
   * *Transpilation Logic:* Extracts pin obstruction geometry (`.lef`) and component instance placements (`.def`).

---

### 4.2 PCB Fabrication & Assembly Targets
1. **Gerber RS-274X / Gerber X2 + Excellon NC Drill (`.zip`):**
   * *Target:* Universal standard for all board manufacturers (JLCPCB, PCBWay, Eurocircuits, OSH Park).
   * *Transpilation Logic:* Slices `.hwx` stackup layers into discrete `.gbr` files with aperture macros, and emits `.drl` tool drill coordinates.
2. **IPC-2581 (IPC-DPMX, Single XML):**
   * *Target:* Enterprise tier-1 automated assembly (Aerospace, Defense, Automotive).
   * *Transpilation Logic:* Serializes copper geometry, stackup dielectric thicknesses, BOM, and pick-and-place centroid data into a single verified XML schema.
3. **ODB++ (`.tgz` Container):**
   * *Target:* High-volume EMS assembly facilities utilizing Siemens/Valor CAM systems.
   * *Transpilation Logic:* Outputs structured directory archives containing matrix tables for copper, drill, netlists, and components.

---

### 4.3 3D Mechanical & CAD Interoperability
1. **STEP (ISO 10303-214 / AP242 - `.step` / `.stp`):**
   * *Target:* Mechanical enclosure design in SolidWorks, Autodesk Fusion 360, FreeCAD, and PTC Creo.
   * *Transpilation Logic:* Extrudes `.hwx` 2D copper and dielectric boundaries along their $Z$-elevations into true B-Rep (Boundary Representation) solid bodies.
2. **glTF / GLB (`.glb`):**
   * *Target:* Ultra-fast, hardware-accelerated 3D rendering in WebGL, Babylon.js, Three.js, and the Studio HUD.
   * *Transpilation Logic:* Uses `earcut` on pre-welded 2D contours to generate triangulated, normal-mapped surface meshes.
3. **DXF (AutoCAD R2018 - `.dxf`):**
   * *Target:* 2D mechanical validation, optical comparator inspection, laser cutting, and CNC milling.

---

### 4.4 Simulation & Modeling Targets
1. **Annotated SPICE Netlist (`.sp` / `.cir`):**
   * *Target:* Analog/RF simulation in ngspice, Xyce, LTSpice, Spectre, and HSPICE.
   * *Transpilation Logic:* Emits device subcircuit calls (`XR1`, `MM1`) with embedded $R_{\text{trace}}$, $C_{\text{ground}}$, $C_{\text{coupling}}$, and $R_{\text{via}}$ lumped elements.
2. **Touchstone S-Parameters (`.s2p`, `.s4p`):**
   * *Target:* RF & High-Speed channel analysis (Keysight ADS, Ansys HFSS).
   * *Transpilation Logic:* Formulates extracted RLCG transmission line matrices into frequency-dependent S-parameter scattering matrices.

---

### 4.5 Open-Source EDA Cross-Compatibility
1. **KiCad 8+ S-Expressions (`.kicad_pcb`):**
   * *Target:* Open-source PCB ecosystem interoperability.
   * *Transpilation Logic:* Transpiles `.hwx` tracks, pads, vias, and zones into native KiCad Lisp S-expression blocks.
2. **KLayout / Magic Python Scripts:**
   * *Target:* Open-source silicon verification and mask visualization.

---

## 5. Summary Workflow: From Code to Silicon/Board

```text
[ Developer writes .hw code ]
             │
             ▼
      hwc build main.hw
             │
             ├─► Emits: "build/project.hwx" (Single unified binary in <20ms)
             └─► Emits: "build/project.lock" (Incremental cache)
             │
             ▼
[ Open in HardwareScript Studio / Monitor ]
             │
             ├─► Memory-maps .hwx with zero-copy (120 FPS 3D/2D Viewport)
             ├─► Runs ngspice.wasm on extracted netlist
             │
             ▼
[ Click "Export for Fabrication" in Studio / Run CLI Exporter ]
             │
             ├── If profile is ASIC ──► Emits GDSII Stream (.gds)
             └── If profile is PCB  ──► Emits Gerber + Drill ZIP (.zip)
```

This ensures the compiler core remains lean, while the `.hwx` binary serves as the permanent, high-performance foundation for all visualization, verification, and manufacturing tools.