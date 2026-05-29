# v0.1.7 TSV Implementation Status (Draft)

**Date**: 2026-05-24
**Current Version**: v0.1.7-alpha (TSV Integration)

## 1. Accomplishments (Completed)

- **Voxel Engine Support**:
    - `LinerStack` and `TSVParams` structures implemented for multi-material TSV cores.
    - `stamp_tsv` operation added for concentric cylindrical rendering (Liner -> Bridge -> Core).
    - `add_tsv_stack` coordinated logic implemented (Drill -> Stamp -> Register).
- **Physics Validation (Bit-Parallel)**:
    - **P47 Substrate Short Circuit**: Detects conductive cores touching silicon without a liner.
    - **P48 Keep-Out Zone (KOZ)**: Validates mechanical stress regions around TSVs.
- **Parser & AST**:
    - Updated `ContactPlacement` to support `liner`, `liner_thickness`, `bridge`, and `koz` properties.
    - Added `FloatLiteral` support to the expression parser for multipliers (e.g., `koz: 3.0`).
- **Compiler IR**:
    - `ContactPlacement` conversion now triggers `add_tsv_stack` when properties are present.

## 2. Current Blockers (The Z-Axis Abstraction Leak)

The TSV implementation is currently blocked by a fundamental issue in the Z-axis coordinate system:

1.  **Coordinate Mismatch**: The engine uses 0-based indices, while the parser/compiler sometimes expects 1-based indices or physical units. This has caused panics during evaluation (e.g., "Expected integer but got measurement" when a physical unit was passed to a layer index field).
2.  **Semantic Layering**: TSVs often span multiple logical layers. The current `z: index` syntax is too fragile to handle the complex stackups required for 3D ICs and advanced PCBs.
3.  **DRC Reporting**: While the back-end detection works, the front-end reporting (miette) was temporarily misaligned with the new violation variants, which has been patched to allow compilation.

## 3. Next Steps (Roadmap)

- [ ] **Z-Axis Refactor**: Implement the new `Elevation` AST type (`Physical` vs `Semantic`).
- [ ] **Stackup Manager**: Build the `StackupManager` in the compiler to translate `layer: l1` into absolute nanometer Z-coordinates.
- [ ] **Engine Cleanup**: Purge `layer_index` from the `hwc-engine` and move entirely to `nm` coordinates.
- [ ] **Verification**: Update `test_tsv.hw` to use the new `layer:` syntax and verify DRC reporting.
