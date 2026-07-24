# Export Keyword: Access Control & Re-exports

**Status**: ✅ **Implemented in v0.2.0**

The `export` keyword provides explicit access control for HardwareScript modules, enabling library authors to declare public APIs while keeping internal implementations private. This feature brings professional encapsulation semantics to hardware design, matching modern software languages like Rust and TypeScript.

---

## 1. Overview

The `export` keyword controls symbol visibility across module boundaries. By default, all definitions are **private** (file-local). Only definitions marked with `export` can be imported by other files.

### Syntax

```hardware
# Private (default) - only visible within this file
material _InternalCopper:
    category: conductor
    symbol: "Cu_Int"
    properties:
        resistivity: 1.72e-8ohm_m

# Public - explicitly exported for external use
export material PublicCopper:
    category: conductor
    symbol: "Cu"
    properties:
        resistivity: 1.68e-8ohm_m

export shape PublicPad(w: Measurement, h: Measurement):
    geometry:
        Rectangle(width: w, height: h)
```

### Supported Definitions

All definition types support the `export` keyword:
- Materials, Profiles, Components, Modules
- Shapes, Devices, Logic blocks
- Enums, Structs, Interfaces
- Patterns, Strategies, Signal Groups
- Units, Constants, Material Aliases

---

## 2. Re-exports: The `pub use` Pattern

HardwareScript supports **re-exporting** imported symbols, enabling intermediate libraries to curate and publish stable APIs from their dependencies.

### Standalone Re-export Syntax

```hardware
import Aluminum, SiO2, Silicon from materials

# Re-export imported symbols (Rust-style pub use)
export Aluminum
export SiO2
export Silicon
```

This allows downstream consumers to import these materials from your PDK without directly depending on the `materials` module.

### Example: Foundry PDK

```hardware
# foundry_pdk.hw
import * from @std/primitives/units
import PublicSilicon, Aluminum, SiO2 from materials

# Re-export foundation materials as part of PDK API
export PublicSilicon
export Aluminum
export SiO2

# Private internal material - NOT re-exported
material _PDK_Silicon_N:
    category: semiconductor
    symbol: "Si_N"
    description: "Private N-type silicon substrate - internal use only"
    properties:
        resistivity: 10.0ohm_m
        # ... other properties

# Public PDK profile using both public and private materials
export profile Silicon_180nm:
    technology: "ASIC"
    stackup:
        substrate:  [material: PublicSilicon, thickness: 100nm, routable: false]
        gate_oxide: [material: SiO2, thickness: 10nm, routable: false]
        metal1:     [material: Aluminum, thickness: 400nm, routable: true]
    # ... other configuration
```

**Downstream usage:**

```hardware
# Can import re-exported materials directly from PDK
import Silicon_180nm, Aluminum from foundry_pdk

# This would fail - _PDK_Silicon_N is private
# import _PDK_Silicon_N from foundry_pdk  # ❌ Compile error
```

---

## 3. Implementation Architecture

The export system is implemented with clean separation of concerns following modern compiler design:

### 3.1 Parser Layer

- Each definition AST node contains an `is_exported: bool` field
- The parser sets this flag when the `export` keyword is encountered
- Re-export statements create `ReExport` entries in the AST

### 3.2 Module Resolver (Clean Architecture)

The module resolver separates two critical concerns:

1. **Stateless File Loading** (Global AST Cache)
   - Files are parsed once and cached in `ast_cache: FxHashMap<PathBuf, Program>`
   - This is purely a performance optimization with zero side effects
   - Cache stores complete AST including imports and re-export declarations

2. **Per-Import Symbol Registration** (Always Executed)
   - Every `import` statement registers its requested symbols
   - Symbol registration happens even if the file was previously loaded
   - For wildcard imports (`import *`), only exported symbols are registered
   - For selective imports, visibility is checked and private access is rejected

### 3.3 Re-export Resolution

When resolving selective imports:

1. Check if the symbol exists as a **local definition** in the module
   - If found and exported → register it
   - If found but private → reject with "private access attempted" error

2. If not found locally, check if it's a **re-exported symbol**
   - The module's `program.re_exports` list is consulted
   - If the symbol is re-exported, it was already registered when the module's own imports were resolved
   - No additional registration needed (symbol is already in the table)

3. If neither local nor re-exported → reject with "symbol not found" error

### 3.4 Key Design Principle

**No global "resolved set" tracking.** The previous architecture had a `resolved: FxHashSet<PathBuf>` that conflated file parsing with symbol registration, causing re-exports to fail. The clean architecture eliminates this by:

- Caching **AST parsing** globally (stateless, pure performance)
- Executing **symbol registration** per-import (stateful, semantic operation)

This matches the architecture of `rustc` and other production compilers.

---

## 4. Benefits to the HardwareScript Ecosystem

### 4.1 Strong Encapsulation for Foundry PDKs

Foundry PDKs contain thousands of internal helpers, custom vias, and sub-micron calculations. The `export` keyword allows foundries to:

- Publish clean, stable public APIs
- Refactor internal implementations without breaking user code
- Version and evolve PDKs safely

**Before (all symbols public):**
```hardware
# foundry_pdk.hw - everything leaks
material _Helper1: ...
material _Helper2: ...
material _Helper3: ...
material PublicMetal: ...  # Only this should be public
```

User code could accidentally depend on `_Helper1`, breaking on PDK updates.

**After (explicit exports):**
```hardware
# foundry_pdk.hw - only public API visible
material _Helper1: ...  # Private
material _Helper2: ...  # Private
material _Helper3: ...  # Private
export material PublicMetal: ...  # Public API
```

Users can only import `PublicMetal`, ensuring stability.

### 4.2 Namespace Cleanliness

Standard library modules can have hundreds of internal helpers but only expose a handful of public primitives. With explicit exports:

- Symbol tables remain lightweight
- Compilation is faster (fewer symbols to resolve)
- Naming collisions are reduced
- IDE autocomplete shows only relevant symbols

### 4.3 Stable Package Dependencies

In the Hardware Package Manager (HPM) ecosystem, packages depend on other packages. Explicit exports ensure:

- Packages only depend on official, public APIs
- Internal refactoring doesn't break downstream packages
- Semantic versioning can be enforced meaningfully
- Package authors have clear API boundaries

---

## 5. Usage Examples

### 5.1 Basic Material Library

```hardware
# materials.hw
import * from @std/primitives/units

# Public materials (exportable API)
export material PublicCopper:
    category: conductor
    symbol: "Cu"
    properties:
        resistivity: 1.68e-8ohm_m
        thermal_conductivity: 401.0W_mK

export material Aluminum:
    category: conductor
    symbol: "Al"
    properties:
        resistivity: 2.82e-8ohm_m
        thermal_conductivity: 237.0W_mK

# Private internal material
material _InternalCopper:
    category: conductor
    symbol: "Cu_Int"
    properties:
        resistivity: 1.72e-8ohm_m  # Slightly different spec
```

### 5.2 Component Library with Shapes

```hardware
# shapes.hw
import * from @std/primitives/units

# Private helper shape
shape _BaseRectangle(w: Measurement, h: Measurement):
    geometry:
        Rectangle(width: w, height: h)

# Public API shape
export shape Pad(w: Measurement, h: Measurement):
    geometry:
        _BaseRectangle(w: w, h: h)  # Can use private helpers internally
```

### 5.3 PDK Re-export Pattern

```hardware
# vendor/tsmc_180nm.hw
import * from @std/primitives/units
import Copper, Aluminum, Silicon from @std/materials/conductors

# Re-export foundation materials
export Copper
export Aluminum
export Silicon

# Private calibration material
material _CalibratedCopper:
    category: conductor
    symbol: "Cu_Cal"
    properties:
        resistivity: 1.69e-8ohm_m  # Factory-calibrated value

# Public process profile
export profile TSMC_180nm_Process:
    technology: "CMOS"
    stackup:
        substrate: [material: Silicon, thickness: 300um, routable: false]
        metal1:    [material: _CalibratedCopper, thickness: 450nm, routable: true]
        metal2:    [material: Copper, thickness: 450nm, routable: true]
    # ... configuration
```

---

## 6. Compiler Errors

### 6.1 Private Symbol Access

Attempting to import a private symbol results in a clear error:

```hardware
import _InternalCopper from materials
```

**Error:**
```
x Import resolution failed: Symbol '_InternalCopper' in module 'materials.hw' is
| not exported (private access attempted)
```

### 6.2 Symbol Not Found

Attempting to import a non-existent symbol:

```hardware
import NonExistentMaterial from materials
```

**Error:**
```
x Import resolution failed: Symbol 'NonExistentMaterial' not found in module
| 'materials.hw'
```

### 6.3 Implicit vs Explicit

All symbols are **private by default**. This is a deliberate design choice that:

- Prevents accidental API surface expansion
- Forces library authors to think about public contracts
- Matches Rust's privacy model (everything is private unless `pub`)

---

## 7. Testing & Validation

The export system is validated through comprehensive tests:

- **`test_valid_import.hw`**: Tests successful import of exported symbols and re-exports
- **`test_invalid_import_shape.hw`**: Tests rejection of private symbol access
- **`foundry_pdk.hw`**: Real-world PDK example with re-exports

All tests pass with proper symbol resolution and visibility enforcement.

---

## 8. Migration Guide

### For Existing Code

If you have existing `.hw` files without `export` keywords:

1. **All definitions are now private by default**
2. Add `export` to symbols you want to make public
3. Use underscore prefix (`_`) for private helpers (convention)

### Example Migration

**Before (v0.1.x - all public):**
```hardware
material Copper: ...
material Helper: ...
```

**After (v0.2.0 - explicit exports):**
```hardware
export material Copper: ...  # Public API
material _Helper: ...        # Private helper
```

---

## 9. Future Enhancements

### 9.1 Export Visibility Levels

Future versions may support fine-grained visibility:

```hardware
export(crate) material CrateLocal: ...    # Visible within package
export(super) material ParentVisible: ... # Visible to parent module
export material Public: ...               # Fully public
```

### 9.2 Export Groups

Syntactic sugar for bulk exports:

```hardware
export {
    Copper,
    Aluminum,
    Silicon,
}
```

### 9.3 Selective Re-exports

Re-export with renaming:

```hardware
import InternalCopper from materials
export InternalCopper as Copper  # Re-export with alias
```

---

## 10. References

- **Implementation**: `hwc-compiler/src/module_resolver.rs`
- **Parser**: `hwc-parser/src/parser/definitions/mod.rs`
- **AST**: `hwc-parser/src/ast/*.rs` (each definition type)
- **Tests**: `hwc/tests/export-keyword-tests/`
- **Symbol Table**: `hwc-compiler/src/symbol_table/`