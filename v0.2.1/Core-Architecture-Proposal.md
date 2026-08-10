# Architectural Specification: Foundry-Grade Silicon Integration

**Document Type:** Core Architecture Proposal & Implementation Spec  
**Status:** 🚧 **PROPOSED** 
**Focus:** Native GDSII Export, Boolean Mask Systems, Parasitic Exemption, Foundry SPICE Inclusion, and Via Arrays  
**Core Philosophy:** Physical Truth & Foundry Compliance

---

## Executive Summary

As HardwareScript scales from board-level PCB generation to monolithic ASIC and silicon chiplet design, it must bridge the gap between "geometric simulator" and "foundry-ready EDA compiler." 

This specification resolves five fatal architectural flaws that prevent current designs from surviving a commercial semiconductor tape-out (e.g., SkyWater SKY130 or TSMC). By deprecating DXF for silicon, decoupling mask operations from 3D Z-heights, and trusting foundry-provided SPICE models, the compiler transitions to a true production-grade silicon synthesizer.

---

## Section 1: Native GDSII Export Pipeline

### 1.1 The Problem: The DXF File Size Apocalypse
DXF is an ASCII-based 2D format excellent for PCBs and the HardwareScript Monitor (`hsm`). However, for ASICs, DXF is fundamentally unscalable. An SoC with 10 million gates exported to DXF would result in tens of gigabytes of text, choking I/O bounds and causing external converters to crash. Furthermore, DXF flattens hierarchical geometries and lacks native Layer/Datatype tuples required by lithography steppers.

### 1.2 The Solution: Native Binary Stream
When a profile declares `technology: "ASIC"`, the compiler will bypass DXF and natively emit binary **GDSII Stream Format** (or OASIS).

**Architectural Requirements:**
1.  **Binary Writer:** Implement a lightweight, zero-allocation binary writer in `hwc-export/src/gdsii.rs` to stream 16-bit tags and 32-bit coordinates.
2.  **Layer/Datatype Tuples:** The AST `Material` block must be extended to support foundry-specific mapping.
3.  **Hierarchical Instantiation:** Components must be exported as GDSII `STRUCT` definitions, and instances placed using `SREF` (Structure Reference) or `AREF` (Array Reference). This reduces a 50MB flattened DXF to a 2MB GDSII file.

**Syntax Extension (`materials.hw`):**
```hardware
export material Polysilicon:
    category: conductor
    gds_mapping: [layer: 66, datatype: 20]   # Required for ASIC technologies
    properties:
        resistivity: 7e-5ohm_m
```

---

## Section 2: The Boolean Mask System (Zero-Thickness DNA)

### 2.1 The Problem: Additive vs. Chemical Modification
In PCB layout, drawing copper implies the chemical removal of everything else. In silicon fabrication, materials are modified by firing ions (implants) or blocking chemical processes (silicide blocking). 
Previously, masks like `rpm` (Resistor Poly Mask) were treated as physical `insulator` layers with a 10nm thickness. This corrupted the stackup Z-heights and broke vertical via traversal calculations.

### 2.2 The Solution: `category: mask`
We introduce the **Boolean Mask System**. Masks exist purely in the 2D XY plane to instruct the foundry's chemical processes. 

**Architectural Rules:**
1.  Materials defined as `category: mask` possess **zero Z-thickness**.
2.  They do not extrude into the 3D GLB mesh.
3.  They do not affect the `StackupManager`'s Z-height calculations.
4.  They are exclusively exported as 2D boundary polygons to the GDSII file.

**Syntax Extension:**
```hardware
export material Resistor_Poly_Mask:
    category: mask              # Marks this as a zero-thickness fab instruction
    gds_mapping: [layer: 86, datatype: 20]
```

---

## Section 3: Semantic Parasitic Exemption

### 3.1 The Problem: Double-Counting Parasitics
Currently, a 4µm × 1µm strip of Polysilicon is extracted twice: once as the physical interior of the resistor `.subckt`, and again by the Sakurai/Wheeler BEM engine as a generic routing trace. This injects phantom series resistance and stray capacitance into the SPICE netlist, destroying analog simulation accuracy.

### 3.2 The Solution: Interior Masking
The compiler must recognize when a pour belongs *exclusively* to the internal physics of a device.

**Architectural Rules:**
1. When a pour is bound to a multi-terminal device using `device: R1.A, R1.B` across a continuous geometry, the compiler tags the `TraceSegment` as `is_device_interior: true`.
2. The `ParasiticSolver` in `hwc-physics` explicitly ignores any segments flagged as device interiors during R/C extraction.
3. Their physical parameters ($W$, $L$) are still passed to the `spice` subcircuit generator, deferring the physics entirely to the foundry model.

---

## Section 4: True Foundry SPICE Integration

### 4.1 The Problem: The Subcircuit Hubris
HardwareScript currently allows users to write inline math equations to approximate foundry subcircuits (e.g., `value: 350.0ohm * (L / W)`). A fab will never accept this. Commercial `.subckt` files contain hundreds of lines of non-linear voltage coefficients, temperature drift models, and mismatch statistics. 

### 4.2 The Solution: `spice_include` Contract
The compiler must abandon synthesizing standard-cell physics and instead act purely as a Netlister. 

**Syntax Extension:**
```hardware
# resistor_pdk.hw
export subcircuit sky130_fd_pr__res_high_po:
    terminals: [PLUS, MINUS, BULK]
    parameters: [W, L]
    # Replaces inline 'elements:' with an explicit trust of the foundry library
    spice_include: "sky130_fd_pr/models/sky130_fd_pr__res_high_po.model.spice"
```
During SPICE export (`circuit.sp`), the compiler will emit `.include "sky130_fd_pr/models/..."` instead of attempting to rebuild the resistor's internal equations.

---

## Section 5: Automated Via Arrays & Contact Matrices

### 5.1 The Problem: Current Crowding
Transitioning from a 1.0µm-wide power rail to another layer using a single 170nm via causes massive localized heating and fails electromigration (EM) DRC checks. Manually writing `add contact` 15 times to build a via array is unmaintainable.

### 5.2 The Solution: The `matrix` and `fill` Directives
The `ContactPlacement` primitive must be extended to automatically generate arrays of vias that satisfy current density rules.

**Syntax Extension A (Explicit Matrix):**
```hardware
add contact(Tungsten) spanning layer: polyres to li1:
    matrix: [rows: 2, cols: 4, pitch: 300nm]
    at: [x: 5um, y: 5um]
    net: VDD
```

**Syntax Extension B (Auto-Fill by Geometry):**
```hardware
add contact(Tungsten) spanning layer: polyres to li1:
    fill: Contact_A_LI.boundary   # Fills the target bounding box
    spacing: 300nm                # Obeys DRC minimum spacing
    net: VDD
```

**Compiler Lowering:** The `hwc-compiler/src/ir/placement` engine will calculate the bounding box, evaluate the `spacing` against the profile's DRC limits, and automatically unroll this into discrete `$N \times $M` via structures in the `EntityGraph`.

---

## 6. Implementation Roadmap

| Phase | Subsystem | Tasks | Target |
| :--- | :--- | :--- | :--- |
| **Phase 1** | **AST & Parser** | Add `category: mask`, `gds_mapping`, `spice_include`, and `matrix:`/`fill:` parameters to parser. | Week 1 |
| **Phase 2** | **Export Isolation** | Implement native `gds2` streaming binary writer. Replace DXF generation for `technology: "ASIC"`. Map AREF/SREF. | Week 2 |
| **Phase 3** | **Z-Axis & Physics** | Exclude `mask` layers from `StackupManager` Z-calculations. Flag `is_device_interior` and bypass BEM extraction. | Week 3 |
| **Phase 4** | **Routing Engine** | Implement `ViaArray` unrolling in `space_builder.rs` based on boundary fills and DRC spacing. | Week 4 |
| **Phase 5** | **Validation** | Generate full SKY130 inverter, stream to GDSII, and verify clean load in KLayout / Magic VLSI. | Week 5 |

---
**Approved by:** HardwareScript Core Architecture Team  
**Date:** August 10, 2026