# Architectural Specification: Technology-Agnostic "Microkernel" Compiler and Engine Architecture

**Version:** 0.1.7  
**Document Type:** Core Architecture Specification  
**Status:** Normative

---

## 1. Executive Architectural Overview & The Problem Statement

### The Core Anti-Pattern

The fundamental architectural liability in domain-specific layout compilers is **structural over-specialization**. Hardcoding physical concepts (such as "Blind Via," "Buried Via," "Microvia," or "Through-Silicon Via") into the compiler's grammar, AST, and rendering pipeline introduces several critical problems:

1. **AST Bloat**: Every new technological innovation (e.g., stacked microvias, active silicon interposers) requires modifying the parser, AST nodes, and compilation passes.

2. **Platform Inflexibility**: A compiler hardcoded for PCB design cannot easily compile Silicon/IC designs, or vice-versa, because the interconnect structures are treated as distinct semantic entities instead of unified physical geometries.

3. **Leaky Abstractions**: The compiler must understand high-level fabrication details, which breaks separation of concerns and complicates compiler maintenance.

### The Solution: The "Microkernel" Compiler Philosophy

The compiler must be designed as a **geometric and physical microkernel**. It remains completely unaware of manufacturing terminology. Instead, it operates on a restricted set of mathematical primitives:

- **Constructive Solid Geometry (CSG)**
- **3D Bounding Volumes**
- **Material Characteristics**
- **Electrical Networks (NetIDs)**
- **Physical Proximity Rules**

All high-level structural concepts are deferred to the **Standard Library (`@std`)** and **User Space** as parameterized configurations of these core primitives.

---

## 2. Define "The Line" (Boundary Classification Matrix)

To maintain a strict separation of concerns, we classify all compiler and language behaviors into three distinct layers:

```
┌─────────────────────────────────────────────────────────────────┐
│                       USER SPACE                                │
│   - Specific PCB Layouts      - Custom Component Placement      │
│   - Padstack Configurations   - Signal Net Routings             │
└─────────────────────────────────────────────────────────────────┘
                               ↓
┌─────────────────────────────────────────────────────────────────┐
│                    STANDARD LIBRARY (@std)                      │
│   - Blind/Buried Via Macros   - FR4 / Silicon Materials         │
│   - SOT/QFN Package Footprints - DRC Rulesets (IPC-2221)        │
└─────────────────────────────────────────────────────────────────┘
                               ↓
┌─────────────────────────────────────────────────────────────────┐
│                    COMPILER ENGINE CORE                         │
│   - CSG Mesh Generators       - Voxel Collision & Net Tracker   │
│   - Generic Parser & AST      - Netlist Spanning & Mesh Export  │
└─────────────────────────────────────────────────────────────────┘
```

### Boundary Classification Matrix

| Feature Domain | Compiler Engine Core (Intrinsics) | Standard Library (`@std`) | User Space (Designs) |
|:---|:---|:---|:---|
| **Lexing & Parsing** | Parses generic structural layout grammar: blocks, property maps, arrays, coordinate spaces, and physical profiles. | None. | None. |
| **AST Representation** | Exposes a single, unified `ContactInstantiation` node containing generic parameters (Z-bounds, shape, diameters, caps). | None. | Instantiates via-types by referencing standard library macros or profiles. |
| **Geometric Primitives** | Rectangles, Cylinders, Tubes, and basic CSG subtraction (holes/cutouts). | Complex multi-primitive composites (e.g., staggered via groupings). | Specific coordinate placements of primitives. |
| **Physical Materials** | Resolves abstract physical categories: Conductor, Insulator, Semiconductor. Tracks physical attributes like conductivity and permittivity. | Declares exact chemical/physical material specifications (e.g., `FR4`, `Copper`, `SiO2`, `GaAs`, `Polyimide`). | Assigns specific materials to layout structures and planes. |
| **Design Rule Checking (DRC)** | Calculates clearances using generic mathematical distance equations (collision detection, minimum spacing bounds). | Defines the rulesets and values for standard manufacturing tolerances (e.g., IPC class 2 vs. TSMC N7 rules). | Specifies safety factor overrides and specific high-voltage boundaries. |
| **Mesh Generation & Export** | Tessellates mathematical models into raw vertex/index data (GLTF, DXF) with precision-alignment algorithms. | None. | Generates output files via compiler flags for target foundries. |

---

## 3. The Unified Parametric Interconnect Primitive

To eliminate distinct code paths for different via types, we collapse all vertical interconnects into a single mathematical construct: **The Parametric Unified Interconnect**.

### Mathematical Definition

Let any vertical connection **V** be represented in the engine as a single tuple:

```
V = (x⃗, z_start, z_end, d_outer, d_inner, d_pad, Cap_top, Cap_bottom, NetId)
```

**Where:**

- **x⃗ = (x, y)** is the center coordinate in 2D space
- **z_start, z_end** are the lower and upper bounds in the Z-axis (calculated from the physical stackup)
- **d_outer** is the drill/hole diameter
- **d_inner** is the finished inner hole diameter (after plating). If d_inner = 0, the interconnect is solid
- **d_pad** is the pad/annular ring diameter
- **Cap_top, Cap_bottom ∈ {None, Annular, Solid}** define the geometry of the top and bottom terminations

### Mapping High-Level Interconnects to the Unified Primitive

Using this parameter set, the lowering engine can represent any arbitrary via type without changing its core Rust logic:

```
PCB PTH Via        PCB Blind Via      PCB Buried Via       Silicon TSV
(Hollow)            (Capped)            (Capped)            (Solid)

Annular             Annular              Solid              Solid
+---+ +---+         +---+ +---+         +---+---+           +-------+
|   | |   |         |   | |   |         |       |           |       |
| Plating |         | Plating |         |Plating|           | Solid |
|  Wall   |         |  Wall   |         | Wall  |           | Core  |
|   | |   |         |   | |   |         |       |           |       |
+---+ +---+         +-------+           +-------+           +-------+
Annular              Solid               Solid              Solid
```

#### 1. Plated Through-Hole (PTH) Via

**Physical Mapping:** Hollow cylinder spanning the entire board.

**Primitive State:**
- d_inner = d_outer - 2 × t_plating
- d_pad = d_outer + 2 × w_annular
- Cap_top = Annular, Cap_bottom = Annular

#### 2. Blind Via

**Physical Mapping:** Hollow cylinder terminating on an inner copper layer with a solid pad on the target layer.

**Primitive State:**
- d_inner = d_outer - 2 × t_plating
- d_pad = d_outer + 2 × w_annular
- Cap_top = Annular, Cap_bottom = Solid (seals the bottom of the drill-hole)

#### 3. Buried Via

**Physical Mapping:** Hollow cylinder hidden inside inner layers, sealed at both ends.

**Primitive State:**
- d_inner = d_outer - 2 × t_plating
- d_pad = d_outer + 2 × w_annular
- Cap_top = Solid, Cap_bottom = Solid

#### 4. Through-Silicon Via (TSV)

**Physical Mapping:** A solid metal pillar (e.g., tungsten) passing through a silicon substrate, surrounded by a thin insulative dielectric liner.

**Primitive State:**
- d_inner = 0 (Solid core)
- d_outer = d_core_metal + 2 × t_liner
- d_pad = d_core_metal
- Cap_top = Solid, Cap_bottom = Solid

---

## 4. AST & Parser Refactoring Specification

To implement this change, we must refactor the compilation pipeline. We will replace specialized keyword processing (such as parsing explicit blind or PTH via blocks) with a single generic contact AST node.

### The Refactored AST Node Structure

```rust
pub struct ContactNode {
    pub name: String,
    pub location: Point2D,
    pub start_layer: String,
    pub end_layer: String,
    pub net_binding: Option<String>,
    pub parameters: HashMap<String, ValueExpr>,
}
```

### The Unified Compiling & Lowering Pipeline

```
Hardware Script Source (.hw)
           ↓
    [ Lexer & Parser ]
           ↓
  Unified ContactNode AST
           ↓
  [ Lowering Engine ] <--- Reads physical properties from Stackup Profile
           ↓
Creates SubstrateLayer (Tube/Cylinder)
           ↓
[ Physical Exporter ] (Generates 3D GLB Meshes / DXF Contours)
```

#### Pipeline Stages:

1. **Syntax Stage**: The parser reads a generic `add contact` statement with an associated property map.

2. **Analysis Stage**: The compiler resolves layer names (top, inner1, bottom) against the active layout stackup profile to compute precise nanometer Z-coordinates (z_min, z_max).

3. **Validation Stage**: The compiler verifies that the assigned geometry meets the active stackup rules. For example, it checks if the aspect ratio (depth / drill_diameter) is manufacturable.

4. **Lowering Stage**: The compiler converts the `ContactNode` into a `SubstrateLayer` struct using `SubstrateLayerShape::Tube` or `SubstrateLayerShape::Cylinder` before sending it to the rendering and validation engines.

---

## 5. Standard Library (@std) Abstract Interface Layer

By moving technology-specific rules to the standard library, we allow engineers to use clean, intuitive declarations while the compiler only has to maintain a single, highly optimized code path.

### Concrete Example: Designing the Via Standard Library

We can define a standard library module (`@std/pcb/vias`) that wraps the generic contact primitive to create helper macros for common PCB structures:

```hw
# File: @std/pcb/vias.hw

# Macro for generating standard Plated Through-Hole Vias
macro PTHVia(name, x, y, net_name, drill, pad):
    add contact(ViaCopper) named name at [x: x, y: y] spanning layer: bottom to top:
        net: net_name
        drill_diameter: drill
        plating_thickness: 25um
        annular_ring: (pad - drill) / 2
        caps: true

# Macro for generating standard Blind Vias
macro BlindVia(name, x, y, net_name, start, end, drill, pad):
    add contact(ViaCopper) named name at [x: x, y: y] spanning layer: start to end:
        net: net_name
        drill_diameter: drill
        plating_thickness: 25um
        annular_ring: (pad - drill) / 2
        caps: true
```

### The Clean User Space Design API

Using these standard library helpers, the end-user design files remain readable and clean, while the compiler backend stays completely generic:

```hw
import * from @std/primitives/units
import FR4 from @std/materials/insulators
import { PTHVia, BlindVia } from @std/pcb/vias

space HighDensityPCB implements PrototypeModule:
    dimensions: 10mm by 10mm by 1.6mm
    profile: AdvancedPCBStackup
    
    # Clean, intuitive via placement via Standard Library Macros
    PTHVia(V1, 2.0mm, 2.0mm, SIG_NET, 0.3mm, 0.6mm)
    BlindVia(V2, 5.0mm, 5.0mm, VDD_NET, top, inner1, 0.15mm, 0.35mm)
```

---

## 6. Unified 3D/GLB Engine Exporter Pipeline

The 3D export engine must process the simplified, parametric primitives and output clean, manifold meshes. To do this reliably, we use a single geometry function: `create_via_mesh`.

```rust
pub fn create_via_mesh(
    name: &str,
    center: (f64, f64, f64),
    drill_dia: f64,
    pad_dia: f64,
    plating_thickness: f64,
    height: f64,
    segments: u32,
    top_cap: CapType,
    bottom_cap: CapType,
    material_name: &str,
    view: SpaceView,
) -> MeshNode
```

### Essential Rendering Guardrails

#### 1. The 1-Micron Epsilon Guard

To prevent co-planar rendering artifacts ("Z-fighting") where the top face of a via meets a copper pad or ground plane, the exporter must apply a 1-micron vertical inset to the via cylinder's height:

```rust
let mut actual_height = height;
let mut cz = center.2;

if actual_height > 0.002 {
    cz += 0.001;        // Move starting position up by 1 micron
    actual_height -= 0.002; // Shorten overall cylinder by 2 microns
}
```

#### 2. Manifold Boundary Rule

When calculating the 3D meshes for surrounding substrate layers, the exporter must subtract the via's drill cylinder (d_outer) from the dielectric boards to create clean, non-overlapping geometric interfaces.

#### 3. Dynamic Segment Scaling

Instead of hardcoding vertex counts, the segment count should scale based on the target via diameter. This keeps low-resolution, high-density via arrays fast and performant, while ensuring high-resolution close-ups remain visually smooth.

```rust
let segments = if drill_dia < 0.2 { 16 } else { 64 };
```

---

## 7. Implementation Roadmap & Guardrails

To execute this refactoring smoothly without causing regressions, follow this systematic implementation plan:

### Phase 1: Engine Alignment (Backend)

1. **Unify Rust Representations**: Retain only the `SubstrateLayerShape::Tube` and `SubstrateLayerShape::Cylinder` variants inside the engine's internal structure, and remove any lingering specialized via logic.

2. **Standardize Mesh Generation**: Ensure that all 3D mesh rendering for vertical connections is routed through the parametric `create_via_mesh` function.

### Phase 2: AST Refactoring (Parser)

1. **Consolidate AST Nodes**: Merge any distinct via syntax rules into a single, generic `ContactNode`.

2. **Parametric Property Parsing**: Update the parser to read all physical dimensions (such as drill sizes, plating depths, and cap styles) from a standard property map.

### Phase 3: Standard Library Migration (@std)

1. **Write standard macros**: Implement the standard library macros (`PTHVia`, `BlindVia`, `BuriedVia`) to wrap the low-level contact primitive.

2. **Update regression tests**: Port your existing test suites to use these standard library macro templates, verifying that they lower into identical engine geometries.

---

## 8. Architectural Benefits & Strategic Impact

By committing to this architecture, you achieve:

### Scalability
- **Technology Independence**: The same compiler handles PCBs, IC layouts, MEMS devices, and future fabrication technologies without core changes.
- **Zero AST Churn**: New interconnect technologies (e.g., laser-drilled microvias, photonic interconnects) are added via standard library extensions, not compiler modifications.

### Maintainability
- **Single Code Path**: One mesh generator, one collision detector, one netlist tracker—drastically reduced code complexity.
- **Clear Separation of Concerns**: Compiler engineers focus on geometric algorithms; domain experts focus on manufacturing rules.

### Performance
- **Optimized Core**: With a unified primitive model, the engine can apply aggressive optimizations (spatial indexing, batch processing) that would be impossible with heterogeneous via types.

### Extensibility
- **User Empowerment**: Advanced users can define custom interconnect types in their own libraries without touching compiler code.
- **Rapid Innovation**: New fabrication processes can be modeled and tested immediately by creating new standard library modules.

---

## 9. Conclusion

This microkernel architecture transforms the compiler from a domain-specific tool into a general-purpose geometric layout engine. By deferring all manufacturing knowledge to the standard library and user space, we protect the core engine from technological churn and create a scalable system capable of designing everything from complex PCBs to sub-micron silicon chips using the exact same compilation pipeline.

**This document defines the boundary line between compiler internals, standard libraries, and user space, and outlines a concrete path forward to keep the compiler scalable, maintainable, and technology-agnostic.**
