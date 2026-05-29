This is the architectural blueprint for **Hardware Script v0.1.7: The Physical Viability Protocol (PVP)**. 

By moving physics validation from Rust into the `.hw` files, we transform the compiler from a "Hardcoded Guard" into a "Universal Law-Enforcement Engine." 

---

# Hardware Script v0.1.7: Physical Viability Protocol (PVP)

**Document Type**: Architectural Authoritative Reference  
**Status**: Target v0.1.7  
**Focus**: Declarative Physics Validation  

## 1. The Core Philosophy: "Atoms obey Code"

In v0.1.6, the compiler "knows" that NMOS bulk connects to Ground because we wrote that logic in Rust. In v0.1.7, the compiler knows **nothing** by default. It reads the **Physical Viability Protocol (PVP)** from the library to understand how atoms are allowed to interact.

This enables:
1.  **Foundry Portability**: Different foundries can define different "physics" (e.g., GaN biasing vs. Silicon biasing).
2.  **Exotic Hardware**: Users can define rules for Photonics, Microfluidics, or Quantum systems without changing the compiler.
3.  **Low-Level Speed**: Rules are compiled into **Predicate Bit-Arrays**, making validation O(1).

## 2. Syntax: The `rules:` Block

The PVP is implemented as a new block inside the `device` definition. It uses the **Boundary Law** (`:`) and follows the standard Hardware Script visual rhythm.

### Example: NMOS Physics Definition
```hw
device NMOS:
    terminals: [gate, source, drain, bulk]
    
    # Static Material Constraints
    materials:
        gate: [Polysilicon, Aluminum]
        bulk: Silicon_P
    
    # THE PVP LAYER: Rules of the Universe
    rules:
        # Rule 1: Enforce Biasing Law
        biasing:
            target: bulk
            condition: material.bias_requirement matches net.classification
            error: "P18: Biasing Violation - {target} requires {material.bias_requirement}"

        # Rule 2: Geometric Separation (Example)
        clearance:
            target: [source, drain]
            minimum: 180nm
            error: "P43: Source/Drain spacing too tight"
```

## 3. The Predicate Logic (How it works)

To maintain sub-second build times, the PVP does not run a "script." It uses a **Predicate Engine** that evaluates simple boolean identities.

### Keywords & Identifiers
*   `target`: The terminal or region being inspected.
*   `material`: Accesses properties from the `material_database` (e.g., `bias_requirement`).
*   `net`: Accesses classifications from the `space` (e.g., `classification`).
*   `matches`: A special operator that links Material Requirements to Net Classifications.

### The `matches` Identity Table (Internal Compiler Logic)
The compiler uses a hardcoded identity map to resolve the `matches` keyword:

| Material Requirement | Net Classification | Result |
| :--- | :--- | :--- |
| `LowestPotential` | `Ground` | ✅ Pass |
| `HighestPotential` | `Power` | ✅ Pass |
| `None` | *Any* | ✅ Pass |
| *Mismatch* | *Any* | ❌ Fail |

## 4. Architectural Implementation (Pass 4.5)

The PVP is injected into the **Device Extraction** phase.

1.  **Parse**: `hwc-parser` captures the `rules:` block into the `DeviceContract` AST.
2.  **Lower**: `hwc-compiler` converts text rules into `ValidationPredicate` objects.
3.  **Execute**: As `DeviceExtractor` identifies a device in the voxels:
    *   It grabs the `Predicate` from the contract.
    *   It performs a direct property lookup (O(1)).
    *   If the boolean result is `false`, it triggers a **Physics Error (P-series)**.

## 5. Why this is "God-Tier" Low-Level Native

### A. Zero Runtime Overhead
Unlike traditional EDA tools that run complex Python or TCL scripts to validate LVS, PVP rules are **static comparisons**. 
`material.bias_requirement == net.classification` 
This is a single CPU instruction once the pointers are resolved.

### B. Error Interpolation
By using the `{target}` and `{material.bias_requirement}` tokens in the error string, the compiler generates **IDE-quality diagnostics** dynamically.

**Output Example:**
```
❌ Physical Viability Error (P18):
   Device: M1 (NMOS)
   Terminal: bulk
   Error: NMOS Bulk requires LowestPotential connection. 
   Found: net 'VCC_CORE' (Classification: Power)
```

## 6. Implementation Roadmap (v0.1.7)

| Task | Component | Effort |
| :--- | :--- | :--- |
| **PVP AST Extension** | `hwc-parser` | 2 Days |
| **Predicate Evaluator** | `hwc-compiler` | 4 Days |
| **Stdlib Migration** | `stdlib/foundry/` | 1 Day |
| **Validation Tests** | `hwc-tests` | 2 Days |

---

### Conclusion for the Roadmap

This feature makes Hardware Script **truly future-proof**. By giving the "Rules of Reality" to the user, you ensure that no matter how complex the future of hardware becomes, your compiler will remain relevant.

**Version 0.1.6 focus remains on Hierarchical Geometry.** Once that "magic" is shipping, 0.1.7 will provide the "enforcement" layer that makes it enterprise-grade.