# HardwareScript v0.3.1: Ergonomic Physical Metaprogramming, Pure Cell Composition, Interface Bundles & Compile-Time Dimensional Algebra Specification

**Document Type:** Authoritative Core Architecture & Subsystem Specification  
**Target Version:** v0.3.1 (Production-Locked Standard)  
**Status:** Approved for Implementation  
**Target Crates:** `crates/hwc-frontend`, `crates/hwc-eval`, `crates/hwc-ir`  
**Reference Standards:** ISO/IEC 80000-1 (Quantities and Units), IEEE 1801 (UPF), OpenAccess Physical Cell Hierarchy Standard, Rust `uom` (Type-Level Dimensional Analysis)

---

## 1. Executive Summary & The Ergonomic Evolution

In HardwareScript v0.3.0, the compiler successfully eliminated AST tree-walking, fragile indentation parsing, and opaque heuristic solvers by introducing a linear Bytecode VM (`hwc-eval`) and deterministic picometer integer math. However, physical layout authoring retained procedural friction points:
1. **Ambient State Side-Effects:** PCell generators mutated the physical database directly via an implicit `space` context, preventing geometric transformation (rotation, mirroring) before placement.
2. **Procedural Parenthesis Hell:** Structs were pure C-style data bags requiring deeply nested free function calls (e.g., `bbox_union(bbox_from_rect(...), tap.bbox)`).
3. **Pin-by-Pin Wiring Boilerplate:** Multi-wire buses, power rails, and differential pairs required manual single-net declarations and point-to-point routing statements.
4. **Dimensionless Unit Degradation:** Compound physical quantities ($[L^2]$, $[R \cdot L]$, $[C / A]$) degraded to unverified floating-point numbers, allowing physical dimension mismatches to pass without compile-time errors.

HardwareScript v0.3.1 resolves these limitations through **Four Ergonomic Pillars** combining **Rust's compile-time invariants and zero-cost type systems** with **Ruby's expressive, human-centric block syntax**:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   THE 4 ERGONOMIC PILLARS OF HARDWARESCRIPT                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. PURE CELL LAYOUTS & FLUENT TRANSFORMS (Ruby Ergonomics + Rust Purity)   │
│     • PCells return self-contained `CellLayout` objects.                    │
│     • Eliminates ambient side-effects; enables fluent chaining:             │
│       `space.place(nmos.rotate(90deg).mirror_y(), at: [10.0um, 5.0um])`    │
│                                                                             │
│  2. STRUCT METHODS VIA `impl` BLOCKS (Rust Separation + Dot-Chaining)       │
│     • Attaches methods to structs without object-oriented class overhead.   │
│     • Replaces nested procedural calls with left-to-right chaining:         │
│       `body_bbox.union(tap.bbox).offset(100nm, 0nm)`                        │
│                                                                             │
│  3. STRUCTURAL INTERFACE BUNDLES & BUS ROUTING (1-Line Connectivity)        │
│     • Any struct containing `Net` handles acts as a first-class bundle.     │
│     • Router natively performs structural field matching:                   │
│       `route tx_diff to rx_diff { intent: Differential, spacing: 200nm }`   │
│                                                                             │
│  4. 7-BASE SI COMPILE-TIME DIMENSIONAL ALGEBRA (Zero-Cost Unit Verification)│
│     • Tracks dimension vectors $[L, M, T, I, \Theta, N, J]$ in the type     │
│       checker.                                                              │
│     • Verifies physical dimensional math at compile time ($L \times L = L^2$)│
│     • Catches unit typos (`min_area: 0.13um` 💥 vs `0.13um2` ✅) instantly. │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Pillar 1: Pure Cell Layouts & Fluent Geometric Transforms

### 2.1 The Problem with Ambient Mutation
When PCells invoke `space.add_polygon(...)` during execution, physical coordinates are baked immediately into the top-level space coordinate system. This prevents common semiconductor layout operations, such as:
* Mirroring a standard cell along the Y-axis to share a common $V_{\text{DD}}$ rail.
* Rotating an analog differential pair by $90^\circ$ or $180^\circ$ for cross-quad common-centroid layout.
* Calculating a component's post-rotation bounding box before committing it to the floorplan.

### 2.2 The `CellLayout` Container
In HardwareScript v0.3.1, a PCell is a **pure function** that returns a self-contained `CellLayout`. A `CellLayout` encapsulates local relative geometries, internal via pillars, device SPICE bindings, and named connection ports.

```
                           PURE CELL COMPOSITION PIPELINE
                           
  PCell Generator ──► Returns Isolated `CellLayout` (Origin at [0, 0])
                                  │
                                  ▼ Fluent Transforms (Pure Math)
                      `cell.rotate(90deg).mirror_y()`
                                  │
                                  ▼ Explicit Space Instantiation
          `space.place(transformed_cell, at: [10.0um, 5.0um])`
                                  │
                                  ▼ Picometer Offset & Merkle Identity Baking
         `FlatGeometryBuffer` Ingestion (`crates/hwc-ir`)
         (Target-agnostic picometer records; zero substrate assumptions)
```

> **Target-Agnostic Isolation:** A `CellLayout` emits purely relative, target-agnostic picometer polygons and contacts into `FlatGeometryBuffer`. It has zero knowledge of whether it will be synthesized into silicon masks or extruded into PCB copper slabs. Substrate realization is deferred entirely to the downstream backend (`crates/hwc-substrate-cmos`, `crates/hwc-substrate-laminate`).

### 2.3 Syntax & Usage in `.hw`

```hardware
# 1. Pure PCell Generator (Returns an isolated CellLayout)
fn sky130_nmos(W: Measurement, L: Measurement) -> CellLayout {
    let mut cell = CellLayout::new(name: "sky130_nmos")

    # Local geometry defined relative to origin [0, 0]
    cell.add_polygon(layer: "diff", rect: [-1.0um, -W/2, 2.0um, W])
    cell.add_polygon(layer: "poly", rect: [-L/2, -W/2 - 200nm, L, W + 400nm])
    cell.add_contact(from: "diff", to: "li1", at: [-500nm, 0nm], diameter: 170nm)

    # Expose local relative connection ports
    cell.add_port("S", at: [-500nm, 0nm], layer: "li1")
    cell.add_port("D", at: [ 500nm, 0nm], layer: "li1")
    cell.add_port("G", at: [   0nm, W/2 + 200nm], layer: "poly")

    return cell
}

# 2. In Space: Pure Geometric Transformation & Explicit Placement
space DifferentialPairSpace {
    dimensions: [50.0um, 30.0um]
    profile: SKY130_1V8_CMOS

    let m1 = sky130_nmos(W: 2.0um, L: 150nm)
    let m2 = sky130_nmos(W: 2.0um, L: 150nm)

    # Instantiate M1 normally, and M2 mirrored along Y to share a common source
    let inst_m1 = space.place(m1, at: [10.0um, 10.0um])
    let inst_m2 = space.place(m2.mirror_y(), at: [15.0um, 10.0um])

    # Connect to placed instance ports directly (World Coordinates are resolved automatically)
    route inst_m1.port("S") to inst_m2.port("S") { intent: Power }
}
```

---

## 3. Pillar 2: Struct Methods via `impl` Blocks

### 3.1 Eliminating Procedural Function Nesting
Legacy layout scripts relied on nested free functions that read backwards:
```hardware
# ❌ Legacy procedural syntax (Hard to read, deeply nested)
let final_box = bbox_offset(bbox_union(bbox_from_rect(pt, sz), tap.bbox), 100nm, 0nm)
```

HardwareScript v0.3.1 introduces Rust-style `impl` blocks. Methods take `self` as their first parameter, enabling left-to-right dot-notation method chaining without the memory overhead or dynamic dispatch cost of classical object-oriented classes.

### 3.2 Method Declaration & Chaining Syntax

```hardware
struct BoundingBox {
    min_x: Measurement,
    min_y: Measurement,
    max_x: Measurement,
    max_y: Measurement,
}

impl BoundingBox {
    fn new(min_x: Measurement, min_y: Measurement, max_x: Measurement, max_y: Measurement) -> BoundingBox {
        return BoundingBox { min_x: min_x, min_y: min_y, max_x: max_x, max_y: max_y }
    }

    fn width(self) -> Measurement {
        return self.max_x - self.min_x
    }

    fn height(self) -> Measurement {
        return self.max_y - self.min_y
    }

    fn union(self, other: BoundingBox) -> BoundingBox {
        return BoundingBox {
            min_x: min(self.min_x, other.min_x),
            min_y: min(self.min_y, other.min_y),
            max_x: max(self.max_x, other.max_x),
            max_y: max(self.max_y, other.max_y),
        }
    }

    fn offset(self, dx: Measurement, dy: Measurement) -> BoundingBox {
        return BoundingBox {
            min_x: self.min_x + dx,
            min_y: self.min_y + dy,
            max_x: self.max_x + dx,
            max_y: self.max_y + dy,
        }
    }
}

# ✅ Clean left-to-right method chaining:
let total_bbox = body_bbox.union(tap.bbox).offset(100nm, 0nm)
let total_area = total_bbox.width() * total_bbox.height()
```

---

## 4. Pillar 3: Structural Interface Bundles & Bus Routing

### 4.1 First-Class Hardware Bundles
Connecting wide data buses, differential analog channels, or standard power rails pin-by-pin introduces repetitive code.

In HardwareScript v0.3.1, **any struct whose fields are `Net` handles automatically qualifies as an Interface Bundle**.

```
                           STRUCTURAL BUNDLE ROUTING
                           
     ┌────────────────────────┐                ┌────────────────────────┐
     │  DiffPair tx           │                │  DiffPair rx           │
     │  ├── tx.p (Net "TX_P") ├═══════════════►│  ├── rx.p (Net "RX_P") │
     │  └── tx.n (Net "TX_N") ├═══════════════►│  └── rx.n (Net "RX_N") │
     └────────────────────────┘                └────────────────────────┘
                      `route tx to rx { intent: Differential }`
```

### 4.2 Field-Matching Semantic Rules
When `route Source to Target` executes:
1. The compiler inspects the field names and types of `Source` and `Target`.
2. Both structures must share identical field names of type `Net`.
3. The compiler lowers the single statement into coordinated physical routes while applying bundle-wide physical constraints (e.g., length matching, differential pair spacing, symmetric layer routing).

```hardware
struct DiffPair {
    p: Net,
    n: Net,
}

struct SpiMasterBus {
    sclk: Net,
    mosi: Net,
    miso: Net,
    cs_n: Net,
}

space TopSoC {
    dimensions: [100.0um, 100.0um]
    profile: SKY130_1V8_CMOS

    # Declare bundles cleanly
    let tx_channel = DiffPair { p: Net("TX_P"), n: Net("TX_N") }
    let rx_channel = DiffPair { p: Net("RX_P"), n: Net("RX_N") }

    # Route entire multi-wire bundle in one declarative statement
    route tx_channel to rx_channel {
        intent: Differential,
        spacing: 200nm,
        skew_tolerance: 50fs
    }
}
```

---

## 5. Pillar 4: Type-Level 7-Base SI Dimensional Algebra

### 5.1 The Mathematical Formulation
To eliminate unit mismatch bugs, HardwareScript implements compile-time dimensional checking based on the **7 Base SI Dimensions**:

$$\text{DimVector} = [L, M, T, I, \Theta, N, J]$$

Where:
* $L$: Length ($\text{meter}$)
* $M$: Mass ($\text{kilogram}$)
* $T$: Time ($\text{second}$)
* $I$: Electric Current ($\text{ampere}$)
* $\Theta$: Thermodynamic Temperature ($\text{kelvin}$)
* $N$: Amount of Substance ($\text{mole}$)
* $J$: Luminous Intensity ($\text{candela}$)

Every physical measurement carries a 7-exponent integer vector:

```
                            DIMENSIONAL EXPONENT ARITHMETIC
                            
  Multiplication (W * L):       [L=1, M=0, T=0, ...] + [L=1, M=0, T=0, ...] = [L=2] (Area)
  Division (L / W):             [L=1, M=0, T=0, ...] - [L=1, M=0, T=0, ...] = [L=0] (Float)
  Addition (Area + Length):     [L=2] != [L=1] ──► 💥 COMPILE ERROR: Dimension Mismatch
```

### 5.2 SI Dimensional Standard Library Registry

| Physical Quantity | SI Exponent Vector $[L, M, T, I, \Theta, N, J]$ | HardwareScript Literal Suffixes |
| :--- | :--- | :--- |
| **Length** | $[+1, 0, 0, 0, 0, 0, 0]$ | `pm`, `nm`, `um`, `mm`, `m`, `mil` |
| **Area** | $[+2, 0, 0, 0, 0, 0, 0]$ | `pm2`, `nm2`, `um2`, `mm2` |
| **Volume** | $[+3, 0, 0, 0, 0, 0, 0]$ | `um3`, `mm3` |
| **Time / Delay** | $[0, 0, +1, 0, 0, 0, 0]$ | `fs`, `ps`, `ns`, `us`, `ms`, `s` |
| **Current** | $[0, 0, 0, +1, 0, 0, 0]$ | `pA`, `nA`, `uA`, `mA`, `A` |
| **Voltage** | $[+2, +1, -3, -1, 0, 0, 0]$ | `nV`, `uV`, `mV`, `V`, `kV` |
| **Resistance** | $[+2, +1, -3, -2, 0, 0, 0]$ | `uOhm`, `mOhm`, `Ohm`, `kOhm`, `MOhm` |
| **Sheet Resistance** | $[0, +1, -3, -2, 0, 0, 0]$ (Resistance / Square) | `Ohm_sq`, `mOhm_sq` |
| **Capacitance** | $[-2, -1, +4, +2, 0, 0, 0]$ | `aF`, `fF`, `pF`, `nF`, `uF` |
| **Capacitance Density** | $[-4, -1, +4, +2, 0, 0, 0]$ (Capacitance / Area) | `fF_um2`, `aF_um2` |
| **Inductance** | $[+2, +1, -2, -2, 0, 0, 0]$ | `pH`, `nH`, `uH`, `mH`, `H` |

---

### 5.3 Rust Type System Implementation (`crates/hwc-compiler`)

```rust
// crates/hwc-compiler/src/eval/value.rs

/// 7-Base SI Dimensional Exponent Vector
#[repr(C)]
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct SiDimension {
    pub length: i8,      // L (meters)
    pub mass: i8,        // M (kilograms)
    pub time: i8,        // T (seconds)
    pub current: i8,     // I (amperes)
    pub temp: i8,        // Theta (kelvin)
    pub amount: i8,      // N (moles)
    pub luminosity: i8,  // J (candelas)
}

impl SiDimension {
    pub const DIMENSIONLESS: Self = Self { length: 0, mass: 0, time: 0, current: 0, temp: 0, amount: 0, luminosity: 0 };
    pub const LENGTH: Self        = Self { length: 1, mass: 0, time: 0, current: 0, temp: 0, amount: 0, luminosity: 0 };
    pub const AREA: Self          = Self { length: 2, mass: 0, time: 0, current: 0, temp: 0, amount: 0, luminosity: 0 };
    pub const RESISTANCE: Self    = Self { length: 2, mass: 1, time: -3, current: -2, temp: 0, amount: 0, luminosity: 0 };
    pub const SHEET_RES: Self     = Self { length: 0, mass: 1, time: -3, current: -2, temp: 0, amount: 0, luminosity: 0 };

    #[inline(always)]
    pub fn mul(self, rhs: Self) -> Self {
        Self {
            length: self.length + rhs.length,
            mass: self.mass + rhs.mass,
            time: self.time + rhs.time,
            current: self.current + rhs.current,
            temp: self.temp + rhs.temp,
            amount: self.amount + rhs.amount,
            luminosity: self.luminosity + rhs.luminosity,
        }
    }

    #[inline(always)]
    pub fn div(self, rhs: Self) -> Self {
        Self {
            length: self.length - rhs.length,
            mass: self.mass - rhs.mass,
            time: self.time - rhs.time,
            current: self.current - rhs.current,
            temp: self.temp - rhs.temp,
            amount: self.amount - rhs.amount,
            luminosity: self.luminosity - rhs.luminosity,
        }
    }
}

/// 128-bit Fixed-Point Dimensional Measurement Representation
#[repr(C)]
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct MeasurementValue {
    /// Scaled integer value in internal base picometer/nanovolt/picoampere units
    pub raw: i128,
    pub dimension: SiDimension,
}
```

---

## 6. End-to-End Concrete Example: Differential Amplifier (`diff_amp.hw`)

This complete, canonical v0.3.1 implementation demonstrates all four pillars working together:

```hardware
# diff_amp.hw - HardwareScript v0.3.1 Canonical Implementation
import * from @std/primitives/units
import { sky130_nmos, sky130_pmos } from @std/pdk/sky130
import { SemiconductorSubstrate } from @std/substrates

# ── 1. Static Nominal Substrate Contract ─────────────────────────────────────

export profile SKY130_1V8_CMOS implements SemiconductorSubstrate {
    manufacturing_grid: 5nm
    lambda: 50nm

    masks {
        nwell:  { gds_layer: 64, datatype: 20 }
        diff:   { gds_layer: 65, datatype: 20 }
        tap:    { gds_layer: 65, datatype: 44 }
        nsdm:   { gds_layer: 93, datatype: 44 }
        psdm:   { gds_layer: 94, datatype: 20 }
        poly:   { gds_layer: 66, datatype: 20 }
        licon:  { gds_layer: 66, datatype: 44 }
        li1:    { gds_layer: 67, datatype: 20 }
        mcon:   { gds_layer: 67, datatype: 44 }
        metal1: { gds_layer: 68, datatype: 20 }
    }

    routing {
        topology: "manhattan"
        grid: 5nm
        preferred_directions {
            li1:    "horizontal"
            metal1: "vertical"
        }
    }

    drc {
        antenna_ratio_max: 400.0
        min_density metal1: 20%
    }
}

# ── 2. Structural Interface Bundles ──────────────────────────────────────────

struct DiffPair {
    p: Net,
    n: Net,
}

struct PowerRail {
    vdd: Net,
    vss: Net,
}

# ── 2. Reusable Math Abstractions with Struct Methods ─────────────────────────

struct LayoutBox {
    bbox: BoundingBox,
}

impl LayoutBox {
    fn from_rect(center: Point2D, width: Measurement, height: Measurement) -> LayoutBox {
        return LayoutBox {
            bbox: BoundingBox::new(
                center.x - width/2,
                center.y - height/2,
                center.x + width/2,
                center.y + height/2
            )
        }
    }

    fn area(self) -> Measurement {
        # Compile-time dimensional multiplication: Length * Length = Area
        return (self.bbox.max_x - self.bbox.min_x) * (self.bbox.max_y - self.bbox.min_y)
    }
}

# ── 3. Top-Level Physical Space ───────────────────────────────────────────────

space OperationalDifferentialPair {
    dimensions: [40.0um, 30.0um]
    profile: SKY130_1V8_CMOS

    nets {
        VDD:   { classification: power,  potential: 1.8V, current: 50.0uA }
        VSS:   { classification: ground, potential: 0.0V, current: 50.0uA }
        IN_P:  { classification: signal, potential: 0.9V }
        IN_N:  { classification: signal, potential: 0.9V }
        OUT_P: { classification: signal }
        OUT_N: { classification: signal }
        TAIL:  { classification: signal }
    }

    # Initialize interface bundles
    let in_diff  = DiffPair { p: IN_P,  n: IN_N }
    let out_diff = DiffPair { p: OUT_P, n: OUT_N }
    let rails    = PowerRail { vdd: VDD, vss: VSS }

    # 4. Pure PCell Instantiation & Fluent Geometric Placement
    let nmos_pcell = sky130_nmos(W: 3.0um, L: 150nm)

    # Place M1 and M2 in common-centroid symmetric orientation
    let m1 = space.place(nmos_pcell, at: [12.0um, 15.0um])
    let m2 = space.place(nmos_pcell.mirror_y(), at: [28.0um, 15.0um])

    # 5. Connect Placed Cell Ports
    route m1.port("S") to m2.port("S") { intent: Signal }
    route m1.port("S") to TAIL         { intent: Signal }

    # 6. One-Line Interface Bundle Routing
    route in_diff to DiffPair { p: m1.port("G"), n: m2.port("G") } {
        intent: Differential,
        spacing: 200nm,
        shield: rails.vss
    }
}
```

---

## 7. Formal EBNF Grammar Deltas

```ebnf
(* Added Struct Implementation Blocks *)
ImplDecl ::= "impl" Identifier "{" ( FunctionDecl )* "}"

(* Added Cell Transformations and Method Chaining *)
PostfixExpr ::= PrimaryExpr ( CallSuffix 
                            | MethodCallSuffix
                            | FieldAccessSuffix 
                            | IndexSuffix )*

MethodCallSuffix ::= "." Identifier "(" ( ExpressionList )? ")"

(* Added Compound Measurement Literals *)
MeasurementLiteral ::= ( IntegerLiteral | FloatLiteral ) CompoundUnitSuffix

CompoundUnitSuffix ::= LengthUnit
                     | AreaUnit
                     | ResistanceUnit
                     | SheetResistanceUnit
                     | CapacitanceDensityUnit

AreaUnit ::= "pm2" | "nm2" | "um2" | "mm2" | "m2"
SheetResistanceUnit ::= "Ohm_sq" | "mOhm_sq" | "kOhm_sq"
CapacitanceDensityUnit ::= "fF_um2" | "aF_um2" | "pC_um2"
```

---

## 8. Compiler Execution & Diagnostic Verification

```text
🔥 hwc COMPILER v0.3.1 (Ergonomic Synthesis & Physical Verification)
================================================================================
[    0.38ms] Source parsed: diff_amp.hw (1 space, 2 structs, 1 impl block)
[    0.62ms] Type Checker: 7-Base SI Dimensional verification active
   • Verified: (W: Length) * (L: Length) ──► Area [L^2]
   • Verified: 0 dimensional mismatches detected
[    0.88ms] Lowering pure PCell 'sky130_nmos': 1 CellLayout generated (0 global mutations)
[    1.15ms] Space 'OperationalDifferentialPair': Placed 2 instances with transform matrices:
   • M1: Identity [dx: 12.0um, dy: 15.0um]
   • M2: MirrorY  [dx: 28.0um, dy: 15.0um]
[    1.42ms] Expanding structural route: DiffPair (2 paired nets)
   • Channel P: IN_P ──► M1.G
   • Channel N: IN_N ──► M2.G (Differential clearance: 200nm enforced)
[   18.40ms] PIVB topological manifold check: 1 Connected Component (Clean)
   ✅ GLB: build/OperationalDifferentialPair/board.glb (9.8ms)
   ✅ SPICE: build/OperationalDifferentialPair/circuit.sp (3.2ms)
    Finished build in 0.042s
```

---

*Approved by the HardwareScript Core Architecture Team — September 2026*