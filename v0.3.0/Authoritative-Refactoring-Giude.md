# HardwareScript v0.3.0: Crate-by-Crate Refactoring & Migration Specification

**Document Type:** Authoritative Refactoring Guide & Execution Roadmap  
**Target Version:** v0.3.0  
**Status:** In Progress (Milestone 1–6 Completed, Phase 7 Release Gate Underway)  
**Focus:** Workspace Architecture, Crate Boundaries, File Deletion Manifest, Comptime Integration, and Execution Plan  

---

## 1. The Executive Audit: Legacy Crate Coupling & Technical Debt

In versions v0.1.0 through v0.2.2, compiler functionality was scattered across crates with **leaky abstraction boundaries, circular dependencies, and duplicate responsibilities**:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    THE HISTORICAL CRATE COUPLING FAILURES                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. `hwc-parser` WAS TOO SMART AND TOO FRAGILE                              │
│     • Maintained complex indentation state machines (`INDENT` / `DEDENT`).  │
│     • Contained 50+ specialized keywords for physical placement.            │
│     • Attempted to parse ad-hoc relational prepositions contextually.       │
│                                                                             │
│  2. `hwc-compiler` WAS RE-BUILDING SYNTHETIC ASTs IN MEMORY                │
│     • `placement_loop.rs` contained 90 lines of boxed AST tree generation.  │
│     • `relational_resolver.rs` attempted to solve placement via opaque      │
│       geometric heuristics instead of letting user scripts evaluate math.   │
│     • Spawned isolated sub-spaces that trapped the router.                  │
│                                                                             │
│  3. `hwc-engine` WAS POLLUTED WITH FOUNDRY-SPECIFIC HACKS                   │
│     • Hardcoded solder mask rules and PDK offsets were baked into Rust.     │
│     • The database had to handle competing coordinate representations.      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### What Would Have Happened If We Kept the Trend?
* **Continuous Merge Conflicts:** Every new layout feature required touchpoints across 6 different crates.
* **Broken Incremental Compilation:** Salsa query memoization cannot cache opaque constraint-solver mutations.
* **Unmaintainable Codebase:** New contributors could not understand where language parsing ended and physical synthesis began.

---

## 2. The Architectural Crate Boundary Contract

HardwareScript v0.3.0 establishes clean, strictly unidirectional crate dependencies:

```
 ┌─────────────────────────────────────────────────────────────────────────┐
 │                                hwc-cli                                  │
 │   CLI Commands: `check`, `eval`, `build`, `test`, package resolution   │
 └────────────────────────────────────┬────────────────────────────────────┘
                                      │
                                      ▼
 ┌─────────────────────────────────────────────────────────────────────────┐
 │                              hwc-compiler                               │
 │   Module resolution, Symbol table, Comptime VM (`hwc-eval`)             │
 └───────────────────────┬─────────────────────────┬───────────────────────┘
                         │                         │
            AST Stream   │                         │ Lowers to EntityGraph
                         ▼                         ▼
 ┌───────────────────────────────┐ ┌───────────────────────────────────────┐
 │          hwc-parser           │ │              hwc-engine               │
 │ Logos Lexer (Brace tokens)    │ │ Single Master `EntityGraph` (DBU pm)  │
 │ Pratt Expression Parser       │ │ Spatial Index (`rstar` / `geo-index`) │
 │ Arena-allocated AST nodes     │ │ Topological Router & QP Legalizer     │
 └───────────────────────────────┘ └───────────┬───────────────┬───────────┘
                                               │               │
                               Physical Queries │               │ Final Geometry
                                               ▼               ▼
 ┌───────────────────────────────────────────────┐ ┌───────────────────────┐
 │                  hwc-physics                  │ │      hwc-export       │
 │ G-Cell Morton-ordered SIMD Sweep DRC          │ │ GDSII / OASIS Stream  │
 │ PIVB Connectivity Solver (Welded Manifolds)   │ │ SPICE (.sp) Netlister │
 │ Wheeler + Sakurai BEM Parasitic Extraction    │ │ DXF (2D) & GLB (3D)   │
 └───────────────────────────────────────────────┘ └───────────────────────┘
```

---

## 3. Crate-by-Crate Surgical Refactoring Specifications

### 3.1 `crates/hwc-parser` (COMPLETED)

**Responsibility:** Pure syntax tokenization and recursive descent / Pratt parsing into an immutable AST. **Zero layout solving, zero indentation tracking.**

#### Files Purged / Modernized:
* ✅ Purged `INDENT` / `DEDENT` whitespace tracking. Whitespace is strictly token-separating.
* ✅ Cleaned up relational and contextual keywords in favor of strong expressions.

#### Implemented Submodules:
* ✏️ `src/lexer/token/token_types.rs` & `tokenizer.rs`:
  * Explicit brace tokens `OpenBrace` (`{`), `CloseBrace` (`}`).
  * 26 Canonical Keywords: `fn`, `let`, `mut`, `const`, `struct`, `enum`, `if`, `else`, `for`, `in`, `return`, `assert`, `match`, `import`, `export`, `from`, `true`, `false`, `and`, `or`, `not`, `space`, `module`, `device`, `material`, `profile`, `route`, `test`, `nets`, `pins`, `implements`, `to`, `with`, `intent`.
  * Complete dimensional measurement literal tokens (pm, nm, um, mm, V, mV, uV, nV, A, mA, uA, pA, Ohm, kOhm, MOhm, etc.).
* ✏️ `src/ast/`:
  * `expression/operators.rs`: Pratt 8-level precedence operators.
  * `expression/types.rs`: Complete `Expression` AST enum.
  * `statement.rs`: `Statement`, `Block`, `TypeExpr`, `AssignmentOperator`, `ElseBranch`.
  * `declarations.rs`: Top-level items (`FunctionDecl`, `StructDecl`, `EnumDecl`, `SpaceDecl`, `ModuleDecl`, etc.).
  * `ast/mod.rs`: `Program` and `TopLevelItem` AST roots.
* ✏️ `src/parser/`:
  * `expression.rs`: Pratt precedence expression parser with full operator support and struct initializers.
  * `statements.rs`: Recursive descent statement, block `{ ... }`, and control flow parser.
  * `declarations.rs`: Declarations parser for functions, spaces, structs, modules, materials, profiles.
  * `mod.rs`: Deterministic top-level synchronization and panic-mode error recovery.

---

### 3.2 `crates/hwc-compiler` (`hwc-eval`) (COMPLETED)

**Responsibility:** Module resolution, standard library loading, type checking, and the **Compile-Time Evaluation Engine (`hwc-eval`)**.

#### Implemented Submodules in `src/eval/`:
* 🆕 `src/eval/value.rs`: `Value` enum (`Int`, `Float`, `Bool`, `String`, `Measurement`, `Point2D`, `Point3D`, `Vector2D`, `BoundingBox`, `Array`, `StructInstance`, `EnumVariant`, `NetHandle`, `PortHandle`, `SpaceHandle`).
  * 128-bit fixed-point integer DBU picometer math.
  * Automatic `Array[Measurement, Measurement]` $\rightarrow$ `Point2D` type coercion.
  * Strict dimensional algebra ($L+L \rightarrow L$, $L \times L \rightarrow \text{Area}$, $V/I \rightarrow R$, $V \times I \rightarrow P$).
* 🆕 `src/eval/context.rs`: `EvaluationContext`, lexical `ScopeFrame` hierarchy, mutability validation (`let` vs `let mut`), and implicit `space` handle injection.
* 🆕 `src/eval/sandbox.rs`: Step-counter ($10^7$ limit) and recursion depth limit ($256$ frames) guarding against infinite loops.
* 🆕 `src/eval/builtins.rs`: Standard diagnostics (`println`, `eprintln`, `dbg`, `assert`) and geometric math (`min`, `max`, `abs`, `sqrt`, `sin`, `cos`, `tan`, `rect_between`).
* 🆕 `src/eval/emitter.rs`: `SpaceEmitter` trait and `MemoryEmitter` bridging `space.add_polygon`, `space.add_contact`, `space.add_device`, and `route` directly into physical geometry records.
* 🆕 `src/eval/mod.rs`: Main comptime execution engine.

---

### 3.3 `stdlib/` (`@std/`) (COMPLETED)

**Responsibility:** The standard library written in pure HardwareScript v0.3.0 syntax with `#` comments.

#### Implemented Hierarchy:
```
stdlib/
├── primitives/
│   ├── units.hw          # Physical unit constants & scale factors
│   └── math.hw           # Vector2D, Point2D, BoundingBox math & rect_between
├── layout/
│   ├── placement.hw      # stack_horizontal, stack_vertical, Align
│   └── via.hw            # via_matrix, fill_vias_in_box
└── pdk/
    └── sky130/           # SkyWater SKY130 1.8V open-source foundry PDK
        ├── rules.hw      # Design rules (SKY130_RULES)
        ├── nmos.hw       # sky130_nmos parameterized generator PCell
        ├── pmos.hw       # sky130_pmos parameterized generator PCell
        ├── tap.hw        # sky130_tap generator (P_Sub, N_Well)
        ├── strap.hw      # route_strap low-resistance bridge
        └── pad.hw        # pad I/O bonding pad generator
```

---

### 3.4 `crates/hwc-engine` (ACTIVE / NEXT PHASE)

**Responsibility:** The canonical physical database (`EntityGraph`), continuous spatial indexing, topological line-search router, and localized QP legalizer. **Zero foundry-specific rules.**

#### Files to Clean Up & Align:
* ✏️ `src/geometry_router/legalizer.rs`: Ensure legalizer treats all unrolled primitives as flat, discrete layer obstacles with port exemptions.
* ✏️ `src/space/mod.rs`: Direct emitter ingestion entry points from `hwc-eval`.
* ✏️ `src/entity_graph/mod.rs`: Ensure all incoming coordinates are strictly 64-bit integer picometers ($1\text{ pm} = 10^{-12}\text{ m}$).

---

### 3.5 `crates/hwc-physics`

**Responsibility:** First-class physical verification gate (DRC, LVS, PIVB, BEM parasitics).

#### Status:
* ✅ **No structural redesign required.**
* ✏️ Ensure `simd_drc.rs` (G-Cell Morton-ordered sweep) and `parasitic_solver.rs` (Sakurai/Wheeler BEM) read cleanly from the single flat `EntityGraph`.

---

### 3.6 `crates/hwc-export`

**Responsibility:** Streaming manufacturing files from the verified `EntityGraph`.

#### Status:
* ✅ **No structural redesign required.**
* ✏️ Ensure `gdsii.rs`, `netlist.rs` (SPICE `.sp`), `dxf.rs`, and `substrate.rs` (GLB 3D) consume finalized `EntityGraph` records directly.

---

### 3.7 `crates/hwc-cli`

**Responsibility:** Command-line developer workflows and package entry points.

#### CLI Command Suite:
* ✏️ `hwc check <file.hw>`: Lexical, grammar, and static type check ($<5\text{ ms}$).
* 🆕 `hwc eval <file.hw>`: Executes comptime functions, prints `println()`/`dbg()` diagnostics without invoking physical meshing ($<10\text{ ms}$).
* ✏️ `hwc build <file.hw>`: Full physical synthesis pipeline (Comptime $\rightarrow$ EntityGraph $\rightarrow$ Routing $\rightarrow$ DRC $\rightarrow$ GDSII/SPICE/GLB) ($<30\text{ ms}$).
* ✏️ `hwc test <file.hw>`: Layout synthesis + automated testbench simulation (`ngspice.wasm`) ($<100\text{ ms}$).

---

## 4. Execution Roadmap & Progress Tracking

```
 ┌─────────────────────────────────────────────────────────────────────────┐
 │                    6-WEEK REALISTIC EXECUTION BLUEPRINT                 │
 ├─────────────────────────────────────────────────────────────────────────┤
 │                                                                         │
 │  [x] WEEK 1: Parser & Lexer Modernization (`hwc-parser`)                │
 │      • Prune `Token` enum in `tokens.rs` (delete `INDENT`/`DEDENT`).    │
 │      • Implement `{}` curly-brace block parser.                         │
 │      • Implement Pratt expression parser with `and`, `or`, `not`.       │
 │      • Status: COMPLETED. All parser tests passing.                     │
 │                                                                         │
 │  [x] WEEK 2: Comptime Bytecode VM (`hwc-eval`) - Core Engine            │
 │      • Implement `Value` enum and 128-bit physical unit arithmetic.     │
 │      • Implement sandbox step-counter ($10^7$ limit) and recursion guard│
 │      • Status: COMPLETED. Unit tests verified.                          │
 │                                                                         │
 │  [x] WEEK 3: Comptime VM Integration & Standard Library Foundation      │
 │      • Implement `EvaluationContext` with scope frames and implicit `space`│
 │      • Implement `SpaceEmitter` connecting `space.*` to `EntityGraph`.  │
 │      • Implement Array[Measurement, Measurement] → Point2D coercion.    │
 │      • Implement `@std/primitives/math` and `@std/layout/placement`.    │
 │      • Status: COMPLETED.                                               │
 │                                                                         │
 │  [x] WEEK 4: SKY130 PCell Migration & Standard Library Completion       │
 │      • Implement `sky130_nmos`, `sky130_pmos`, `sky130_tap` in `.hw`.   │
 │      • Implement `via_matrix`, `fill_vias_in_box`, `route_strap` helpers│
 │      • Create canonical `cmos_inverter.hw` example.                     │
 │      • Status: COMPLETED.                                               │
 │                                                                         │
 │  [x] WEEK 5: 3-Stage Guided Routing - Phase 1 (Global + Track Assignment│
 │      • Implement 3D Volumetric Occupancy Tensor (DOPHR Stage 1).        │
 │      • Implement Panel Track Assignment Heuristic (DOPHR Stage 2).      │
 │      • Status: COMPLETED.                                               │
 │                                                                         │
 │  [x] WEEK 6: 3-Stage Guided Routing - Phase 2 (Detailed + DRC Verif)    │
 │      • Implement Guided Detailed A* Router with Spatial 4-Coloring.     │
 │      • Implement Localized RRR* (Rip-up & Re-route) for DRC violations. │
 │      • Status: COMPLETED.                                               │
 │                                                                         │
 │  [ ] WEEK 7: Tapeout Verification Gauntlet & v0.3.0 Release             │
 │      • Synthesize 28-line `cmos_inverter.hw` in `hwc build`.            │
 │      • Verify 100% clean DRC, PIVB connectivity, and extracted SPICE.   │
 │      • Verify bit-identical GDSII stream output against Golden layout.  │
 │      • Tag and release HardwareScript v0.3.0.                           │
 │                                                                         │
 └─────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Verification Gates & CI Regression Suite

To ensure no physical synthesis regressions occur during refactoring, the continuous integration pipeline enforces four mandatory gates:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        CI/CD VERIFICATION GATES                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  GATE 1: LEXICAL & PARSER DETERMINISM (PASSED)                              │
│  • Verify single-token syntax errors trigger exactly 1 error diagnostic.    │
│  • Assert zero cascading error storms across malformed test files.          │
│                                                                             │
│  GATE 2: COMPTIME EVALUATION & SANDBOX SECURITY (PASSED)                    │
│  • Assert infinite loops halt at exactly 10,000,000 steps (Error C01).      │
│  • Assert dimensional unit mismatches halt with Error S22.                  │
│  • Assert immutable re-assignment halts with Error S14.                     │
│                                                                             │
│  GATE 3: SKY130 PHYSICAL DRC CONFORMANCE (PENDING ROUTING)                  │
│  • Run G-Cell SIMD sweep on synthesized `cmos_inverter.hw`.                 │
│  • Assert 0 DRC violations (poly.4, diff.4, nwell.4, licon.1, npc.2).       │
│                                                                             │
│  GATE 4: SPICE EXTRACTION & LVS MANIFOLD FIDELITY (PENDING ROUTING)         │
│  • Assert PIVB connectivity solver proves single connected component (SCC). │
│  • Assert extracted `circuit.sp` includes correct device cards (`XPMOS_1`,   │
│    `XNMOS_1`) and parasitic interconnect resistors without phantom nodes.   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Summary

* **`hwc-parser` is simplified:** Clean, brace-delimited parser with zero whitespace hacks.
* **`hwc-compiler` becomes an executable engine:** Houses the new `hwc-eval` engine and connects directly to `EntityGraph`.
* **`@std` standard library:** Fully implemented in pure HardwareScript v0.3.0 with `#` comments.
* **`hwc-engine`, `hwc-physics`, and `hwc-export`:** Backend solvers, routers, DRC sweeps, and GDSII writers remain intact and are next in the execution pipeline.