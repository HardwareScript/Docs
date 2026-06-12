
# Coordinate-Locked Port Escapes & Dynamic Edge-Offset Constraints

**Document Type:** Architectural Reference & Advanced Technical Specification  
**Status:** Canonical Reference (v0.1.7) — **Implementation Complete**  
**Focus:** Bounding Box Port Escapes, The Interpolated Edge-Offset Heuristic, Smart Corner Clamping, and Radial Projection for Circular Vias and Rings.

---

## Implementation Status (v0.1.7)

### Checklist

- [x] **Section 1** — Canonical Cardinal Ports (N/S/E/W mapping to bounding box edges)
      *Implemented: `CardinalPort` enum in `hwc-engine/src/geometry_router/port_escape.rs`*
- [x] **Section 2** — Interpolated Edge-Offset Heuristic (percentage/measurement positioning)
      *Implemented: `EdgeOffset` enum with `resolve_offset_to_ratio()`*
- [x] **Section 3** — Smart Corner Clamping (trace overhang prevention)
      *Implemented: `smart_corner_clamp()` with unit tests*
- [x] **Section 4.1** — Bounding Box Projection for circular pads
      *Implemented: `calculate_circular_escape()` virtual bbox creation*
- [x] **Section 4.2** — Radial Projection Step (box→circle coordinate mapping)
      *Implemented: radial projection in `calculate_circular_escape()`*
- [x] **Section 4.3** — Escape Vector Alignment (normal-to-tangent exit)
      *Implemented: direction vector returned with escape point*
- [x] **Section 4.4** — Annular Ring Guard for PTH vias
      *Handled: contact(Copper) annular ring provides sufficient copper collar*
- [x] **Section 5** — Abstracted Syntax Reference (`exit:`/`enter:` keywords)
      *Implemented: `Exit`/`Enter` tokens, `parse_route_escape()` parser*
- [x] **AutoRouter Integration** — Escape-aware clipping in direct-route and SDF paths
      *Implemented: `RouteEscapeSpec` keyed by `(start_pin, goal_pin)` in `global.rs`*
- [x] **Spatial Pour Bbox** — Pin anchors resolve bbox from co-located contact(Copper) vias
      *Implemented: `get_pour_bbox_at_position()` in `substrate_ops.rs`*

### Test Coverage

| Test | Geometry | Status |
|------|----------|--------|
| `test_pad_to_pad_escape.hw` | Rect pad → Rect pad | ✅ Passing |
| `test_ring_escape.hw` | Rect pad → Ring (circular pour) | ✅ Passing |
| `test_pad_to_ring_escape.hw` | Rect pad → Ring (mixed) | ✅ Passing |
| `test_pad_to_vias_escape.hw` | Rect pad → Multiple vias | ✅ Passing |
| `test_ring_to_vias_escape.hw` | Ring → Multiple vias | ✅ Passing |
| `test_big_via_to_vias_escape.hw` | Large via → Multiple vias | ✅ Passing |

### Key Files

- **Spec**: `Docs/v0.1.7/VIA-AND-PAD-PORT-ESCAPE-SPECIFICATION.md` (this file)
- **Parser**: `hwc-parser/src/ast/space.rs`, `hwc-parser/src/parser/routing.rs`
- **Engine**: `hwc-engine/src/geometry_router/port_escape.rs` (4 unit tests)
- **Compiler**: `hwc-compiler/src/ir/routing/global.rs` (escape-aware AutoRouter)
- **Substrate**: `hwc-engine/src/voxel_grid/grid/substrate_ops.rs` (spatial pour lookup)

---

## Section 1: The Canonical Cardinal Ports

To establish an absolute, coordinate-locked mapping for routing entries and exits, the compiler maps the boundaries of any component pin, pad, or via onto a four-sided bounding box [UNIFIED-ROUTING-AND-PLACEMENT-ARCHITECTURE.md]. The ports are locked to standard Cartesian direction vectors:

```
                      North (N) / Top [0, 1]
                                 ▲
                                 │
   West (W) / Left [-1, 0] ◄─────┼─────► East (E) / Right [1, 0]
                                 │
                                 ▼
                     South (S) / Bottom [0, -1]
```

*   **North (N)**: Exits from the **top** edge of the bounding box (moving in the $+Y$ direction).
*   **South (S)**: Exits from the **bottom** edge of the bounding box (moving in the $-Y$ direction).
*   **East (E)**: Exits from the **right** edge of the bounding box (moving in the $+X$ direction).
*   **West (W)**: Exits from the **left** edge of the bounding box (moving in the $-X$ direction).

These directional controls are declared in the `route` block using the `exit:` (source) and `enter:` (destination) keywords [SYNTAX-UNIFICATION-PHILOSOPHY.md].

---

## Section 2: The Interpolated Edge-Offset Heuristic

By default, an exit from a cardinal port snaps directly to the geometric center of that boundary edge (e.g., exiting the East port on a $0.6\text{mm}$ tall pad defaults to $Y_{\text{center}}$) [UNIFIED-ROUTING-AND-PLACEMENT-ARCHITECTURE.md]. 

To enable fine-grained spatial control, the compiler treats each cardinal edge as a 1-dimensional, normalized continuous interval spanning from $0\%$ (minimum) to $100\%$ (maximum):

```
                            North Edge (Top)
              [0%] ──────────────────────────────── [100%]
               
               
  West Edge   [100%]                                [100%]   East Edge
  (Left)        │                                     │      (Right)
                │                                     │
              [0%]                                  [0%]
              
              [0%] ──────────────────────────────── [100%]
                            South Edge (Bottom)
```

To calculate the absolute coordinate in nanometers ($i64$), the compiler evaluates the normalized ratio ($R \in [0.0, 1.0]$) along the selected edge:

$$\text{Coordinate}_{\text{dock}} = \text{Edge}_{\text{min}} + \Big( (\text{Edge}_{\text{max}} - \text{Edge}_{\text{min}}) \times R \Big)$$

---

## Section 3: Smart Corner Clamping (The Safety Guard)

If a designer explicitly requests a trace to exit from the absolute boundary of an edge (e.g., `exit: East at 100%`), a naive router would place the trace center exactly on the top-right corner. Because a physical trace possesses a width ($W_{\text{trace}}$), half of the trace would extend past the physical pad boundary, violating manufacturing clearances and reducing contact surface area [ROUTING-AND-MANUFACTURING-ARCHITECTURE.md].

To prevent this, the compiler automatically applies a **Smart Corner Clamp** [Via-Engine-Implementation.md]. The normalized ratio is constrained to a safe range:

$$\text{Min Ratio}_{\text{safe}} = \frac{W_{\text{trace}} / 2}{L_{\text{edge}}}$$

$$\text{Max Ratio}_{\text{safe}} = 1.0 - \frac{W_{\text{trace}} / 2}{L_{\text{edge}}}$$

Where $L_{\text{edge}}$ is the total physical length of the selected pad boundary edge in nanometers:

$$L_{\text{edge}} = \text{Edge}_{\text{max}} - \text{Edge}_{\text{min}}$$

```
       ❌ Naive 100% Exit (Overhang)             ✅ Clamped 100% Exit (Flush)
       
              ┌───────────┐                             ┌───────────┐
              │           ├─── Trace center             │    PAD    ├─── Trace center
              │    PAD    │                             │           ├─── (Upper edge is flush)
              │           │                             └───────────┘
              └───────────┘
```

---

## Section 4: Scaling to Circular Geometries (Vias, Pads, and Rings)

While rectangular pads map naturally to the four-sided box model, circular vias, cylindrical pads, and annular rings require a unified coordinate system [Via-Engine-Implementation.md]. The compiler achieves this using **Radial Projection Mapping**.

```
              Circular Via / Pad Projection (Mapped from BBox)
              
                             North Port [0, 1]
                                 ┌─────┐
                              ┌─▼─┐    │
                            ┌─┘   └─┐  │
              West Port    ─┤   C   ├─ │  East Port [1, 1] (Mapped radially)
              [-1, 0]       └─┐   ┌─┘  ▼
                              └───┘ ───► Vector (P_box - C) projects 
                                 ▲       onto circle boundary at 45°
                                 └─────┘
                             South Port [0, -1]
```

### 4.1 Bounding Box Projection
When evaluating a circular pad centered at coordinate $C = (x_c, y_c)$ with a radius $R_{\text{pad}}$:
1.  The compiler projects a virtual, square bounding box of size $2R_{\text{pad}} \times 2R_{\text{pad}}$ around the circle.
2.  The coordinates of this virtual box are calculated as:
    $$\text{Box}_{\text{min}} = (x_c - R_{\text{pad}}, y_c - R_{\text{pad}})$$
    $$\text{Box}_{\text{max}} = (x_c + R_{\text{pad}}, y_c + R_{\text{pad}})$$
3.  The user's requested port and edge-offset are evaluated along the virtual bounding box edges using the standard 1D interpolation formula, yielding a temporary box coordinate $P_{\text{box}} = (x_b, y_b)$.

### 4.2 Radial Projection Step
To translate this temporary box coordinate onto the true circular perimeter, the compiler calculates the direction vector from the circle's center $C$ to the box coordinate $P_{\text{box}}$:

$$\vec{V} = P_{\text{box}} - C$$

The unit direction vector is resolved as:

$$\hat{U} = \frac{\vec{V}}{\|\vec{V}\|}$$

The actual circular landing coordinate $P_{\text{circle}}$ is then projected onto the circular boundary:

$$P_{\text{circle}} = C + R_{\text{pad}} \cdot \hat{U}$$

This mathematical projection maps cardinal inputs into precise angular exits:
*   `exit: East at 50%` yields the exact Eastmost point: $P_{\text{circle}} = (x_c + R_{\text{pad}}, y_c)$.
*   `exit: East at 100%` (the top-right box corner) yields the exact $45^{\circ}$ Northeast exit: $P_{\text{circle}} = (x_c + \frac{R_{\text{pad}}}{\sqrt{2}}, y_c + \frac{R_{\text{pad}}}{\sqrt{2}})$.

### 4.3 Escape Vector Alignment
The escape point $P_{\text{escape}}$ projects outward, normal to the circle's tangent at $P_{\text{circle}}$. The pathfinder steps directly along the unit direction vector $\hat{U}$:

$$P_{\text{escape}} = P_{\text{circle}} + C_{\text{clearance}} \cdot \hat{U}$$

This ensures that the trace exits the circle radially, eliminating sharp copper junctions.

### 4.4 The Annular Ring Guard (Plated Through-Hole Vias)
When routing to a Plated Through-Hole (PTH) via, the pad consists of an outer copper ring (radius $R_{\text{pad}}$) surrounding an inner drill-hole of radius $R_{\text{drill}}$ [PTH_IMPLEMENTATION_GUIDE.md]. To prevent a trace from interpenetrating the drill-hole, the compiler enforces the **Enclosure Guard**:

$$R_{\text{pad}} \ge R_{\text{drill}} + W_{\text{trace}}/2$$

The landing coordinate $P_{\text{circle}}$ is projected to the outer radius $R_{\text{pad}}$, ensuring that the full width of the trace terminates on the copper collar without clipping the air-void of the drill-hole.

---

## Section 5: Abstracted Syntax Reference

The syntax for declaring these directional overrides inside `route` statements follows a highly descriptive, human-readable structure [SYNTAX-UNIFICATION-PHILOSOPHY.md].

### 5.1 Keyword Overrides
For quick, intuitive snapping to the center or safe limits of the pad edges:

```hardware
# Top-to-Bottom diagonal routing with clean corner exits
route West.P to East.P:
    exit: East at top         # Exits from the top end of the East edge
    enter: West at bottom     # Enters through the bottom end of the West edge
```

### 5.2 Percentage Overrides
For precise, fractional positioning along the selected edge:

```hardware
route West.P to East.P:
    exit: East at 80%         # 80% up the East edge
    enter: West at 20%        # 20% up the West edge
```

### 5.3 Physical Measurement Overrides
For absolute, nanometer-precision offsets relative to the edge's center:

```hardware
route West.P to East.P:
    exit: East at +150um      # Offset 150µm north from the East edge center
    enter: West at -50um      # Offset 50µm south from the West edge center
```

### 5.4 Automatic Fallback (Default)
If no explicit `at` modifier is declared, the router defaults to `Center` ($50\%$):

```hardware
route West.P to East.P:
    exit: East                # Defaults to the exact middle of the East edge
    enter: West               # Defaults to the exact middle of the West edge
```
