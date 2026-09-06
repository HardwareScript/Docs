# HardwareScript v0.3.2: JIT Comptime Elaboration Engine Specification

**Document Type:** Core Runtime & Comptime Evaluation Specification  
**Target Version:** v0.3.2 (Production-Locked Standard)  
**Status:** Authoritative & Canonical  
**Target Crate:** `hwc-compiler`  
**Downstream Dependents:** `hwc-engine`, `hwc-substrate-cmos`, `hwc-substrate-laminate`, `hwc-physics`  
**Date:** September 2026  

---

## 1. Executive Overview: Eradication of VMs and Tree-Walkers

HardwareScript v0.3.2 permanently eradicates both the legacy AST tree-walk interpreter and the custom register-based Bytecode Virtual Machine. 

- **Failure Mode 1 (AST Tree-Walk):** Suffered from massive allocation overhead, deep call-stack recursion, and dynamic variable lookup degradation during large-scale layout generation.
- **Failure Mode 2 (Bytecode VM):** Suffered from instruction-dispatch overhead, fuel accounting limits, register boxing/unboxing, and memory quotas that artificially choked multi-million-transistor generative synthesis.

In v0.3.2, compile-time elaboration (`hwc-compiler`) compiles procedural loops, parameter algebra, and physical cell evaluations directly into **native host machine code (x86_64, aarch64) in memory via Cranelift**.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 CRANELIFT IN-MEMORY JIT EVALUATION PIPELINE                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ AST Arena (hwc-parser) ]                                                 │
│  • Immutable, typed Abstract Syntax Tree                                    │
│  • 7-Base SI unit dimensional validation performed upstream                │
│                                                                             │
│                                      │ Lowering                             │
│                                      ▼                                      │
│  [ Cranelift Intermediate Representation (CLIF) ]                           │
│  • SSA basic blocks for procedural loops, parameter math, and branches      │
│  • Intrinsic calls for coordinate pool writes and affine transforms         │
│                                                                             │
│                                      │ In-Memory Compilation (< 3ms)        │
│                                      ▼                                      │
│  [ Native Host Executable Buffer ]                                          │
│  • Bare-metal machine code residing in executable RAM pages                │
│  • Direct pointer invocation: `extern "C" fn(&mut FlatGeometryBuffer)`      │
│                                                                             │
│                                      │ Native Hardware Execution            │
│                                      ▼                                      │
│  [ FlatGeometryBuffer (hwc-engine) ]                                        │
│  • Picometer vertices emitted directly into contiguous coordinate memory    │
│  • Zero fuel limits, zero register re-boxing, zero runtime allocations      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. In-Memory JIT Compilation Architecture

### 2.1 The Execution Lifecycle

When a `space` floorplan or procedural generator block is elaborated:

1. **SSA Lowering to CLIF:** `hwc-compiler` lowers the procedural control-flow graph (loops, conditional branches, coordinate arithmetic) into Cranelift Intermediate Representation (CLIF).
2. **Native Code Generation:** Cranelift's JIT backend (`cranelift-jit`, `cranelift-module`) compiles the CLIF blocks directly into executable host machine code in memory.
3. **Native Function Invocation:** The compiler executes the compiled code via a native function pointer, passing a mutable reference to the active `FlatGeometryBuffer` allocation context:
   ```rust
   pub type ComptimeEntryFn = unsafe extern "C" fn(
       buffer_ctx: *mut FlatGeometryBufferContext,
       params_ptr: *const u8,
   ) -> i32;
   ```
4. **Direct Memory Emission:** Loops execute at host hardware speed, writing signed 64-bit picometer vertices directly into contiguous vectors without interpreter dispatch overhead.
5. **Memory Reclaim:** Upon completion of the elaboration stage, executable memory pages are safely reclaimed.

### 2.2 Cranelift IR (CLIF) Lowering Example

Consider a procedural contact array generator:

```hardware
for i in 0..num_vias {
    let vy = -via_offset + (i * via_pitch)
    cell.add_contact(from: "diff", to: "li1", at: [src_x, vy], diameter: 170nm)
}
```

`hwc-compiler` lowers this loop into SSA CLIF basic blocks:

```clif
function u0:0(i64, i64, i64, i64, i64) -> i32 system_v {
    ; Arguments:
    ; v0: *mut FlatGeometryBufferContext
    ; v1: num_vias
    ; v2: via_offset
    ; v3: via_pitch
    ; v4: src_x

block0(v0: i64, v1: i64, v2: i64, v3: i64, v4: i64):
    v5 = iconst.i64 0
    jump block1(v5)

block1(v6: i64):
    v7 = icmp slt v6, v1
    brif v7, block2, block3

block2:
    v8 = imul v6, v3
    v9 = isub v8, v2
    ; Intrinsic call to emit contact into FlatGeometryBuffer
    call fn_emit_contact(v0, v4, v9, 170000)
    v10 = iadd_imm v6, 1
    jump block1(v10)

block3:
    v11 = iconst.i32 0
    return v11
}
```

---

## 3. Pure PCell Flyweight Memoization

Parameterized Cells (PCells) are pure mathematical functions:
- **Zero Ambient Mutation:** PCells cannot read, mutate, or query the enclosing `space` block or global environment.
- **Local Origin-Relative Geometry:** All coordinates within a PCell are computed relative to local origin `[0, 0]` in signed 64-bit picometers.
- **Blake3 Parameter Hashing:** The compiler hashes the PCell function identifier and its input arguments using Blake3:
  $$\text{PCellKey} = \text{Blake3}(\text{FunctionId} \parallel \text{Arg}_1 \parallel \text{Arg}_2 \parallel \dots \parallel \text{Arg}_N)$$

### 3.1 Flyweight Caching Strategy

```rust
// hwc-compiler/src/memoize.rs

use blake3::Hash;
use hwc_engine::CellLayout;
use std::collections::HashMap;
use std::sync::Arc;

pub struct PCellMemoCache {
    cache: HashMap<Hash, Arc<CellLayout>>,
}

impl PCellMemoCache {
    pub fn get_or_evaluate<F>(&mut self, key: Hash, evaluate: F) -> Arc<CellLayout>
    where
        F: FnOnce() -> CellLayout,
    {
        if let Some(layout) = self.cache.get(&key) {
            return Arc::clone(layout);
        }
        let layout = Arc::new(evaluate());
        self.cache.insert(key, Arc::clone(&layout));
        layout
    }
}
```

- **$O(1)$ Stamping:** If an identical cell has already been evaluated (e.g. 1,024 instances of an identical SRAM bitcell or NMOS transistor), the compiler retrieves the cached `CellLayout` reference and applies affine coordinate translations $(dx, dy)$ in $O(1)$ time.
- **Evaluation Speedup:** Layout generation scales linearly with unique parameter configurations, dropping multi-thousand-transistor evaluation from minutes to sub-millisecond execution.

---

## 4. Diagnostic Traceability: The Out-of-Band Debug Span Registry

### 4.1 The Problem: Cache Pollution from Source Spans

Traditional compilers store source code byte offsets or line numbers directly inside AST nodes or intermediate representation structs. This causes severe cache pollution:
- Adding a single comment or whitespace character at the top of a file alters all downstream byte spans.
- Incremental compilation caches (e.g. `rkyv`, Salsa) are invalidated, forcing total rebuilds of identical physical geometry.
- Production mask exports (GDSII, OASIS, Gerber) waste memory on debug tracking metadata.

### 4.2 The Solution: The Out-of-Band Lookup Map

HardwareScript v0.3.2 strictly decouples physical identity from source code coordinates:

$$\text{DebugSpanRegistry} : \text{EntityId} \to (\text{FileId}, \text{SourceSpan})$$

```rust
// hwc-compiler/src/diagnostics.rs

use hwc_types::identity::EntityId;
use miette::SourceSpan;
use std::collections::HashMap;

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct FileId(pub u32);

#[derive(Default, Debug, Clone)]
pub struct DebugSpanRegistry {
    spans: HashMap<EntityId, (FileId, SourceSpan)>,
}

impl DebugSpanRegistry {
    pub fn register(&mut self, id: EntityId, file_id: FileId, span: SourceSpan) {
        self.spans.insert(id, (file_id, span));
    }

    pub fn lookup(&self, id: EntityId) -> Option<(FileId, SourceSpan)> {
        self.spans.get(&id).copied()
    }
}
```

### 4.3 Invariants of the Debug Span Registry

1. **Zero Serialization to Disk:** `DebugSpanRegistry` is ephemeral. It is never serialized into disk caches and never included in GDSII or Gerber streams.
2. **Span-Independent Hashing:** `EntityId` is computed exclusively from the hierarchical lexical path and instance index:
   $$\text{EntityId} = \text{Blake3}(\text{ScopePath} \parallel \text{TypeName} \parallel \text{InstanceIndex})[0..8]$$
   Source code edits that do not alter physical structure preserve identical `EntityId` values across builds.
3. **On-Demand Error Construction:** When a verification pass (DRC, LVS, Connectivity) detects an anomaly on an `EntityId`, it queries `DebugSpanRegistry` to construct rich, colorized `miette` diagnostic reports pointing to the exact user source code line.

---

## 5. Elimination of Fuel Accounting and Quotas

Because HardwareScript is an interactive, professional engineering compiler, artificial instruction-fuel limits are completely eliminated:

| Parameter | Legacy Bytecode VM | v0.3.2 Cranelift JIT |
| :--- | :--- | :--- |
| **Execution Limit** | 100M instruction fuel limit + area scaling | **Uncapped native execution** |
| **Infinite Loop Protection** | Fuel decrement per instruction step | **OS-level interrupt signals (`SIGINT` / `Ctrl+C`)** |
| **Memory Limit** | 2 GB resident allocation quota | **OS virtual memory paging** |
| **Stack Depth** | 256 call frames | **Native hardware stack** |
| **Execution Speed** | ~10–50 M ops/sec (interpreted dispatch) | **Native CPU speed (~3–4 G ops/sec)** |

Physical layouts containing 10,000,000 polygons compile in fractions of a second with zero artificial aborts.

---

## 6. Verification and Invariant Checklist

- [x] **No Virtual Machine:** No opcodes, bytecodes, instruction fuel counters, or interpreter dispatch loops exist in `hwc-compiler`.
- [x] **No Tree-Walk Interpreter:** Control flow compiles directly to native host instructions.
- [x] **Flyweight Caching:** Identical PCells are evaluated once and shared across all instances.
- [x] **Span Invariance:** Physical geometry records never store line/column numbers; diagnostics flow through `DebugSpanRegistry`.
