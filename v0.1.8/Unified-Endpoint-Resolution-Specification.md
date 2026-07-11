# Unified Endpoint Resolution Architecture Specification

**Document Type:** Architectural Specification  
**Status:** Proposed for v0.1.8 Implementation  
**Subsystem:** Route Endpoint Resolution (Semantic Lowering Phase)  

---

## Executive Summary

The current v0.1.6 router performs **string-based name lookups** during the routing phase, which:
1. Degrades performance (repeated hash table lookups during hot-path routing)
2. Violates separation of concerns (router handles semantic resolution)
3. Causes silent failures (string lookups return `None` without error context)
4. Creates architectural smell (overloaded function parameters)

This specification defines a **Compile-Time Endpoint Resolution** architecture where all route endpoints are resolved to stable, typed **EntityIds** during the Semantic Lowering Phase, before the router executes.

---

## Problem Statement

### Current Architecture (v0.1.6) - String-Based Runtime Resolution

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Parse Phase                                              │
│    Input:  "route M1.gate to VIN_Pad"                       │
│    Output: Route { from: "M1.gate", to: "VIN_Pad" }        │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. Semantic Lowering Phase                                  │
│    ❌ NO ENDPOINT RESOLUTION                                │
│    Route objects passed through unchanged                   │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. Routing Phase                                            │
│    ❌ Router calls get_pour_bbox_for_pin("VIN_Pad", "")    │
│    ❌ String lookup during hot-path routing                │
│    ❌ Silent failure when entity not found                 │
│    ❌ No error context (span, suggestion)                  │
└─────────────────────────────────────────────────────────────┘
```

### Issues with This Approach

1. **Performance Degradation**: String hash lookups executed repeatedly during A* pathfinding
2. **Silent Failures**: `if let Ok` pattern swallows lookup failures
3. **Poor Error Messages**: No source span, no typo suggestions
4. **Architectural Smell**: Function parameter overloading (`component: "VIN_Pad", pin: ""`)
5. **Tight Coupling**: Router depends on string-based entity naming


---

## Proposed Architecture - Compile-Time EntityId Resolution

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Parse Phase                                              │
│    Input:  "route M1.gate to VIN_Pad"                       │
│    Output: RouteSpec {                                      │
│              from: RouteEndpointSpec::ComponentPin {        │
│                component: "M1", pin: "gate"                 │
│              },                                             │
│              to: RouteEndpointSpec::SpaceEntity {           │
│                name: "VIN_Pad"                              │
│              }                                              │
│            }                                                │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. Semantic Lowering Phase (space_builder.rs)              │
│    ✅ Resolve "M1.gate" → EntityId(0x1F)                   │
│    ✅ Resolve "VIN_Pad" → EntityId(0x2A)                   │
│    ✅ Validate entities exist                              │
│    ✅ Generate actionable errors with source spans         │
│    Output: ResolvedRoute {                                 │
│              from: EntityId(0x1F),                         │
│              to: EntityId(0x2A)                            │
│            }                                                │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. Routing Phase (automatic.rs)                            │
│    ✅ Query entity_graph.get_geometry_bbox(0x1F) → O(1)   │
│    ✅ Query entity_graph.get_geometry_bbox(0x2A) → O(1)   │
│    ✅ No string lookups                                    │
│    ✅ Guaranteed valid EntityIds                           │
└─────────────────────────────────────────────────────────────┘
```

### Benefits of This Approach

1. **O(1) Resolution Speed**: Integer-indexed lookups during routing hot-path
2. **Strict Error Propagation**: Failures halt compilation with exact source spans
3. **Early Validation**: Typos caught during lowering, not routing
4. **Zero-Magic Design**: No parameter overloading or string conventions
5. **Loose Coupling**: Router decoupled from entity naming

---

## Implementation Specification

### Phase 1: AST Modifications (hwc-parser)

**File:** `crates/hwc-parser/src/ast/space.rs`

Define a typed endpoint enum in the AST:

```rust
/// Route endpoint specification in the parsed AST
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub enum RouteEndpointSpec {
    /// Component pin reference: `M1.gate`, `R1.A`, etc.
    ComponentPin {
        component_name: CompactString,
        pin_name: CompactString,
        span: Span,
    },
    
    /// Space-level entity reference: `VIN_Pad`, `GND_Rail`, etc.
    SpaceEntity {
        name: CompactString,
        span: Span,
    },
}
```


Update the `Route` AST structure:

```rust
/// Route statement in the parsed AST
#[derive(Debug, Clone)]
pub struct Route {
    pub from: RouteEndpointSpec,  // ✅ Changed from PinReference
    pub to: RouteEndpointSpec,    // ✅ Changed from PinReference
    pub width: Option<Expression>,
    pub layer: Option<LayerId>,
    pub strategy: Option<RoutingStrategy>,
    pub pattern: Option<PatternInstantiation>,
    pub current_limit_ac: Option<CurrentLimitAc>,
    pub span: Span,
}
```

**Parser Changes:**

```rust
// Old (v0.1.6):
// route M1.gate to VIN_Pad
//       ^^^^^^^ parsed as PinReference { component: "M1", pin: "gate" }
//                 ^^^^^^^ parsed as PinReference { component: "VIN_Pad", pin: "" }

// New (v0.1.8):
// route M1.gate to VIN_Pad
//       ^^^^^^^ parsed as RouteEndpointSpec::ComponentPin { "M1", "gate" }
//                 ^^^^^^^ parsed as RouteEndpointSpec::SpaceEntity { "VIN_Pad" }
```

Parser logic detects whether the endpoint contains a `.` to distinguish types:
- Contains `.` → `ComponentPin { component, pin }`
- No `.` → `SpaceEntity { name }`

---

### Phase 2: EntityId System (hwc-engine)

**File:** `crates/hwc-engine/src/entity_graph/mod.rs`

Define stable EntityIds for all routing targets:

```rust
/// Stable, unique identifier for any entity in the design
/// Generated via cryptographic hash of semantic path + type
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct EntityId(pub u64);

impl EntityId {
    pub fn new(id: u64) -> Self {
        EntityId(id)
    }
    
    pub fn raw(&self) -> u64 {
        self.0
    }
}
```

Add registration methods to EntityGraph:

```rust
impl EntityGraph {
    /// Register a component pin and return its EntityId
    pub fn register_component_pin(
        &mut self,
        component_name: &str,
        pin_name: &str,
        bbox: BoundingBox,
        net_id: NetId,
    ) -> EntityId {
        let id = self.compute_entity_id(&format!("pin:{}:{}", component_name, pin_name));
        self.entity_registry.insert(id, EntityData {
            entity_type: EntityType::ComponentPin,
            bbox,
            net_id: Some(net_id),
            name: format!("{}.{}", component_name, pin_name).into(),
        });
        id
    }
    
    /// Register a space-level pour/pad and return its EntityId
    pub fn register_space_entity(
        &mut self,
        name: &str,
        bbox: BoundingBox,
        net_id: NetId,
        layer_z: i64,
    ) -> EntityId {
        let id = self.compute_entity_id(&format!("space:{}", name));
        self.entity_registry.insert(id, EntityData {
            entity_type: EntityType::SpacePour,
            bbox,
            net_id: Some(net_id),
            name: name.into(),
        });
        id
    }
```

