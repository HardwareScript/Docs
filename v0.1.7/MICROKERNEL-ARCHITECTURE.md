# Architectural Reference: Technology-Agnostic Microkernel Compilation for Spatial Layouts

**Version:** 0.1.7  
**Document Type:** Core Architecture Reference  
**Status:** Authoritative / Normative

---

This document serves as the authoritative conceptual reference for the technology-agnostic "microkernel" architecture of the compiler and engine. It explains how high-level physical design declarations in Hardware Script are compiled, validated, and translated into mathematically accurate, simulation-ready 3D solid geometries without hardcoding domain-specific layout concepts into the compiler binary.

## 1. The Core Philosophy: "No Magic, Just Primitives"

Traditional Electronic Design Automation (EDA) compilers are structurally rigid. They are built on high-level, domain-specific concepts—such as "Blind Vias," "Buried Vias," "Microvias," "Solder Mask Openings," and "Through-Silicon Vias"—and hardcode those terms into their syntax, parsers, and rendering engines.

This creates a leaky abstraction and an inflexible compilation pipeline. If a silicon foundry or packaging facility invents a new spatial structure, the entire compiler binary must be rewritten.

### The Microkernel Strategy

The microkernel approach strips the compiler of all manufacturing-specific knowledge. The compiler remains completely ignorant of what a "Via-in-Pad" or a "Tapered Microvia" is.

Instead, the compiler is a pure Geometric and Physical Synthesis Engine that operates on a minimal set of highly generic mathematical primitives:

1.  **Constructive Solid Geometry (CSG)**: Subtracting and adding shapes (such as drilling a cylinder hole out of a flat box).
2.  **Concentric 2D Contours**: Circles, rings, and polygons that can be nested and unioned on discrete layer planes.
3.  **Parametric 3D Extrusions**: Extruding a 2D boundary along a vertical Z-axis.
4.  **Physical Properties Map**: Exposing a generic key-value dictionary for dimensions, allowing standard libraries (@std) to configure physical attributes.

By shifting all specialized design terminology to the Standard Library (@std) and User Space, the compiler's codebase remains small, highly optimized, and infinitely scalable across both printed circuit boards (PCBs) and silicon chip layouts.

## 2. The Multi-Stage Compilation Pipeline

A spatial layout compiled under the microkernel architecture follows a strict, one-way synthesis pipeline. The compiler transforms abstract, human-readable source code into exact physical solids.

```
1. SYNTAX STAGE
- Parses generic declarations (add, contact, pour)
- Populates generic properties dictionary
       |
       v
2. SEMANTIC LOWERING STAGE
- Resolves elevations (Semantic layers -> Nanometer coordinates)
- Retrieves stackup clearances and default diameters
       |
       v
3. GEOMETRIC SEPARATION STAGE
- Divides vias into 3D Vertical Barrels and 2D Layer Pads
       |
       v
4. 2D POLYGON UNIONING STAGE (Clipper)
- Collects all copper on a layer (traces + via pads)
- Performs Boolean Union using the Non-Zero winding rule
       |
       v
5. MANIFOLD TESSELLATION STAGE (Earcut)
- Triangulates flat unioned profiles (handles outer borders & holes)
- Extrudes faces vertically into a single solid 3D mesh
```

## 3. The Property Map Pattern (How We Prevent AST Creep)

To ensure that the parser and Abstract Syntax Tree (AST) never have to be modified when new manufacturing technologies emerge, we use the Property Map Pattern.

Instead of declaring dedicated fields in the Rust struct for every possible physical dimension (such as `drill_diameter`, `liner_thickness`, or `plating_thickness`), the AST node for vertical connections (`ContactPlacement`) is stripped down to its core identity:

- **Material**: The primary conductive metal.
- **Name**: The spatial identifier.
- **Coordinate**: The XY placement center.
- **Z-Elevation Span**: The start and end layer boundaries.
- **Net Name**: The electrical network binding.
- **Properties Map**: A generic key-value dictionary of string identifiers and expressions.

### The Standard Library Layer

When a designer uses a high-level helper from the standard library (such as a `BlindVia` macro), the macro maps user-friendly arguments to the compiler's generic properties map:

**User writes:** `BlindVia(x: 5mm, y: 5mm, drill: 0.3mm, pad: 0.6mm)`  
↓  
**Macro compiles to:** `add contact at [5mm, 5mm]` with properties:
- `"drill_diameter"`: 0.3mm
- `"annular_ring"`: (0.6mm - 0.3mm) / 2
- `"top_cap"`: "annular"
- `"bottom_cap"`: "solid"

During compilation, the lowering phase queries this dictionary. If a property is present, the compiler evaluates it and applies it; if it is missing, the compiler quietly falls back to the default values specified in the Stackup Profile.

## 4. The "Split-Via" Geometric Strategy

Vias are inherently hybrid structures: they are partly 3D vertical pipes (passing through dielectric cores) and partly 2D flat sheets (merging with traces on copper planes).

Treating a via as a single, monolithic 3D cylinder inside a 2D layout engine is an anti-pattern. It prevents the via pads from merging with connecting traces, causing Z-fighting and overlapping geometry.

The microkernel solves this by splitting the via's geometry during the export phase:

```
        Physical Via Structure             Split Exporter Model
        
             Top Pad                             Top Pad (Flat 2D Circle) 
          +===========+                       +===========+ 
          | \       / |                       |           | 
          |  \     /  |                       |           | 
Dielectric|   \   /   | Dielectric            |           |   Bare Tube 
Core (FR4)|    \ /    | Core (FR4)            |           |  (Hollow Cone, 
          |     |     |                       |           |    No Caps) 
          |     |     |                       |           | 
          |    / \    |                       |           | 
          |   /   \   |                       |           | 
          +==/=====\==+                       +===========+ 
           Bottom Pad                          Bottom Pad (Flat 2D Circle) 
```

### 1. The Vertical Barrel (Bare Tube)

The central shaft of the via is rendered analytically as a hollow tube (using `create_via_mesh`).

- Its caps are set to `CapType::None`. It is a bare, open pipe spanning vertically from the bottom of the start layer to the top of the destination layer.
- If it is a microvia, the tube's geometry is generated as a tapered frustum (cone).

### 2. The Landing Pads (Annular Rings)

The flat copper rings (the caps) are extracted as 2D circular boundaries.

- If a cap is `CapType::Annular`, the compiler generates two concentric circular paths: an outer path (pad diameter, counter-clockwise) and an inner path (drill hole diameter, clockwise).
- If a cap is `CapType::Solid`, the compiler generates a single solid circular path.
- These flat 2D paths are injected directly into the 2D Clipper pool of the corresponding copper layer.

## 5. 2D Polygon Unioning (Geometric Welding)

Once all copper traces and via landing pads have been collected and grouped by their layer Z-boundaries, the compiler runs them through the 2D Boolean Clipper Engine.

```
     Concentric Via Pad Paths                    Trace Path 
   (CCW Outer Ring, CW Inner Hole)         (Closed Rectangle Path) 
                \                                     / 
                 \                                   / 
                  v                                 v 
              [ Clipper2-Rust Boolean Union Engine ] (Non-Zero Rule) 
                                  | 
                                  v 
                     Single, Unified Outer Contour 
                     with a Hollow Inner Hole (Donut) 
                                  | 
                                  v 
                      [ Mesh Extrusion (Earcut) ] 
                                  | 
                                  v 
                Unified Solid Copper 3D Manifold Mesh 
```

### Why the Winding Rule Matters (Non-Zero vs. Even-Odd)

When Clipper merges overlapping paths, it must use the **Non-Zero Winding Rule**:

- **Even-Odd Winding**: Counts overlapping boundaries. An overlap count of 2 is considered "outside," which incorrectly punches hollow rectangular holes in the middle of overlapping copper pads and traces.
- **Non-Zero Winding**: Treats any region enclosed by one or more boundaries as solid. This correctly merges (welds) overlapping boundaries into a single continuous solid copper volume.

### Extrusion & Triangulation

The unified 2D contour generated by the Clipper engine is triangulated using `earcut` (the zero-allocation polygon triangulator). It translates the flat 2D shape—including any complex boundaries and hollow inner holes—into raw, non-degenerate triangles.

These triangles are then duplicated at the top and bottom of the layer's thickness ($35\mu m$) and connected with vertical side walls to create a single, continuous, solid 3D mesh node.

## 6. How the System Prevents Visual and Physical Defects

By combining Unified Primitives, Dynamic Cap Extraction, and 2D Clipper Unioning, the microkernel resolves the core geometric defects of layout compilers:

| Defect Type | Root Cause in Legacy Engines | How the Microkernel Resolves It |
| :--- | :--- | :--- |
| **Z-Fighting (Flickering)** | Co-planar faces sharing the exact same Z-coordinate (e.g., via pad resting on a trace). | The via pad and the trace are merged in 2D and extruded as a **single mesh node**. Because there are no overlapping faces, Z-fighting is mathematically impossible. |
| **The "Hollow-Trough" Bug** | Blindly culling the top/bottom faces of trace slices due to local overlaps. | Face culling is protected by a **Bounding Box Match Guard**. Slices only cull their faces if they overlap completely (stackup layers). Local overlaps (pads on traces) are left solid. |
| **The "Solid-Via" Bug** | Treating 3D vertical vias as 2D flat copper layers, losing their hollow cores. | **Dynamic Cap Extraction** separates the 3D hollow tube from the flat caps. The tube remains a hollow cylinder, while only the flat rings are merged with the trace. |
| **Simulation Failures** | Overlapping, interpenetrating meshes confuse boundary calculation engines. | The exporter output is a **perfect, non-overlapping solid manifold**. There are no duplicate volumes or internal hidden walls, ensuring accurate simulation in thermal and electromagnetic solvers. |

## 7. The 4-Mode Shape Design System

The shape system extends the microkernel's generic primitives with a unified shape design paradigm that supports four levels of abstraction, allowing designers to work at their preferred level of complexity.

### Four Design Paradigms

| Paradigm | Abstraction Level | Example |
| :--- | :--- | :--- |
| **Manual** | Lowest level, full control | Direct polygon coordinates: `Point(0,0), Point(1000,0), ...` |
| **Parametric** | Equation-driven with variables | `let r = width/2; Circle(r)` with variable substitution |
| **CSG** | Boolean operations on primitives | `Union(circle, rect) - Circle(...)` for complex outlines |
| **Procedural** | Algorithmic generation | `star_generator_contour(points=5, width=1000nm)` |

These paradigms are implemented via geometry blocks in the AST and evaluated by dedicated shape generators.

### Integration with Via System

The shape system integrates with the via insertion pipeline through profile parameters:

- **`via: shape:`** parameter in stackup profiles specifies which shape generator to use for auto-via insertion
- The compiler evaluates shape contours and scales them to the via diameter using the `width` parameter
- Custom contacts with `shape:` parameter enable multi-shape stitching within a single space, with collision detection to prevent overlaps

### Multi-Shape Stitching

Multiple shapes can coexist in a single layout space through custom contact declarations that reference different shape generators. The compiler performs collision detection and union operations to create unified copper geometries from diverse shape sources.

For complete details on the shape system architecture, see [SHAPE-SYSTEM-ARCHITECTURE.md](./SHAPE-SYSTEM-ARCHITECTURE.md).
