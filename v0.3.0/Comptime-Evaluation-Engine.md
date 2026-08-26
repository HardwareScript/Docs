# `hwc-eval` Compile-Time Evaluation Engine: Architecture, Bytecode VM & Deprecation Plan

---

## 1. The Total Purge Manifest: Dead Code & Legacy Removal

HardwareScript v0.3.0 shifts physical synthesis from a **declarative markup DSL solved by heuristic solvers** to a **Generative Compile-Time HDL (`comptime`) executing on a Linear Bytecode VM**. 

To prevent architectural drift and technical debt, all legacy AST-mutating passes, heuristic solvers, and voxel artifacts must be deleted from the workspace.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       DEPRECATION & PURGE MANIFEST                          │
├─────────────────────────────────────────────────────────────────────────────┤
│ ❌ PURGE FROM `hwc-compiler`:                                               │
│   ├── src/ir/compilation/placement_loop.rs   (90 lines of boxed AST trees) │
│   ├── src/ir/relational_resolver.rs          (Opaque heuristic solver)      │
│   ├── src/ir/routing/boundary_sorting.rs     (Replaced by DOPHR Stage 2)    │
│   ├── src/ir/routing/macro_fusion.rs         (Legacy heuristic)             │
│   ├── src/auto_via_inserter/                 (Replaced by @std/layout/via)  │
│   ├── src/ir/parametric_unroller/            (Replaced by Bytecode loops)   │
│   └── src/ir/errors.rs                       (Monolithic 856-line enum)     │
│                                                                             │
│ ❌ PURGE FROM `hwc-engine`:                                                 │
│   ├── src/geometry_router/shift_boundary_port.rs                            │
│   ├── src/geometry_router/pathfinding/sdf_router.rs                         │
│   ├── src/voxel_grid/                        (Entire legacy voxel system)   │
│   └── src/geometry_router/heuristic.rs                                      │
│                                                                             │
│ ❌ PURGE SYNTACTIC ANTI-PATTERNS (User & PDK Space):                       │
│   ├── `origin:`, `resolution:`, and 3D Z-dimensions in `space` blocks       │
│   ├── `absolute:` keyword block wrapper                                     │
│   ├── 1nm `Air` dummy pours for SPICE virtual terminals                     │
│   ├── Quoted material symbols (`symbol: "Poly"`)                            │
│   └── 35-line manual transistor pour boilerplate                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Architectural Replacement Matrix

| Legacy Subsystem / Hack | Failure Mode in v0.1–v0.2 | v0.3.0 Canonical Replacement |
| :--- | :--- | :--- |
| **AST Tree Re-Boxing** (`placement_loop.rs`) | Synthesized fake `Expr::Binary(Box::new(...))` to recalculate bounds; $O(N^2)$ memory explosion. | **Bytecode VM registers & activation records**: Values compute directly in fixed-point picometers. |
| **Relational Guessing Solver** (`relational_resolver.rs`) | Guessed coordinate offsets using heuristics; impossible to debug when missing by 20nm. | **Pure comptime math**: `@std/layout/placement.hw` (`stack_horizontal`, `stack_vertical`). |
| **Heuristic Transistor Wiring** | Dummy $1\text{nm}$ `Air` pours to bind SPICE substrate nodes. | **Direct compact model contract**: `space.add_device()` in `@std/pdk/sky130/nmos.hw`. |
| **Voxel Grid & Slower A\*** | $O(N^3)$ memory blowup, staircased traces, phantom vias. | **DOPHR 3-Stage Guided Routing**: 3D DoD capacity tensor + continuous track anchors. |

---

## 2. `hwc-eval` Engine Architecture

The compile-time evaluation engine compiles parsed, arena-allocated immutable AST expressions into a **flat, linear bytecode instruction stream (`Chunk`)** and executes them within an isolated, deterministic **Virtual Machine (`VM`)** using static activation records (`StackFrame`).

```
 ┌─────────────────────────────────────────────────────────────────────────┐
 │                           AST PARSER (hwc-parser)                       │
 │  Immutable Arena-Allocated AST (Program, FunctionDecl, SpaceDecl, Expr) │
 └────────────────────────────────────┬────────────────────────────────────┘
                                      │
                                      ▼
 ┌─────────────────────────────────────────────────────────────────────────┐
 │                   TYPE CHECKER & COMPTIME COMPILER                      │
 │  • Type inference & strict dimensional algebra validation               │
 │  • Array[Measurement, 2] ──► Point2D automatic coercion pass            │
 │  • Emits linear Instruction stream into `Chunk`                         │
 └────────────────────────────────────┬────────────────────────────────────┘
                                      │ Chunk (Bytecode)
                                      ▼
 ┌─────────────────────────────────────────────────────────────────────────┐
 │                      BYTECODE VM (hwc-compiler/src/eval)                │
 │                                                                         │
 │  ┌─────────────────────────┐             ┌───────────────────────────┐  │
 │  │ Fixed Activation Stack  │             │ Step Counter & Sandbox    │  │
 │  │ • Vec<Value> registers  │             │ • 10,000,000 step limit   │  │
 │  │ • CallFrame stack (256) │             │ • Zero OS clock/IO leaks  │  │
 │  │ • ScopeFrame hierarchy  │             │ • 100% bit-deterministic  │  │
 │  └────────────┬────────────┘             └─────────────┬─────────────┘  │
 │               │                                        │                │
 │               └────────────────────┬───────────────────┘                │
 │                                    │                                    │
 │                                    ▼                                    │
 │                    Linear Instruction Dispatch Loop                     │
 │                     (OP_ADD, OP_CALL, OP_EMIT_POLY...)                  │
 └────────────────────────────────────┬────────────────────────────────────┘
                                      │
                   Direct Native SpaceEmitter Ingestion
                                      │
                                      ▼
 ┌─────────────────────────────────────────────────────────────────────────┐
 │                 CANONICAL DATABASE (hwc-engine::EntityGraph)            │
 │  Receives flat picometer polygons, contacts, devices & route intents     │
 └─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. The Unified `Value` Model & 128-Bit Picometer Arithmetic

### 3.1 Type Definition (`src/eval/value.rs`)

```rust
use compact_str::CompactString;
use std::sync::Arc;

/// Canonical base units for dimensional checking
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum UnitDimension {
    Length,          // pm (picometers, 10^-12 m)
    Voltage,         // nV (nanovolts, 10^-9 V)
    Current,         // pA (picoamperes, 10^-12 A)
    Resistance,      // uOhm (micro-ohms, 10^-6 Ohm)
    Capacitance,     // aF (attofarads, 10^-18 F)
    Inductance,      // pH (picohenries, 10^-12 H)
    Time,            // fs (femtoseconds, 10^-15 s)
    Frequency,       // Hz
    Power,           // pW (picowatts, 10^-12 W)
    Angle,           // micro-degrees (10^-6 deg)
    Temperature,     // mK (millikelvin)
    Conductivity,    // S/m
    Resistivity,     // ohm-m
    Area,            // pm^2
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct MeasurementValue {
    /// 128-bit signed integer value scaled to the dimension's canonical internal unit
    pub raw: i128,
    pub dimension: UnitDimension,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct NetId(pub u32);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct DeviceId(pub u32);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct SpaceId(pub u32);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct FunctionId(pub u32);

#[derive(Debug, Clone, PartialEq)]
pub enum Value {
    // ── Primitive Literals ──
    Void,
    Bool(bool),
    Int(i64),
    Float(f64),
    String(CompactString),

    // ── Physical & Geometric Primitives (Evaluated in Picometers) ──
    Measurement(MeasurementValue),
    Point2D { x: i64, y: i64 },              // Coordinates in picometers
    Point3D { x: i64, y: i64, z: i64 },      // Coordinates in picometers
    Vector2D { dx: i64, dy: i64 },
    BoundingBox { min_x: i64, min_y: i64, max_x: i64, max_y: i64 },

    // ── Composites & References ──
    Array(Arc<Vec<Value>>),
    StructInstance {
        name: CompactString,
        fields: Arc<Vec<(CompactString, Value)>>,
    },
    EnumVariant {
        enum_name: CompactString,
        variant_name: CompactString,
        payload: Option<Arc<Vec<Value>>>,
    },
    FunctionRef(FunctionId),

    // ── Hardware Domain Handles ──
    NetHandle(NetId),
    SpaceHandle(SpaceId),
    DeviceHandle(DeviceId),
}
```

### 3.2 Strict 128-Bit Dimensional Algebra Specification

Every physical measurement is stored in an absolute 128-bit integer scaled to canonical base units ($1\text{ pm} = 10^{-12}\text{ m}$, $1\text{ nV} = 10^{-9}\text{ V}$, $1\text{ pA} = 10^{-12}\text{ A}$, $1\,\mu\Omega = 10^{-6}\,\Omega$).

The evaluator enforces physical dimensional algebra at runtime:

```rust
impl MeasurementValue {
    pub fn add(self, rhs: Self) -> Result<Self, EvalError> {
        if self.dimension != rhs.dimension {
            return Err(EvalError::UnitMismatch {
                expected: self.dimension,
                found: rhs.dimension,
                op: "+",
            });
        }
        Ok(Self {
            raw: self.raw + rhs.raw,
            dimension: self.dimension,
        })
    }

    pub fn sub(self, rhs: Self) -> Result<Self, EvalError> {
        if self.dimension != rhs.dimension {
            return Err(EvalError::UnitMismatch {
                expected: self.dimension,
                found: rhs.dimension,
                op: "-",
            });
        }
        Ok(Self {
            raw: self.raw - rhs.raw,
            dimension: self.dimension,
        })
    }

    pub fn mul_scalar(self, scalar: f64) -> Self {
        Self {
            raw: (self.raw as f64 * scalar) as i128,
            dimension: self.dimension,
        }
    }

    pub fn mul_measurement(self, rhs: Self) -> Result<Value, EvalError> {
        match (self.dimension, rhs.dimension) {
            // Length * Length -> Area (pm^2)
            (UnitDimension::Length, UnitDimension::Length) => Ok(Value::Measurement(Self {
                raw: self.raw * rhs.raw,
                dimension: UnitDimension::Area,
            })),
            // Voltage * Current -> Power
            // (1 nV * 1 pA = 10^-9 V * 10^-12 A = 10^-21 W = 10^-9 pW)
            (UnitDimension::Voltage, UnitDimension::Current)
            | (UnitDimension::Current, UnitDimension::Voltage) => Ok(Value::Measurement(Self {
                raw: (self.raw * rhs.raw) / 1_000_000_000,
                dimension: UnitDimension::Power,
            })),
            // Current * Resistance -> Voltage
            // (1 pA * 1 uOhm = 10^-12 A * 10^-6 Ohm = 10^-18 V = 10^-9 nV)
            (UnitDimension::Current, UnitDimension::Resistance)
            | (UnitDimension::Resistance, UnitDimension::Current) => Ok(Value::Measurement(Self {
                raw: (self.raw * rhs.raw) / 1_000_000_000,
                dimension: UnitDimension::Voltage,
            })),
            _ => Err(EvalError::InvalidDimensionalMultiplication(self.dimension, rhs.dimension)),
        }
    }

    pub fn div_measurement(self, rhs: Self) -> Result<Value, EvalError> {
        if rhs.raw == 0 {
            return Err(EvalError::DivisionByZero);
        }
        if self.dimension == rhs.dimension {
            // Dimensionless ratio
            return Ok(Value::Float(self.raw as f64 / rhs.raw as f64));
        }
        match (self.dimension, rhs.dimension) {
            // Voltage / Current -> Resistance
            (UnitDimension::Voltage, UnitDimension::Current) => Ok(Value::Measurement(Self {
                raw: (self.raw * 1_000_000_000) / rhs.raw,
                dimension: UnitDimension::Resistance,
            })),
            // Area / Length -> Length
            (UnitDimension::Area, UnitDimension::Length) => Ok(Value::Measurement(Self {
                raw: self.raw / rhs.raw,
                dimension: UnitDimension::Length,
            })),
            _ => Err(EvalError::InvalidDimensionalDivision(self.dimension, rhs.dimension)),
        }
    }
}
```

### 3.3 Array-to-`Point2D` Coercion Algorithm

Whenever a function call or struct instantiation targets a `Point2D` type, the VM checks the argument:

```rust
impl Value {
    pub fn coerce_to_point2d(&self) -> Result<Value, EvalError> {
        match self {
            Value::Point2D { .. } => Ok(self.clone()),
            Value::Array(items) => {
                if items.len() != 2 {
                    return Err(EvalError::CoercionFailed {
                        expected: "Point2D",
                        found: format!("Array of length {}", items.len()),
                        hint: "Point2D array literal coercion requires exactly [x: Measurement, y: Measurement]",
                    });
                }
                match (&items[0], &items[1]) {
                    (
                        Value::Measurement(MeasurementValue { raw: x, dimension: UnitDimension::Length }),
                        Value::Measurement(MeasurementValue { raw: y, dimension: UnitDimension::Length }),
                    ) => Ok(Value::Point2D {
                        x: *x as i64,
                        y: *y as i64,
                    }),
                    (a, b) => Err(EvalError::CoercionFailed {
                        expected: "Point2D (both Length measurements)",
                        found: format!("[{:?}, {:?}]", a, b),
                        hint: "Array elements must both be Length measurements (e.g., [10.0um, 5.0um])",
                    }),
                }
            }
            other => Err(EvalError::TypeMismatch {
                expected: "Point2D or [Measurement, Measurement]",
                found: format!("{:?}", other),
            }),
        }
    }
}
```

---

## 4. Bytecode Instruction Set Architecture (ISA)

The VM uses a stack-register hybrid instruction format. Local variables and temporary expression results reside on the contiguous `Vec<Value>` activation stack.

```rust
// crates/hwc-compiler/src/eval/bytecode.rs

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Register(pub u16);

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct ConstantIndex(pub u16);

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct JumpOffset(pub i32);

#[derive(Debug, Clone, PartialEq)]
pub enum OpCode {
    // ── Constants & Stack Management ──
    LoadConst { dst: Register, const_idx: ConstantIndex },
    Move { dst: Register, src: Register },
    LoadNull { dst: Register },
    LoadBool { dst: Register, val: bool },
    LoadInt { dst: Register, val: i64 },

    // ── Arithmetic & Dimensional Math ──
    Add { dst: Register, lhs: Register, rhs: Register },
    Sub { dst: Register, lhs: Register, rhs: Register },
    Mul { dst: Register, lhs: Register, rhs: Register },
    Div { dst: Register, lhs: Register, rhs: Register },
    Mod { dst: Register, lhs: Register, rhs: Register },
    Neg { dst: Register, src: Register },

    // ── Comparison & Logic ──
    Eq { dst: Register, lhs: Register, rhs: Register },
    Ne { dst: Register, lhs: Register, rhs: Register },
    Lt { dst: Register, lhs: Register, rhs: Register },
    Le { dst: Register, lhs: Register, rhs: Register },
    Gt { dst: Register, lhs: Register, rhs: Register },
    Ge { dst: Register, lhs: Register, rhs: Register },
    Not { dst: Register, src: Register },

    // ── Control Flow & Loops ──
    Jump { offset: JumpOffset },
    JumpIfTrue { cond: Register, offset: JumpOffset },
    JumpIfFalse { cond: Register, offset: JumpOffset },
    LoopStep { iter_reg: Register, end_reg: Register, step_val: i64, offset: JumpOffset },

    // ── Functions & Call Stack ──
    Call { func: Register, args_start: Register, arg_count: u8, dst: Register },
    Return { val: Register },

    // ── Data Structures & Coercion ──
    AllocArray { dst: Register, start_reg: Register, count: u16 },
    AllocStruct { dst: Register, struct_name_idx: ConstantIndex, fields_start: Register, count: u16 },
    GetField { dst: Register, obj: Register, field_idx: ConstantIndex },
    SetField { obj: Register, field_idx: ConstantIndex, src: Register },
    GetIndex { dst: Register, obj: Register, index: Register },
    CoercePoint2D { dst: Register, src: Register },

    // ── Native Emitter Operations (`space.*`) ──
    EmitPolygon { layer_reg: Register, net_reg: Register, points_or_rect_reg: Register },
    EmitContact { from_layer_reg: Register, to_layer_reg: Register, at_reg: Register, dia_reg: Register, net_reg: Register },
    EmitDevice { type_idx: ConstantIndex, name_reg: Register, terminals_reg: Register, params_reg: Register },
    EmitRoute { from_reg: Register, to_reg: Register, intent_idx: ConstantIndex },

    // ── Built-in Functions & Diagnostics ──
    Assert { cond: Register, msg_idx: ConstantIndex },
    BuiltinCall { builtin_id: u8, args_start: Register, arg_count: u8, dst: Register },
}

#[derive(Debug, Clone, Default)]
pub struct Chunk {
    pub code: Vec<OpCode>,
    pub constants: Vec<Value>,
    pub spans: Vec<miette::SourceSpan>,
}
```

---

## 5. VM Execution Engine & Activation Stack

### 5.1 CallFrame & Stack Architecture (`src/eval/context.rs`)

```rust
// crates/hwc-compiler/src/eval/context.rs

use super::bytecode::{Chunk, OpCode, Register};
use super::value::{SpaceId, Value};
use miette::Diagnostic;
use thiserror::Error;

const MAX_CALL_STACK: usize = 256;
const MAX_STEP_LIMIT: u64 = 10_000_000;

#[derive(Debug, Clone)]
pub struct CallFrame {
    pub chunk: std::sync::Arc<Chunk>,
    pub ip: usize,
    pub stack_base: usize,
    pub return_register: Option<Register>,
}

pub struct VM<'a> {
    pub stack: Vec<Value>,
    pub frames: Vec<CallFrame>,
    pub step_count: u64,
    pub current_space_id: Option<SpaceId>,
    pub emitter: &'a mut dyn super::emitter::SpaceEmitter,
}

#[derive(Error, Diagnostic, Debug)]
pub enum EvalError {
    #[error("Step limit exceeded: Execution surpassed 10,000,000 comptime operations")]
    #[diagnostic(code(C01), help("Check for infinite loops or non-terminating recursion in layout generator."))]
    StepLimitExceeded,

    #[error("Recursion depth limit exceeded (max 256 stack frames)")]
    #[diagnostic(code(C02), help("Comptime generators cannot recurse deeper than 256 frames."))]
    RecursionDepthExceeded,

    #[error("Cannot perform 'space.{method}()' outside of an active space block")]
    #[diagnostic(code(S30), help("Physical geometry emitters must be invoked inside 'space SpaceName implements ... {{ }}'."))]
    NoActiveSpaceContext { method: &'static str },

    #[error("Type mismatch: expected {expected}, found {found}")]
    #[diagnostic(code(S22))]
    TypeMismatch { expected: &'static str, found: String },

    #[error("Dimensional unit mismatch in operation '{op}': cannot combine {expected:?} with {found:?}")]
    #[diagnostic(code(S22))]
    UnitMismatch { expected: super::value::UnitDimension, found: super::value::UnitDimension, op: &'static str },

    #[error("Assertion failed: {message}")]
    #[diagnostic(code(C03))]
    AssertionFailed { message: String },

    #[error("Coercion failed: expected {expected}, found {found}. {hint}")]
    #[diagnostic(code(S22))]
    CoercionFailed { expected: &'static str, found: String, hint: &'static str },

    #[error("Invalid dimensional multiplication: {0:?} * {1:?}")]
    #[diagnostic(code(S22))]
    InvalidDimensionalMultiplication(super::value::UnitDimension, super::value::UnitDimension),

    #[error("Invalid dimensional division: {0:?} / {1:?}")]
    #[diagnostic(code(S22))]
    InvalidDimensionalDivision(super::value::UnitDimension, super::value::UnitDimension),

    #[error("Division by zero in compile-time evaluation")]
    #[diagnostic(code(S23))]
    DivisionByZero,
}
```

### 5.2 The VM Dispatch Loop (`src/eval/mod.rs`)

```rust
impl<'a> VM<'a> {
    pub fn run(&mut self) -> Result<Value, EvalError> {
        while !self.frames.is_empty() {
            // 1. Hermetic Sandbox Step Check
            self.step_count += 1;
            if self.step_count > MAX_STEP_LIMIT {
                return Err(EvalError::StepLimitExceeded);
            }

            let frame_idx = self.frames.len() - 1;
            let frame = &mut self.frames[frame_idx];

            if frame.ip >= frame.chunk.code.len() {
                self.frames.pop();
                continue;
            }

            let op = frame.chunk.code[frame.ip].clone();
            frame.ip += 1;
            let base = frame.stack_base;

            match op {
                OpCode::LoadConst { dst, const_idx } => {
                    let val = frame.chunk.constants[const_idx.0 as usize].clone();
                    self.stack[base + dst.0 as usize] = val;
                }

                OpCode::LoadInt { dst, val } => {
                    self.stack[base + dst.0 as usize] = Value::Int(val);
                }

                OpCode::LoadBool { dst, val } => {
                    self.stack[base + dst.0 as usize] = Value::Bool(val);
                }

                OpCode::Move { dst, src } => {
                    self.stack[base + dst.0 as usize] = self.stack[base + src.0 as usize].clone();
                }

                OpCode::Add { dst, lhs, rhs } => {
                    let l = &self.stack[base + lhs.0 as usize];
                    let r = &self.stack[base + rhs.0 as usize];
                    let res = match (l, r) {
                        (Value::Int(a), Value::Int(b)) => Value::Int(a + b),
                        (Value::Float(a), Value::Float(b)) => Value::Float(a + b),
                        (Value::Measurement(a), Value::Measurement(b)) => Value::Measurement(a.add(*b)?),
                        _ => return Err(EvalError::TypeMismatch { expected: "Addable types", found: format!("{:?} + {:?}", l, r) }),
                    };
                    self.stack[base + dst.0 as usize] = res;
                }

                OpCode::Sub { dst, lhs, rhs } => {
                    let l = &self.stack[base + lhs.0 as usize];
                    let r = &self.stack[base + rhs.0 as usize];
                    let res = match (l, r) {
                        (Value::Int(a), Value::Int(b)) => Value::Int(a - b),
                        (Value::Float(a), Value::Float(b)) => Value::Float(a - b),
                        (Value::Measurement(a), Value::Measurement(b)) => Value::Measurement(a.sub(*b)?),
                        _ => return Err(EvalError::TypeMismatch { expected: "Subtractable types", found: format!("{:?} - {:?}", l, r) }),
                    };
                    self.stack[base + dst.0 as usize] = res;
                }

                OpCode::Mul { dst, lhs, rhs } => {
                    let l = &self.stack[base + lhs.0 as usize];
                    let r = &self.stack[base + rhs.0 as usize];
                    let res = match (l, r) {
                        (Value::Int(a), Value::Int(b)) => Value::Int(a * b),
                        (Value::Float(a), Value::Float(b)) => Value::Float(a * b),
                        (Value::Measurement(m), Value::Int(n)) | (Value::Int(n), Value::Measurement(m)) => {
                            Value::Measurement(m.mul_scalar(*n as f64))
                        }
                        (Value::Measurement(m), Value::Float(n)) | (Value::Float(n), Value::Measurement(m)) => {
                            Value::Measurement(m.mul_scalar(*n))
                        }
                        (Value::Measurement(a), Value::Measurement(b)) => a.mul_measurement(*b)?,
                        _ => return Err(EvalError::TypeMismatch { expected: "Multipliable types", found: format!("{:?} * {:?}", l, r) }),
                    };
                    self.stack[base + dst.0 as usize] = res;
                }

                OpCode::Div { dst, lhs, rhs } => {
                    let l = &self.stack[base + lhs.0 as usize];
                    let r = &self.stack[base + rhs.0 as usize];
                    let res = match (l, r) {
                        (Value::Int(a), Value::Int(b)) => {
                            if *b == 0 { return Err(EvalError::DivisionByZero); }
                            Value::Int(a / b)
                        }
                        (Value::Float(a), Value::Float(b)) => {
                            if *b == 0.0 { return Err(EvalError::DivisionByZero); }
                            Value::Float(a / b)
                        }
                        (Value::Measurement(m), Value::Int(n)) => {
                            if *n == 0 { return Err(EvalError::DivisionByZero); }
                            Value::Measurement(m.mul_scalar(1.0 / *n as f64))
                        }
                        (Value::Measurement(m), Value::Float(n)) => {
                            if *n == 0.0 { return Err(EvalError::DivisionByZero); }
                            Value::Measurement(m.mul_scalar(1.0 / *n))
                        }
                        (Value::Measurement(a), Value::Measurement(b)) => a.div_measurement(*b)?,
                        _ => return Err(EvalError::TypeMismatch { expected: "Dividable types", found: format!("{:?} / {:?}", l, r) }),
                    };
                    self.stack[base + dst.0 as usize] = res;
                }

                OpCode::CoercePoint2D { dst, src } => {
                    let val = &self.stack[base + src.0 as usize];
                    let coerced = val.coerce_to_point2d()?;
                    self.stack[base + dst.0 as usize] = coerced;
                }

                OpCode::Jump { offset } => {
                    let frame = &mut self.frames[frame_idx];
                    frame.ip = (frame.ip as i32 + offset.0) as usize;
                }

                OpCode::JumpIfFalse { cond, offset } => {
                    let c = match &self.stack[base + cond.0 as usize] {
                        Value::Bool(b) => *b,
                        other => return Err(EvalError::TypeMismatch { expected: "Bool", found: format!("{:?}", other) }),
                    };
                    if !c {
                        let frame = &mut self.frames[frame_idx];
                        frame.ip = (frame.ip as i32 + offset.0) as usize;
                    }
                }

                OpCode::Call { func, args_start, arg_count, dst } => {
                    if self.frames.len() >= MAX_CALL_STACK {
                        return Err(EvalError::RecursionDepthExceeded);
                    }
                    let func_val = &self.stack[base + func.0 as usize];
                    match func_val {
                        Value::FunctionRef(fn_id) => {
                            let (chunk, _) = self.emitter.lookup_function(*fn_id)?;
                            let new_base = self.stack.len();
                            // Reserve stack space for callee
                            for i in 0..arg_count {
                                let arg_val = self.stack[base + args_start.0 as usize + i as usize].clone();
                                self.stack.push(arg_val);
                            }
                            // Grow local variable registers
                            self.stack.resize(new_base + 64, Value::Void);

                            self.frames.push(CallFrame {
                                chunk,
                                ip: 0,
                                stack_base: new_base,
                                return_register: Some(dst),
                            });
                        }
                        other => return Err(EvalError::TypeMismatch { expected: "Function", found: format!("{:?}", other) }),
                    }
                }

                OpCode::Return { val } => {
                    let return_val = self.stack[base + val.0 as usize].clone();
                    let popped = self.frames.pop().unwrap();
                    self.stack.truncate(popped.stack_base);
                    if let Some(parent_frame) = self.frames.last() {
                        if let Some(ret_reg) = popped.return_register {
                            self.stack[parent_frame.stack_base + ret_reg.0 as usize] = return_val;
                        }
                    } else {
                        return Ok(return_val);
                    }
                }

                // ── Native Emitter Hooks into `EntityGraph` ──
                OpCode::EmitPolygon { layer_reg, net_reg, points_or_rect_reg } => {
                    let space_id = self.current_space_id.ok_or(EvalError::NoActiveSpaceContext { method: "add_polygon" })?;
                    let layer = self.stack[base + layer_reg.0 as usize].as_string()?;
                    let net = self.stack[base + net_reg.0 as usize].as_net_handle()?;
                    let geometry = &self.stack[base + points_or_rect_reg.0 as usize];

                    self.emitter.emit_polygon(space_id, layer, net, geometry)?;
                }

                OpCode::EmitContact { from_layer_reg, to_layer_reg, at_reg, dia_reg, net_reg } => {
                    let space_id = self.current_space_id.ok_or(EvalError::NoActiveSpaceContext { method: "add_contact" })?;
                    let from_layer = self.stack[base + from_layer_reg.0 as usize].as_string()?;
                    let to_layer = self.stack[base + to_layer_reg.0 as usize].as_string()?;
                    let at = self.stack[base + at_reg.0 as usize].coerce_to_point2d()?;
                    let dia = self.stack[base + dia_reg.0 as usize].as_measurement_raw()?;
                    let net = self.stack[base + net_reg.0 as usize].as_net_handle()?;

                    self.emitter.emit_contact(space_id, from_layer, to_layer, at, dia, net)?;
                }

                OpCode::EmitDevice { type_idx, name_reg, terminals_reg, params_reg } => {
                    let space_id = self.current_space_id.ok_or(EvalError::NoActiveSpaceContext { method: "add_device" })?;
                    let dev_type = frame.chunk.constants[type_idx.0 as usize].as_string()?;
                    let name = self.stack[base + name_reg.0 as usize].as_string()?;
                    let terminals = &self.stack[base + terminals_reg.0 as usize];
                    let params = &self.stack[base + params_reg.0 as usize];

                    self.emitter.emit_device(space_id, dev_type, name, terminals, params)?;
                }

                OpCode::Assert { cond, msg_idx } => {
                    let c = match &self.stack[base + cond.0 as usize] {
                        Value::Bool(b) => *b,
                        other => return Err(EvalError::TypeMismatch { expected: "Bool", found: format!("{:?}", other) }),
                    };
                    if !c {
                        let msg = frame.chunk.constants[msg_idx.0 as usize].as_string()?;
                        return Err(EvalError::AssertionFailed { message: msg.to_string() });
                    }
                }

                OpCode::BuiltinCall { builtin_id, args_start, arg_count, dst } => {
                    let args = &self.stack[base + args_start.0 as usize..base + args_start.0 as usize + arg_count as usize];
                    let res = super::builtins::dispatch_builtin(builtin_id, args)?;
                    self.stack[base + dst.0 as usize] = res;
                }

                _ => unimplemented!("OpCode handler"),
            }
        }

        Ok(Value::Void)
    }
}
```

---

## 6. Native Geometry & Database Emitter Bridge

The `SpaceEmitter` trait establishes the boundary between the compile-time evaluator and the physical database.

```rust
// crates/hwc-compiler/src/eval/emitter.rs

use super::value::{FunctionId, NetId, SpaceId, Value};
use super::EvalError;
use std::sync::Arc;

pub trait SpaceEmitter {
    fn lookup_function(&self, id: FunctionId) -> Result<(Arc<super::bytecode::Chunk>, usize), EvalError>;
    
    fn emit_polygon(
        &mut self,
        space_id: SpaceId,
        layer: &str,
        net: NetId,
        geometry: &Value,
    ) -> Result<(), EvalError>;

    fn emit_contact(
        &mut self,
        space_id: SpaceId,
        from_layer: &str,
        to_layer: &str,
        at: Value,
        diameter_pm: i128,
        net: NetId,
    ) -> Result<(), EvalError>;

    fn emit_device(
        &mut self,
        space_id: SpaceId,
        device_type: &str,
        instance_name: &str,
        terminals: &Value,
        params: &Value,
    ) -> Result<(), EvalError>;

    fn emit_route(
        &mut self,
        space_id: SpaceId,
        from_port: &Value,
        to_port: &Value,
        intent: &str,
    ) -> Result<(), EvalError>;
}
```

When `space.add_polygon` runs:
1. Coordinates are extracted directly as 64-bit integer picometers ($10^{-12}\text{ m}$).
2. If given a `rect: [x, y, w, h]` array, it converts to 4 orthogonal bounding points.
3. The primitive inserts directly into `hwc-engine::EntityGraph` without intermediary AST syntax trees or heap string reparsing.

---

## 7. Standard Built-in Functions & Math Library

`src/eval/builtins.rs` implements all hardware-domain mathematical utilities:

| Built-in Function ID | Signature | Comptime Behavior |
| :--- | :--- | :--- |
| `0x01` (`println`) | `println(fmt: String, ...args)` | Formats interpolated string and writes to compiler stdout. |
| `0x02` (`eprintln`) | `eprintln(fmt: String, ...args)` | Formats warning and writes to compiler stderr. |
| `0x03` (`dbg`) | `dbg(val: T) -> T` | Dumps value with line number to stdout, returns unmodified value. |
| `0x04` (`assert`) | `assert(cond: Bool, msg: String)` | Validates condition; halts evaluation with `Error C03` if false. |
| `0x05` (`min`) | `min(a: T, b: T) -> T` | Returns smaller of two numbers or measurements. |
| `0x06` (`max`) | `max(a: T, b: T) -> T` | Returns larger of two numbers or measurements. |
| `0x07` (`abs`) | `abs(a: T) -> T` | Computes absolute magnitude. |
| `0x08` (`sqrt`) | `sqrt(val: Float) -> Float` | Computes scalar square root. |
| `0x09` (`sin`) | `sin(angle: Measurement) -> Float` | Trigonometric sine from `deg` or `rad` measurement. |
| `0x0A` (`cos`) | `cos(angle: Measurement) -> Float` | Trigonometric cosine from `deg` or `rad` measurement. |
| `0x0B` (`tan`) | `tan(angle: Measurement) -> Float` | Trigonometric tangent from `deg` or `rad` measurement. |
| `0x0C` (`rect_between`) | `rect_between(p1: Point2D, p2: Point2D, w: Measurement) -> Array[Point2D]` | Generates a 4-point bounding polygon connecting two ports. |

---

## 8. End-to-End Execution Trace: Evaluating `sky130_nmos`

To verify how the bytecode VM completely bypasses legacy AST re-boxing, trace a single invocation from `cmos_inverter.hw`:

```hardware
let nmos = sky130_nmos(name: "M_N", W: 1.0um, L: 150nm, at: [10.0um, 5.0um], source: VSS, drain: Out, gate: In, bulk: VSS)
```

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    COMPTIME BYTECODE EXECUTION TRACE                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│ 1. CALL DISPATCH                                                            │
│    • VM loads `FunctionRef(FnId: sky130_nmos)` into R0                      │
│    • Pushes arguments to Stack[Base..Base+7]:                               │
│      - R1: "M_N"                                                            │
│      - R2: Measurement(1_000_000_000_000, Length)   # 1.0um                 │
│      - R3: Measurement(150_000_000_000, Length)     # 150nm                 │
│      - R4: Array([10.0um, 5.0um]) ──► Auto-coerced to Point2D(10um, 5um)   │
│      - R5: NetHandle(NetId: VSS)                                            │
│      - R6: NetHandle(NetId: Out)                                            │
│      - R7: NetHandle(NetId: In)                                             │
│    • VM executes `OpCode::Call`, pushes new CallFrame; IP = 0.              │
│                                                                             │
│ 2. ARITHMETIC IN VM REGISTERS                                               │
│    • `OP_MUL R8, R_sd_len, 2` ──► 1_500_000_000_000 pm                     │
│    • `OP_ADD R9, R8, R_L`      ──► 1_650_000_000_000 pm (diff_len)          │
│    • `OP_SUB R10, R_W, 200nm`  ──► 800_000_000_000 pm                       │
│    • `OP_DIV R11, R10, 400nm`  ──► Int(2) (num_vias)                       │
│                                                                             │
│ 3. NATIVE DEVICE REGISTRATION                                               │
│    • VM executes `OpCode::EmitDevice`:                                      │
│      - space_id: SpaceId(0)                                                 │
│      - type: "NMOS"                                                         │
│      - name: "M_N"                                                          │
│      - terminals: { S: VSS, D: Out, G: In, B: VSS }                         │
│      - params: { W: 1.0um, L: 150nm }                                       │
│    • Instantiates device contract in `EntityGraph` for SPICE extraction.    │
│                                                                             │
│ 4. GEOMETRY EMISSION                                                        │
│    • VM executes `OpCode::EmitPolygon` (diff layer):                        │
│      - Inserts rect [9.175um, 4.5um, 1.65um, 1.0um] into `EntityGraph`.     │
│    • VM executes `OpCode::EmitPolygon` (nsdm layer):                        │
│      - Inserts rect [9.045um, 4.37um, 1.91um, 1.26um] into `EntityGraph`.   │
│    • VM executes `OpCode::EmitPolygon` (poly layer):                        │
│      - Inserts channel stripe [9.925um, 4.3um, 150nm, 1.4um].               │
│                                                                             │
│ 5. UNROLLED VIA GENERATION LOOP                                             │
│    • Loop iteration 0:                                                      │
│      - Evaluates via Y: `5.0um - 200nm = 4.8um`.                            │
│      - Executes `OpCode::EmitContact`: Inserts via at [9.625um, 4.8um].     │
│    • Loop iteration 1:                                                      │
│      - Evaluates via Y: `5.0um + 200nm = 5.2um`.                            │
│      - Executes `OpCode::EmitContact`: Inserts via at [9.625um, 5.2um].     │
│                                                                             │
│ 6. RETURN VALUE PACKING                                                     │
│    • VM executes `OpCode::AllocStruct` for `NMOSLayout`.                    │
│    • `OpCode::Return` pops CallFrame, binds `nmos` in caller space scope.   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 9. Implementation Roadmap & Checklist

```
crates/hwc-compiler/src/eval/
├── mod.rs             // Public interface, compile_and_eval(), top-level VM execution
├── bytecode.rs        // OpCode enum, Chunk, Register, Constant pool
├── compiler.rs        // AST -> Bytecode compiler & expression code generation
├── value.rs           // Value enum, UnitDimension, MeasurementValue, coercion
├── context.rs         // VM struct, CallFrame, stack frames, error types
├── sandbox.rs         // Step counter (10M limit), stack depth tracker
├── builtins.rs        // Standard library built-ins (math, diagnostics, rect_between)
└── emitter.rs         // SpaceEmitter trait & MemoryEmitter bridging EntityGraph
```

### Execution Phases

- [ ] **Phase 1: Value Model & 128-bit Arithmetic (`value.rs`)**
  - Implement `Value`, `UnitDimension`, and `MeasurementValue`.
  - Implement full dimensional algebra operations ($L \pm L$, $L \times L$, $V/I$, $V \times I$).
  - Implement `coerce_to_point2d()` for `[Measurement, Measurement]` array literals.
  - Write unit tests for physical unit calculations and unit mismatch errors (`Error S22`).

- [ ] **Phase 2: Bytecode Compiler (`compiler.rs`, `bytecode.rs`)**
  - Implement AST expression visitor emitting linear `OpCode`s.
  - Implement compiler passes for `let`, `let mut`, `for .. in`, `if/else`, and function calls.
  - Implement `nets { ... }` block declaration lowering to `Value::NetHandle(NetId)`.

- [ ] **Phase 3: VM Dispatch Loop & Sandbox (`context.rs`, `mod.rs`, `sandbox.rs`)**
  - Implement stack-frame activation record allocator.
  - Implement instruction dispatch loop with $10^7$ step limiter (`Error C01`) and recursion guard (`Error C02`).
  - Wire implicit `space` handle contextual injection (`Error S30` outside space).

- [ ] **Phase 4: Built-ins & SpaceEmitter Bridge (`builtins.rs`, `emitter.rs`)**
  - Implement diagnostic built-ins (`println`, `eprintln`, `dbg`, `assert`).
  - Implement geometric built-ins (`sin`, `cos`, `tan`, `sqrt`, `min`, `max`, `abs`, `rect_between`).
  - Implement `SpaceEmitter` connecting `space.add_polygon`, `space.add_contact`, `space.add_device`, `route` directly to `hwc-engine::EntityGraph`.

- [ ] **Phase 5: Legacy Subsystem Deletion & Integration**
  - Delete `placement_loop.rs` and `relational_resolver.rs`.
  - Run the complete CMOS Inverter tapeout verification gauntlet (`cmos_inverter.hw`).

---

## Conclusion

This specification provides the blueprint for `hwc-eval`. By eliminating all AST tree re-boxing, deleting legacy relational guessing solvers, and implementing a **deterministic 128-bit linear Bytecode VM**, HardwareScript v0.3.0 compiles complete silicon and PCB layouts in milliseconds with zero dead code and zero backward-compatibility baggage.