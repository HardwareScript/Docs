# HardwareScript v0.1.9: Connection Interface Routing (CIR)

**Document Type:** Core Subsystem Reference  
**Status:** Implementation Ready with Integration Plan  
**Target Version:** v0.1.9  
**Focus:** Interface Abstraction Layer, Integration with Existing Port Escape, Extensible Cost Evaluation  

---

## 1. Architecture Philosophy: Integration Over Replacement

CIR extends the existing routing infrastructure rather than replacing it. The current system has:
- **Port Escape Engine** (`port_escape.rs`): Cardinal directions, edge offsets, boundary docking
- **EntityGraph**: Component metadata, spatial queries, substrate tracking
- **TopologicalRouter**: Continuous-coordinate pathfinding with Minkowski inflation
- **RoutingParams**: Explicit penalty weights for cost calculation

CIR adds a **semantic layer above these systems** to handle:
- Multi-interface component footprints (components with redundant contacts)
- Interface capability tracking (current limits, bandwidth, thermal)
- Routing intent abstraction (clock trees, power meshes, differential pairs)
- Connection candidate optimization (choosing best interface pair before routing)

---

## 2. The Five-Tier Abstraction Hierarchy

```
  Logical Terminal ──► Routing Intent ──► Physical Interface ──► Access Regions ──► Existing Port Escape
       (Netlist)         (Signal Class)      (Component Boundary)    (Approach Zones)    (Cardinal/Edge)
```

### 2.1 Tier Descriptions

1.  **Logical Terminal:** An entity in the netlist representing an electrical connection point (e.g., `CPU.clk`, `VDD`).

2.  **Routing Intent:** The non-geometric descriptor of the net's signal class and performance targets (e.g., `Clock`, `HighCurrent`, `DifferentialPair`).

3.  **Physical Interface:** The concrete geometric boundary or boundaries in the layout where connections are permitted. A component may expose multiple interfaces for the same logical pin (e.g., left VDD contact and right VDD contact).

4.  **Access Regions:** A prioritized collection of derived geometric zones outlining how the pathfinder can approach the interface. Generated once and cached as immutable data owned by the interface.

5.  **Port Escape Strategy:** The existing port escape engine (`CardinalPort`, `EdgeOffset`) that computes exact docking coordinates. **This is not replaced—it becomes the implementation strategy selected based on interface geometry.**

### 2.2 Geometry-Specific Escape Strategies

Different interface geometries use different escape strategies:

| **Interface Geometry** | **Escape Strategy** | **Implementation** |
|:---|:---|:---|
| Rectangle | Cardinal Ports (N, S, E, W) | Existing `calculate_rect_escape()` |
| Polygon | Boundary Segment Selection | New: per-edge normal derivation |
| Circle | Radial Projection | Existing `calculate_circular_escape()` |
| Waveguide | Face-Normal Alignment | Future: photonics extension |

**Key Insight:** `CardinalPort` is not the architecture—it's one implementation strategy for rectangular footprints. CIR provides the abstraction that selects the appropriate strategy based on geometry type.

---

## 2. Declarative Language Interface

The updated syntax separates logical design intent from routing-style overrides [SYNTAX-UNIFICATION-PHILOSOPHY.md].

### 2.1 Intent & Profile Declarations (PDK)
The PDK defines standard **Routing Intents** and associates them with cost evaluation weights.

```hardware
# foundry_pdk.hw

profile Silicon_180nm:
    technology: "ASIC"
    
    constraints:
        min_trace_width: 180nm
        min_clearance: 180nm
        min_via_diameter: 220nm

    # Declarative Routing Intents (The Designer's Policy)
    intent Clock:
        routing_style: symmetrical_tree
        layer_preference: [metal3, metal4]
        max_crosstalk_db: -40dB
        cost_weights:
            via_penalty: 50000      # Strong via avoidance for timing-critical
            direction_penalty: 5      # Weak preference for preferred direction
            crosstalk_penalty: 100    # Strong crosstalk avoidance

    intent PowerRail:
        routing_style: trunk_mesh
        max_current_density: 35A_mm2
        cost_weights:
            via_penalty: 10           # Vias are cheap for power (more is better)
            tight_clearance_penalty: 5
```

### 2.2 Component Interface & Capability Declarations
Components declare physical interfaces, assign them logical pins, and map their physical capabilities.

```hardware
# standard_cells.hw

component NAND_Gate:
    pins: [A, B, OUT, VDD, GND]
    
    layout:
        shape: Rectangle(1.2um, 2.4um)
        
        interfaces:
            # Multiple redundant physical interfaces map to the same VDD logical terminal
            # Router will select best candidate based on ConnectionCandidate scoring
            vdd_contact_left:
                pin: VDD
                geometry: Edge([x: -0.4um, y: 1.0um] to [x: -0.2um, y: 1.0um])
                capability: [CarryCurrent(100uA), CarryHeat(0.01uW_K)]

            vdd_contact_right:
                pin: VDD
                geometry: Edge([x: 0.2um, y: 1.0um] to [x: 0.4um, y: 1.0um])
                capability: [CarryCurrent(100uA)]
            
            # Signal pins typically have single interface (no redundancy)
            a_input:
                pin: A
                geometry: Point([x: -0.6um, y: 0.5um])
                capability: [SignalBandwidth(10GHz)]
```

---

## 3. EntityGraph Integration

### 3.1 Ownership Model

Interfaces are owned and indexed by the EntityGraph, not stored as standalone entities:

```
EntityGraph
  ├── Component (ComponentId: 42)
  │     ├── InterfaceId: 101  (VDD left contact)
  │     ├── InterfaceId: 102  (VDD right contact)
  │     └── InterfaceId: 103  (GND contact)
  │
  └── Interface Database
        ├── 101 → PhysicalInterface { geometry, capabilities, cached_normals, ... }
        ├── 102 → PhysicalInterface { geometry, capabilities, cached_normals, ... }
        └── 103 → PhysicalInterface { ... }
```

**Allocation Strategy:**
- `InterfaceId` is allocated by EntityGraph during component registration
- Each component stores `Vec<InterfaceId>` mapping logical pins to physical interfaces
- Interface Database is a dense arena indexed by InterfaceId
- Spatial queries use existing R*-tree; interfaces are not separately indexed

### 3.2 Type Definitions

```rust
// hwc-engine/src/geometry_router/connection_interface.rs

use hwc_engine::{Point3D, BoundingBox};
use crate::geometry_router::port_escape::{CardinalPort, EdgeOffset}; // Reuse existing

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct InterfaceId(pub u32);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct ComponentId(pub u32);

/// Fixed-point integer normal vector scaled by 10^9 (Q31-like)
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct Normal2D {
    pub x: i32, // x component * 10^9
    pub y: i32, // y component * 10^9
}

impl Normal2D {
    pub const SCALE: i64 = 1_000_000_000;

    #[inline(always)]
    pub fn new(x: i32, y: i32) -> Self {
        Self { x, y }
    }
}

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub enum InterfaceGeometry {
    Point(Point3D),
    Edge { start: Point3D, end: Point3D },
    Polygon(Vec<Point3D>),
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum Orientation {
    Derived,
    Explicit(Normal2D),
    None,
}

/// Interface capability with constraint derivation
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum InterfaceCapability {
    /// Maximum current through this interface
    /// Derives: minimum trace width, minimum via count, allowed materials
    CarryCurrent { max_ma: u32 },
    
    /// Signal bandwidth constraint
    /// Derives: maximum trace length, impedance matching requirements
    SignalBandwidth { max_ghz: u32 },
    
    /// Thermal dissipation capability
    /// Derives: thermal via requirements, keepout zones
    CarryHeat { max_uw_k: u32 },
    
    /// Optical coupling (future: photonics)
    OpticalCoupling { wavelength_nm: u32 },
}

impl InterfaceCapability {
    /// Convert capability into routing constraint
    /// Consumed by: Router (trace width), DRC (clearance), EM Solver, IR Drop
    pub fn derive_constraint(&self, db: &dyn RoutingDatabase) -> DerivedConstraint {
        match self {
            Self::CarryCurrent { max_ma } => {
                let current_density = db.get_max_current_density(); // From material DB
                let min_width_nm = (*max_ma as i64 * 1_000_000) / current_density;
                DerivedConstraint::MinimumTraceWidth(min_width_nm)
            }
            Self::SignalBandwidth { max_ghz } => {
                let max_length_mm = 3_000 / max_ghz; // Speed of light approximation
                DerivedConstraint::MaximumTraceLength(max_length_mm * 1_000_000)
            }
            Self::CarryHeat { max_uw_k } => {
                DerivedConstraint::ThermalViaRequired
            }
            Self::OpticalCoupling { .. } => {
                DerivedConstraint::None // Future work
            }
        }
    }
}

/// Constraints derived from interface capabilities
#[derive(Debug, Clone, Copy)]
pub enum DerivedConstraint {
    MinimumTraceWidth(i64),
    MaximumTraceLength(i64),
    ThermalViaRequired,
    None,
}
```

### 3.3 Cached Geometric Properties

To avoid recomputing geometry on every routing query, derived properties are cached as immutable data owned by the interface:

```rust
pub struct PhysicalInterface {
    pub id: InterfaceId,
    pub component_id: ComponentId,
    pub geometry: InterfaceGeometry,
    pub capabilities: Vec<InterfaceCapability>,
    pub routing_intent: RoutingIntent,
    
    // ── Cached Derived Properties (computed once, immutable) ──
    pub boundary_normals: Arc<Vec<Normal2D>>,      // Pre-computed outward normals
    pub access_regions: Arc<Vec<AccessRegion>>,    // Pre-computed approach zones
    pub expanded_keepouts: Arc<Vec<BoundingBox>>,  // Minkowski-inflated boundaries
}
```

**Performance Rationale:**
- Routing queries are hot-path operations (millions of calls per design)
- Computing normals/access regions on-demand is wasteful
- Immutable caching enables lock-free concurrent access via `Arc`

### 3.4 Pure Integer Square Root with Polygon Support

All geometric vector operations utilize a deterministic, non-overflowing integer square root to ensure bit-identical builds across platforms:

```rust
// hwc-engine/src/geometry/math.rs

/// Deterministic Newton-Heron integer square root
#[inline(always)]
pub fn integer_sqrt(n: u128) -> u128 {
    if n == 0 { return 0; }
    let mut x = n;
    let mut y = (x + 1) >> 1;
    while y < x {
        x = y;
        y = (x + n / x) >> 1;
    }
    x
}

impl InterfaceGeometry {
    /// Derive the outward perpendicular normal vector without floating-point math
    /// Returns Vec<Normal2D> because polygons have one normal per edge
    pub fn derive_normals(&self, orientation: Orientation) -> Vec<Normal2D> {
        match orientation {
            Orientation::None => vec![],
            Orientation::Explicit(normal) => vec![normal],
            Orientation::Derived => match self {
                InterfaceGeometry::Point(_) => vec![],
                InterfaceGeometry::Edge { start, end } => {
                    vec![Self::compute_edge_normal(start, end)]
                }
                InterfaceGeometry::Polygon(vertices) => {
                    // Compute normal for each edge of the polygon
                    vertices
                        .windows(2)
                        .map(|pair| Self::compute_edge_normal(&pair[0], &pair[1]))
                        .collect()
                }
            },
        }
    }
    
    /// Compute outward normal for a single edge using signed area method
    fn compute_edge_normal(start: &Point3D, end: &Point3D) -> Normal2D {
        let dx = (end.x - start.x) as i128;
        let dy = (end.y - start.y) as i128;
        let d2 = (dx * dx + dy * dy) as u128;
        
        if d2 == 0 {
            return Normal2D::new(0, 0); // Degenerate edge
        }
        
        let len = integer_sqrt(d2) as i128;
        
        // Perpendicular vector: rotate 90° counterclockwise, then scale
        let nx = ((-dy * Normal2D::SCALE) / len) as i32;
        let ny = ((dx * Normal2D::SCALE) / len) as i32;
        
        Normal2D { x: nx, y: ny }
    }
}
```

---

## 4. The Extensible Cost Evaluation Architecture

To keep the router core clean and independent of evolving analysis modules (timing, electromigration, thermal, crosstalk), the cost calculation uses a **composable evaluator system** with zero-cost abstraction.

### 4.1 Design Principle: Plugin Architecture Without Dynamic Dispatch

**Problem:** `Vec<Box<dyn CostEvaluator>>` adds virtual function overhead in the routing inner loop (millions of calls).

**Solution:** Use an enum-based dispatch with SmallVec optimization for the common case:

```rust
// hwc-engine/src/geometry_router/pathfinding/cost_evaluator.rs

use hwc_engine::Point3D;
use crate::ir::query_engine::RoutingDatabase;
use smallvec::SmallVec;

/// Cost evaluator types (enum dispatch, not trait objects)
#[derive(Debug, Clone)]
pub enum CostEvaluator {
    /// Base geometric movement cost
    GeometricMove,
    
    /// Via transition penalty
    ViaTransition { penalty: i64 },
    
    /// Direction penalty (against preferred layer direction)
    Direction { penalty: i64 },
    
    /// Thermal hotspot avoidance
    Thermal { threshold_mk: i64, penalty: i64 },
    
    /// Electromigration risk zones
    Electromigration { current_density_limit: i64, penalty: i64 },
    
    /// Crosstalk risk (parallel trace proximity)
    Crosstalk { min_spacing_nm: i64, penalty: i64 },
    
    /// Reference plane void crossing
    ReferenceVoid { penalty: i64 },
}

impl CostEvaluator {
    /// Evaluate cost at a specific position
    /// Inlined for zero-cost abstraction
    #[inline]
    pub fn evaluate(&self, db: &dyn RoutingDatabase, pos: Point3D) -> i64 {
        match self {
            Self::GeometricMove => 1,
            
            Self::ViaTransition { penalty } => *penalty,
            
            Self::Direction { penalty } => *penalty,
            
            Self::Thermal { threshold_mk, penalty } => {
                let temp = db.get_local_temperature_at(pos);
                if temp > *threshold_mk { *penalty } else { 0 }
            }
            
            Self::Electromigration { current_density_limit, penalty } => {
                let density = db.get_current_density_at(pos);
                if density > *current_density_limit { *penalty } else { 0 }
            }
            
            Self::Crosstalk { min_spacing_nm, penalty } => {
                let spacing = db.get_nearest_parallel_trace_distance(pos);
                if spacing < *min_spacing_nm { *penalty } else { 0 }
            }
            
            Self::ReferenceVoid { penalty } => {
                if db.is_in_reference_void(pos) { *penalty } else { 0 }
            }
        }
    }
}

/// Cost composer with stack-allocated storage for common case
/// SmallVec<T, 8> avoids heap allocation for up to 8 evaluators
pub struct CostComposer {
    evaluators: SmallVec<[CostEvaluator; 8]>,
}

impl CostComposer {
    pub fn new() -> Self {
        Self {
            evaluators: smallvec![CostEvaluator::GeometricMove],
        }
    }
    
    pub fn with_evaluator(mut self, evaluator: CostEvaluator) -> Self {
        self.evaluators.push(evaluator);
        self
    }
    
    /// Accumulate total path cost across all evaluators
    /// Fully inlined, no virtual dispatch
    #[inline]
    pub fn calculate_step_cost(&self, db: &dyn RoutingDatabase, pos: Point3D) -> i64 {
        self.evaluators
            .iter()
            .map(|eval| eval.evaluate(db, pos))
            .sum()
    }
}

impl Default for CostComposer {
    fn default() -> Self {
        Self::new()
    }
}
```

---

## 5. Connection Candidate Optimization

Before routing, the system selects the best **interface pair** to connect. This separates topology optimization from pathfinding.

### 5.1 The Connection Candidate Problem

Consider routing VDD from a processor to a power mesh:

```
Processor (4 VDD contacts)          Power Mesh (200 via candidates)
    ├── VDD_left                        ├── Via @(10mm, 5mm)
    ├── VDD_right                       ├── Via @(12mm, 5mm)
    ├── VDD_top                         ├── Via @(10mm, 7mm)
    └── VDD_bottom                      └── ... 197 more
```

**Naive approach:** Try routing all 4 × 200 = 800 combinations  
**Optimized approach:** Select best interface pair first (e.g., VDD_right → Via @(12mm, 5mm)), then route once

### 5.2 ConnectionCandidate Type

```rust
/// Represents a potential connection between two interfaces
#[derive(Debug, Clone)]
pub struct ConnectionCandidate {
    pub source_interface: InterfaceId,
    pub sink_interface: InterfaceId,
    pub estimated_cost: i64,       // Euclidean distance + capability mismatch penalty
    pub requires_via: bool,
    pub layer_span: i64,
}

impl ConnectionCandidate {
    /// Heuristic scoring for candidate selection
    pub fn score(&self, routing_intent: &RoutingIntent) -> i64 {
        let mut cost = self.estimated_cost;
        
        if self.requires_via {
            cost += 10_000; // Via penalty
        }
        
        if routing_intent.is_critical_path && self.layer_span > 1 {
            cost += 50_000; // Strong via penalty for timing-critical nets
        }
        
        cost
    }
}

/// Select best interface pairs before routing
pub fn select_connection_candidates(
    db: &dyn RoutingDatabase,
    net_id: NetId,
    routing_intent: &RoutingIntent,
) -> Vec<ConnectionCandidate> {
    let terminals = db.get_net_terminals(net_id);
    let mut candidates = Vec::new();
    
    // Generate all possible interface pairs
    for i in 0..terminals.len() {
        for j in (i + 1)..terminals.len() {
            let source_interfaces = db.get_component_interfaces(terminals[i]);
            let sink_interfaces = db.get_component_interfaces(terminals[j]);
            
            for src in source_interfaces {
                for snk in sink_interfaces {
                    candidates.push(ConnectionCandidate::estimate(db, src, snk));
                }
            }
        }
    }
    
    // Sort by score and return top N
    candidates.sort_by_key(|c| c.score(routing_intent));
    candidates.truncate(10); // Keep only best 10 candidates
    candidates
}
```

**Benefits:**
- Reduces pathfinding search space by 10-100×
- Enables power mesh optimization (redundant via placement)
- Supports BGA escape routing (choose best ball from array)
- Critical for chiplet and TSV interconnect

## 6. Salsa-Tracked Queries with Cached Properties

Access regions are pre-computed and stored, not regenerated on every query:

```rust
// hwc-compiler/src/ir/query_engine.rs

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct RoutingIntent {
    pub name: String,
    pub is_critical_path: bool,
    pub target_impedance_milliohms: u32,
}

/// Query interface by ID (returns cached structure)
#[salsa::tracked]
pub fn get_physical_interface(
    db: &dyn RoutingDatabase,
    interface_id: InterfaceId,
) -> Arc<PhysicalInterface> {
    // PhysicalInterface already contains pre-computed access_regions
    // No geometry recomputation happens here
    db.interface_database().get(interface_id).clone()
}

/// Generate access regions during interface creation (called once)
pub fn create_interface_with_access_regions(
    geometry: InterfaceGeometry,
    capabilities: Vec<InterfaceCapability>,
    trace_width_nm: i64,
    escape_stub_length_nm: i64,
) -> PhysicalInterface {
    // Compute normals once
    let boundary_normals = Arc::new(geometry.derive_normals(Orientation::Derived));
    
    // Generate access regions once
    let mut access_regions = Vec::new();
    match &geometry {
        InterfaceGeometry::Edge { start, end } => {
            access_regions.push(AccessRegion::generate(
                start,
                end,
                &boundary_normals[0],
                escape_stub_length_nm,
                trace_width_nm,
            ));
        }
        InterfaceGeometry::Polygon(vertices) => {
            // Generate one access region per polygon edge
            for (i, normal) in boundary_normals.iter().enumerate() {
                let start = &vertices[i];
                let end = &vertices[(i + 1) % vertices.len()];
                access_regions.push(AccessRegion::generate(
                    start,
                    end,
                    normal,
                    escape_stub_length_nm,
                    trace_width_nm,
                ));
            }
        }
        InterfaceGeometry::Point(_) => {
            // Point interfaces have no directional access region
        }
    }
    
    PhysicalInterface {
        geometry,
        capabilities,
        boundary_normals,
        access_regions: Arc::new(access_regions),
        // ... other fields
    }
}
```

---

## 7. Implementation Phases

### Phase 1: Core Integration (Critical Path)

**Goal:** Bridge CIR with existing systems without breaking current functionality

1. **EntityGraph Extension**
   - Add `InterfaceId` allocation mechanism
   - Add `interface_database: HashMap<InterfaceId, PhysicalInterface>` to EntityGraph
   - Extend component metadata to track `Vec<InterfaceId>` per logical pin

2. **Interface-Port Escape Bridge**
   - Modify existing `calculate_rect_escape()` to accept `InterfaceId` parameter
   - Add geometry-based dispatch: Rectangle → Cardinal, Polygon → Edge, Circle → Radial
   - Reuse all existing `CardinalPort` and `EdgeOffset` logic

3. **Access Region Generation**
   - Implement `AccessRegion::generate()` with cached normals
   - Store as immutable `Arc<Vec<AccessRegion>>` on PhysicalInterface
   - Use during candidate selection, not during routing inner loop

4. **Capability Constraint Derivation**
   - Implement `InterfaceCapability::derive_constraint()`
   - Wire into existing `RoutingParams` struct (extend, don't replace)
   - Add validation: emit compile error if `CarryCurrent(100mA)` but trace too narrow

**Success Criteria:**
- Existing tests still pass
- Can declare interfaces in `.hw` syntax
- Router queries EntityGraph for interfaces
- Capabilities enforce trace width constraints

### Phase 2: Optimization (Performance & Quality)

**Goal:** Add advanced features that improve routing quality

1. **Connection Candidate Selection**
   - Implement `ConnectionCandidate` type
   - Add `select_connection_candidates()` pre-routing pass
   - Measure: routing time reduction for components with redundant contacts

2. **Routing Intent System**
   - Implement `RoutingIntent` enum (Clock, Power, Signal, DifferentialPair)
   - Wire into cost composer for intent-aware penalties
   - Add syntax: `route CPU.clk to RAM.clk with intent: Clock`

3. **Cost Composer Integration**
   - Replace `RoutingParams` penalty fields with `CostComposer`
   - Use enum dispatch (not trait objects) for zero overhead
   - Add thermal and EM evaluators (optional, gated by features)

4. **Salsa Query Optimization**
   - Add `#[salsa::tracked]` queries for interface lookups
   - Measure cache hit rate during incremental compilation

**Success Criteria:**
- 10-100× reduction in pathfinding candidates for multi-contact components
- Cost evaluation has zero runtime overhead vs. current system
- Incremental compilation invalidates only affected interfaces

### Phase 3: Advanced Features (Future Work)

**Goal:** Enable next-generation capabilities

1. **Thermal-Aware Routing**
   - Implement `CostEvaluator::Thermal` with temperature gradient fields
   - Wire into physics engine for junction temperature estimation

2. **Electromigration Analysis**
   - Implement `CostEvaluator::Electromigration` with current density limits
   - Integrate with existing EM checker from v0.1.8

3. **Timing-Driven Routing**
   - Add delay estimation to cost function
   - Wire into static timing analysis (STA) engine

4. **Photonics Extension**
   - Add `InterfaceGeometry::WaveguideFace`
   - Implement modal coupling constraint evaluator

**Success Criteria:**
- Thermal hotspots avoided in power delivery networks
- EM violations caught at synthesis time, not post-layout
- Critical paths routed with minimal skew

---

## 8. Key Design Decisions & Rationale

### Decision 1: Integrate, Don't Replace
**Rationale:** The existing port escape system works. CIR adds abstraction, not replacement. Avoids rewriting 1000+ lines of tested code.

### Decision 2: Enum Dispatch for Cost Evaluators
**Rationale:** Plugin architecture without virtual call overhead. Keeps extensibility while maintaining performance in routing inner loop.

### Decision 3: Cached Geometric Properties
**Rationale:** Normals and access regions are immutable once computed. Caching eliminates millions of redundant calculations during routing.

### Decision 4: EntityGraph Ownership
**Rationale:** Interfaces belong to components. EntityGraph is already the spatial database. Adding a parallel ownership model would create synchronization bugs.

### Decision 5: Connection Candidate Pre-Selection
**Rationale:** Reduces pathfinding search space by 10-100×. Critical for power meshes, BGA escape, TSV arrays, and chiplet interconnect.

---

## Conclusion

This updated architecture:
- **Integrates** with existing port escape, EntityGraph, and TopologicalRouter systems
- **Extends** capabilities without breaking current functionality
- **Optimizes** for zero-cost abstraction (enum dispatch, cached properties)
- **Enables** advanced features (thermal-aware, EM-aware, timing-driven routing)

The specification is now **implementation-ready** with a clear phased approach. Phase 1 focuses on integration; Phases 2-3 add value incrementally. Real-world routing benchmarks will validate the architecture and guide future optimizations.

---
**Version:** v0.1.9  
**Last Updated:** 2026-07-20  
**Status:** Locked for Implementation  
 
---