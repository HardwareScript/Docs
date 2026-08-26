# HardwareScript v0.3.0: Specification Leak Fixes & Cross-Document Alignment

**Document Type:** Specification Audit Resolution  
**Target Version:** v0.3.0  
**Status:** Completed  
**Date:** 2026-08-26  

---

## Executive Summary

This document addresses and resolves **four critical cross-document contradictions** identified in the v0.3.0 specification suite. These specification leaks created implementation ambiguity that could have resulted in architectural drift, type system inconsistencies, and undocumented runtime behavior.

All four leaks have been **resolved** through explicit documentation updates and architectural clarifications across:
- `Comptime-Evaluation-Engine.md`
- `Authoritative-Refactoring-Giude.md`
- `Standard-Library-Architecture.md`
- `Canonical-Language-Grammar.md`

---

## 1. ✅ RESOLVED: `hwc-eval` Execution Model Drift (Bytecode VM vs. Tree-Walker)

### The Contradiction

**Original State:**
- `Authoritative-Refactoring-Giude.md` defined `hwc-eval` as a **Linear Bytecode VM with Static Activation Records (`Vec<Value>`)**.
- `Comptime-Evaluation-Engine.md` Section 9 (Comparison Table) stated: **"Pure tree-walk interpreter on immutable AST"**.

This contradiction meant developers could implement the wrong execution paradigm, losing the performance benefits of bytecode dispatch.

### The Fix

**Updated:** `Comptime-Evaluation-Engine.md` Section 10 (formerly Section 9)

**Before:**
```
| Execution Paradigm | Multi-pass AST re-boxing | Pure tree-walk interpreter on immutable AST |
```

**After:**
```
| Execution Paradigm | Multi-pass AST re-boxing | Linear Bytecode VM with Static Activation Records |
```

**Implementation Guidance:**
- The `hwc-eval` engine must compile AST expressions into a linear bytecode instruction stream.
- Execution uses a register-based or stack-based VM with static activation records (`Vec<Value>`).
- This architecture enables:
  - Constant-time instruction dispatch
  - Zero AST tree traversal overhead
  - Efficient loop unrolling
  - Predictable memory allocation patterns

**Location:** `crates/hwc-compiler/src/eval/mod.rs`

---

## 2. ✅ RESOLVED: Type-System Mismatch (`struct Point2D` vs. Array Literal `[x, y]`)

### The Contradiction

**Original State:**
- Function signatures in `@std/pdk/sky130/nmos.hw` declared: `at: Point2D`
- Calling sites in `cmos_inverter.hw` used: `at: [10.0um, 5.0um]`
- The formal EBNF grammar in `Canonical-Language-Grammar.md` defined `[10.0um, 5.0um]` as an `ArrayLiteral (Array[Measurement])`, **not** a `StructInstanceExpr`.

Without explicit coercion rules, this would trigger:
```
Error S22: Type Mismatch
Expected Point2D, found Array[Measurement]
```

### The Fix

**Added:** `Comptime-Evaluation-Engine.md` Section 6.2: "Array-to-Point2D Type Coercion"

**Coercion Rule:**
When a function parameter is declared with type `Point2D` and the caller provides a 2-element array literal `[x, y]` where both elements are of type `Measurement`, the evaluator **automatically coerces** the array into:
```hardware
Point2D { x: x_value, y: y_value }
```

**Type Checking Rules:**
1. ✅ `at: [10.0um, 5.0um]` → Coerced to `Point2D { x: 10.0um, y: 5.0um }`
2. ❌ `at: [10.0um, 5.0um, 3.0um]` → Error: Array has 3 elements (expected exactly 2)
3. ❌ `at: [10.0um, 5]` → Error: Mixed types (Measurement, Int)
4. ✅ `at: Point2D { x: 10.0um, y: 5.0um }` → Always allowed (explicit syntax)

**Implementation Location:** `crates/hwc-compiler/src/eval/value.rs` in `coerce_to_type()` function.

**Documentation Updates:**
- `Comptime-Evaluation-Engine.md` Section 6.2 (new)
- `Standard-Library-Architecture.md` Section 4.1 (`@std/primitives/math.hw`)
- Function signatures in `@std/pdk/sky130/nmos.hw` and `pmos.hw` now include comments:
  ```hardware
  at: Point2D,  // Accepts Point2D struct OR 2-element Measurement array [x, y]
  ```

---

## 3. ✅ RESOLVED: Contextual Scope Leaks (The Implicit `space.*` Handle)

### The Contradiction

**Original State:**
In standard library functions (`nmos.hw`, `strap.hw`), the code called:
```hardware
space.add_polygon(...)
space.add_contact(...)
space.add_device(...)
```

But the `space` variable was **never passed as a function argument** into `sky130_nmos(name, W, L, at, source, drain, gate, bulk)`.

**Question:** Where did the `space` variable come from?

### The Fix

**Added:** `Comptime-Evaluation-Engine.md` Section 6.1: "The Implicit `space` Handle"

**Contextual Handle Injection Rules:**

1. **Inside `space` Block:** The `space` handle is implicitly bound to the current active space context in the `EvaluationContext`. All `space.add_*` methods emit geometry directly into the master `EntityGraph`.

2. **Outside `space` Block:** Attempting to call `space.add_polygon()` or any `space.*` method outside of a `space` block results in:
   ```
   Error S30: No Active Space Context
   Cannot call 'space.add_polygon()' outside of a space block.
   ```

3. **Function Calls From Within Space:** When a user-defined function (e.g., `sky130_nmos()`) is called from within a `space` block, the `space` handle **remains accessible** to that function and all nested function calls in the call stack.

**Implementation Detail:**

The `EvaluationContext` maintains a `current_space_id: Option<SpaceId>` field. When entering a `space` block:
1. Allocates a new `SpaceId` in the `EntityGraph`.
2. Sets `current_space_id = Some(space_id)`.
3. Injects a synthetic immutable binding: `space = Value::SpaceHandle(space_id)` into the local `ScopeFrame`.

When exiting the `space` block, the evaluator clears `current_space_id = None`.

**Implementation Location:**
- `crates/hwc-compiler/src/eval/context.rs` - `EvaluationContext` struct
- `crates/hwc-compiler/src/eval/emitter.rs` - `SpaceEmitter` trait

**Documentation Updates:**
- `Comptime-Evaluation-Engine.md` Section 6.1 (new)
- `Standard-Library-Architecture.md` Section 5.2 (added contextual handle usage note)
- `Authoritative-Refactoring-Giude.md` Section 3.2 (updated `context.rs` description)

---

## 4. ✅ RESOLVED: Symbol Resolution of `NetHandle` in Space Lexical Scope

### The Contradiction

**Original State:**
In `cmos_inverter.hw`:
```hardware
nets {
    VDD: { classification: power, potential: 1.8V }
    Out: { classification: signal }
}
let nmos = sky130_nmos(..., source: VSS, drain: Out, ...)
```

**Question:** How does the evaluator resolve `VSS` and `Out` as valid identifiers? How does it inject them as `NetHandle` values?

### The Fix

**Clarified:** The `nets { ... }` block declaration mechanism is now explicitly documented.

**Symbol Resolution Mechanism:**

When the evaluator parses a `nets { ... }` block:

1. **Entity Graph Allocation:** Every net declaration allocates a unique `NetId` in the `EntityGraph`.
   ```rust
   let net_id = entity_graph.allocate_net("VDD", NetProperties { ... });
   ```

2. **Scope Injection:** The identifier `VDD` or `Out` is injected into the local `ScopeFrame` as an immutable `Value::NetHandle(NetId)`.
   ```rust
   scope_frame.bind("VDD", Value::NetHandle(net_id), is_mutable: false);
   ```

3. **Strongly-Typed Passing:** When `source: VSS` or `drain: Out` is evaluated, the compiler resolves the identifier to a `Value::NetHandle(NetId)` and passes it as a strongly-typed function argument.

4. **Type Safety:** Attempting to pass a non-NetHandle value (e.g., `source: 10.0um`) to a parameter expecting `Net` triggers a compile-time type error.

**Implementation Location:**
- `crates/hwc-compiler/src/eval/mod.rs` - `eval_nets_block()` function
- `crates/hwc-engine/src/entity_graph.rs` - `allocate_net()` method

**Documentation Updates:**
- `Comptime-Evaluation-Engine.md` Section 6.1 (implicit handle injection)
- `Canonical-Language-Grammar.md` Section 4.4 (Hardware Domain Declarations)

---

## 5. ⚠️ ACKNOWLEDGED: The "4-Week Timeline" Reality Check

### The Original Timeline Risk

The original `Authoritative-Refactoring-Giude.md` proposed a **4-week execution blueprint**:
- Week 1: Parser & Lexer
- Week 2: Comptime Bytecode VM
- Week 3: @std Standard Library & SKY130 PCells
- Week 4: Full 3-Stage DOPHR Router (3D Volumetric Tensor, Track Assigner, Spatial 4-Coloring, Adaptive RRR, SIMD DRC, SPICE validation, **and Release**)

**The Warning:**

Attempting to implement an industrial-grade **3-Stage Guided Router** (Global PathFinder + Panel Track Assigner + Guided Detailed A* + Localized RRR*) in a **single week** (Week 4) is unrealistic.

In commercial EDA teams, writing and hardening a 3-stage guided router takes **3 to 6 weeks** of focused engineering.

### The Fix

**Updated:** `Authoritative-Refactoring-Giude.md` Section 5: "The 6-Week Realistic Migration & Execution Roadmap"

**New Schedule:**
- **Weeks 1-3:** Parser + Bytecode VM + @std Library
- **Weeks 4:** SKY130 PCell Migration
- **Weeks 5-6:** 3-Stage Guided Router (Global + Track Assignment + Detailed + RRR)
- **Week 7:** DRC/LVS Verification, SKY130 Tapeout Gauntlet, and v0.3.0 Release

**Timeline Rationale:**

1. **Parser + Bytecode VM (Weeks 1-3):** Transitioning from a tree-walk interpreter to a linear bytecode VM requires careful design of instruction encoding, register allocation, and scope frame management. Commercial compilers spend **2-3 weeks** on VM design alone.

2. **3-Stage Guided Router (Weeks 5-6):** Production-grade routers require **3-6 weeks** in commercial environments. The extended timeline prevents:
   - Incomplete spatial conflict resolution
   - DRC violations in congested regions
   - Missing adaptive re-routing logic
   - Insufficient testing on multi-net designs

3. **Standard Library (Week 4):** Writing certified foundry PCells with proper DRC margins, via matrices, and SPICE model bindings requires validation against actual SKY130 design rules.

**The 6-7 week schedule is realistic, achievable, and maintains quality.**

---

## Summary & Sign-Off

| Domain | Status | Files Modified |
| :--- | :--- | :--- |
| **Execution Model (Bytecode VM)** | ✅ Resolved | `Comptime-Evaluation-Engine.md` Section 10 |
| **Type Coercion (Point2D)** | ✅ Resolved | `Comptime-Evaluation-Engine.md` Section 6.2<br>`Standard-Library-Architecture.md` Section 4.1, 5.2, 5.3 |
| **Implicit `space` Handle** | ✅ Resolved | `Comptime-Evaluation-Engine.md` Section 6.1<br>`Standard-Library-Architecture.md` Section 5.2<br>`Authoritative-Refactoring-Giude.md` Section 3.2 |
| **NetHandle Symbol Resolution** | ✅ Documented | `Comptime-Evaluation-Engine.md` Section 6.1<br>`Canonical-Language-Grammar.md` Section 4.4 |
| **Execution Schedule** | ✅ Adjusted | `Authoritative-Refactoring-Giude.md` Section 5 |

---

## Implementation Checklist for Engineers

When implementing the v0.3.0 compiler, use this checklist to ensure specification alignment:

### Bytecode VM Implementation
- [ ] Implement linear bytecode instruction stream in `hwc-compiler/src/eval/mod.rs`
- [ ] Use static activation records (`Vec<Value>`) instead of tree-walking
- [ ] Verify instruction dispatch is constant-time ($O(1)$)

### Type Coercion System
- [ ] Implement `coerce_to_type()` in `hwc-compiler/src/eval/value.rs`
- [ ] Add unit tests for `Array[Measurement, Measurement]` → `Point2D` coercion
- [ ] Verify error messages for invalid array sizes and mixed types

### Contextual Handle Injection
- [ ] Add `current_space_id: Option<SpaceId>` to `EvaluationContext`
- [ ] Inject `space = Value::SpaceHandle(space_id)` when entering `space` block
- [ ] Emit `Error S30: No Active Space Context` when `space.*` called outside space
- [ ] Preserve `space` handle across nested function calls

### NetHandle Symbol Resolution
- [ ] Implement `eval_nets_block()` to allocate `NetId` in `EntityGraph`
- [ ] Inject net identifiers as `Value::NetHandle(NetId)` into scope
- [ ] Add type checking to prevent non-NetHandle values in `Net` parameters

### Timeline Adherence
- [ ] Follow 6-7 week realistic schedule
- [ ] Allocate 2 weeks minimum for 3-stage router implementation
- [ ] Include buffer time for DRC/LVS verification

---

**End of Specification Leak Resolution Document**
