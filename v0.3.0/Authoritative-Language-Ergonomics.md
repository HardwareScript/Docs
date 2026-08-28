# HardwareScript v0.3.0: Authoritative Language Ergonomics & Add-On Specification

**Document Type:** Authoritative Specification & Compiler Extension Roadmap  
**Target Version:** v0.3.0
**Status:** Approved for Implementation  
**Focus:** Complete Ergonomic Add-On Manifest, Parser Grammar Delta, Bytecode VM Opcodes, Emitter Upgrades, and the Bloat Prevention Contract  

---

## 1. The Core Philosophy: The Comptime Boundary Contract

HardwareScript is a **Compile-Time Generative Hardware Description Language (Comptime HDL)**. The compilation pipeline operates on a single immutable law:

$$\text{User Source } (.hw) \xrightarrow{\text{Comptime VM (hwc-eval)}} \text{EntityGraph (Flat Picometers)} \xrightarrow{\text{Synthesis Engine}} \text{GDSII / SPICE / GLB}$$

The evaluation engine inside `hwc-compiler` (`hwc-eval`) exists solely to compute spatial geometry, evaluate parameterized cell (PCell) mathematics, and emit verified physical polygons. 

To eliminate repetitive refactoring cycles and prevent future language churn, this specification establishes:
1. **The Complete Ergonomic Add-On Manifest:** Every language feature required for industrial-grade PDK and PCell development, specified for a single implementation pass.
2. **The Forbidden Zone Contract:** Hard architectural boundaries prohibiting general-purpose software bloat to preserve sub-30ms compilation and deterministic silicon builds.

---

## 2. The Complete Ergonomic Add-On Manifest

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    COMPLETE COMPTIME LANGUAGE ADD-ONS                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. EXPRESSION-ORIENTED CONTROL FLOW                                        │
│     • `if` / `else` as expressions (`let x = if cond { a } else { b }`)     │
│     • `match` as expressions returning values                               │
│     • Block expressions `{ ...; value }` with implicit tail returns         │
│                                                                             │
│  2. LOOP CONTROL & COMPOUND OPERATORS                                       │
│     • `break` and `continue` inside `for` loops                             │
│     • Compound assignments: `+=`, `-=`, `*=`, `/=`, `%=`                   │
│                                                                             │
│  3. COLLECTION & STRING PRIMITIVES                                          │
│     • Array methods: `.len()`, `.push(item)`, `.pop()`, `.is_empty()`       │
│     • Array slice syntax: `arr[start..end]`                                 │
│     • String primitives: `.len()`, `.is_empty()`, universal `{}` format     │
│                                                                             │
│  4. FUNCTION & RETURN ERGONOMICS                                            │
│     • Tuple / Multi-Value Destructuring (`let (rows, cols) = get_grid()`)   │
│     • Named parameter call sites (`pad(name: "P1", at: [0um, 0um])`)        │
│     • Default parameter values in function declarations                     │
│     • Early `return` with optional expression                               │
│                                                                             │
│  5. EXPLICIT UNIT & NUMERIC CONVERSIONS                                     │
│     • Unit scalar extractors: `.to_float()`, `.to_int()`, `.to_nm()`, etc.  │
│     • Numeric cast builtins: `int(float_val)`, `float(int_val)`             │
│                                                                             │
│  6. EMITTER IDENTITY & SPATIAL METADATA                                     │
│     • Named polygon & contact emitters (`space.add_polygon(name: ...)`)     │
│     • Geometric queries: `bbox.width()`, `bbox.height()`, `bbox.center()`   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 2.1 Expression-Oriented Control Flow

#### A. `if` / `else` as Expressions
An `if` statement with an `else` branch can be used anywhere an `Expression` is expected. Both branches must evaluate to the same compatible type.

```hardware
# Clean conditional assignment without dummy mutable variables
let c_name = if name != "" and count > 1 {
    "{name}_{count}"
} else if name != "" {
    name
} else {
    ""
}

let trace_w = if is_wide_driver { 2.0um } else { 300nm }
```

#### B. `match` as Expressions
Pattern matching resolves directly to a value:

```hardware
let implant_layer = match tap_type {
    TapType.P_Sub => "psdm",
    TapType.N_Well => "nsdm",
    _ => "tap"
}
```

#### C. Block Expressions
Blocks `{ ... }` evaluate to their trailing expression if no semicolon terminates the final statement:

```hardware
let dynamic_pitch = {
    let base = SKY130_RULES.licon_size + SKY130_RULES.licon_spacing
    let margin = 30nm
    base + margin  # Implicit tail return (No semicolon)
}
```

---

### 2.2 Loop Controls & Compound Assignment

#### A. `break` and `continue`
Essential for boundary searches, track snapping, and obstacle-avoidance heuristics in comptime generators:

```hardware
for i in 0..max_tracks {
    let offset = i * track_pitch
    if is_obstructed(offset) {
        continue
    }
    if offset > target_boundary {
        break
    }
    place_track(offset)
}
```

#### B. Compound Assignment Operators
Supported operators: `+=`, `-=`, `*=`, `/=`, `%=`.

```hardware
let mut count = 0
let mut current_x = at.x

for w in widths {
    current_x += w + gap
    count += 1
}
```

---

### 2.3 Collection & String Operations

#### A. Built-in Array Methods
The compiler provides core methods on `Value::Array`:

| Method | Signature | Description |
| :--- | :--- | :--- |
| `.len()` | `arr.len() -> Int` | Returns number of elements |
| `.push()` | `arr.push(val: T) -> Void` | Appends element to mutable array |
| `.pop()` | `arr.pop() -> T?` | Removes and returns last element |
| `.is_empty()` | `arr.is_empty() -> Bool` | Returns `true` if `len == 0` |

```hardware
let mut ports = []
for i in 0..num_vias {
    ports.push(TransistorPort { center: [x, y], layer: "metal1", net: net })
}
let total = ports.len()
```

#### B. Array Slicing Syntax
Half-open slicing returns a new array reference:

```hardware
let all_pins = get_bus_pins()
let low_byte = all_pins[0..8]    # Pins 0 through 7
let high_byte = all_pins[8..16]  # Pins 8 through 15
```

#### C. String Built-ins
| Method / Property | Signature | Description |
| :--- | :--- | :--- |
| `.len()` | `str.len() -> Int` | Returns byte length |
| `.is_empty()` | `str.is_empty() -> Bool` | Returns `true` if `str == ""` |
| `{expr}` | `"Text {expr}"` | Inline compile-time string interpolation |

---

### 2.4 Function & Return Ergonomics

#### A. Tuple / Multi-Value Destructuring
Functions can return tuples `(T1, T2, ...)` that unpack cleanly into `let` bindings:

```hardware
fn calculate_grid_dims(box_w: Measurement, box_h: Measurement, pitch: Measurement) -> (Int, Int) {
    let cols = max(1, box_w / pitch)
    let rows = max(1, box_h / pitch)
    return (cols, rows)
}

# Call-site destructuring:
let (cols, rows) = calculate_grid_dims(10.0um, 5.0um, 400nm)
```

#### B. Named Arguments at Call Sites
Any function or PCell call can use named arguments for self-documenting layout declarations:

```hardware
let nmos = sky130_nmos(
    name: "M1",
    W: 2.0um,
    L: 150nm,
    at: [10.0um, 5.0um],
    source: VSS,
    drain: Out,
    gate: In,
    bulk: VSS
)
```

#### C. Early `return`
Functions can exit early using `return` or `return <expr>`:

```hardware
fn validate_drc(w: Measurement, l: Measurement) -> Bool {
    if l < 150nm {
        eprintln("Gate length below foundry minimum!")
        return false
    }
    return true
}
```

---

### 2.5 Explicit Unit & Numeric Conversions

HardwareScript enforces rigid dimensional arithmetic ($L \pm L \rightarrow L$, $L \times L \rightarrow \text{Area}$, $V/I \rightarrow R$). When dimension-free math or unit-scaling is necessary, explicit conversion methods prevent silent precision loss:

| Conversion Method | Input Type | Output Type | Description |
| :--- | :--- | :--- | :--- |
| `meas.to_float()` | `Measurement` | `Float` | Scales canonical picometer/nanovolt value to base SI float (e.g. `1.5um.to_float()` $\rightarrow$ `1.5e-6`) |
| `meas.to_int()` | `Measurement` | `Int` | Returns raw 128-bit internal picometer/picoamp integer |
| `meas.to_pm()` | `Measurement (Length)` | `Int` | Returns integer picometers ($10^{-12}\text{ m}$) |
| `meas.to_nm()` | `Measurement (Length)` | `Float` | Returns fractional nanometers ($10^{-9}\text{ m}$) |
| `meas.to_um()` | `Measurement (Length)` | `Float` | Returns fractional micrometers ($10^{-6}\text{ m}$) |
| `int(x)` | `Float / String` | `Int` | Truncates float to integer |
| `float(x)` | `Int / String` | `Float` | Converts integer to float |

```hardware
# Safe unit conversions in layout math:
let count = int(width / 400nm)
let ratio = (L / W).to_float()
let r_val = ratio * 350.0  # Ohms
```

---

### 2.6 Emitter Identity & Spatial Metadata

To preserve semantic naming in BOM, SPICE, and DXF exports (fixing anonymous items like `polyres_0`), all `space.add_*` emitters accept optional `name:` identifiers.

```hardware
# Emitter API Upgrades:
space.add_polygon(
    name: "Resistor_Body",
    layer: "polyres",
    net: term_a,
    rect: [x, y, w, h]
)

space.add_contact(
    name: "Via_A_0",
    from: "polyres",
    to: "li1",
    at: [vx, vy],
    diameter: 170nm,
    net: net
)
```

#### Bounding Box Geometric Methods (`BoundingBox`)
All `BoundingBox` structs provide instant geometry queries:

```hardware
let bbox = BoundingBox { min_x: 0um, min_y: 0um, max_x: 10um, max_y: 4um }

let width = bbox.width()        # 10.0um
let height = bbox.height()      # 4.0um
let center = bbox.center()      # Point2D { x: 5.0um, y: 2.0um }
let overlaps = bbox_intersects(bbox_a, bbox_b) # Bool
let merged = bbox_union(bbox_a, bbox_b)        # BoundingBox
```

---

## 3. The Forbidden Zone: What HardwareScript Must NEVER Add

To ensure compile times remain under **30 ms**, memory usage remains under **5 MB**, and layout synthesis remains 100% deterministic, the following language features are permanently excluded:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         THE FORBIDDEN BLOAT MANIFEST                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ❌ 1. NO BORROW CHECKER OR LIFETIMES                                       │
│     All comptime values are either small `Copy` primitives (Int, Float,     │
│     Measurement, Point2D) or reference-counted `Arc` / Arena buffers.       │
│                                                                             │
│  ❌ 2. NO USER-LAND MULTI-THREADING OR ASYNC / AWAIT                        │
│     The compiler parallelizes execution internally via Rayon and DOPHR     │
│     Spatial 4-Coloring. User scripts are purely synchronous and sandboxed.  │
│                                                                             │
│  ❌ 3. NO OOP CLASSES, INHERITANCE, OR VIRTUAL DISPATCH                    │
│     Hardware structure is modeled purely through algebraic `struct`, `enum`, │
│     `match`, and functions.                                                 │
│                                                                             │
│  ❌ 4. NO RUNTIME EXCEPTIONS OR TRY / CATCH                                │
│     Errors halt execution immediately at compile time with exact source     │
│     spans and error codes (e.g., `Error C01`, `Error S22`).                 │
│                                                                             │
│  ❌ 5. NO ARBITRARY OPERATOR OVERLOADING                                    │
│     Operators (`+`, `-`, `*`, `/`) are reserved for numbers and rigid       │
│     physical unit algebra ($L \pm L \rightarrow L$, $V/I \rightarrow R$).   │
│                                                                             │
│  ❌ 6. NO C-STYLE PREPROCESSOR MACROS                                       │
│     Comptime evaluation (`fn`, `for`, `if`) *is* the macro system.          │
│                                                                             │
│  ❌ 7. NO IMPLICIT TYPE WIDENING OR COERCION OF INCOMPATIBLE UNITS          │
│     Adding `10um + 1.8V` is always a hard compile error (`Error S22`).      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Formal Grammar & AST Delta (EBNF)

Below is the exact grammar specification for the additions:

### 4.1 Statements & Control Flow

```ebnf
Statement ::= LetStmt
            | AssignmentStmt
            | IfStmt
            | ForStmt
            | BreakStmt
            | ContinueStmt
            | ReturnStmt
            | AssertStmt
            | ExpressionStmt

BreakStmt ::= "break" ( ";" )?
ContinueStmt ::= "continue" ( ";" )?
ReturnStmt ::= "return" ( Expression )? ( ";" )?

AssignmentStmt ::= TargetExpr AssignmentOp Expression ( ";" )?
AssignmentOp   ::= "=" | "+=" | "-=" | "*=" | "/=" | "%="

LetStmt ::= "let" ( "mut" )? BindingPattern ( ":" TypeExpr )? "=" Expression ( ";" )?
BindingPattern ::= Identifier
                 | "(" Identifier ( "," Identifier )* ( "," )? ")"
```

### 4.2 Expressions

```ebnf
Expression ::= IfExpr
             | MatchExpr
             | BlockExpr
             | LogicalOrExpr

IfExpr ::= "if" Expression Block ( "else" ( IfExpr | Block ) )?

BlockExpr ::= "{" ( Statement )* ( Expression )? "}"

MatchExpr ::= "match" Expression "{" ( MatchArm )* "}"
MatchArm  ::= MatchPattern "=>" ( Expression | Block ) ( "," )?
MatchPattern ::= EnumPattern | Identifier | "_"

PostfixExpr ::= PrimaryExpr ( CallSuffix 
                            | FieldAccessSuffix 
                            | IndexSuffix 
                            | SliceSuffix )*

SliceSuffix ::= "[" ( Expression )? ".." ( Expression )? "]"
```

---

## 5. Bytecode Instruction Set Architecture (ISA) Updates

To execute these additions on the linear register-based VM (`hwc-eval`), the following opcodes are added to `crates/hwc-compiler/src/eval/bytecode.rs`:

```rust
#[derive(Debug, Clone, PartialEq)]
pub enum OpCode {
    // ── Existing ISA (LoadConst, Move, Add, Sub, Mul, Div, Jump, Call, Return)... ──

    // ── NEW: Compound & Local Register Arithmetic ──
    AddAssign { dst: Register, src: Register },
    SubAssign { dst: Register, src: Register },
    MulAssign { dst: Register, src: Register },
    DivAssign { dst: Register, src: Register },

    // ── NEW: Control Flow Jumps ──
    JumpForward { offset: JumpOffset },
    JumpBack { offset: JumpOffset },

    // ── NEW: Array & Collection Operations ──
    ArrayPush { array_reg: Register, val_reg: Register },
    ArrayPop { dst: Register, array_reg: Register },
    ArrayLen { dst: Register, array_reg: Register },
    ArraySlice { dst: Register, array_reg: Register, start_reg: Register, end_reg: Register },

    // ── NEW: Tuple & Destructuring Ops ──
    AllocTuple { dst: Register, start_reg: Register, count: u8 },
    UnpackTuple { dst_start: Register, tuple_reg: Register, count: u8 },

    // ── NEW: Unit Conversion Ops ──
    MeasToFloat { dst: Register, src: Register },
    MeasToInt { dst: Register, src: Register },
}
```

---

## 6. End-to-End Concrete Implementation Example

Here is a production-grade standard library helper module (`@std/layout/via.hw`) utilizing the complete ergonomic specification:

```hardware
# stdlib/layout/via.hw
import * from @std/primitives/units
import * from @std/primitives/math

export fn via_matrix(
    name: String = "",
    from_layer: String, 
    to_layer: String, 
    rows: Int, 
    cols: Int, 
    pitch: Measurement, 
    diameter: Measurement, 
    at: Point2D, 
    net: Net
) -> Array[Point2D] {
    let offset_x = (cols - 1) * pitch / 2
    let offset_y = (rows - 1) * pitch / 2
    
    let mut count = 0
    let mut placed_centers = []

    for r in 0..rows {
        for c in 0..cols {
            let vx = at.x - offset_x + (c * pitch)
            let vy = at.y - offset_y + (r * pitch)
            let pt = Point2D { x: vx, y: vy }

            # Expression-based conditional naming
            let contact_name = if name != "" and (rows * cols > 1) {
                "{name}_{count}"
            } else if name != "" {
                name
            } else {
                ""
            }

            space.add_contact(
                name: contact_name,
                from: from_layer,
                to: to_layer,
                at: pt,
                diameter: diameter,
                net: net
            )

            placed_centers.push(pt)
            count += 1
        }
    }

    return placed_centers
}

export fn fill_vias_in_box(
    from_layer: String,
    to_layer: String,
    box_w: Measurement,
    box_h: Measurement,
    diameter: Measurement,
    min_spacing: Measurement,
    enclosure: Measurement,
    at: Point2D,
    net: Net
) -> (Int, Int) {
    let usable_w = box_w - (2 * enclosure)
    let usable_h = box_h - (2 * enclosure)
    let pitch = diameter + min_spacing

    let cols = max(1, int(usable_w / pitch))
    let rows = max(1, int(usable_h / pitch))

    via_matrix(
        from_layer: from_layer,
        to_layer: to_layer,
        rows: rows,
        cols: cols,
        pitch: pitch,
        diameter: diameter,
        at: at,
        net: net
    )

    # Clean tuple return destructuring
    return (rows, cols)
}
```

---

## 7. Implementation Checklist for Engineers

- [ ] **Parser Phase (`hwc-parser`)**:
  - [ ] Add `IfExpr` production to Pratt expression parser in `expression.rs`.
  - [ ] Add compound assignment tokens `+=`, `-=`, `*=`, `/=`, `%=` to `token_types.rs`.
  - [ ] Support tuple destructuring patterns `let (a, b) = ...` in `statements.rs`.
  - [ ] Add `break` and `continue` tokens and AST nodes in `statement.rs`.
  - [ ] Support array slicing `arr[start..end]` in postfix expression parsing.

- [ ] **Bytecode Compiler Phase (`hwc-compiler/src/eval/compiler.rs`)**:
  - [ ] Lower `IfExpr` to `OpCode::JumpIfFalse` / `OpCode::Jump`.
  - [ ] Lower `+=`, `-=`, `*=`, `/=` to in-place register opcodes.
  - [ ] Lower `break` and `continue` jumps to the loop header/exit blocks.
  - [ ] Lower tuple unpacking to `OpCode::UnpackTuple`.

- [ ] **VM Runtime Phase (`hwc-compiler/src/eval/context.rs`)**:
  - [ ] Implement `ArrayPush`, `ArrayPop`, `ArrayLen`, `ArraySlice` handlers.
  - [ ] Implement `MeasToFloat` and `MeasToInt` dimensional extractors.
  - [ ] Update `SpaceEmitter` trait to accept `name: &str` across `emit_polygon` and `emit_contact`.

- [ ] **Standard Library Migration (`stdlib/`)**:
  - [ ] Refactor `@std/layout/via.hw` and `@std/layout/placement.hw` with expression-based `if` and `+=`.
  - [ ] Re-run tapeout validation test on `cmos_inverter.hw` and `simple_resistor_test.hw`.