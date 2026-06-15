
# 4-Mode Shape Design System Architecture

**Document Type:** Authoritative Architecture Reference  
**Status:** Canonical Reference (v0.1.7) — **Implementation Complete**  
**Focus:** The four-mode radial/polygonal geometry definition paradigm for auto-via insertion, CSG boolean operations, procedural generator calls, and geometry block evaluation.

---

## Implementation Status (v0.1.7)

### Checklist

- [x] **Mode A** — Manual Points (`points:` block with `Point(x, y)` entries)
      *Implemented: `GeometryStatement::Point` in `hwc-parser/src/ast/shape.rs`*
- [x] **Mode B** — Parametric Loops (`for i in 0..N: let angle = ...; Point(x: ..., y: ...)`)
      *Implemented: `GeometryBlock::ForLoop`, `GeometryStatement::Variable`, `GeometryStatement::Point`*
- [x] **Mode C** — CSG Boolean (`union`, `intersect`, `subtract` of primitives)
      *Implemented: `CsgExpression`, `CsgPrimitive` AST nodes, Clipper2 integration*
- [x] **Mode D** — Procedural Generators (`star_generator(...)`, `gear_generator(...)`)
      *Implemented: `ShapeGenerator` AST, `star_generator_contour`, `gear_generator_contour`*
- [x] **AST Extensions** — All four mode AST nodes defined
      *Implemented: `crates/hwc-parser/src/ast/shape.rs`*
- [x] **Parser Extensions** — `geometry:` block, `for` loops, `let` bindings, CSG expressions, generator calls
      *Implemented: `crates/hwc-parser/src/parser/definitions/shape.rs`*
- [x] **Shape Generators** — Star and gear contour generation
      *Implemented: `crates/hwc-compiler/src/shape_generators.rs`*
- [x] **Via Integration** — Shape point evaluation, parameter substitution, math evaluation
      *Implemented: `crates/hwc-compiler/src/auto_via_inserter/library.rs`*
- [x] **`evaluate_pure_math` Fix** — Parenthesized expression support
      *Implemented: resolves `-(width / 4)` from 0 to correct value*

### Test Coverage

| Test File | Mode | Geometry | Status |
|-----------|------|----------|--------|
| `shapes/circle_via.hw` | A | 8-point circle via (parameterized) | ✅ Passing |
| `shapes/diamond_via.hw` | A | 4-point diamond via (parameterized) | ✅ Passing |
| `shapes/triangle_via.hw` | A | 3-point triangle via (parameterized) | ✅ Passing |
| `shapes/octagon_via.hw` | A | 8-point octagon via (parameterized) | ✅ Passing |
| `shapes/star_via.hw` | D | 8-point star via (parameterized) | ✅ Passing |
| `individual/hex_mode_a.hw` | A | 6-point hexagon (manual points) | ✅ Passing |
| `individual/hex_mode_b.hw` | B | 6-point hexagon (for-loop) | ✅ Passing |
| `individual/hex_mode_c.hw` | C | Circle (CSG primitive) | ✅ Passing |
| `individual/hex_mode_d.hw` | D | Gear (procedural generator) | ✅ Passing |
| `mode_b_math/hex_v2.hw` | B | Hexagon using `DEG_TO_RAD` from stdlib | ✅ Passing |
| `mode_b_math/star_v2.hw` | B | 16-point star using `DEG_TO_RAD` from stdlib | ✅ Passing |

### Key Files

- **Spec**: `Docs/v0.1.7/SHAPE-SYSTEM-ARCHITECTURE.md` (this file)
- **AST**: `crates/hwc-parser/src/ast/shape.rs` (`CsgExpression`, `CsgPrimitive`, `ShapeGenerator`, `GeometryBlock`, `ShapePoint`)
- **Parser**: `crates/hwc-parser/src/parser/definitions/shape.rs` (geometry block, CSG, generators)
- **Generators**: `crates/hwc-compiler/src/shape_generators.rs` (`star_generator_contour`, `gear_generator_contour`)
- **Via Integration**: `crates/hwc-compiler/src/auto_via_inserter/library.rs` (`evaluate_shape_points`, `evaluate_geometry_blocks`, `evaluate_expr`)
- **Stdlib Math**: `stdlib/primitives/math.hw` (constants: `PI`, `DEG_TO_RAD`, `RAD_TO_DEG`, etc.)
- **Tests**: `tests/shapes/four_modes/individual/` (per-mode tests), `tests/shapes/four_modes/mode_b_math/` (stdlib integration tests)

---

## Section 1: The 4-Mode Paradigm

The shape system allows developers to define complex radial and polygonal geometries via four paradigms, all converging on the same mathematical output. Each mode is suited to a different design intent.

```
                    ┌──────────────────────────────────────┐
                    │      4-Mode Shape Definition         │
                    └──────────┬───────────────────────────┘
                               │
            ┌──────────┬───────┴───────┬──────────┐
            │          │               │          │
        ┌───▼───┐  ┌───▼───┐     ┌───▼───┐  ┌───▼───┐
        │Mode A │  │Mode B │     │Mode C │  │Mode D │
        │Manual │  │Para-  │     │CSG    │  │Proc-  │
        │Points │  │metric │     │Bool   │  │edural │
        │       │  │Loops  │     │       │  │Gen    │
        └───┬───┘  └───┬───┘     └───┬───┘  └───┬───┘
            │          │               │          │
            └──────────┴───────┬───────┴──────────┘
                               │
                    ┌──────────▼───────────────────────────┐
                    │   Mathematically Identical Output    │
                    │   32-vertex 16-point star geometry    │
                    └──────────────────────────────────────┘
```

All four modes produce **mathematically identical 32-vertex 16-point star geometry** — the same set of points, in the same order, regardless of the definition paradigm.

---

## Section 2: Mode Definitions

### Mode A — Manual Points

Explicit vertex coordinates declared in a `points:` block. Each point is a `- [x: expr, y: expr]` entry with expression strings for `x` and `y`.

```hardware
shape star_via(width: Measurement):
    points:
        - [x: 0, y: -(width / 2)]
        - [x: width / 4, y: -(width / 4)]
        - [x: width / 2, y: 0]
        - [x: width / 4, y: width / 4]
        - [x: 0, y: width / 2]
        - [x: -(width / 4), y: width / 4]
        - [x: -(width / 2), y: 0]
        - [x: -(width / 4), y: -(width / 4)]
```

**AST Representation:**

```rust
// crates/hwc-parser/src/ast/shape.rs

pub struct ShapePoint {
    pub x_expr: String,  // e.g., "0", "width / 4", "-(width / 2)"
    pub y_expr: String,  // e.g., "-(width / 2)", "0", "width / 4"
}
```

### Mode B — Parametric Loops

For-loop parameterized vertex generation. Variables are computed iteratively and `Point(x, y)` entries are emitted from within the loop body. Mode B benefits from the standard library math constants (`DEG_TO_RAD`, `PI`, etc.) to avoid manual unit conversion.

**Basic syntax (no stdlib):**

```hardware
shape circle_via(width: Measurement):
    geometry:
        for i in 0..8:
            let angle = (i * 45) * 3.14159265 / 180
            let r = width / 2
            Point(x: r * cos(angle), y: r * sin(angle))
```

**With stdlib math constants (recommended):**

```hardware
import * from @std/primitives/math

shape hexagon_via(width: Measurement):
    geometry:
        for i in 0..6:
            let angle = i * 60
            let rad = angle * DEG_TO_RAD
            Point(x: width / 2 * cos(rad), y: width / 2 * sin(rad))
```

**With custom constant library:**

```hardware
import * from my_constants

shape custom_via(width: Measurement):
    geometry:
        for i in 0..N:
            let angle = i * MY_STEP
            let rad = angle * MY_DEG_TO_RAD
            Point(x: r * cos(rad), y: r * sin(rad))
```

> **Note:** Custom constants declared in user files supersede stdlib constants of the same name. This allows projects to define domain-specific constants (e.g., `MY_DEG_TO_RAD`) that override the standard library values.

**AST Representation:**

```rust
GeometryBlock::ForLoop {
    variable: "i".to_string(),
    start: 0,
    end: 6,
    body: vec![
        GeometryStatement::Variable {
            name: "angle".to_string(),
            value: Expr::BinOp { op: BinOp::Mul, left: Expr::Identifier("i"), right: Expr::Literal(60.0) },
        },
        GeometryStatement::Variable {
            name: "rad".to_string(),
            value: Expr::BinOp { op: BinOp::Mul, left: Expr::Identifier("angle"), right: Expr::Identifier("DEG_TO_RAD") },
        },
        GeometryStatement::Point {
            x: Expr::BinOp { op: BinOp::Mul, left: ..., right: Expr::Call { name: "cos", args: [Expr::Identifier("rad")] } },
            y: Expr::BinOp { op: BinOp::Mul, left: ..., right: Expr::Call { name: "sin", args: [Expr::Identifier("rad")] } },
        },
    ],
}
```

### Mode C — CSG Boolean

Constructive Solid Geometry using `union`, `intersect`, and `subtract` operations on primitives. Clipper2 handles the boolean contour operations at evaluation time. CSG expressions go inside a `geometry:` block.

```hardware
shape cross_via(width: Measurement):
    geometry:
        Rectangle(width: width, height: width / 4) + Rectangle(width: width / 4, height: width)
```

**AST Representation:**

```rust
pub enum CsgPrimitive {
    Circle { diameter: String },
    Rectangle { width: String, height: String },
    ShapeRef(String),  // Reference to another shape definition
}

pub enum CsgExpression {
    Primitive(CsgPrimitive),
    Union(Box<CsgExpression>, Box<CsgExpression>),
    Difference(Box<CsgExpression>, Box<CsgExpression>),
    Intersection(Box<CsgExpression>, Box<CsgExpression>),
    Transformed { expr: Box<CsgExpression>, rotation: Option<f64>, translation: Option<(f64, f64)> },
    LetBinding { name: String, value: Box<CsgExpression>, body: Box<CsgExpression> },
}
```

**Supported Primitives:**

| Primitive | Parameters | Description |
|-----------|-----------|-------------|
| `circle(radius)` | `radius`: expression | Circle with given radius |
| `rectangle(width, height)` | `width`, `height`: expressions | Axis-aligned rectangle |
| `polygon(radius, vertices)` | `radius`: expression, `vertices`: integer | Regular polygon |

**Supported Operations:**

| Operation | Semantics |
|-----------|-----------|
| `union(A, B)` | Union of two contours (A ∪ B) |
| `intersect(A, B)` | Intersection of two contours (A ∩ B) |
| `subtract(A, B)` | Difference of two contours (A \ B) |

### Mode D — Procedural Generators

Calls to built-in generator functions that produce vertex contours algorithmically. Generator calls go inside a `geometry:` block.

```hardware
shape star_via(width: Measurement):
    geometry:
        StarGenerator(teeth: 16, outer: width / 2, inner: width / 4)
```

**AST Representation:**

```rust
pub struct ShapeGenerator {
    pub name: String,  // "StarGenerator" or "GearGenerator"
    pub params: FxHashMap<String, String>,
}
```

**Built-in Generators:**

| Generator | Output | Parameters |
|-----------|--------|------------|
| `StarGenerator(teeth, outer, inner)` | Star contour | `teeth`: point count, `outer`: outer radius, `inner`: inner radius |
| `GearGenerator(teeth, outer, inner)` | Gear contour | `teeth`: tooth count, `outer`: outer radius, `inner`: inner radius |

---

## Section 3: Parser Extensions

The parser in `crates/hwc-parser/src/parser/definitions/shape.rs` was extended to handle all four modes.

### 3.1 Geometry Block Parsing

The `geometry:` keyword introduces a block containing `GeometryStatement` entries:

```rust
// Parser entry point for geometry blocks
fn parse_geometry_block(input: &str) -> Result<GeometryBlock, ParseError> {
    // Parses:
    //   geometry:
    //       Point(x, y)
    //       for i in 0..N { ... }
    //       let name = value
    //       if condition { ... } else { ... }
}
```

### 3.2 For-Loop Parsing

```hardware
for i in 0..8:
    let angle = (i * 45) * DEG_TO_RAD
    let r = width / 2
    Point(x: r * cos(angle), y: r * sin(angle))
```

The parser resolves `variable`, `start`, `end` (exclusive range), and `body` (vector of nested `GeometryStatement`). The for-loop uses exclusive range (`0..6` = 0,1,2,3,4,5).

### 3.3 Let Binding Parsing

```hardware
let r = width / 2
```

Binds a name to an expression for later substitution. The expression is stored as an `Expr` AST node (not a string).

### 3.4 Point Expression Parsing

```hardware
Point(x: r * cos(angle), y: -(width / 4))
```

Both `x` and `y` are stored as `Expr` AST nodes. Evaluation occurs during via insertion via `evaluate_expr`, which walks the AST tree directly (no string re-parsing).

### 3.5 CSG Expression Parsing

```hardware
geometry:
    Circle(diameter: width) - Circle(diameter: width * 0.8)
```

CSG expressions go inside a `geometry:` block. The parser detects CSG by checking for `Rectangle`, `Circle`, or operators (`+`, `-`, `*`). Nested expressions are parsed into tree-structured `CsgExpression` nodes.

### 3.6 Procedural Generator Parsing

```hardware
geometry:
    GearGenerator(teeth: 12, outer: width / 2, inner: width * 0.35)
```

Generator calls go inside a `geometry:` block. The parser resolves the generator name and parameter expressions into a `ShapeGenerator` struct.

### 3.7 `check_identifier` Fix

The `check_identifier` function was updated to recognize reserved keywords that appear in geometry blocks:

```
Token::For, Token::In, Token::If, Token::Else, Token::Mod
```

Without this fix, `for`, `in`, `if`, `else`, and `mod` inside geometry blocks would be incorrectly treated as identifiers.

---

## Section 4: Standard Library Math Integration

Mode B (Parametric Loops) benefits from the standard library math constants defined in `stdlib/primitives/math.hw`. These constants are available globally when imported.

### 4.1 Available Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `PI` | 3.14159265358979323846 | Mathematical constant π |
| `E` | 2.71828182845904523536 | Mathematical constant e |
| `DEG_TO_RAD` | 0.017453292519943295 | Degrees to radians conversion factor |
| `RAD_TO_DEG` | 57.29577951308232 | Radians to degrees conversion factor |
| `SQRT2` | 1.4142135623730951 | Square root of 2 |
| `SQRT3` | 1.7320508075688772 | Square root of 3 |
| `TAU` | 6.283185307179586 | Full circle constant (2π) |

### 4.2 Usage

```hardware
import * from @std/primitives/math

shape hexagon_via(width: Measurement):
    geometry:
        for i in 0..6:
            let angle = i * 60
            let rad = angle * DEG_TO_RAD
            Point(x: width / 2 * cos(rad), y: width / 2 * sin(rad))
```

### 4.3 Custom Constant Libraries

Users can define their own constant libraries that supersede stdlib constants. Constants declared in user files take priority over stdlib constants of the same name.

**Create a custom constants file:**

```hardware
# my_constants.hw
const MY_PI: 3.14159
const MY_DEG_TO_RAD: 0.01745
const ViasPerRow: 8
```

**Use in shape definitions:**

```hardware
import * from my_constants

shape custom_via(width: Measurement):
    geometry:
        for i in 0..ViasPerRow:
            let angle = i * (360 / ViasPerRow)
            let rad = angle * MY_DEG_TO_RAD
            Point(x: width / 2 * cos(rad), y: width / 2 * sin(rad))
```

**Override stdlib constants:**

```hardware
# Override the stdlib PI with a custom value
const PI: 3.14159265358979323846

import * from @std/primitives/math  # PI from my_constants.hw takes priority
```

### 4.4 How Constants Flow Through the Compiler

1. **Parse time:** Constants from `stdlib/primitives/math.hw` are loaded into the Prelude layer
2. **User imports:** Constants from user files are loaded into HPM layers (higher priority)
3. **Symbol table:** `get_all_constants()` merges all layers (Core < Prelude < HPM < Local)
4. **Geometry evaluation:** `evaluate_geometry_blocks()` receives the merged constants map
5. **Expression evaluation:** `evaluate_expr()` resolves identifiers against params (including constants)

---

## Section 5: Shape Generators (Detailed)

The procedural generator implementations live in `crates/hwc-compiler/src/shape_generators.rs`.

### 4.1 Star Generator

```rust
pub fn star_generator_contour(
    teeth: u32,
    outer_radius: f64,
    inner_radius: f64,
) -> Vec<Point> {
    let mut points = Vec::new();
    let angle_step = std::f64::consts::PI / teeth as f64;

    for i in 0..(teeth * 2) {
        let angle = i as f64 * angle_step - std::f64::consts::FRAC_PI_2;
        let radius = if i % 2 == 0 { outer_radius } else { inner_radius };
        let x = radius * angle.cos();
        let y = radius * angle.sin();
        points.push(Point { x, y });
    }

    points
}
```

**Geometry:** Alternating outer/inner radii at evenly spaced angles, producing a star polygon.

```
              *  (outer_radius)
             / \
            /   \
    (inner) *---* (inner)
          /       \
         /         \
    (outer) *   * (outer)
         \         /
          \       /
    (inner) *---* (inner)
            \   /
             \ /
              *  (outer_radius)
```

### 4.2 Gear Generator

```rust
pub fn gear_generator_contour(
    teeth: u32,
    outer_radius: f64,
    inner_radius: f64,
) -> Vec<Point> {
    // Similar alternating structure, but with flat-topped teeth
    // Each tooth has two outer vertices (top-left, top-right)
    // and two inner vertices (valley-left, valley-right)
}
```

**Geometry:** Each tooth is a trapezoid — two outer points at the tooth tip, two inner points at the valley.

```
    ____      ____
   /    \    /    \
  /      \  /      \
 /        \/        \
|    tooth  |  tooth |
 \        /\        /
  \      /  \      /
   \____/    \____/
```

---

## Section 6: Via Integration (Auto-Via Inserter)

The auto-via inserter in `crates/hwc-compiler/src/auto_via_inserter/library.rs` resolves shapes from the symbol table and evaluates them into concrete point lists.

### 5.1 Evaluation Order

`evaluate_shape_points(shape_def, size_nm)` resolves shapes in the following priority order:

```
┌─────────────────────────────────┐
│  1. Check for CSG expression    │  ← CSG boolean operations
│     (union/intersect/subtract)  │
├─────────────────────────────────┤
│  2. Check for generator call    │  ← Procedural generators
│     (star/gear)                 │
├─────────────────────────────────┤
│  3. Check for geometry block    │  ← For-loops, let bindings,
│     (geometry: statements)      │     points in block
├─────────────────────────────────┤
│  4. Fall through to points:     │  ← Manual point lists
│     evaluation                  │
└─────────────────────────────────┘
```

### 5.2 Parameter Substitution

`evaluate_nm_expr(expr, params)` substitutes parameter values (e.g., `width`) into raw expression strings:

```
Input:  expr = "width / 2", params = { "width": 800 }
Output: "800 / 2"
```

### 5.3 Math Evaluation

`evaluate_pure_math(expr)` evaluates arithmetic expressions after parameter substitution.

**Critical Fix:** `evaluate_pure_math` was patched to handle **parenthesized expressions**. Without this fix, expressions like `-(width / 4)` collapsed to `0` because the negation operator was applied before the parenthesized division was evaluated.

```
Before fix:  -(800 / 4) → -0 / 4 → 0     ← WRONG
After fix:   -(800 / 4) → -(200) → -200   ← CORRECT
```

### 5.4 CSG Evaluation

CSG expressions are evaluated by generating contours from each primitive, then applying Clipper2 boolean operations:

```
Primitive A (circle) ──► contour_A
                              │
                         Clipper2 boolean
                              │
Primitive B (circle) ──► contour_B ──► result_contour
```

### 5.5 Geometry Block Evaluation

Geometry blocks are evaluated by iterating over statements:

1. **`ForLoop`** — Evaluate range, bind loop variable, evaluate body for each iteration
2. **`LetBinding`** — Bind name to evaluated expression in local scope
3. **`Point(x, y)`** — Evaluate `x_expr` and `y_expr` via `evaluate_nm_expr` + `evaluate_pure_math`
4. **`IfStatement`** — Evaluate condition, execute matching branch

### 5.6 Parameterized Via Shapes

All parameterized via shapes must use the `width` parameter to scale correctly to the via diameter. The `size_nm` argument passed to `evaluate_shape_points` provides the via diameter in nanometers.

```
size_nm = 800 (via diameter)
         │
         ▼
width = 800
         │
         ▼
All "width" references resolve to 800nm
```

---

## Section 7: Multi-Shape Stitching

Multiple shapes can be used in a single space via `add contact()` with different `shape:` parameters, enabling stitched copper chains.

### 6.1 Declaration

```hardware
space Main:
    add contact(Copper) at (1000, 1000):
        shape: shapes/star_via.hw

    add contact(Copper) at (1050, 1000):
        shape: shapes/circle_via.hw

    add contact(Copper) at (1100, 1000):
        shape: shapes/diamond_via.hw

    route net1:
        from: contact(Copper) at (1000, 1000)
        to: contact(Copper) at (1100, 1000)
        waypoints: (1050, 1000)
```

### 6.2 Collision Detection

Auto-via insertion detects collisions with custom contacts and skips duplicates:

```
Custom contact at (1000, 1000)  ← exists
         │
         ▼
Auto-via wants to insert at (1000, 1000)
         │
         ▼
Collision detected → skip insertion
```

### 6.3 Route Waypoint Stitching

Route waypoints can pass through custom contact positions to create stitched chains. The route engine recognizes that a waypoint coincides with an existing contact and treats the chain as a single continuous copper structure.

```
    ┌──────────┐     ┌──────────┐     ┌──────────┐
    │  Star    │─────│  Circle  │─────│ Diamond  │
    │  Via     │     │  Via     │     │  Via     │
    └──────────┘     └──────────┘     └──────────┘
     (1000, 1000)     (1050, 1000)     (1100, 1000)

    Route waypoints: (1000, 1000) → (1050, 1000) → (1100, 1000)
    Result: stitched copper chain across 3 custom shapes
```

---

## Section 8: Shape Import Convention

Each via shape is defined in a separate `.hw` file under the `shapes/` directory and imported by reference.

```
project/
├── shapes/
│   ├── circle_via.hw
│   ├── diamond_via.hw
│   ├── triangle_via.hw
│   ├── octagon_via.hw
│   └── star_via.hw
├── main.hw
└── ...
```

Import and usage:

```hardware
add contact(Copper) at (500, 500):
    shape: shapes/star_via.hw
```

The compiler resolves the path relative to the project root, loads the shape definition, and evaluates it at the via insertion point.

---

## Appendix A: Mathematical Equivalence Proof

All four modes converge on the same 32-vertex 16-point star:

| Mode | Definition Approach | Vertex Count |
|------|-------------------|--------------|
| A (Manual) | Explicit `Point(x, y)` × 32 | 32 |
| B (Parametric) | `for i in 0..32 { Point(...) }` | 32 |
| C (CSG) | `subtract(circle(400), star(8, 400, 200))` → contour | 32 |
| D (Generator) | `star_generator(teeth=16, outer=400, inner=200)` | 32 |

The star generator produces alternating outer/inner radii at 16 evenly spaced angles, yielding 16 outer vertices + 16 inner vertices = 32 total points — identical to manually specifying 32 `Point(x, y)` entries.

---

## Appendix B: Expression Evaluation Pipeline

```
Raw expression: "-(width / 4)"
         │
         ▼
Parameter substitution (evaluate_nm_expr):
    params = { "width": 800 }
    → "-(800 / 4)"
         │
         ▼
Math evaluation (evaluate_pure_math):
    → Parse parenthesized expression
    → Evaluate inner: 800 / 4 = 200
    → Apply negation: -(200) = -200
         │
         ▼
Result: -200 (nanometers)
```
