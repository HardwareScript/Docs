# HardwareScript Middle-End Architecture & Declarative Intent Specification (v0.2.0-Alpha)

**Document Type:** Core Compiler Architecture & Middle-End Synthesis Reference  
**Status:** Approved for Implementation (v0.2.0-Alpha)  
**Focus:** Declarative Intent Paradigm, Relational Placement Lowering, Region Floorplanning, and CIR Intent Synthesis

---

## Executive Summary

HardwareScript is a **pure declarative Hardware Description Language (HDL)**. It is not an imperative scripting language, nor does it expose procedural compiler APIs to script-land [11-ZERO-MAGIC-COMPILER.md]. 

The middle-end architecture enforces a fundamental separation of concerns:
1. **The Compiler Core (`hwc-engine` & `hwc-compiler`)** exclusively owns all synthesis algorithms, numerical solvers, pathfinding routines, Boolean polygon engines, and physical verification checks [Core-System-Architecture.md].
2. **HardwareScript Source (`.hw`)** exclusively expresses physical intent, relational placement rules, floorplanning regions, routing policies, and domain knowledge [10-WIRING-INTEGRATION-GAPS.md].
3. **The Standard Library (`@std`)** functions strictly as a **data warehouse** of physical constants, unit definitions, device contracts, standard footprints, and PDK profiles—it contains zero procedural execution hooks [HPM-ARCHITECTURE.md].

This specification defines the remaining middle-end synthesis pipeline required to lower high-level declarative code into exact, verified 3D physical geometry [01-DATABASE-SPATIAL-FOUNDATION.md].

---

## Section 1: The Core Architecture Boundary

```
  ┌─────────────────────────────────────────────────────────────┐
  │                 HARDWARESCRIPT DECLARATIONS                 │
  │  - Physical Intent (Materials, PCells, Device Contracts)    │
  │  - Placement Intent (Relational Constraints, Regions)       │
  │  - Routing Intent (Signal Classes, Performance Targets)     │
  │  - PDK Profiles (Stackup Rules, Bridges, Cost Weights)      │
  └──────────────────────────────┬──────────────────────────────┘
                                 │
           Pure Immutable Declarative Data (Salsa Inputs)
                                 │
                                 ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                 RUST COMPILER MIDDLE-END                    │
  │  1. Relational Constraint Resolver (FixedTransform2D)       │
  │  2. Bottom-Up Region Floorplanner                           │
  │  3. Connection Interface Routing (CIR) Intent Synthesizer   │
  │  4. Topological Line-Search Router & Port Selector          │
  │  5. Refinement Pipeline (Legalizer, Compactor, Miter)      │
  │  6. Clipper2 Copper Welder & Earcut Triangulator            │
  └─────────────────────────────────────────────────────────────┘
```

### System Responsibility Matrix

| Subsystem | Location | Responsibility | Execution Model |
| :--- | :--- | :--- | :--- |
| **Grammar & AST** | `hwc-parser` | Parses declarative syntax, enforces punctuation laws (`:` vs `=`). | Deterministic AST construction. |
| **Data Warehouse** | `stdlib/` | Stores SI units, physical constants, device contracts, and PDK rules. | Pure declarative data files. |
| **Dependency DAG** | `hwc-compiler` | Manages incremental compilation and query invalidation. | Salsa-driven demand query graph [06-CACHING-INCREMENTAL.md]. |
| **Relational Solver** | `hwc-compiler` | Resolves relative anchoring (`right_of`, `align`) to absolute coordinates. | 128-bit fixed-point integer math [01-DATABASE-SPATIAL-FOUNDATION.md]. |
| **Floorplanner** | `hwc-engine` | Partitions design into logical zones (`region`) and bounds G-cells. | Convex QP solver / `clarabel` [03-LEGALIZATION-COMPACTION.md]. |
| **CIR Synthesizer** | `hwc-engine` | Maps `intent` declarations to physical widths, stubs, and cost weights. | Rule-driven parameter derivation [Connection-Interface-Routing.md]. |
| **Pathfinder** | `hwc-engine` | Calculates 2.5D/3D trace paths around obstacles using ray-casting. | Axis-Aligned Slab Method over `geo-index` [Unified-2.5D-3D-Routing-and-Placement.md]. |
| **Copper Welder** | `hwc-export` | Fuses coplanar same-net polygons into clean 2D manifold contours. | Clipper2 Non-Zero Winding Union [PHYSICAL-SYNTHESIS-MIDDLE-END-SPEC.md]. |

---

## Section 2: The Four Declarative Domains

All HardwareScript source declarations belong to one of four structured domains.

### 2.1 Domain 1: Physical Intent
Declares the atomic reality of the hardware—materials, device contracts, and parameterized cell (PCell) shapes [transistors.hw, SHAPE-SYSTEM-ARCHITECTURE.md].

```hardware
# Physical Intent Declaration
material Aluminum:
    category: conductor
    properties:
        resistivity: 2.82e-8ohm_m
        max_current_density: 20.0A_mm2

shape Pad(w: Measurement, h: Measurement):
    geometry:
        Rectangle(width: w, height: h)
```

### 2.2 Domain 2: Placement Intent
Declares relational spatial constraints and floorplanning partitions without hardcoding absolute coordinates [10-WIRING-INTEGRATION-GAPS.md].

```hardware
# Placement Intent Declaration
region Analog
region Digital:
    right_of: Analog with spacing: pdk.min_spacing * 10
    align: top with Analog

add plane(Aluminum) named Pad_A inside: Analog:
    shape: Pad(w: 100um, h: 100um)
    at: Analog.center

add plane(Aluminum) named Pad_B inside: Digital:
    shape: Pad(w: 100um, h: 100um)
    right_of: Pad_A with spacing: pdk.min_spacing * 4
    align: center_y with Pad_A
```

### 2.3 Domain 3: Routing Intent
Declares connection requirements, signal classes, and optional directional overrides [Connection-Interface-Routing.md, VIA-AND-PAD-PORT-ESCAPE-SPECIFICATION.md].

```hardware
# Routing Intent Declaration
route Pad_A to Pad_B:
    intent: Clock      # Maps to Clock cost-weights in PDK profile
    prefer exit: East  # Optional routing constraint (Soft Constraint)
```

### 2.4 Domain 4: Compiler Context & PDK Profile
Defines the active physical space boundaries, snapping grid resolution, stackup profile, and manufacturing rules [foundry_pdk.hw].

```hardware
# Compiler Context Declaration
space Mixed_Signal_SoC implements Schematic_Netlist:
    dimensions: 2mm by 2mm by 136um
    resolution: 100nm
    origin: bl by b
    profile: Silicon_180nm
```

---

## Section 3: Middle-End Technical Gap & Execution Tasks

To complete the v0.2.0 Middle-End pipeline, the compiler must execute five core lowering tasks.

```
                   The Five Middle-End Lowering Tasks
                   
  1. Relational Lowering  ──►  2. Region Partitioning  ──►  3. CIR Intent Mapping
  (Evaluates anchors)           (Bottom-up BBox bounds)       (Derives width/stubs)
                                                                       │
  5. Refinement & Weld    ◄──  4. Topological Routing   ◄──────────────┘
  (Clipper2 + earcut)           (Obstacle-aware raycast)
```

### Task 1: Relational Constraint Lowering (`FixedTransform2D`)
*   **Input:** Relational placement expressions (e.g., `right_of: Pad_A with spacing: pdk.min_spacing * 4`).
*   **Execution:** 
    1. Query `Pad_A`'s bounding box from the `EntityGraph` [01-DATABASE-SPATIAL-FOUNDATION.md].
    2. Retrieve `min_spacing` from the active PDK profile.
    3. Perform 128-bit fixed-point arithmetic scaled by $10^9$ to evaluate the offset [01-DATABASE-SPATIAL-FOUNDATION.md]:
       $$X_{\text{Pad\_B}} = X_{\text{Pad\_A.max}} + (4 \times \text{PDK}_{\text{min\_spacing}})$$
    4. Register resolved picometer coordinates directly into the `rstar` dynamic spatial index [01-DATABASE-SPATIAL-FOUNDATION.md].

### Task 2: Region Floorplanning & Dynamic Bounding Box Solvers
*   **Input:** Bounded or un-bounded `region` declarations with relational constraints [Core-System-Architecture.md].
*   **Execution:**
    1. **Bottom-Up Child Aggregation:** For regions lacking an explicit `boundary:`, calculate the collective bounding box of all components assigned to that region via `inside: RegionName`.
    2. **Container Placement:** Treat the aggregated bounding box as a dynamic container and evaluate inter-region constraints (e.g., `Digital right_of Analog`).
    3. **G-Cell Partitioning:** Divide each resolved region into coarse G-cells and initialize parallel worker threads for local detailed routing [ROUTING-ENGINE-MODERNIZATION.md].

### Task 3: CIR Intent-to-Cost-Weight Synthesis
*   **Input:** `route` statements containing an `intent:` property and the net's declared electrical parameters (`current`, `voltage`).
*   **Execution:**
    1. Query the net's declared peak current ($I_{\text{peak}}$) from the `nets` block [Syntax-&-Definition.md].
    2. Query the material's `max_current_density` from `MaterialRegistry` [11-ZERO-MAGIC-COMPILER.md].
    3. Synthesize the minimum trace cross-sectional area:
       $$A_{\text{min}} = \frac{I_{\text{peak}}}{J_{\text{limit}}}$$
    4. Calculate physical trace width: $W_{\text{trace}} = \max(W_{\text{PDK\_min}}, A_{\text{min}} / T_{\text{layer\_thickness}})$.
    5. Retrieve `escape_stub` length and cost penalties (`via_penalty`, `direction_penalty`) from the PDK's `net_type` configuration matching the declared `intent` [foundry_pdk.hw].
    6. Construct an immutable `CostComposer` instance for the pathfinder [Connection-Interface-Routing.md].

### Task 4: Obstacle-Aware Port Selection with Overrides
*   **Input:** Start/goal interfaces and optional exit constraints (`prefer exit: East`, `require exit: West`).
*   **Execution:**
    1. Cast rays outward from all cardinal faces of the source and target component boundaries [Obstacle-Aware-Port-Selection.md].
    2. Compute clearance distance ($C$) and geometric alignment toward goal ($A$).
    3. Calculate score for each candidate port:
       $$\text{Score} = (A \times 0.3) + (\min(1.0, C / 1\text{mm}) \times 0.7)$$
    4. **Constraint Enforcement:** If `prefer exit: Direction` is set, apply a $+0.5$ bonus to the specified port score. If `require exit: Direction` is set, force selection of that port regardless of score; if blocked, trigger a build-halt error [13-PHYSICAL-SYNTHESIS-GUARDRAILS.md].

### Task 5: Refinement Pipeline & Export Boundary Isolation
*   **Input:** Raw orthogonal coordinate paths from the topological line-search router [ROUTING-ENGINE-MODERNIZATION.md].
*   **Execution:**
    1. **Legalization:** Apply QP/DAG localized solvers to nudge colliding parallel trace vectors apart up to clearance limits [03-LEGALIZATION-COMPACTION.md].
    2. **45° Miter Pass:** Scan for 90° corners. Insert diagonal chamfers at a distance of $1.5 \times W_{\text{trace}}$. If the straight segment length between adjacent corners is less than $3.0 \times W_{\text{trace}}$, reduce chamfer distance proportionally to prevent miter collision spikes [PHYSICAL-SYNTHESIS-MIDDLE-END-SPEC.md].
    3. **Clipper2 Copper Weld:** Execute 2D Boolean Union under the Non-Zero Winding Rule on all same-net, same-layer copper shapes strictly at the export boundary [2D-POLYGON-UNIONING-IMPLEMENTATION.md].
    4. **Tessellation:** Pass clean, unioned 2D contours to `earcut` for cap triangulation during 3D mesh (GLB) generation [PHYSICAL-SYNTHESIS-MIDDLE-END-SPEC.md].

---

## Section 4: Canonical Design Code Example

This complete, production-ready example demonstrates pure **Declarative Intent and Policy** in HardwareScript, completely free of low-level coordinate leaks and procedural API calls [10-WIRING-INTEGRATION-GAPS.md]:

```hardware
# Canonical Middle-Level Test Case (v0.2.0-Alpha)
# Demonstrates pure declarative intent, relational placement, and CIR intent synthesis

import * from @std/primitives/units
import * from @std/primitives/math
import * from ../materials
import Silicon_180nm from ../foundry_pdk

module Mixed_Signal_System:
    pins: [
        input Analog_IN,
        output Digital_OUT,
        power VDD,
        ground GND
    ]
    route Analog_IN to Digital_OUT

shape CubicPad(w: Measurement, h: Measurement):
    geometry:
        Rectangle(width: w, height: h)

space System_Layout implements Mixed_Signal_System:
    dimensions: 2mm by 2mm by 136um
    resolution: 100nm
    origin: bl by b
    profile: Silicon_180nm

    # Net electrical declarations — Mandatory under Zero Magic rules
    nets:
        Analog_IN:   { classification: signal, potential: 1.8V, current: 0.1uA }
        Digital_OUT: { classification: signal, potential: 1.8V, current: 0.1uA }
        VDD:         { classification: power,  potential: 1.8V, current: 10mA }
        GND:         { classification: ground, potential: 0V,   current: 10mA }

    # ========================================================================
    # 1. FLOORPLANNING REGIONS (Declarative Placement Intent)
    #    The compiler partitions placement and parallelizes DRC per region.
    # ========================================================================
    region AnalogRegion:
        at: space.bottom_left + [100um, 100um]

    region DigitalRegion:
        right_of: AnalogRegion with spacing: pdk.min_spacing * 15
        align: top with AnalogRegion

    # ========================================================================
    # 2. RELATIONAL COMPONENT PLACEMENT (No absolute coordinates)
    # ========================================================================
    add plane(Aluminum) named Pad_Analog_In inside: AnalogRegion:
        shape: CubicPad(w: 120um, h: 120um) # Intrinsic footprint dimension
        at: AnalogRegion.center
        net: Analog_IN

    add plane(Aluminum) named Pad_Digital_Out inside: DigitalRegion:
        shape: CubicPad(w: 120um, h: 120um)
        at: DigitalRegion.center
        net: Digital_OUT

    # ========================================================================
    # 3. ROUTING INTENT (CIR Synthesis)
    #    Widths, stubs, and cost weights derived automatically from profile.
    # ========================================================================
    route Pad_Analog_In to Pad_Digital_Out:
        intent: Signal
        prefer exit: East  # Soft directional constraint
```

---

## Section 5: The Unidirectional Compilation Pipeline

The compiler lowers source declarations through a strictly unidirectional pipeline with zero feedback loops [ROUTING-ENGINE-MODERNIZATION.md]:

```
1. AST Parsing & Validation (hwc-parser)
   ├── Validate grammar and punctuation rules (: for facts, = for logic/actions)
   └── Construct immutable AST data structures
       │
       ▼
2. Semantic Lowering & Symbol Resolution (hwc-compiler)
   ├── Resolve PDK profile, stackup layers, and material physical properties
   └── Register physical device contracts and PCell shape definitions in SymbolTable
       │
       ▼
3. Relational Placement & Region Floorplanning (hwc-engine)
   ├── Bottom-Up Aggregation: Calculate dynamic bounding boxes for un-bounded regions
   ├── Relational Solver: Lower relative statements (`right_of`, `align`) via FixedTransform2D
   └── Register resolved picometer coordinates into DynamicSpatialIndex (rstar)
       │
       ▼
4. Connection Interface Routing (CIR) Synthesis (hwc-engine)
   ├── Query net current limits and lookup PDK `net_type` cost weights
   ├── Calculate physical trace widths and perpendicular escape stubs
   └── Select optimal exit/entry ports via ray-casting (respecting prefer/require hints)
       │
       ▼
5. Topological Pathfinding & Refinement (hwc-engine)
   ├── Execute continuous ray-casting pathfinder within soft G-cell corridors
   ├── Run Localized Legalizer (QP/DAG) to resolve clearance nudges
   └── Apply 45° Miter Pass (with short-segment overlap protection)
       │
       ▼
6. Physical Verification & Export Boundary (hwc-export)
   ├── Verify LVS-by-construction, PIVB connectivity, and G-Cell sweep-line DRC
   ├── Execute Clipper2 2D Boolean Union on coplanar same-net copper shapes
   └── Triangulate clean manifold contours via earcut for GLB/DXF/GDSII export
```

This specification completes the architectural blueprint for the HardwareScript Middle-End. By strictly keeping algorithms inside Rust and intent inside HardwareScript declarations, the system delivers sub-second compilation, physical correctness, and complete design portability.