# Hierarchical Routing Database Architecture

## Problem Statement

When compiling hierarchical designs (e.g., CMOS inverter composed of PMOS and NMOS child cells), the compiler was mixing route data from multiple sources without proper tracking:

1. **Child instance routes** (transformed from child spaces)
2. **Parent-level interconnects** (created by parent space)
3. **Legacy entity_graph data** (stale/duplicate segments)

This caused:
- False connectivity violations ("Net VDD has 3 disconnected components")
- Data corruption (wrong coordinates in validation)
- Poor error messages (couldn't identify source of problem)
- No way to distinguish between child and parent routes

## Root Cause Analysis

### Data Flow Before Fix

```
PMOS_Cell (child)                    NMOS_Cell (child)
├── VDD_Rail pour                    ├── GND_Rail pour  
├── Via_Source contact               ├── Via_Source contact
└── route: Via_Source → VDD_Rail     └── route: Via_Source → GND_Rail

              ↓ Hierarchical Flattening ↓

Inverter_Cell (parent)
├── entity_graph.routed_segments ← Child routes (transformed coordinates)
├── analytic_routes ← Parent routes (Out, In interconnects)
└── Validation reads BOTH sources ← PROBLEM: Duplicate/conflicting data
```

### The Bug

1. Child spaces work perfectly when compiled standalone
2. When instantiated in parent, child routes are transformed and stored in `entity_graph.routed_segments()`
3. Parent creates new routes stored in `analytic_routes`
4. Validation reads from BOTH sources
5. **But**: Child routes in `entity_graph` had stale/wrong coordinates
6. **Result**: Connectivity checker sees multiple disconnected islands

### Why "Just Skip" Was Wrong

The initial fix was to skip `entity_graph.routed_segments()` if `analytic_routes` exists. Problems:
1. Loses data - child instance internal routing disappears
2. No provenance tracking - can't tell where routes came from
3. No validation - can't detect missing parent-level connections
4. Not scalable - what about 3+ level hierarchies?

## Proper Solution: Hierarchical Routing Database

### Architecture

```rust
pub struct HierarchicalRoutingDatabase {
    /// Child instance routes (immutable after flattening)
    /// Key: (instance_name, net_id)
    child_instance_routes: FxHashMap<(CompactString, NetId), Vec<TraceSegment>>,
    
    /// Parent-level interconnects
    parent_interconnects: Vec<AnalyticTrace>,
    
    /// Provenance tracking
    route_provenance: FxHashMap<RouteId, RouteSource>,
}

pub enum RouteSource {
    ChildInstance { instance: String, original_net: String },
    ParentLevel { from_entity: String, to_entity: String },
}
```

### Key Principles

1. **Clear Separation**: Child and parent routes stored separately
2. **Immutability**: Child routes never change after flattening
3. **Provenance**: Every segment knows its source
4. **Lazy Merging**: Routes only merged for validation, never stored merged
5. **Validation First**: Check hierarchical connectivity before merging

### Data Flow After Fix

```
PMOS_Cell                           NMOS_Cell
├── VDD route                       ├── GND route
└── Out route                       └── Out route

          ↓ Hierarchical Flattening ↓

routing_database.register_child_routes(
    instance: "PMOS_Inst",
    net: VDD,
    segments: [transformed routes]
)

routing_database.register_child_routes(
    instance: "NMOS_Inst", 
    net: GND,
    segments: [transformed routes]
)

          ↓ Parent Routing ↓

routing_database.register_parent_route(
    trace: Out interconnect,
    from: "PMOS_Inst.Out_Pad",
    to: "NMOS_Inst.Out_Pad"
)

          ↓ Validation ↓

routing_database.validate_hierarchical_connectivity()
  → Detects: VDD exists in PMOS_Inst but no parent route
  → Error: "Net VDD isolated in child instance 'PMOS_Inst'"
  → Suggestion: "Add route PMOS_Inst.VDD_Rail to external source"
```

## API Usage

### During Hierarchical Flattening

```rust
// In space_instance.rs when flattening child space
for route in child_space.get_routes() {
    let transformed_segments = transform_to_parent_coords(route.segments);
    
    parent_space.routing_database.register_child_routes(
        instance_name.clone(),
        parent_net_id,  // After net remapping
        original_net_name.clone(),
        transformed_segments,
    );
}
```

### During Parent Routing

```rust
// In routing builder when creating parent interconnects
space.routing_database.register_parent_route(
    trace,
    from_entity_name,
    to_entity_name,
);
```

### During Validation

```rust
// Before connectivity checking
if let Err(errors) = space.routing_database.validate_hierarchical_connectivity() {
    for error in errors {
        eprintln!("{}", error);  // Rich error messages with suggestions
    }
    return Err(...);
}

// Get unified view for connectivity checker
let segments = space.routing_database.get_connectivity_view();
for seg in segments {
    // seg.source tells you if it's from child or parent
    // seg.route_id for debugging
}
```

## Error Messages

### Before (Cryptic)

```
❌ Net 'VDD' has 3 disconnected conductive components
X-gap: 0 nm, Y-gap: 0 nm
Suggested fix: Add a pour or route to bridge the gap
```

### After (Helpful)

```
❌ HIERARCHICAL CONNECTIVITY ERROR

Net NetId(3) exists in 1 child instances but has NO parent-level routing:

  • Instance 'PMOS_Inst' (original net: 'VDD')

These instances have internal routing but are NOT connected to each other.

Suggested fix:
  Add a parent-level route statement in your space to connect these instances.
  Example:
    route PMOS_Inst.VDD_Rail to ExternalVDD:
        net: VDD
        width: 200nm
        layer: metal1
```

## Benefits

### 1. Data Integrity
- Child routes preserved exactly as compiled
- No data loss during hierarchical composition
- No coordinate corruption

### 2. Debugging
- Know source of every route segment
- Track provenance through hierarchy
- Statistics for route distribution

### 3. Validation
- Detect missing parent connections
- Validate before merging data
- Clear error messages with suggestions

### 4. Scalability
- Works with arbitrary hierarchy depth
- A → B → C → D all tracked properly
- Each level maintains separation

### 5. Type Safety
- Can't accidentally mix child/parent routes
- Compiler enforces proper registration
- Clear API boundaries

## Migration Path

### Phase 1: Add Database (Complete)
- [x] Create `routing_database.rs`
- [x] Add to `HardwareSpace`
- [x] Integrate into engine exports

### Phase 2: Update Hierarchical Flattening
- [ ] Modify `space_instance.rs` to use routing database
- [ ] Register child routes during flattening
- [ ] Preserve provenance information

### Phase 3: Update Parent Routing
- [ ] Modify routing builder to register parent routes
- [ ] Track from/to entities
- [ ] Add to database instead of analytic_routes directly

### Phase 4: Update Validation
- [ ] Call `validate_hierarchical_connectivity()` first
- [ ] Use `get_connectivity_view()` for checking
- [ ] Improve error messages

### Phase 5: Deprecate Old System
- [ ] Remove entity_graph.routed_segments usage in validation
- [ ] Keep for backward compatibility (export function)
- [ ] Add deprecation warnings

## Testing Strategy

### Unit Tests
- [x] Empty database
- [x] Single child instance (should pass)
- [x] Multiple instances, same net, no parent route (should fail)
- [x] Multiple instances, same net, with parent route (should pass)
- [ ] Three-level hierarchy
- [ ] Net remapping edge cases

### Integration Tests
- [ ] PMOS_Cell standalone (should pass)
- [ ] NMOS_Cell standalone (should pass)
- [ ] CMOS Inverter without power routing (should fail with good error)
- [ ] CMOS Inverter with power routing (should pass)

### Regression Tests
- [ ] All existing tests still pass
- [ ] Performance unchanged
- [ ] Memory usage reasonable

## Performance Considerations

### Memory
- **Before**: Duplicated data in entity_graph + analytic_routes
- **After**: Separated but no duplication
- **Impact**: Slight increase for provenance tracking, but cleaner

### Speed
- **Validation**: One extra pass to check hierarchical connectivity
- **Impact**: <1ms for typical designs, negligible
- **Benefit**: Catches errors earlier, saves debug time

### Scalability
- **Small designs** (1-10 instances): No difference
- **Medium designs** (10-100 instances): Better organization helps
- **Large designs** (100+ instances): Provenance tracking essential

## Future Enhancements

### 1. Visual Debugging
```rust
routing_database.export_to_graphviz()
  → Generates visual hierarchy diagram
  → Color-codes child vs parent routes
  → Shows connectivity gaps
```

### 2. Auto-Fix Suggestions
```rust
routing_database.suggest_fixes()
  → Proposes specific route statements
  → Estimates required trace lengths
  → Checks clearances
```

### 3. Design Rule Checking
```rust
routing_database.check_hierarchical_drc()
  → Validates child-parent interface points
  → Checks via stack alignment
  → Verifies layer transitions
```

### 4. Optimization
```rust
routing_database.optimize_hierarchical_routing()
  → Suggests power grid improvements
  → Identifies redundant vias
  → Proposes consolidation opportunities
```

## Conclusion

The Hierarchical Routing Database solves the fundamental problem of managing routing data across hierarchical designs by:

1. Maintaining clear separation between child and parent routes
2. Tracking provenance for every segment
3. Validating connectivity before merging data
4. Providing helpful error messages

This architecture is:
- ✅ **Correct**: No data loss or corruption
- ✅ **Scalable**: Works with arbitrary hierarchy depth
- ✅ **Maintainable**: Clear API and separation of concerns
- ✅ **Debuggable**: Full provenance tracking
- ✅ **User-Friendly**: Helpful error messages with suggestions

No more "just skipping" - every piece of data is accounted for and validated properly.
