# HardwareScript Spatial Synthesis Abstraction Specification (v0.1.9)

**Document Type:** Core Language Abstraction & Compiler Integration Reference  
**Status:** Approved for Implementation (v0.1.9)  
**Focus:** Tiered Spatial Abstractions, Relational Constraints, Parameterized Cells, and Bare-Metal Escape Hatches

---

## Executive Summary

To ensure the HardwareScript compilation pipeline is built upon an uncompromised physical foundation, all development up to this point has focused strictly on the **low-level "Assembly" tier** [ROUTING-AND-MANUFACTURING-ARCHITECTURE.md]. By forcing the early parser, compiler, and database to operate on absolute picometer coordinates, we have established a robust, mathematically sound backend:
*   **The Entity Graph** serves as the canonical source of truth, avoiding the pointer invalidation and index-shifting problems of legacy tools [01-DATABASE-SPATIAL-FOUNDATION.md].
*   **The Hybrid Spatial Index** (`rstar` + `geo-index`) delivers $O(\log N)$ spatial query performance [01-DATABASE-SPATIAL-FOUNDATION.md].
*   **The 2D Clipper2 Union Engine** ("Copper Welder") successfully dissolves overlapping geometric boundaries to prevent visual Z-fighting and mesh intersections [PHYSICAL-SYNTHESIS-MIDDLE-END-SPEC.md].
*   **The Planar Island & Via Bridge (PIVB) Solver** deterministically validates physical connectivity on continuous coordinate planes [PIVB-Solver.md].

With this low-level core securely in place, we can now introduce **high-level spatial abstractions** [10-WIRING-INTEGRATION-GAPS.md]. These abstractions free both human developers and code-generation models (LLMs) from the burden of manual coordinate arithmetic [12-ERROR-SYSTEM-OVERHAUL.md]. This document defines the syntax, keywords, compiler lowering mechanisms, and integration paths for:
1.  **The Middle-Level (Relational & Parameterized) Syntax**
2.  **The High-Level (Declarative & Constraint-Driven) Syntax**
3.  **Direct Coordinate Placement (No Wrapper Needed)**

---

# Section 1: Middle-Level (Relational & Parameterized) Synthesis

The middle-level syntax abstracts raw polygon drawing and absolute positioning into **relational constraints** and **parameterized cells (PCells)** [BIT-BLIT-UNROLLER-IMPLEMENTATION.md, 10-WIRING-INTEGRATION-GAPS.md]. Instead of calculating coordinate coordinates by hand, the designer describes how components and pours are physically sized and geometrically anchored relative to each other [10-WIRING-INTEGRATION-GAPS.md].

```
                             C-Level Spatial Relationship
                             
   [ Pad_A ]  ───────►  [ Pad_A.right + 2.0um ]  ───────►  [ Pad_B ]
                          (Relational Anchor)
```

### 1.1 What It Replaces

| Legacy Assembly-Level Syntax (Leaving) [Syntax-&-Definition.md] | New Middle-Level Syntax (Entering) [Syntax-&-Definition.md] |
| :--- | :--- |
| Hardcoded, absolute boundary coordinates for every pour (e.g., `boundary: [500nm, 500nm] to [1500nm, 1500nm]`) | Parameterized `shape` definitions (PCells) that scale dynamically based on input arguments. |
| Absolute coordinate placement for every component (e.g., `at [x: 3500nm, y: 500nm]`) | Relational alignment constraints (`align:`, `with:`) and relative edge offsets (`at: last.right + 1.0um`). |
| Hardcoded waypoint routing paths (e.g., `path: [[1000nm, 1000nm], [4000nm, 1000nm]]`) | Port-aware logical routing (`exit:`, `enter:`) where the pathfinder calculates mitered 3D paths [Unified-2.5D-3D-Routing-and-Placement.md]. |

---

### 1.2 Syntax and Keywords

*   **`shape`**: Declares a parameterized geometric template (PCell) in the symbol table [SHAPE-SYSTEM-ARCHITECTURE.md].
*   **`align`**: Specifies a co-planar axis constraint (`center_x`, `center_y`, `center_z`, `top`, `bottom`, `left`, `right`).
*   **`with`**: Binds the alignment constraint to a target component instance or anchor [13-PHYSICAL-SYNTHESIS-GUARDRAILS.md].
*   **`above` / `below` / `right_of` / `left_of`**: Relational directional prepositions for simple layout spacing.
*   **`last`**: Refers to the previously placed component in the space block, enabling loop-based array offsets [11-ZERO-MAGIC-COMPILER.md].

---

### 1.3 Concrete Code Example

This script places two $1.0\mu\text{m} \times 1.0\mu\text{m}$ metal pads separated by a $2.0\mu\text{m}$ gap and routes a $250\text{nm}$ trace between them on `metal1`.

```hardware
# C-Level: Parameterized cells, relative constraints, and port docking
import Silicon_250nm from foundry_pdk
import Aluminum from materials

# 1. Parameterized Cell (PCell) Definition
shape Pad(w: Measurement, h: Measurement):
    pins: [port]
    geometry:
        # Subtract the hole from the base rectangle natively using CSG
        Rectangle(w, h) - Circle(500nm) at center
        
        # Expose the right boundary face as a logical docking portal
        port at [w, h / 2]

space Pad_Connect:
    resolution: 1nm
    profile: Silicon_250nm

    # 2. Instantiate Pad_A at an absolute reference point
    add plane(Aluminum) named Pad_A:
        shape: Pad(1.0um, 1.0um)
        on layer: metal1
        at: [500nm, 500nm]

    # 3. Place Pad_B relative to Pad_A without manual math
    add plane(Aluminum) named Pad_B:
        shape: Pad(1.0um, 1.0um)
        on layer: metal1
        align: center_y with Pad_A
        at: Pad_A.right + 2.0um

    # 4. Port-aware topological routing (Pathfinder calculates the waypoints)
    route Pad_A.port to Pad_B.port:
        width: 250nm
        exit: East      # Terminates at the exact East boundary of Pad_A
        enter: West     # Terminates at the exact West boundary of Pad_B
```

---

### 1.4 Compiler Lowering and Execution

The compiler processes the middle-level relative constraints through a deterministic three-stage pipeline [01-DATABASE-SPATIAL-FOUNDATION.md]:

```
   1. AST Parsing  ──►  2. Relational Resolving  ──►  3. 2.5D Pathfinding
  (Prose expression)      (FixedTransform2D)            (Boundary Docking)
```

1.  **Relational Resolving:** 
    During the semantic lowering phase, the compiler's constraint manager evaluates the relative layout instructions [10-WIRING-INTEGRATION-GAPS.md]. When resolving `Pad_A.right + 2.0um`, it queries `Pad_A`'s bounding box from the `EntityGraph` [01-DATABASE-SPATIAL-FOUNDATION.md].
    The `FixedTransform2D` engine promotes the calculations to 128-bit integer arithmetic to prevent overflow, multiplying the coordinates by a scaling factor of $10^9$ to yield absolute picometer values [01-DATABASE-SPATIAL-FOUNDATION.md]:
    $$\text{Pad\_A.right} = 1{,}500{,}000\text{ nm} \quad \Rightarrow \quad 1{,}500{,}000\text{ nm} + 2{,}000\text{ nm} = 3{,}500{,}000\text{ nm}$$
    `Pad_B`'s global origin is locked to $X = 3{,}500{,}000\text{ pm}$ and registered in the R\*-tree [01-DATABASE-SPATIAL-FOUNDATION.md].

2.  **Slab-Method Boundary Docking:**
    The `TopologicalRouter` executes the pathfinding between the two logical ports [02-ROUTING-PIPELINE.md]. Instead of targeting the pins' centers (which would penetrate the component and trigger DRC errors) [13-PHYSICAL-SYNTHESIS-GUARDRAILS.md], the router projects rays outwards from the bounding box's co-planar outer faces [VIA-AND-PAD-PORT-ESCAPE-SPECIFICATION.md]:
    *   `exit: East` locks the start node to the right edge of `Pad_A` ($X = 1{,}500\text{ nm}$, $Y = 1{,}000\text{ nm}$) [VIA-AND-PAD-PORT-ESCAPE-SPECIFICATION.md].
    *   `enter: West` locks the end node to the left edge of `Pad_B` ($X = 3{,}500\text{ nm}$, $Y = 1{,}000\text{ nm}$) [VIA-AND-PAD-PORT-ESCAPE-SPECIFICATION.md].
    The Axis-Aligned Slab Method evaluates ray-AABB intersections over the static `geo-index` in $O(\log N)$ time, generating a straight, orthogonal trace segment [02-ROUTING-PIPELINE.md, Unified-2.5D-3D-Routing-and-Placement.md].

3.  **Clipper2 Weld:**
    At the final compilation boundary, the 2D Boolean Union engine merges the generated trace and the pads on the `metal1` layer into a single, clean manifold mesh [PHYSICAL-SYNTHESIS-MIDDLE-END-SPEC.md, 2D-POLYGON-UNIONING-IMPLEMENTATION.md].

---

# Section 2: High-Level (Declarative & Constraint-Driven) Synthesis

The high-level syntax introduces schematic-driven physical synthesis [Core-System-Architecture.md]. The designer declares the logical schematic netlist and sets high-level structural constraints, while the compiler's middle-end solvers automate placement, floorplanning, and routing entirely [Core-System-Architecture.md].

```
                     Schematic-Driven Layout Synthesis
                     
     [ Schematic Netlist ] ──┐
                             ├──► [ Floorplanner & Solvers ] ──► [ Realized Layout ]
     [ Layout Constraints] ──┘      (Automated Placement)
```

### 2.1 What It Replaces

| Middle-Level Syntax (Leaving) [10-WIRING-INTEGRATION-GAPS.md] | New High-Level Syntax (Entering) [Core-System-Architecture.md] |
| :--- | :--- |
| Manual component positioning (even with relative offsets like `Pad_A.right + 2.0um`). | Abstract component instantiation without physical coordinates. |
| Explicit layout block positioning for every component. | General, declarative floorplanning constraints (e.g., `Pad_B right_of Pad_A with spacing: 2.0um`). |
| Explicit `route` declarations for every point-to-point connection. | Automated routing: the compiler extracts nets directly from the logical schematic and routes them [Core-System-Architecture.md]. |

---

### 2.2 Syntax and Keywords

*   **`implements`**: Binds a physical `space` to a logical `module` schematic, establishing LVS-by-construction [LOGICAL-BEHAVIOR-GUIDE.md, 12-ERROR-SYSTEM-OVERHAUL.md].
*   **`layout:`**: Introduces the declarative floorplanning block [15-DRC-SIMPLIFICATION.md].
*   **`at bottom` / `at top` / `at center_y`**: Declares absolute structural alignment zones relative to the board boundaries [Core-System-Architecture.md].
*   **`with spacing`**: Specifies the clearance interval between relatives, allowing the constraint solver to calculate layout density [Core-System-Architecture.md].

---

### 2.3 Concrete Code Example

This script defines the logical schematic for the two-pad connection and synthesizes the physical board layout automatically using high-level constraints.

```hardware
# Python/JS-Level: Pure schematic intent with layout constraints
import Silicon_250nm from foundry_pdk
import Aluminum from materials

# 1. Define the Logical Schematic (RTL / Netlist)
module Logic_Connection:
    pins: [io Pad_A_Port, io Pad_B_Port]
    
    # Declarative routing connection
    route Pad_A_Port to Pad_B_Port

# 2. Synthesize the Physical Space
space Pad_Connect implements Logic_Connection:
    resolution: 1nm
    profile: Silicon_250nm

    # Instantiate the components (no physical coordinates specified)
    add Pad named Pad_A
    add Pad named Pad_B

    # 3. Declarative Layout Constraints (Compiler solves the placement)
    layout:
        GND_Rail at bottom
        Pad_A at center_y
        Pad_B right_of Pad_A with spacing: 2.0um
```

---

### 2.4 Compiler Lowering and Execution

The compiler executes the high-level declarative constraints through a closed-loop, three-stage synthesis pipeline:

```
   1. Constraint Parser  ──►  2. Force-Directed Placer  ──►  3. Global & Detailed Router
   (Extracts netlist)          (Convex QP Solver)            (Negotiated Congestion)
```

1.  **Force-Directed Floorplanning:**
    The compiler's constraint manager reads the logical connections and the `layout:` block [15-DRC-SIMPLIFICATION.md]. It translates the spatial constraints (`Pad_B right_of Pad_A with spacing: 2.0um`) into a system of linear equalities and inequalities [10-WIRING-INTEGRATION-GAPS.md]:
    $$X_{\text{Pad\_B}} - X_{\text{Pad\_A}} \ge \text{Width}_{\text{Pad\_A}} + 2{,}000\text{ nm}$$
    The mathematical relationships are passed to the `clarabel` interior-point solver or a sparse active-set QP solver to calculate the non-overlapping, optimized coordinate coordinates [01-DATABASE-SPATIAL-FOUNDATION.md]:
    *   `Pad_A` is placed at the vertical center: $Y = 500\text{ nm}$.
    *   `Pad_B` is placed $2.0\mu\text{m}$ to the right, aligning perfectly on the $Y$ axis.

2.  **Global Routing & Partitioning:**
    The Partition Stage divides the board area into G-cells and plans the global route across boundaries, pre-negotiating G-cell crossings as locked interface ports to prevent routing congestion [Core-System-Architecture.md].

3.  **Topological Auto-Routing:**
    The Negotiated Congestion Engine (`PathFinder-style`) routes the nets [Core-System-Architecture.md]. Since the logic schematic is the single source of truth, the router extracts the `route Pad_A_Port to Pad_B_Port` directive, queries the spatial index for the components' boundary ports, and executes the topological line-search router to generate the mitered trace segments [Core-System-Architecture.md, 12-ERROR-SYSTEM-OVERHAUL.md].

---

# Section 3: Direct Coordinate Placement (No Wrapper Needed)

While higher-level abstractions are essential for development velocity, physical hardware eventually confronts raw physical boundaries (e.g., custom RF matching, analog standard-cell gates, high-voltage clearances) where the compiler's auto-solvers must be bypassed.

**Direct coordinate specification** with `at: [x: ..., y: ...]` acts as the explicit, opt-in bare-metal escape hatch of HardwareScript [Architectural-Specification.md]. It is the spatial equivalent of Rust's `unsafe` or `asm!` blocks. When you specify explicit coordinates, the compiler suspends all automatic layout, boundary-docking, and alignment solvers, allowing the designer (or LLM) to write raw, uncompromised picometer coordinate paths directly.

```hardware
# Using direct coordinates for precise control (no wrapper needed)
space Mixed_Signal_SoC:
    profile: Silicon_250nm

    # High-level relational placements
    add Processor_Core named CPU at [x: 50%, y: 50%]
    add RF_Transceiver named Radio right_of CPU with spacing: 1.0mm

    # ========================================================================
    # DIRECT COORDINATE PLACEMENT: Bypasses auto-solvers for precise control
    # ========================================================================
    
    # 1. Paint custom copper shielding to exact coordinates
    add pour(Copper) named RF_Shield_Trace on layer: metal1:
        boundary: [x: 450000pm, y: 300000pm] to [x: 455000pm, y: 800000pm]
        
    # 2. Force a manual routing path with explicit 3D waypoints
    route Radio.RF_OUT to Antenna_Port:
        width: 150nm
        layer: metal1
        path: [
            [450000pm, 300000pm],
            [450000pm, 800000pm]
        ]
```

### 3.1 Compiler Behavior With Direct Coordinates
*   **Auto-Solver Suspension:** When explicit coordinates are provided (`at: [x: ..., y: ...]`), the layout constraint solver and Topological AutoRouter are suspended for those statements.
*   **Direct Picometer Mapping:** Component placements and manual route `path` coordinates are mapped directly to the `EntityGraph` using 64-bit picometers, bypassing the relative transform step [01-DATABASE-SPATIAL-FOUNDATION.md].
*   **Active Verification (No Safety Sacrifice):** While layout and routing auto-solvers are suspended, **physical verification remains 100% active** [13-PHYSICAL-SYNTHESIS-GUARDRAILS.md]. The G-Cell sweep-line DRC, the PIVB connectivity solver, and the Sakurai parasitic extractor still analyze the manual geometry [PHYSICAL-SYNTHESIS-MIDDLE-END-SPEC.md, PIVB-Solver.md]. If a manual coordinate violates clearance rules or creates a short circuit, the compiler halts the build immediately with a specific violation code [15-DRC-SIMPLIFICATION.md].

This guarantees that you can write bare-metal physical code when necessary without sacrificing the safety and correctness of the overall design.