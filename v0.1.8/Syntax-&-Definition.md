

# Hardware Script (HWS) v0.1.8 - Syntax & Definition Specification

**Document Type:** Core Language Grammar & Syntax Specification  
**Status:** Feature Freeze (v0.1.8-alpha)  
**Focus:** The transition from Voxel-First to Vector-First Syntax and Abstractions  

---

## 1. Introduction: The Syntax Unification Boundary

The primary objective of the version 0.1.8 syntax refactoring is the complete elimination of **Z-axis abstraction leaks, grid quantization dependencies, and geometric implementation details** from the user's source code. 

By removing these structural leaks, the Hardware Script language becomes simpler, more declarative, and significantly easier for compilers to optimize and AI models to generate. 

Our grammar follows a strict rule: **The user describes physical intent (what should exist), and the compiler's middle-end optimization solvers handle physical realization (how to draw, route, and mesh it).**

---

## 2. Leaving vs. Entering/Promoted Syntax Map

| v0.1.7 Syntax (Leaving) | v0.1.8 Syntax (Entering / Promoted) | Architectural Rationale |
| :--- | :--- | :--- |
| **`grid: <X> by <Y> by <Z>`** (voxel grid cell count) | **`resolution: <Measurement>`** (continuous database snap-step) | Removes the 3D voxel grid memory footprint. The database transitions to continuous coordinates snapped to a physical step. |
| **`z: <Measurement>`** or raw index `z: 1` | **`on layer: <Identifier>`** (Strict Semantic Layer Abstraction) | Eliminates vertical elevation leaks. The `StackupManager` calculates and aligns Z heights dynamically on compile. |
| **`spanning z: <Start> to <End>`** | **`spanning layer: <Start> to <End>`** | Prevents physical via-to-pad floating gap errors. Vias snap exactly to coplanar layer boundaries. |
| **`pour` with `cutouts:`** | **`substrate(Material)` for dielectric, `plane(Material)` for conductive** | Semantic separation. `pour` is strictly additive copper. `substrate` = dielectric core (FR4, Si). `plane` = conductive sheet (Copper) with subtractive cutouts (antipads). |
| **`points:`** (manual coordinate lists for radial shapes) | **`geometry:` with generators / math loops** | Eliminates self-intersecting polygon errors that cause normal-warping and 3D rendering defects. |
| **Manual route `path:` arrays** | **Port-Aware Logical `route`** (`exit:`, `enter:`) | Bypasses the "inside-routing" bug. Pathfinder targets outer bounding box edges, keeping trace paths out of pad interiors. |
| **`current_limit: <Value>`** (single DC value for both EM and thermal) | **`current_limit: [rms: <Value>, peak: <Value>]`** (separate AC declarations) | EM verification uses $I_{\text{peak}}$; thermal-rise verification uses $I_{\text{RMS}}$. For DC-only pins, a single value is interpreted as both. Prevents clock traces with high peak currents from bypassing safety checks. |

---

## 3. In-Depth Code Transitions (Before vs. After)

### A. Space Block Definition

#### ❌ Before (v0.1.7 - Grid and Voxel Dependent)
The space definition forced the user to calculate and declare a rigid 3D voxel grid size. If the grid cell size was too coarse, the router suffered from quantization noise, creating stair-step trace jigs and phantom vias.

```hardware
space PCB_Board:
    dimensions: 20mm by 20mm by 1.27mm
    grid: 300 by 300 by 2             # 3D voxel array declaration
    origin: bl by b
    profile: StandardPCB
```

#### ✅ After (v0.1.8 - Continuous and Gridless)
The coordinate grid is completely removed. We replace it with a continuous database snap-step (`resolution:`). The compiler represents coordinates as absolute 64-bit integer picometers (pm), using zero memory footprint on compile. The `resolution:` attribute acts as a user-facing snapping constraint, while the engine always executes in picometers for sub-atomic precision.

```hardware
space PCB_Board:
    dimensions: 20mm by 20mm by 1.27mm
    resolution: 1nm                   # User-facing snap-step (engine uses 1pm internally)
    origin: bl by b
    profile: StandardPCB
```

---

### B. Route Statement and Pad Connection

#### ❌ Before (v0.1.7 - Manual Coordinate Paths & Inside-Routing)
To bypass routing obstacles, the user had to manually write absolute 3D coordinate path arrays inside the route statement. Furthermore, because the pathfinder targeted the pin's center and exempted the pad from collision, the trace entered the pad's interior, creating loops and hooks.

```hardware
# Traces are forced to route to pin centers and loop inside
route J1.p to J2.p:
    net: NET_AUTO
    path:
        - [x: 2mm, y: 10mm, z: 1235um]
        - [x: 8mm, y: 15mm, z: 1235um]
        - [x: 16mm, y: 15mm, z: 1235um]
```

#### ✅ After (v0.1.8 - Bounding Box Edge Docking)
The user specifies the connection logically using the component pins. The Topological Line-Search Router targets the outer bounding box edge of the selected cardinal port (North, South, East, West) and terminates the path the instant it touches the edge. The pad's interior is marked as impenetrable, preventing any internal loops.

```hardware
# Symmetrical, port-aware logical routing
route J1.p to J2.p:
    net: NET_AUTO
    exit: East      # Pathfinder docks on the outer East edge port
    enter: West     # Pathfinder docks on the outer West edge port
```

---

### C. Custom Via and Pad Shapes

#### ❌ Before (v0.1.7 - Fragile Point Lists)
Custom shapes were drawn using long, manual lists of points. These manual calculations frequently suffered from minor rounding errors and self-intersecting edges, which broke 3D triangulation and caused wavy normal-warping.

```hardware
shape Starburst(width: Measurement):
    points:
        - [x: 0,                y: -width / 2]
        - [x: width / 19.3,     y: -width / 4.1]
        - [x: width / 10.2,     y: -width / 2.2]
        # ... (32 lines of manual coordinates) ...
```

#### ✅ After (v0.1.8 - Parametric Geometry Blocks)
The manual point list is removed. It is replaced with a **`geometry:`** block that supports mathematical loops (`for i in 0..N`), local variable assignments, trigonometric functions (`sin`, `cos`, `tan`), and standard library math constants (`PI`, `DEG_TO_RAD`). It also natively supports procedural generator calls.

```hardware
import * from @std/primitives/math

# Option 1: Native Procedural Generator (Mode D)
shape Starburst_Gen(width: Measurement):
    geometry:
        StarGenerator(points: 24, outer: width / 2, inner: width / 3.5)

# Option 2: Parametric Loop (Mode B)
shape Starburst_Loop(width: Measurement):
    geometry:
        for i in 0..31:
            let angle = i * 11.25
            let rad = angle * DEG_TO_RAD
            let r = if i mod 2 = 0: width / 2 else: width / 3.5
            Point(x: r * cos(rad), y: r * sin(rad))
```

---

### D. Ground Planes and Subtractive Geometry

#### ❌ Before (v0.1.7 - Mixed Additive & Subtractive Pours)
Pours, which are physically additive copper structures, were allowed to declare subtractive `cutouts:` blocks to define voids and plane clearances. This created a semantic conflict in the physical database.

```hardware
add pour(Copper) named GND_Plane:
    boundary: [x: 0, y: 0] to [x: 20mm, y: 20mm]
    net: GND
    thickness: 35um
    cutouts:                                      # Semantic conflict on pour
        Rectangle(width: 8mm, height: 16mm) at [x: 10mm, y: 10mm]
```

#### ✅ After (v0.1.8 - Semantic Plane Separation)
We cleanly segregate subtractive conductive sheets from subtractive dielectric sheets. `substrate` is reserved strictly for non-conductive dielectric core materials (FR4, Silicon, Glass). `plane` is introduced as a first-class keyword for large conductive sheets (Copper, Aluminum) that support subtractive `cutouts:` (antipads, plane voids). This eliminates the semantic conflation between dielectric mounting holes (which cut through everything) and conductive antipads (which cut through copper only, leaving FR4 intact).

```hardware
# Non-conductive structural base (cuts through everything)
add substrate(FR4) named MainBoard:
    spanning layer: top_copper to bottom_copper
    cutouts:
        Circle(3.2mm) at [5mm, 5mm]      # Mounting hole: cuts FR4 AND copper

# Conductive reference plane with electrical net binding (cuts copper only)
add plane(Copper) named GND_Plane:
    spanning layer: gnd_layer to gnd_layer
    net: GND
    cutouts:
        Rectangle(width: 8mm, height: 16mm) at [x: 10mm, y: 10mm] # Antipad: cuts copper, FR4 intact
```

---

## 4. AST and Grammatical Impact

By moving to this unified vector-first representation, we can significantly simplify the parser and AST structures inside `hwc-parser`:

1.  **Deletion of Soft Keyword Hacks:** Property keys inside definitions (such as `tolerance`, `clearance`, `trace`, `via`) are no longer parsed as custom tokens. They are tokenized directly as standard `IDENTIFIER` strings, reducing the lexer's token variants by 20%.
2.  **Removal of the String-Reparsing Loop:** In earlier versions, geometry expressions were serialized back to strings and re-parsed in a secondary, hand-rolled parser. In v0.1.8, the parser does its job once—parsing expressions directly into recursive `Expr` AST nodes. The geometry engine evaluates this tree directly, completely eliminating parsing crashes due to token spacing variations.
3.  **Strict Semantic Separation:** The AST enforces the boundary between declarative facts (colons `:` inside property blocks) and behavioral actions (equals `=` inside logic blocks), making the compiler's backend extremely lean and robust.
