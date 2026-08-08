# HardwareScript Error Architecture Refactoring

**Document Type:** Compiler Architecture Refactoring  
**Status:** 🚧 **PLANNED** - Modular Error System  
**Target:** v0.3.0 (6-month roadmap)  
**Companion Docs:** `Clippy-Level-Error-Intelligence.md`, `Parasitic-Error-Intelligence.md`  
**Author:** HardwareScript Core Team  
**Date:** 2024

---

## Problem Statement

**Current:** `ir/errors.rs` is 856 lines containing ALL error types (compilation, routing, placement, materials, devices, shapes) in one monolithic enum.

**Issue:** Violates modular architecture. Other crates follow the pattern:
- `alignment/error.rs` - owns alignment errors
- `conversions/error.rs` - owns conversion errors
- `symbol_table/error.rs` - owns symbol errors

But `ir/` has one giant file for everything instead of domain-specific error files.

---

## Target Architecture

### File Structure

```
ir/
├── errors.rs                    # Top-level IrError enum (~50 lines)
├── compilation/
│   ├── mod.rs
│   └── errors.rs                # CompilationError enum (~250 lines, C** codes)
├── routing/
│   ├── mod.rs
│   ├── errors.rs                # RoutingError enum (~200 lines, R** codes)
│   └── ...
├── placement/
│   ├── mod.rs
│   ├── errors.rs                # PlacementError enum (~150 lines, P** codes)
│   └── ...
├── materials/
│   ├── mod.rs
│   └── errors.rs                # MaterialError enum (~50 lines, M** codes)
├── devices/
│   ├── mod.rs
│   └── errors.rs                # DeviceError enum (~50 lines, D** codes)
└── shapes/
    ├── mod.rs
    └── errors.rs                # ShapeError enum (~50 lines, S** codes)
```

### Error Code Organization by Prefix

| Prefix | Domain | File Location | Line Count |
|--------|--------|---------------|------------|
| C** | Compilation | `ir/compilation/errors.rs` | ~250 |
| R** | Routing | `ir/routing/errors.rs` | ~200 |
| P** | Placement/Physical | `ir/placement/errors.rs` | ~150 |
| M** | Materials | `ir/materials/errors.rs` | ~50 |
| D** | Devices | `ir/devices/errors.rs` | ~50 |
| S** | Shapes | `ir/shapes/errors.rs` | ~50 |
| CIR** | Circuit/Interface | `ir/routing/errors.rs` | (included in routing) |

---

## Implementation Pattern

### Top-Level Wrapper (ir/errors.rs)

```rust
//! Top-level IR error wrapper

use thiserror::Error;
use miette::Diagnostic;

#[derive(Error, Diagnostic, Debug)]
pub enum IrError {
    #[error(transparent)]
    Compilation(#[from] compilation::CompilationError),
    
    #[error(transparent)]
    Routing(#[from] routing::RoutingError),
    
    #[error(transparent)]
    Placement(#[from] placement::PlacementError),
    
    #[error(transparent)]
    Material(#[from] materials::MaterialError),
    
    #[error(transparent)]
    Device(#[from] devices::DeviceError),
    
    #[error(transparent)]
    Shape(#[from] shapes::ShapeError),
    
    #[error(transparent)]
    Symbol(#[from] crate::SymbolError),
}
```

### Domain-Specific Error Example (ir/routing/errors.rs)

```rust
//! Routing subsystem errors

use compact_str::CompactString;
use miette::Diagnostic;
use thiserror::Error;

#[derive(Error, Diagnostic, Debug)]
pub enum RoutingError {
    #[error("No route path found from {from_pin} to {to_pin}")]
    #[diagnostic(
        code(R16),
        url("https://docs.hw-script.org/errors/R16"),
        help("Check that components are within routing reach, or reduce congestion.")
    )]
    NoPathFound {
        net: CompactString,
        from_pin: CompactString,
        to_pin: CompactString,
    },

    #[error("Route for net '{net}' has no waypoints")]
    #[diagnostic(
        code(R20),
        url("https://docs.hw-script.org/errors/R20"),
        help("Add waypoints or use auto-routing")
    )]
    EmptyRoute { net: CompactString },

    // All other R** errors...
}

// Detail structs for boxed error data
#[derive(Debug, Clone)]
pub struct DisconnectedNetDetails {
    pub route_name: CompactString,
    pub waypoint_type: CompactString,
    pub waypoint_pos: String,
    pub distance: String,
    pub pin_name: CompactString,
    pub pin_pos: String,
}
```

---

## Migration Strategy

### Phase 1: Create Parallel Structure (v0.2.1)

1. Create new error files alongside existing `ir/errors.rs`
2. Move error variants to domain files
3. Keep old file for backward compatibility
4. Add deprecation warnings

**No breaking changes yet.**

### Phase 2: Update Call Sites (v0.2.2-v0.2.5)

**Pattern matches require updates:**

```rust
// OLD (v0.2.0)
match err {
    IrError::NoPathFound { net, .. } => { /* handle */ }
}

// NEW (v0.3.0)
match err {
    IrError::Routing(RoutingError::NoPathFound { net, .. }) => { /* handle */ }
}
```

**The `?` operator works automatically:**

```rust
// No changes needed - auto-converts via #[from]
fn route_thing() -> Result<(), RoutingError> {
    Err(RoutingError::NoPathFound { /* ... */ })
}

fn compile() -> Result<(), IrError> {
    route_thing()?; // Auto-converts RoutingError -> IrError
    Ok(())
}
```

**Migration tool:** Create a script to find and update pattern matches:

```bash
# Find all pattern matches that need updating
rg "IrError::(NoPathFound|EmptyRoute|InvalidRouteExpression)" --type rust
```

### Phase 3: Delete Monolith (v0.3.0)

1. Remove old `ir/errors.rs` after all call sites updated
2. Update documentation
3. Confirm all tests pass

---

## Benefits

### 1. Maintainability
- Each error file ~50-250 lines (manageable)
- Easy to find errors by domain
- Clear ownership boundaries

### 2. Consistency
- Matches existing pattern in codebase
- `alignment/error.rs`, `conversions/error.rs`, `symbol_table/error.rs` all follow this pattern
- IR subsystem should too

### 3. Scalability
- Adding new error to routing: edit `ir/routing/errors.rs` (~200 lines)
- Not: edit monolithic file (856+ lines and growing)

### 4. Parallel Development
- Multiple developers can add errors to different domains without merge conflicts
- Clear module boundaries

---

## Testing Strategy

### Test Coverage

```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn routing_error_converts_to_ir_error() {
        let routing_err = RoutingError::NoPathFound {
            net: "TestNet".into(),
            from_pin: "A".into(),
            to_pin: "B".into(),
        };
        
        let ir_err: IrError = routing_err.into();
        
        assert!(matches!(ir_err, IrError::Routing(_)));
    }
    
    #[test]
    fn question_mark_operator_auto_converts() -> Result<(), IrError> {
        fn returns_routing_error() -> Result<(), RoutingError> {
            Err(RoutingError::EmptyRoute {
                net: "Test".into(),
            })
        }
        
        // Should auto-convert via ?
        returns_routing_error()?;
        Ok(())
    }
}
```

### Integration Tests

1. Compile existing test suite - all should pass
2. Check error messages unchanged (user-facing)
3. Verify diagnostic output format preserved

---

## Timeline

| Phase | Version | Weeks | Description |
|-------|---------|-------|-------------|
| **Phase 1** | v0.2.1 | 2 | Create parallel structure, no breaking changes |
| **Phase 2** | v0.2.2-v0.2.5 | 8 | Incremental call site migration |
| **Phase 3** | v0.3.0 | 1 | Delete old monolith, release |

**Total: 11 weeks** (can be done alongside error intelligence work)

---

## Open Questions

1. **Should detail structs move too?**
   - `DisconnectedNetDetails`, `GeometricCollisionDetails`, etc.
   - **Answer:** Yes, move to same file as parent error

2. **What about shared error details?**
   - Some detail structs used by multiple error types
   - **Answer:** Keep in `ir/errors.rs` or create `ir/error_details.rs`

3. **How to handle SymbolError from outside crate?**
   - Already works via `#[error(transparent)]`
   - No changes needed

---

## Conclusion

This refactoring brings the IR error system in line with the rest of the codebase's modular architecture. It's a straightforward mechanical refactoring that improves maintainability without changing user-facing behavior.

**When combined with the error intelligence upgrades (Clippy-Level and Parasitic), v0.3.0 will have:**
- ✅ Modular, maintainable error architecture (this document)
- ✅ Intelligent multi-span errors with causality (Clippy-Level)
- ✅ Rich parasitic validation reporting (Parasitic)

All three work together to create a world-class error system.


---

## Detailed Implementation Guide

### Section 1: Current State Analysis

#### Current ir/errors.rs Breakdown (856 lines)

| Error Code Range | Count | Category | Lines |
|------------------|-------|----------|-------|
| C12, C25, C27-C45 | 25 | Compilation | ~280 |
| R12, R15-R44 | 30 | Routing | ~220 |
| P12, P42-P46 | 8 | Placement/Physical | ~160 |
| M01 | 1 | Materials | ~15 |
| D01 | 1 | Devices | ~20 |
| S15-S16 | 2 | Shapes | ~25 |
| CIR1 | 1 | Circuit/Interface | ~15 |
| Detail Structs | 5 | Supporting | ~85 |
| Imports/Docs | - | Boilerplate | ~36 |

#### Call Site Analysis

```bash
# Find all IrError usage in codebase
$ rg "IrError::" --type rust -c

crates/hwc-compiler/src/ir/space_builder.rs:47
crates/hwc-compiler/src/ir/routing/automatic.rs:23
crates/hwc-compiler/src/ir/routing/manual.rs:18
crates/hwc-compiler/src/ir/placement/component.rs:31
crates/hwc-compiler/src/ir/placement/pour.rs:28
# ... ~70 total call sites
```

**Estimated migration impact:** ~70 call sites across 25 files

---

### Section 2: Step-by-Step Implementation

#### Step 2.1: Create ir/compilation/errors.rs

```rust
//! Compilation phase error types (C** error codes)

use compact_str::CompactString;
use miette::{Diagnostic, SourceSpan};
use thiserror::Error;

/// Errors that occur during the compilation phase (C** codes)
#[derive(Error, Diagnostic, Debug)]
pub enum CompilationError {
    #[error("No space definition found in program")]
    #[diagnostic(
        code(C28),
        url("https://docs.hw-script.org/errors/C28"),
        help("Hardware Script files must contain a 'space Name:' block")
    )]
    NoSpaceDefinition,

    #[error("Space dimensions not specified")]
    #[diagnostic(
        code(C31),
        url("https://docs.hw-script.org/errors/C31"),
        help("Add 'dimensions: <width> by <height>' or similar to the space definition")
    )]
    MissingDimensions {
        #[label("dimensions must be specified for this space")]
        span: SourceSpan,
    },

    #[error("Space origin not specified for '{space_name}'")]
    #[diagnostic(
        code(C32),
        url("https://docs.hw-script.org/errors/C32"),
        help("{hint}")
    )]
    MissingOrigin {
        space_name: String,
        hint: String,
    },

    #[error("Space grid not specified")]
    #[diagnostic(
        code(C33),
        url("https://docs.hw-script.org/errors/C33"),
        help("Add 'grid: <x> by <y> by <z>' or 'resolution: <step>' to the space definition")
    )]
    MissingGrid,

    #[error("Invalid coordinate: [{0}, {1}, {2}] exceeds grid bounds")]
    #[diagnostic(
        code(C27),
        url("https://docs.hw-script.org/errors/C27"),
        help("Coordinates must be within the grid dimensions specified in the space definition")
    )]
    InvalidCoordinate(usize, usize, usize),

    #[error("Pin reference not found: {component}.{pin}")]
    #[diagnostic(
        code(C12),
        url("https://docs.hw-script.org/errors/C12"),
        help("Verify component name and pin name are correct")
    )]
    PinNotFound {
        component: CompactString,
        pin: String,
    },

    #[error("Invalid resolution: {value} nm")]
    #[diagnostic(
        code(C39),
        url("https://docs.hw-script.org/errors/C39"),
        help("Resolution must be a positive value, e.g., `resolution: 1nm`.")
    )]
    InvalidResolution { value: i64 },

    #[error("Profile '{name}' not found")]
    #[diagnostic(
        code(C40),
        url("https://docs.hw-script.org/errors/C40"),
        help("Import or declare profile '{name}'.")
    )]
    ProfileNotFound { name: CompactString },

    #[error("Logic synthesis failed: {message}")]
    #[diagnostic(
        code(C41),
        url("https://docs.hw-script.org/errors/C41"),
        help("Check the logic definition for errors.")
    )]
    LogicSynthesisFailed { message: String },

    #[error("Compilation aborted: {error_count} previous error{}", if *error_count == 1 { "" } else { "s" })]
    #[diagnostic(
        code(C42),
        url("https://docs.hw-script.org/errors/C42"),
        help("Fix the preceding errors and try again.")
    )]
    CompilationAborted { error_count: usize },

    #[error("Missing profile constraint: {field}")]
    #[diagnostic(
        code(C34),
        url("https://docs.hw-script.org/errors/C34"),
        help("The profile must specify '{field}'. Add it to the profile definition.")
    )]
    MissingProfileConstraint { field: String },

    #[error("Invalid expression in loop: {0}")]
    #[diagnostic(
        code(C43),
        url("https://docs.hw-script.org/errors/C43"),
        help("Check the expression syntax and ensure all variables are defined")
    )]
    InvalidExpression(String),

    #[error("Dimensional unit mismatch in expression: {expression}")]
    #[diagnostic(
        code(C44),
        url("https://docs.hw-script.org/errors/C44"),
        help("Cannot {operation} incompatible units. {detail}\n\nHardware Script enforces dimensional type safety to prevent physically nonsensical operations.")
    )]
    DimensionalUnitMismatch {
        expression: String,
        operation: String,
        detail: String,
    },

    #[error("Invalid Z elevation: {value} nm")]
    #[diagnostic(
        code(C25),
        url("https://docs.hw-script.org/errors/C25"),
        help("Z elevations cannot be negative. The evaluated height is below the origin.\n\nEvaluated: {value} nm\n\nUse z: 0mm or higher, or adjust expressions so the result is non-negative.")
    )]
    NegativeLayerIndex {
        value: i64,
        #[label("negative layer index evaluated here")]
        span: SourceSpan,
    },

    #[error("Circular spatial dependency detected: {path}")]
    #[diagnostic(
        code(C35),
        url("https://docs.hw-script.org/errors/C35"),
        help(
            "Component positioning forms a cycle (e.g., A depends on B, and B depends on A). \
              Hardware Script requires a directed acyclic graph (DAG) for spatial placement."
        )
    )]
    CircularReference { path: String },

    #[error("ASIC compile failed: {message}")]
    #[diagnostic(
        code(C36),
        url("https://docs.hw-script.org/errors/C36"),
        help("Under ASIC technology, all physical constraints must be explicitly declared. No implicit defaults are permitted.\n\n{hint}")
    )]
    MissingAsicConstraint { message: String, hint: String },

    #[error("ASIC compile failed: {message}")]
    #[diagnostic(
        code(C36),
        url("https://docs.hw-script.org/errors/C36"),
        help("Under ASIC technology, all physical constraints must be explicitly declared. No implicit defaults are permitted.\n\n{hint}")
    )]
    MissingAsicConstraintWithSpan {
        message: String,
        hint: String,
        #[label("missing constraint here")]
        span: SourceSpan,
    },

    #[error("Material '{material}' is missing required property '{property}'")]
    #[diagnostic(
        code(C37),
        url("https://docs.hw-script.org/errors/C37"),
        help("Declare the required property '{property}' in the material definition for '{material}'")
    )]
    MissingPhysicalProperty {
        material: CompactString,
        property: String,
    },

    #[error("Active net '{net}' is missing electrical specifications")]
    #[diagnostic(
        code(C38),
        url("https://docs.hw-script.org/errors/C38"),
        help("Add explicit voltage, current, or classification properties to the net '{net}': {detail}")
    )]
    MissingElectricalSpecification { net: CompactString, detail: String },

    #[error("Unit conversion error: {message}")]
    #[diagnostic(
        code(C45),
        url("https://docs.hw-script.org/errors/C45"),
        help("Check that the unit matches the expected dimension for this field.")
    )]
    UnitConversion {
        message: String,
        #[label("invalid unit here")]
        span: Option<SourceSpan>,
    },

    #[error("Placement error: {0}")]
    PlacementError(String),
}
```

#### Step 2.2: Update ir/compilation/mod.rs

```rust
//! IR compilation phase

pub mod errors;

pub use errors::CompilationError;

// Existing compilation module code...
```



#### Step 2.3: Create ir/routing/errors.rs

```rust
//! Routing subsystem error types (R** error codes)

use compact_str::CompactString;
use miette::{Diagnostic, SourceSpan};
use thiserror::Error;

/// Errors that occur during routing phase (R** codes)
#[derive(Error, Diagnostic, Debug)]
pub enum RoutingError {
    #[error("Bridge material transition invalid: {from_material} -> {to_material}: {reason}")]
    #[diagnostic(
        code(R15),
        url("https://docs.hw-script.org/errors/R15"),
        help("Check bridge material compatibility")
    )]
    BridgeValidationFailed {
        from_material: CompactString,
        to_material: CompactString,
        reason: String,
    },

    #[error("No route path found from {from_pin} to {to_pin}")]
    #[diagnostic(
        code(R16),
        url("https://docs.hw-script.org/errors/R16"),
        help("No valid path exists for this net. With the legalization-only workflow, this is a terminal error — there is no rip-up or retry mechanism.\n\nCheck that components are within routing reach, or reduce congestion.")
    )]
    NoPathFound {
        net: CompactString,
        from_pin: CompactString,
        to_pin: CompactString,
    },

    #[error("Failed to resolve coordinate expression '{coordinate_str}': {reason}")]
    #[diagnostic(
        code(R17),
        url("https://docs.hw-script.org/errors/R17"),
        help("Verify coordinate syntax and that all variables are defined")
    )]
    CoordinateResolutionFailed {
        coordinate_str: String,
        reason: String,
    },

    #[error("Failed to resolve layer '{layer_name}': {reason}")]
    #[diagnostic(
        code(R18),
        url("https://docs.hw-script.org/errors/R18"),
        help("Verify layer name exists in the profile stackup")
    )]
    StackupResolutionFailed {
        layer_name: CompactString,
        reason: String,
    },

    #[error("Placement constraint violation: {message}")]
    #[diagnostic(
        code(R19),
        url("https://docs.hw-script.org/errors/R19"),
        help("Check placement constraints for this component")
    )]
    PlacementConstraint { message: String, component: String },

    #[error("Route for net '{net}' has no waypoints")]
    #[diagnostic(
        code(R20),
        url("https://docs.hw-script.org/errors/R20"),
        help("Add waypoints or use auto-routing")
    )]
    EmptyRoute { net: CompactString },

    #[error("Invalid route expression '{expression}': {reason}")]
    #[diagnostic(
        code(R21),
        url("https://docs.hw-script.org/errors/R21"),
        help("Check route expression syntax")
    )]
    InvalidRouteExpression { expression: String, reason: String },

    #[error("Manual route missing required field: {missing_field}")]
    #[diagnostic(
        code(R22),
        url("https://docs.hw-script.org/errors/R22"),
        help("Add the missing field to the route definition")
    )]
    ManualRouteIncomplete { missing_field: String },

    #[error("Disconnected net: {}", .0.route_name)]
    #[diagnostic(
        code(R12),
        url("https://docs.hw-script.org/errors/R12"),
        help("The {} waypoint at {}mm is {} away from pin {} at {}mm. Manual waypoints must start and end at the exact pin positions.", .0.waypoint_type, .0.waypoint_pos, .0.distance, .0.pin_name, .0.pin_pos)
    )]
    DisconnectedNet(Box<DisconnectedNetDetails>),

    #[error("Trace placed on non-routable layer '{layer}' (material: {material})")]
    #[diagnostic(
        code(R25),
        url("https://docs.hw-script.org/errors/R25"),
        help(
            "Layer '{layer}' is declared as routable: false in the profile stackup. \
              Routing is only permitted on layers with routable: true. \
              Route the trace on a different layer, or change the layer's routable attribute."
        )
    )]
    NonRoutableLayer {
        layer: CompactString,
        material: CompactString,
    },

    #[error("Route penetrates interior of component '{component}' at ({x_nm}, {y_nm}, {z_nm})")]
    #[diagnostic(
        code(R30),
        url("https://docs.hw-script.org/errors/R30"),
        help(
            "A routed trace segment passes through the physical body of component '{component}'. \
              Traces must terminate at boundary ports (exit:/enter: cardinal directions) \
              and must not penetrate the component's bounding box. \
              Adjust the route's exit/enter directions to dock at the component boundary."
        )
    )]
    RoutePenetratesComponent {
        component: CompactString,
        x_nm: i64,
        y_nm: i64,
        z_nm: i64,
    },

    #[error("Missing routing heuristic '{field}' in profile")]
    #[diagnostic(
        code(R25),
        url("https://docs.hw-script.org/errors/R25"),
        help(
            "The PDK profile must declare all routing heuristic weights in the 'routing:' block. \
              The compiler is a deterministic engine — no hardcoded fallbacks. \
              {hint}"
        )
    )]
    MissingRoutingHeuristics { field: CompactString, hint: String },

    #[error("No corridor found from ({start_x}, {start_y}, {start_z}) to ({end_x}, {end_y}, {end_z}) in G-cell {gcell_id}")]
    #[diagnostic(
        code(R31),
        url("https://docs.hw-script.org/errors/R31"),
        help(
            "The spatial decomposer could not extract a navigable corridor between these points. \
              Possible causes: \
              1. All corridors are narrower than the required width (trace_width + 2 * clearance). \
              2. Obstacles completely block the route. \
              3. Start or end point is inside an inflated obstacle.\n\n\
              Suggestions: \
              - Reduce trace_width_nm or min_clearance_nm for this net. \
              - Check that start/end ports are in free space. \
              - Verify board_bounds encompass the routing area."
        )
    )]
    CorridorExtractionFailed {
        gcell_id: u32,
        start_x: i64,
        start_y: i64,
        start_z: i64,
        end_x: i64,
        end_y: i64,
        end_z: i64,
    },

    #[error("Hard constraint violation for net {net_id}: {description}")]
    #[diagnostic(
        code(R37),
        url("https://docs.hw-script.org/errors/R37"),
        help(
            "A hard constraint was violated and cannot be resolved by the optimizer.\n\n\
              {description}"
        )
    )]
    HardConstraintViolation { net_id: u32, description: String },

    #[error("No connection point for entity '{entity}' on routing layer '{layer}'")]
    #[diagnostic(
        code(R40),
        url("https://docs.hw-script.org/errors/R40"),
        help(
            "Entity '{entity}' does not have a registered connection on layer '{layer}'. \
              Check that a via or contact connects this entity to the target routing layer.\n\n\
              {hint}"
        )
    )]
    NoConnectionPoint {
        entity: CompactString,
        layer: CompactString,
        hint: String,
    },

    #[error("Interface capability constraint violated: trace width {actual_nm}nm < required {required_nm}nm")]
    #[diagnostic(
        code(CIR1),
        url("https://docs.hw-script.org/errors/CIR1"),
        help("Increase the trace width or reduce the current requirement on this interface")
    )]
    InterfaceConstraintViolation {
        actual_nm: i64,
        required_nm: i64,
        #[label("interface capability requires wider trace")]
        span: SourceSpan,
    },

    #[error("Routing error: {0}")]
    RoutingError(String),

    #[error("Failed to resolve route endpoint '{endpoint}'")]
    #[diagnostic(code(R22), url("https://docs.hw-script.org/errors/R22"))]
    UnresolvedEndpoint {
        endpoint: String,
        #[label("this endpoint could not be resolved")]
        span: SourceSpan,
        #[help]
        help_message: String,
    },

    #[error("Expression evaluation failed: {message}")]
    #[diagnostic(
        code(R25),
        url("https://docs.hw-script.org/errors/R25"),
        help("Check expression syntax and ensure all variables are defined")
    )]
    ExpressionEvaluation { message: String },
}

/// Details for disconnected net errors (boxed to reduce enum size)
#[derive(Debug, Clone)]
pub struct DisconnectedNetDetails {
    pub route_name: CompactString,
    pub waypoint_type: CompactString,
    pub waypoint_pos: String,
    pub distance: String,
    pub pin_name: CompactString,
    pub pin_pos: String,
}
```

#### Step 2.4: Create ir/placement/errors.rs

```rust
//! Placement subsystem error types (P** error codes)

use compact_str::CompactString;
use miette::{Diagnostic, SourceSpan};
use thiserror::Error;

/// Errors that occur during placement and physical validation (P** codes)
#[derive(Error, Diagnostic, Debug)]
pub enum PlacementError {
    #[error(
        "Static short circuit: net '{net_a}' overlaps net '{net_b}' at ({x_nm},{y_nm},{z_nm}) nm"
    )]
    #[diagnostic(
        code(P42),
        url("https://docs.hw-script.org/errors/P42"),
        help(
            "Coplanar conductors on different nets overlap in the XY and Z planes. \
              Separate the overlapping geometry or verify that these nets should be connected. \
              Detected by the static geometry guard before routing to fail fast."
        )
    )]
    StaticGeometryShort {
        net_a: CompactString,
        net_b: CompactString,
        x_nm: i64,
        y_nm: i64,
        z_nm: i64,
    },

    #[error("Material interpenetration detected at z = {z_nm} nm")]
    #[diagnostic(
        code(P43),
        url("https://docs.hw-script.org/errors/P43"),
        help("Pour '{pour_a}' (material: {mat_a}) overlaps with pour '{pour_b}' (material: {mat_b}). Different materials cannot occupy the same physical space. Adjust boundaries so pours touch at edges but do not overlap.")
    )]
    MaterialInterpenetration {
        pour_a: CompactString,
        mat_a: CompactString,
        pour_b: CompactString,
        mat_b: CompactString,
        z_nm: i64,
    },

    #[error("Component '{component}' overlaps with substrate material", component = .0.component)]
    #[diagnostic(
        code(P44),
        url("https://docs.hw-script.org/errors/P44"),
        help("Place component at z:{suggested_z_layer} or higher (above substrate surface). Computed position: ({x_mm:.3}mm, {y_mm:.3}mm, {z_mm:.3}mm)",
            suggested_z_layer = .0.suggested_z_layer,
            x_mm = .0.x_mm,
            y_mm = .0.y_mm,
            z_mm = .0.z_mm)
    )]
    #[label("component overlaps substrate here", .0.span)]
    SubstrateOverlap(Box<SubstrateOverlapDetails>),

    #[error("Component '{component}' is floating in air above substrate", component = .0.component)]
    #[diagnostic(
        code(P44),
        url("https://docs.hw-script.org/errors/P44"),
        help("Place component at z:{substrate_max_layer} (substrate surface). Computed position: ({x_mm:.3}mm, {y_mm:.3}mm, {z_mm:.3}mm)",
            substrate_max_layer = .0.substrate_max_layer,
            x_mm = .0.x_mm,
            y_mm = .0.y_mm,
            z_mm = .0.z_mm)
    )]
    #[label("component floats {gap_mm:.3}mm above substrate here", .0.span)]
    ComponentFloatingInAir(Box<ComponentFloatingInAirDetails>),

    #[error("Component '{component}' is buried below substrate", component = .0.component)]
    #[diagnostic(
        code(P44),
        url("https://docs.hw-script.org/errors/P44"),
        help("Place component at z:{substrate_max_layer} or higher (on or above substrate surface). Computed position: ({x_mm:.3}mm, {y_mm:.3}mm, {z_mm:.3}mm)",
            substrate_max_layer = .0.substrate_max_layer,
            x_mm = .0.x_mm,
            y_mm = .0.y_mm,
            z_mm = .0.z_mm)
    )]
    #[label("component buried {gap_mm:.3}mm below substrate here", .0.span)]
    ComponentBuriedInSubstrate(Box<ComponentBuriedInSubstrateDetails>),

    #[error("Geometric collision in array '{array_name}': instances {instance_a} and {instance_b} have overlapping geometry", array_name = .0.array_name, instance_a = .0.instance_a, instance_b = .0.instance_b)]
    #[diagnostic(
        code(P12),
        url("https://docs.hw-script.org/errors/P12"),
        help("Array instances have overlapping geometry without explicit merge intent.\n\nPhysical Reality: Two pieces of material cannot occupy the same physical space.\n\nProblem: Pour '{}' in instances {} and {} overlap at:\n  Instance {}: [{:.3}mm, {:.3}mm] to [{:.3}mm, {:.3}mm]\n  Instance {}: [{:.3}mm, {:.3}mm] to [{:.3}mm, {:.3}mm]\n\nSolutions:\n1. Increase pitch to prevent overlap: pitch: {:.3}mm (currently {:.3}mm)\n2. Add explicit merge intent if overlap is intentional:\n   merge: [{}]  # Declares: \"I know these overlap. Melt them.\"\n\nPhilosophy: Hardware Script has NO IMPLICIT MAGIC. Overlapping geometry must be explicitly declared.",
            .0.pour_name, .0.instance_a, .0.instance_b,
            .0.instance_a, .0.bbox_a_min_x, .0.bbox_a_min_y, .0.bbox_a_max_x, .0.bbox_a_max_y,
            .0.instance_b, .0.bbox_b_min_x, .0.bbox_b_min_y, .0.bbox_b_max_x, .0.bbox_b_max_y,
            .0.suggested_pitch, .0.current_pitch,
            .0.terminal_name)
    )]
    GeometricCollision(Box<GeometricCollisionDetails>),

    #[error(
        "Forbidden junction: {mat_a} touching {mat_b} at ({x_nm}, {y_nm}, {z_nm}) without a bridge"
    )]
    #[diagnostic(
        code(P45),
        url("https://docs.hw-script.org/errors/P45"),
        help(
            "Material '{mat_a}' (category: {cat_a}) is in direct coplanar contact with \
              '{mat_b}' (category: {cat_b}) without an intermediate ohmic contact bridge. \
              Declare a bridge rule in the profile: \
              'bridge {mat_a} to {mat_b}: <BridgeMaterial>' \
              where <BridgeMaterial> is a material with category: ohmic_contact."
        )
    )]
    ForbiddenJunction {
        mat_a: CompactString,
        cat_a: CompactString,
        mat_b: CompactString,
        cat_b: CompactString,
        x_nm: i64,
        y_nm: i64,
        z_nm: i64,
    },

    #[error("Clearance violation during placement of '{entity_name}'")]
    #[diagnostic(
        code(P46),
        url("https://docs.hw-script.org/errors/P46"),
        help("{reason}")
    )]
    ClearanceViolation {
        entity_type: CompactString,
        entity_name: CompactString,
        reason: CompactString,
    },
}

/// Detail structs for boxed error data
#[derive(Debug, Clone)]
pub struct SubstrateOverlapDetails {
    pub component: CompactString,
    pub component_z_layer: usize,
    pub suggested_z_layer: usize,
    pub x_mm: f64,
    pub y_mm: f64,
    pub z_mm: f64,
    pub span: SourceSpan,
}

#[derive(Debug, Clone)]
pub struct ComponentFloatingInAirDetails {
    pub component: CompactString,
    pub component_z_layer: usize,
    pub substrate_max_layer: usize,
    pub gap_mm: f64,
    pub x_mm: f64,
    pub y_mm: f64,
    pub z_mm: f64,
    pub span: SourceSpan,
}

#[derive(Debug, Clone)]
pub struct ComponentBuriedInSubstrateDetails {
    pub component: CompactString,
    pub component_z_layer: usize,
    pub substrate_max_layer: usize,
    pub gap_mm: f64,
    pub x_mm: f64,
    pub y_mm: f64,
    pub z_mm: f64,
    pub span: SourceSpan,
}

#[derive(Debug, Clone)]
pub struct GeometricCollisionDetails {
    pub array_name: CompactString,
    pub instance_a: usize,
    pub instance_b: usize,
    pub pour_name: CompactString,
    pub terminal_name: CompactString,
    pub bbox_a_min_x: f64,
    pub bbox_a_min_y: f64,
    pub bbox_a_max_x: f64,
    pub bbox_a_max_y: f64,
    pub bbox_b_min_x: f64,
    pub bbox_b_min_y: f64,
    pub bbox_b_max_x: f64,
    pub bbox_b_max_y: f64,
    pub suggested_pitch: f64,
    pub current_pitch: f64,
}
```



#### Step 2.5: Create Smaller Domain Error Files

**ir/materials/errors.rs:**
```rust
//! Material system error types (M** error codes)

use compact_str::CompactString;
use miette::Diagnostic;
use thiserror::Error;

#[derive(Error, Diagnostic, Debug)]
pub enum MaterialError {
    #[error("Undeclared material: '{material}'")]
    #[diagnostic(
        code(M01),
        url("https://docs.hw-script.org/errors/M01"),
        help("Material '{material}' is used but never declared or imported. Add a 'material {material}: category: conductor|semiconductor|insulator' declaration, or import it from a standard library: 'import * from @std/materials/conductors'")
    )]
    UndeclaredMaterial { material: CompactString },
}
```

**ir/devices/errors.rs:**
```rust
//! Device system error types (D** error codes)

use compact_str::CompactString;
use miette::Diagnostic;
use thiserror::Error;

#[derive(Error, Diagnostic, Debug)]
pub enum DeviceError {
    #[error("Device terminal pour '{pour_name}' missing explicit net assignment")]
    #[diagnostic(
        code(D01),
        url("https://docs.hw-script.org/errors/D01"),
        help("HardwareScript Philosophy: Zero Compiler Magic\n\nDevice terminal pours MUST have explicit 'net:' assignments. The compiler does NOT infer connectivity from physical layout.\n\nFix: Add 'net: <NetName>' to the pour definition:\n\n  add pour({material}) named {pour_name}:\n      device: {device}.{terminal}\n      net: YourNetName  ← ADD THIS LINE\n      dimensions: ...")
    )]
    DeviceTerminalMissingNet {
        pour_name: CompactString,
        device: CompactString,
        terminal: CompactString,
        material: CompactString,
    },
}
```

**ir/shapes/errors.rs:**
```rust
//! Shape system error types (S** error codes)

use compact_str::CompactString;
use miette::Diagnostic;
use thiserror::Error;

#[derive(Error, Diagnostic, Debug)]
pub enum ShapeError {
    #[error("Undeclared shape: '{shape}'")]
    #[diagnostic(
        code(S15),
        url("https://docs.hw-script.org/errors/S15"),
        help("Shape '{shape}' is used but never declared. Add a 'shape {shape}(...)' definition.")
    )]
    UndeclaredShape { shape: CompactString },

    #[error("Failed to resolve shape '{shape}': {reason}")]
    #[diagnostic(
        code(S16),
        url("https://docs.hw-script.org/errors/S16"),
        help("The shape could not be instantiated or evaluated")
    )]
    ShapeResolutionFailed {
        shape: CompactString,
        reason: String,
    },
}
```

#### Step 2.6: Update Top-Level ir/errors.rs

```rust
//! Top-level IR error wrapper
//!
//! This module provides a unified `IrError` enum that wraps all domain-specific
//! error types. The `#[from]` attribute enables automatic conversion via the `?` operator.

use miette::Diagnostic;
use thiserror::Error;

// Re-export domain errors for convenience
pub use compilation::CompilationError;
pub use devices::DeviceError;
pub use materials::MaterialError;
pub use placement::PlacementError;
pub use routing::RoutingError;
pub use shapes::ShapeError;

/// Top-level IR error type
///
/// This enum wraps all domain-specific error types. Use of `#[error(transparent)]`
/// ensures the underlying error's Display implementation is used directly.
///
/// The `#[from]` attribute enables automatic conversion:
/// ```rust
/// fn do_routing() -> Result<(), RoutingError> {
///     Err(RoutingError::NoPathFound { /* ... */ })
/// }
///
/// fn compile() -> Result<(), IrError> {
///     do_routing()?; // Auto-converts RoutingError -> IrError
///     Ok(())
/// }
/// ```
#[derive(Error, Diagnostic, Debug)]
pub enum IrError {
    /// Compilation phase errors (C** codes)
    #[error(transparent)]
    Compilation(#[from] CompilationError),

    /// Routing phase errors (R** codes, CIR** codes)
    #[error(transparent)]
    Routing(#[from] RoutingError),

    /// Placement and physical validation errors (P** codes)
    #[error(transparent)]
    Placement(#[from] PlacementError),

    /// Material system errors (M** codes)
    #[error(transparent)]
    Material(#[from] MaterialError),

    /// Device system errors (D** codes)
    #[error(transparent)]
    Device(#[from] DeviceError),

    /// Shape system errors (S** codes)
    #[error(transparent)]
    Shape(#[from] ShapeError),

    /// Symbol table errors (external crate)
    #[error(transparent)]
    Symbol(#[from] crate::SymbolError),
}
```

---

### Section 3: Call Site Migration Patterns

#### Pattern 1: Direct Construction (No Changes Needed)

```rust
// OLD - Still works!
fn check_space(space: &Space) -> Result<(), IrError> {
    if space.dimensions.is_none() {
        return Err(CompilationError::MissingDimensions {
            span: space.span,
        }.into()); // Explicit .into()
    }
    Ok(())
}

// NEW - Even cleaner with ?
fn check_space_new(space: &Space) -> Result<(), CompilationError> {
    if space.dimensions.is_none() {
        return Err(CompilationError::MissingDimensions {
            span: space.span,
        });
    }
    Ok(())
}

fn caller() -> Result<(), IrError> {
    check_space_new(&space)?; // Auto-converts!
    Ok(())
}
```

#### Pattern 2: Pattern Matching (Requires Updates)

```rust
// OLD (v0.2.0)
match result {
    Err(IrError::NoPathFound { net, from_pin, to_pin }) => {
        eprintln!("Failed to route {} from {} to {}", net, from_pin, to_pin);
    }
    Err(IrError::EmptyRoute { net }) => {
        eprintln!("Route for {} has no waypoints", net);
    }
    Ok(()) => {}
}

// NEW (v0.3.0)
match result {
    Err(IrError::Routing(RoutingError::NoPathFound { net, from_pin, to_pin })) => {
        eprintln!("Failed to route {} from {} to {}", net, from_pin, to_pin);
    }
    Err(IrError::Routing(RoutingError::EmptyRoute { net })) => {
        eprintln!("Route for {} has no waypoints", net);
    }
    Ok(()) => {}
}

// BETTER (v0.3.0) - Match on specific error type
fn handle_routing_error(err: &RoutingError) {
    match err {
        RoutingError::NoPathFound { net, from_pin, to_pin } => {
            eprintln!("Failed to route {} from {} to {}", net, from_pin, to_pin);
        }
        RoutingError::EmptyRoute { net } => {
            eprintln!("Route for {} has no waypoints", net);
        }
        _ => {}
    }
}

match result {
    Err(IrError::Routing(e)) => handle_routing_error(e),
    Ok(()) => {}
}
```

#### Pattern 3: Error Propagation with ? (No Changes)

```rust
// Works in both old and new architecture!
fn do_compilation() -> Result<(), IrError> {
    validate_space()?;      // Returns CompilationError, auto-converts
    route_nets()?;          // Returns RoutingError, auto-converts
    place_components()?;    // Returns PlacementError, auto-converts
    Ok(())
}
```

#### Pattern 4: Type Annotations in Function Signatures

```rust
// OLD (v0.2.0)
impl SpaceBuilder {
    pub fn build(&self) -> Result<Space, IrError> {
        // Returns generic IrError
    }
}

// NEW (v0.3.0) - More specific return types
impl SpaceBuilder {
    pub fn build(&self) -> Result<Space, IrError> {
        self.validate_dimensions()?;  // CompilationError
        self.build_routing()?;         // RoutingError
        Ok(Space::default())
    }
    
    fn validate_dimensions(&self) -> Result<(), CompilationError> {
        // Returns specific error type
        if self.dimensions.is_none() {
            return Err(CompilationError::MissingDimensions {
                span: self.span,
            });
        }
        Ok(())
    }
    
    fn build_routing(&self) -> Result<(), RoutingError> {
        // Returns specific error type
        if self.routes.is_empty() {
            return Err(RoutingError::EmptyRoute {
                net: "TestNet".into(),
            });
        }
        Ok(())
    }
}
```

---

### Section 4: Migration Tooling

#### Automated Pattern Finder Script

```bash
#!/bin/bash
# find_error_patterns.sh - Find pattern matches that need updating

echo "Finding IrError pattern matches..."
echo "=================================="
echo

echo "1. Match expressions that need updating:"
rg "IrError::(NoPathFound|EmptyRoute|InvalidRouteExpression|PlacementConstraint)" \
   --type rust --line-number --color always | head -20

echo
echo "2. Type annotations (review for specificity):"
rg "-> Result<.*IrError>" --type rust --line-number --color always | head -10

echo
echo "3. Direct error construction:"
rg "Err\(IrError::" --type rust --line-number --color always | head -10

echo
echo "Run with: bash find_error_patterns.sh"
```

#### Semi-Automated Replacement Script

```python
#!/usr/bin/env python3
# migrate_error_patterns.py - Semi-automated migration helper

import re
import sys

ERROR_MAPPING = {
    # Routing errors (R**)
    'NoPathFound': ('Routing', 'RoutingError'),
    'EmptyRoute': ('Routing', 'RoutingError'),
    'InvalidRouteExpression': ('Routing', 'RoutingError'),
    'BridgeValidationFailed': ('Routing', 'RoutingError'),
    'CorridorExtractionFailed': ('Routing', 'RoutingError'),
    
    # Compilation errors (C**)
    'NoSpaceDefinition': ('Compilation', 'CompilationError'),
    'MissingDimensions': ('Compilation', 'CompilationError'),
    'InvalidCoordinate': ('Compilation', 'CompilationError'),
    'CircularReference': ('Compilation', 'CompilationError'),
    
    # Placement errors (P**)
    'StaticGeometryShort': ('Placement', 'PlacementError'),
    'SubstrateOverlap': ('Placement', 'PlacementError'),
    'GeometricCollision': ('Placement', 'PlacementError'),
    
    # Material/Device/Shape errors
    'UndeclaredMaterial': ('Material', 'MaterialError'),
    'DeviceTerminalMissingNet': ('Device', 'DeviceError'),
    'UndeclaredShape': ('Shape', 'ShapeError'),
}

def migrate_match_pattern(line):
    """Convert old match pattern to new nested pattern"""
    for error_variant, (wrapper, error_type) in ERROR_MAPPING.items():
        old_pattern = f'IrError::{error_variant}'
        new_pattern = f'IrError::{wrapper}({error_type}::{error_variant}'
        if old_pattern in line:
            return line.replace(old_pattern, new_pattern)
    return line

def process_file(filename):
    """Process a single Rust file"""
    with open(filename, 'r') as f:
        lines = f.readlines()
    
    modified = False
    new_lines = []
    
    for line in lines:
        new_line = migrate_match_pattern(line)
        if new_line != line:
            modified = True
            print(f"CHANGED: {line.strip()}")
            print(f"     TO: {new_line.strip()}")
        new_lines.append(new_line)
    
    if modified:
        print(f"\nWrite changes to {filename}? [y/N] ", end='')
        if input().lower() == 'y':
            with open(filename, 'w') as f:
                f.writelines(new_lines)
            print("✓ Updated")
    else:
        print(f"No changes needed in {filename}")

if __name__ == '__main__':
    if len(sys.argv) < 2:
        print("Usage: python3 migrate_error_patterns.py <file.rs>")
        sys.exit(1)
    
    for filename in sys.argv[1:]:
        process_file(filename)
```

Usage:
```bash
# Find all files with IrError patterns
rg -l "IrError::" --type rust > files_to_migrate.txt

# Migrate each file interactively
cat files_to_migrate.txt | while read f; do
    python3 migrate_error_patterns.py "$f"
done
```

