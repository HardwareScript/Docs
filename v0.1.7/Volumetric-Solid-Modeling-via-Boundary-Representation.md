# Architectural Specification: Volumetric Solid Modeling via Boundary Representation (B-Rep) in the hwc Compiler

---

## 1. Foundations of Boundary Representation (B-Rep)

In computer-aided design (CAD) and 3D computer graphics, **Boundary Representation (B-Rep)** is the standard method for representing physical, volumetric solids.

### The Mathematical Concept of B-Rep

B-Rep operates on the principle that a physical solid is uniquely defined by the boundary surfaces that separate its internal points (the solid substance) from its external points (empty space or surrounding air).

Mathematically, a solid $S$ in three-dimensional space $\mathbb{R}^3$ is defined by its boundary:

$$\partial S = \bigcup_i F_i$$

where $F_i$ represents the finite set of oriented 2D faces (surfaces) that enclose the volume. For the volume to be physically valid, the boundary must satisfy the **2-Manifold Property**:

* Every point on the boundary surface must be topologically equivalent to a flat 2D disk.
* Every edge on the boundary must be shared by exactly two adjacent faces.
* The surface must be completely closed with no holes, cracks, or self-intersections (creating a "water-tight" or manifold solid).

### The "Eggshell" Reality of Graphics Engines

While physical manufacturing treats materials as solid volumes of atoms, standard Graphics Processing Units (GPUs) and 3D file formats (such as glTF, GLB, STL, and OBJ) are surface-based. They have no concept of "internal volume."

To a GPU, a solid cube or cylinder is merely a hollow, infinitely thin "eggshell" composed of triangles. The compiler's task is to generate these surface boundaries with mathematical precision so that downstream physics engines, thermal simulators, and rendering pipelines can reconstruct and validate the physical solid correctly.

## 2. GPU and Graphics Pipeline Solid Modeling Mechanics

To render these hollow boundaries in a way that represents solid materials, the graphics pipeline relies on several low-level geometric behaviors.

```
       Vertices (V0, V1, V2)
                 |
                 v
     [ Winding Order (CCW vs CW) ] --> Determines Normal Vector (n)
                 |
                 v
       Normal Vector (n)
                 |
                 +---> Facing Camera (n . v < 0)  --> Render Face
                 |
                 +---> Facing Away  (n . v >= 0) --> Cull Face (Backface Culling)
```

### A. Normal Vectors

A Normal Vector ($\vec{n}$) is a 3D unit vector perpendicular to a surface face. In B-Rep, normal vectors are directional indicators:

* **Outward-pointing normal:** Indicates that the solid material lies behind the face, and empty space lies in front.
* **Inward-pointing normal:** Indicates that the solid material lies in front of the face, and a hollow void lies behind it.

For a triangle defined by three vertices $V_0$, $V_1$, and $V_2$, the surface normal is calculated mathematically using the cross product of the edge vectors:

$$\vec{n} = \frac{(V_1 - V_0) \times (V_2 - V_0)}{\|(V_1 - V_0) \times (V_2 - V_0)\|}$$

### B. Winding Order and the Right-Hand Rule

GPUs use the **winding order** of vertices to calculate the direction of the normal vector without running expensive cross-product operations.

* **Counter-Clockwise (CCW):** Vertices are arranged in a counter-clockwise sequence when viewed from the outside. By the Right-Hand Rule, the normal vector points **outward** (toward the camera).
* **Clockwise (CW):** Vertices are arranged in a clockwise sequence. The normal vector points **inward** (away from the camera).

### C. Backface Culling

To optimize rendering performance, GPUs discard faces that face away from the camera. This is called **Backface Culling**.

* If the dot product of the normal vector ($\vec{n}$) and the camera's view vector ($\vec{v}$) is greater than or equal to zero ($\vec{n} \cdot \vec{v} \ge 0$), the face is culled (not drawn).
* This is why entering a hollow 3D mesh appears transparent. Once the camera passes through the front surface, it looks at the back of the triangles where the normals point away from the camera, causing them to be culled.

### D. Depth Buffering (Z-Buffer) & Z-Fighting

When two co-planar faces share the exact same Z-coordinate, the GPU's rasterizer receives identical depth values. Microscopic floating-point rounding errors during camera movement cause the pixels to alternate between the two surfaces, producing a flickering visual defect called **Z-fighting**.

To prevent this, the compiler must resolve overlapping boundaries using 2D/3D Boolean operations (unions and subtractions) before exporting the meshes, ensuring that no two faces occupy the same coordinate plane.

---

## 3. Concrete Example 1: The Solid Cylinder (1-Shell Manifold)

A solid, conductive metal cylinder (such as a solid tungsten via plug) is modeled using a single, watertight closed shell.

```
                         The Solid Cylinder Mesh

                              [Top Cap] (Triangulated CCW Fan)
                             +-----------------------+
                             |        \   |   /      |
                             |          \ | /        |
                             |------------*----------|
                             |          / | \        |
                             |        /   |   \      |
                             +-----------------------+
                             | | | | | | | | | | | | |
                             | | | | | | | | | | | | |  Side Walls
                             | | | | | | | | | | | | |  (CCW Quads / Triangles)
                             | | | | | | | | | | | | |
                             +-----------------------+
                             |        \   |   /      |
                             |          \ | /        |
                             |------------*----------|
                             |          / | \        |
                             |        /   |   \      |
                             +-----------------------+
                            [Bottom Cap] (Triangulated CW Fan)
```

### Geometric Construction

The cylinder is composed of:

1. **The Side Walls:** A sequence of vertical rectangular quads wrapped around the circumference, split into triangles.
2. **The Top Cap:** A flat circular disk sealing the top.
3. **The Bottom Cap:** A flat circular disk sealing the bottom.

### Normal and Winding Configuration

To make this cylinder render as a solid volume, all faces must have outward-pointing normals:

* **Side Walls:** The vertices of the side triangles are wound in a **Counter-Clockwise (CCW)** direction, forcing the normal vectors to point radially outward.
* **Top Cap:** To triangulate this cap stably for GPU rasterizers, the compiler generates a central vertex at the circle's center $(x,\, y,\, z_{\text{top}})$. It then generates a **triangle fan** connecting the center to the outer ring. The vertices of these triangles are wound in a **Counter-Clockwise (CCW)** direction, forcing the normal vectors to point straight up.
* **Bottom Cap:** A similar triangle fan is generated at the bottom center $(x,\, y,\, z_{\text{bottom}})$. To ensure the normals point straight down (away from the interior), the triangles are wound in a **Clockwise (CW)** direction.

---

## 4. Concrete Example 2: The 3-Cylinder Stack (Concentric Plated Via with Epoxy Fill)

A complex, filled vertical interconnect—such as a Via-in-Pad Plated Over (VIPPO) filled with epoxy and capped with solid copper—cannot be modeled as a single cylinder. It consists of multiple concentric materials that must nest together.

To represent this physically and visually, the compiler constructs a **3-Cylinder Stack**:

```
                       Concentric 3-Cylinder Stack

                         [Cylinder 1]    [Cylinder 2]    [Cylinder 3]
                          Outer Plating   Inner Plating   Solid Epoxy
                              Wall            Wall         Core Plug
                                |               |              |
                                v               v              v
                              +---+           +---+          +---+
                              | C |           | C |          | E |
                              | o |           | o |          | p |
                              | p |           | p |          | o |
                              | p |           | p |          | x |
                              | e |           | e |          | y |
                              | r |           | r |          |   |
                              +---+           +---+          +---+
```

### The Three Concentric Shells

#### Shell 1: The Outer Plating Wall (Cylinder 1)

* **What it represents:** The interface where the plated copper meets the drilled FR4 dielectric wall.
* **Geometry:** A cylinder of diameter $d_{\text{outer}}$ (the physical drill size).
* **Winding & Normals:** Wound **Counter-Clockwise (CCW)**. The normals point radially **outward**, indicating copper substance lies inside this boundary.

#### Shell 2: The Inner Plating Wall (Cylinder 2)

* **What it represents:** The inside boundary of the hollow copper plating tube.
* **Geometry:** A cylinder of diameter $d_{\text{inner}}$ (the finished hole size, which is $d_{\text{outer}} - 2 \cdot t_{\text{plating}}$).
* **Winding & Normals:** Wound **Clockwise (CW)**. The normals point radially **inward** (toward the center axis), indicating that the copper substance lies behind the wall, and the central core is a void.

#### Shell 3: The Solid Epoxy Core (Cylinder 3)

* **What it represents:** The insulative epoxy plug filled inside the hollow copper tube.
* **Geometry:** A solid cylinder of diameter $d_{\text{inner}}$ (exactly matching the inner diameter of the copper tube).
* **Winding & Normals:** Wound **Counter-Clockwise (CCW)**. The normals point radially **outward**, indicating solid epoxy substance lies inside this cylinder.

### Concentric Boundary Nesting

When these three shells are placed concentrically:

* **The Copper Volume:** The space between Shell 1 (normal pointing out) and Shell 2 (normal pointing in) is evaluated as solid copper.
* **The Epoxy Volume:** The space inside Shell 3 (normal pointing out) is evaluated as solid epoxy.
* **Perfect Contact:** Because Shell 2 and Shell 3 share the exact same diameter ($d_{\text{inner}}$), they sit perfectly flush with one another. Since their normal vectors point in opposite directions, they form a clean, non-overlapping geometric boundary.

---

## 5. Hardware Script Translation Pipeline

The compiler uses this concentric shell model to translate high-level design descriptions into physically accurate 3D models.

### Step 1: Parsing and Abstract Representation

The compiler reads the design file and instantiates a generic `ContactPlacement` AST node. All technology-specific parameters are stored in the generic `properties` map:

```
Properties Map:
  - "drill_diameter":    0.6 mm
  - "plating_thickness": 25 um
  - "filled":            true
  - "fill_material":     "Epoxy"
  - "top_cap":           "solid"
  - "bottom_cap":        "solid"
```

### Step 2: The Semantic Lowering Phase

The compiler checks the Active Stackup Profile to calculate the absolute Z-boundaries. It then retrieves the parameters from the properties map and converts them into physical nanometer coordinates:

* $d_{\text{outer}} = 600{,}000 \text{ nm}$
* $t_{\text{plating}} = 25{,}000 \text{ nm}$
* $d_{\text{inner}} = 600{,}000 - (2 \times 25{,}000) = 550{,}000 \text{ nm}$
* $d_{\text{pad}} = 600{,}000 + (2 \times 200{,}000) = 1{,}000{,}000 \text{ nm}$

### Step 3: Concentric Assembly Generation

Instead of invoking a single hardcoded via mesh generator, the compiler lowers the parsed properties into separate concentric geometric structures in the database:

1. It adds a hollow **`Tube`** layer of `ViaCopper` to represent the plated walls (using $d_{\text{outer}}$ and $d_{\text{inner}}$).
2. Because `filled: true` is set, it adds a concentric solid **`Cylinder`** layer of `Epoxy` to represent the fill (using $d_{\text{inner}}$).
3. Because `top_cap: solid` and `bottom_cap: solid` are set, it generates solid flat circular pads of diameter $d_{\text{pad}}$ on the top and bottom copper layers.

### Step 4: The 2D Clipper Union Pass

To prevent rendering artifacts and overlapping meshes, the compiler groups all copper shapes on the same layer. The via pads are extracted as flat circular boundaries and merged with the rectangular traces using a **2D Boolean Union (Non-Zero Winding Rule)**.

Because the pads are welded directly into the traces in 2D, the trace and pad are extruded as a single, unified 3D solid copper mesh, completely eliminating Z-fighting.

### Step 5: Exporting the Manifold Solid

The vertical bare tube and the solid epoxy cylinder are exported as distinct, non-overlapping mesh nodes alongside the unioned copper trace meshes.

Because all boundaries are closed and normals are correctly oriented, the final GLB file is a water-tight manifold solid. It contains no redundant interior walls or overlapping volumes, making it completely stable for 3D visualization, 3D printing, and thermal/electromagnetic simulations.
