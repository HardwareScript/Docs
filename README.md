# Hardware Script — Documentation

This directory contains the **official language specifications, architecture guides, vision documents, and technical references** for the [Hardware Script](https://github.com/hardware-script) language and its compiler toolchain.

Each version directory tracks the evolving documentation for a specific release — from the earliest Python-based proof-of-concept through the current Rust compiler with 2.5D analytic routing, HSM desktop application, and ecosystem tooling.

---

## Directory Structure

```
Docs/
├── README.md              ← This file
│
├── v0.0.1/  — Pre-Alpha Research & Prototyping
│   ├── ARCHITECTURE.md          System architecture planning
│   ├── LANGUAGE-SPEC.md         Initial language specification
│   ├── MODULARITY-AND-DOCUMENTATION.md   Module system design
│   ├── SILICON-DESIGN.md        Silicon/ASIC design considerations
│   ├── SPECIFICATION.md         Technical specification
│   ├── TOOLING.md               Development toolchain planning
│   └── WHITE-PAPER.md           Original white paper / thesis
│
├── v0.1/  — MVP Release (Python Proof-of-Concept)
│   ├── README.md                Project overview & quick start
│   ├── INDEX.md                 Documentation index with learning paths
│   ├── GETTING-STARTED.md       Tutorial — first hardware design
│   ├── LANGUAGE-SPEC.md         Syntax reference & EBNF grammar
│   ├── ARCHITECTURE.md          Compiler implementation guide
│   └── ACHIEVEMENTS.md          Success report & verification
│
├── v0.1.1/  — Refinement & File Standardization
│   ├── 3D-ASSET-MAPPING.md      3D model ↔ component mapping
│   ├── ARCHITECTURE-COMPLETE.md  Complete architecture documentation
│   └── FILE-EXTENSIONS.md       File extension convention decisions
│
├── v0.1.2/  — Compiler Foundations & Design Philosophy
│   Core philosophy documents:
│   ├── COMPILER-ARCHITECTURE-PHILOSOPHY.md  Design principles
│   ├── LANGUAGE-DESIGN-PHILOSOPHY.md        Language design rationale
│   ├── FIRST-PRINCIPLES-IMPLEMENTATION.md   First-principles approach
│   ├── HYPER-LEAN-ARCHITECTURE.md           Minimalist architecture
│   ├── VISION-AND-CORE-IDEOLOGY.md          Project vision & ideology
│   ├── HISTORICAL-CONTEXT-AND-LESSONS.md    Lessons from prior EDA
│   │
│   Technical architecture:
│   ├── MULTI-LEVEL-IR-ARCHITECTURE.md       Multi-level IR pipeline
│   ├── ARCHITECTURAL-VALIDATION-AND-IR-PIPELINE.md  IR validation
│   ├── RUST-IMPLEMENTATION-STRATEGY.md      Rust migration strategy
│   ├── COORDINATE-SYSTEM-ABSTRACTION.md     Coordinate abstraction
│   ├── COMPILER-FOUNDATIONS-AND-PERFORMANCE.md  Perf foundations
│   │
│   Routing & layout:
│   ├── MANHATTAN-ROUTING-STRATEGY.md        Manhattan routing design
│   ├── ROUTING-GEOMETRY-RULES.md            Geometry constraint rules
│   ├── DETERMINISTIC-ROUTING-IMPLEMENTATION.md  Deterministic routing
│   ├── NETLIST-AND-ROUTING-PHILOSOPHY.md    Netlist model design
│   │
│   Materials & physics:
│   ├── MATERIAL-DATABASE-ARCHITECTURE.md    Materials DB architecture
│   ├── MATERIALS-DATABASE-PHILOSOPHY.md     Materials DB rationale
│   ├── PHYSICS-TO-CONSTRAINTS.md            Physics → routing constraints
│   │
│   Cross-cutting:
│   ├── ACADEMIC-POSITIONING.md              Academic / research context
│   ├── SILICON-AND-PCB-UNIVERSALITY.md      Unified PCB + silicon
│   ├── ERROR-HANDLING-PHILOSOPHY.md         Error handling design
│   ├── EXPORT-OPTIMIZATION-AND-POLISH.md    Export pipeline optimization
│   ├── IDE-INTEGRATION-AND-LAUNCH.md        IDE / LSP integration
│   ├── FILE-EXTENSION-DEBATE-RESOLUTION.md  File extension resolution
│   ├── FILE-EXTENSIONS-VALIDATION.md        Extension validation
│   ├── DOCUMENTATION-UPGRADE-SUMMARY.md     Documentation audit
│   ├── FUTURE-PROOF-VALIDATION.md           Future-proofing analysis
│   ├── TECHNICAL-CHALLENGES-AND-TRADEOFFS.md  Known trade-offs
│   └── CORE-PROBLEMS-AND-RETHINKS.md        Fundamental re-examinations
│
├── v0.1.3/  — Routing, Physics & Ecosystem
│   ├── COMPILER-INTERNALS.md       Compiler internals & pipeline
│   ├── LANGUAGE-SPEC.md            Language specification
│   ├── ROUTING-AND-PHYSICS.md      Routing engine & physics
│   ├── EXPORTS-AND-ASSETS.md       3D export & asset pipeline
│   ├── ECOSYSTEM.md                Ecosystem & tooling overview
│   ├── VISION.md                   Project vision
│   └── v0.1.3.1/                   Sub-release refinements
│
├── v0.1.4/  — Constraint-Aware Routing & Standard Library
│   ├── COMPILER-INTERNALS.md            Updated compiler internals
│   ├── LANGUAGE-SPEC.md                 Updated language spec
│   ├── ROUTING-AND-PHYSICS.md           Constraint-aware routing
│   ├── EXPORTS-AND-ASSETS.md            Updated export pipeline
│   ├── ECOSYSTEM.md                     Ecosystem expansion
│   ├── ERROR-HANDLING-PHILOSOPHY.md     Error handling evolution
│   ├── STANDARD-LIBRARY-ARCHITECTURE.md  Stdlib architecture
│   ├── GENERIC-MEASUREMENT-ARCHITECTURE.md  Generic measurement system
│   └── v0.1.4.1/                        Sub-release refinements
│
├── v0.1.5/  — Analytic Router & Native Simulation
│   ├── COMPILER-INTERNALS.md            2.5D analytic routing engine
│   ├── LANGUAGE-SPEC.md                  Updated language spec
│   ├── ROUTING-AND-PHYSICS.md            Shape-based pathfinding
│   ├── ERROR-HANDLING-PHILOSOPHY.md      Error handling for analytic path
│   ├── NATIVE-SIMULATION-AND-SPICE.md    Built-in SPICE simulation
│   ├── FUTURE-PARADIGMS-AND-SCALE.md     Future scalability vision
│   ├── SOFTWARE-HERITAGE.md              Design lineage & influences
│   ├── THE-DOMAIN-BOUNDARY.md            Domain boundary analysis
│   ├── SAFE-VOXELGRID-REFACTORING.md     Safe voxel grid refactoring
│   ├── EDApipeline-fix.md                EDA pipeline corrections
│   ├── architectural-decision.md         Key architectural decisions
│   ├── Consolidation_ideas.md            Consolidation proposals
│   └── v0.1.5.1/                         Sub-release refinements
│
├── v0.1.6/  — Syntax Unification & Assembly Completeness
│   The authoritative documentation version. Supersedes v0.1.1–v0.1.5 specs.
│   ├── MANIFESTO.md                      Syntax Unification Manifesto
│   ├── LANGUAGE-SPEC.md                  Unified language specification
│   ├── SYNTAX-UNIFICATION-PHILOSOPHY.md  Unification design rationale
│   │
│   Architecture:
│   ├── ARCHITECTURAL-AUDIT-AND-ROADMAP.md    Full architecture audit
│   ├── LAZY-REALIZATION-ARCHITECTURE.md       Lazy geometry evaluation
│   ├── HPM-ARCHITECTURE.md                   HPM (Hardware Package Manager)
│   ├── HPM-BOOTSTRAP-ARCHITECTURE.md          HPM bootstrap strategy
│   ├── LOD-AND-SCALE-ARCHITECTURE.md          Level-of-detail & scale
│   │
│   Physics & materials:
│   ├── MATERIAL-DRIVEN-PHYSICS-VALIDATION.md  Material-driven validation
│   ├── MATERIAL-INTEGRITY-AND-VERTICAL-INTERCONNECTS.md  VIAs & material integrity
│   ├── PHYSICAL-VIABILITY-PROTOCOL.md          Physical viability checks
│   │
│   Routing & layout:
│   ├── BIT-BLIT-UNROLLER-IMPLEMENTATION.md    Bit-blit unroller design
│   ├── MERGED-REGIONS-AND-COLLISION-WAIVER.md  Collision waiver system
│   ├── WAIVER-SYSTEM.md                        Design rule waivers
│   ├── COMPONENT-GEOMETRIC-CONTROLLER.md       Component geometry control
│   │
│   Logic & behavior:
│   ├── LOGICAL-BEHAVIOR-GUIDE.md               Logical behavior specification
│   ├── COMPLETED-FEATURES.md                    Feature completion log
│   ├── COMPILER-CHANGES.md                      Compiler change log
│   │
│   Devices & contracts:
│   ├── DEVICE-CONTRACTS-AND-INDUSTRY-TRANSFORMATION.md  Device contracts
│   ├── MATURITY-AND-EXPANSION-VISION.md         Maturity model & growth
│   ├── AUTHORITY-AND-LIBRARY-ARCHITECTURE.md    Authority & trust system
│   ├── STANDARD-LIBRARY-EXAMPLES.md             Standard library examples
│   │
│   Viewport & tooling:
│   ├── HARDWARE-SCRIPT-STUDIO-VIEWPORT.md       Studio viewport design
│   ├── THE-PULSE-ARCHITECTURE.md                Pulse architecture (event system)
│   ├── NATIVE-DIAGNOSTIC-SYSTEM.md              Built-in diagnostics
│   └── ERROR-HANDLING-PHILOSOPHY.md             Error handling for unified syntax
│
└── v0.1.7/  — 2.5D Manufacturing & HSM Desktop
    Current release documentation:
    ├── HARDWARE-SCRIPT-MONITOR.md          HSM desktop app specification
    ├── ROUTING-AND-MANUFACTURING-ARCHITECTURE.md  Routing ↔ manufacturing
    ├── ADVANCED-ROUTING-AND-MANUFACTURING-ARCHITECTURE.md  Advanced routing
    ├── MANUFACTURING-PROCESS-DNA.md        Manufacturing process description
    ├── Z-AXIS-ABSTRACTION.md               Z-axis elevation abstraction
    ├── Z_FIGHTING_FIX.md                   Z-fighting (depth conflict) fix
    └── TSV-STATUS.md                       Through-Silicon Via status
```

---

## Version Summaries

### v0.0.1 — Pre-Alpha Research & Prototyping
The earliest design documents. Explores the core thesis — can hardware be described in human-readable text and compiled deterministically into manufacturing files? Contains the original white paper, initial language spec, and architecture planning for what would become the discrete 3D tensor grid system.

### v0.1 — MVP Release (Python Proof-of-Concept)
The first working release. A ~180-line Python compiler that transforms `.hw` text files into Gerber, Blender Python scripts, and OBJ 3D models. Proves the core concept: single-source text → multi-format EDA output. Includes a complete documentation index, getting-started tutorial, and achievements report.

**Key innovation**: Discrete 3D tensor grid with Bresenham line interpolation and O(1) collision detection.

### v0.1.1 — Refinement & File Standardization
Consolidates the file extension conventions and completes the architecture documentation. Focuses on standardizing `.hw` file format and 3D asset mapping for visualization.

### v0.1.2 — Compiler Foundations & Design Philosophy
The most philosophically rich version. 30+ documents covering:
- **Design philosophy** — Language design, compiler architecture, first-principles approach, hyper-lean architecture
- **Technical architecture** — Multi-level IR pipeline, Rust implementation strategy, coordinate system abstraction
- **Routing** — Manhattan routing strategy, deterministic routing, geometry rules, netlist philosophy
- **Materials & physics** — Materials database architecture, physics-to-constraints mapping
- **Cross-cutting** — Academic positioning, PCB + silicon universality, IDE integration, error handling, file extensions

**Key transition**: Beginning of the Rust migration from the Python prototype.

### v0.1.3 — Routing, Physics & Ecosystem
First structured documentation set with 6 core documents:
- Compiler internals, language spec, routing & physics, exports & assets, ecosystem, vision
- Sub-release refinements in `v0.1.3.1/`

### v0.1.4 — Constraint-Aware Routing & Standard Library
Expands on v0.1.3 with constraint-aware routing, standard library architecture, and a generic measurement system. Error handling philosophy evolves to match the growing compiler complexity.

### v0.1.5 — Analytic Router & Native Simulation
Major technical pivot:
- 2.5D **shape-based pathfinding** (analytic router replacing voxel-crawling)
- **Native SPICE simulation** integration
- Safe voxel grid refactoring strategy
- Domain boundary analysis and software heritage documentation
- EDA pipeline fixes and consolidation proposals

### v0.1.6 — Syntax Unification & Assembly Completeness
The **authoritative documentation version**. Supersedes all v0.1.1–v0.1.5 language specifications. Key additions:
- **Syntax Unification Manifesto** — The definitive language design document
- **HPM Architecture** — Hardware Package Manager design
- **Lazy Realization** — On-demand geometry evaluation for performance
- **Material-Driven Physics Validation** — Physics checked against material properties
- **Device Contracts** — Industry-transformation device specification model
- **Maturity & Expansion Vision** — Long-term growth roadmap
- **Waiver System** — Design rule violation waivers
- **Pulse Architecture** — Event-driven system design
- **Studio Viewport** — HSM visual interface design

### v0.1.7 — 2.5D Manufacturing & HSM Desktop
Current release. Focuses on:
- **HSM Desktop** — The Hardware Script Monitor Tauri v2 application
- **Manufacturing Architecture** — Manufacturing process DNA, routing-to-manufacturing pipeline
- **Z-Axis Abstraction** — Layer-to-nanometer elevation resolution
- **Z-Fighting Fix** — GPU depth conflict resolution for coincident surfaces
- **TSV Status** — Through-Silicon Via feasibility analysis

---

## How to Use These Documents

1. **New to Hardware Script?** → Start at `v0.1/README.md` for the MVP overview, then `v0.1/GETTING-STARTED.md` for a tutorial.

2. **Learning the language?** → Read `v0.1.6/MANIFESTO.md` (the definitive language design document) and `v0.1.6/LANGUAGE-SPEC.md` (the current authoritative spec).

3. **Understanding architecture?** → Read `v0.1.6/ARCHITECTURAL-AUDIT-AND-ROADMAP.md` for the full architecture review, then drill into specific topics:
   - **Routing**: `v0.1.5/ROUTING-AND-PHYSICS.md` → `v0.1.7/ROUTING-AND-MANUFACTURING-ARCHITECTURE.md`
   - **Compiler pipeline**: `v0.1.6/COMPILER-CHANGES.md` → `v0.1.6/LAZY-REALIZATION-ARCHITECTURE.md`
   - **Materials**: `v0.1.6/MATERIAL-DRIVEN-PHYSICS-VALIDATION.md`
   - **HSM Desktop**: `v0.1.7/HARDWARE-SCRIPT-MONITOR.md`

4. **Exploring design philosophy?** → Read `v0.1.2/` documents:
   - `COMPILER-ARCHITECTURE-PHILOSOPHY.md`
   - `LANGUAGE-DESIGN-PHILOSOPHY.md`
   - `FIRST-PRINCIPLES-IMPLEMENTATION.md`
   - `VISION-AND-CORE-IDEOLOGY.md`

5. **Contributing to the compiler?** → Read `v0.1.6/COMPILER-CHANGES.md`, `v0.1.5/ROUTING-AND-PHYSICS.md`, and the `ROADMAP/` directory for implementation plans.

6. **Working on the HSM desktop app?** → Read `v0.1.7/HARDWARE-SCRIPT-MONITOR.md` and the `hsm/` source directory.

---

## Key Concepts Referenced Across Versions

| Concept | First Introduced | Description |
|---------|-----------------|-------------|
| **Discrete 3D Tensor Grid** | v0.0.1 | Physical space as a 3D matrix of voxels for O(1) collision |
| **Bresenham Line Interpolation** | v0.1 | Automatic trace filling between waypoints |
| **Multi-Level IR** | v0.1.2 | Intermediate representation pipeline (HSX → HWSB → ...) |
| **Manhattan Routing** | v0.1.2 | Grid-aligned routing with 90° trace bends |
| **Minkowski Obstacle Inflation** | v0.1.5 | AABB + Width/2 + Clearance for O(1) collision |
| **Planar Lock** | v0.1.5 | Lock A* pathfinder to specific Z-height |
| **Syntax Unification** | v0.1.6 | Unified grammar across all language constructs |
| **Lazy Realization** | v0.1.6 | Deferred geometry evaluation until actually needed |
| **Device Contracts** | v0.1.6 | Formal interface contracts for components |
| **HPM** | v0.1.6 | Hardware Package Manager — dependency resolution |
| **Aggregate Engine** | v0.1.7 | Rust Data Factory + JS Beauty Layer for HSM |
| **PourID Picking** | v0.1.7 | Babylon.js raycast → Rust PourID → DeviceBinding |
| **Z-Axis Abstraction** | v0.1.7 | Dual-paradigm (Assembly / Layer-Based) Z resolution |

---

## Cross-Cutting Documents

| Document | Location | Scope |
|----------|----------|-------|
| `HARDWARE-SCRIPT-MONITOR.md` | `v0.1.7/` | HSM desktop application specification and architecture |
| `MANIFESTO.md` | `v0.1.6/` | Syntax Unification Manifesto — authoritative language design |
| `VISION-AND-CORE-IDEOLOGY.md` | `v0.1.2/` | Foundational project vision |
| `ACADEMIC-POSITIONING.md` | `v0.1.2/` | Research context and academic relevance |
| `WHITE-PAPER.md` | `v0.0.1/` | Original thesis / whitepaper |

---

## Relationship to ROADMAP/

The `Docs/` and `ROADMAP/` directories serve complementary purposes:

| Directory | Purpose | Content |
|-----------|---------|---------|
| **`Docs/`** | Specification & Reference | Language specs, architecture guides, philosophy, vision — *what* the system is and *why* it's designed that way |
| **`ROADMAP/`** | Implementation Tracking | Task checklists, gap analyses, system plans, bug trackers — *how* to build it and *what's left* to build |

For implementation plans, build checklists, and gap analyses, see the [`ROADMAP/`](../ROADMAP/README.md) directory.