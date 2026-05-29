# Intentional Overlap Waivers (v0.1.6)

## Overview
The **Waiver System** allows designers to intentionally bypass physical safety checks for specific design patterns. This is primarily used in **Silicon Design (Doping Wells)** and **RF Shielding**, where components or pours must intentionally interpenetrate the substrate or each other.

By default, the `hwc` compiler enforces strict physical isolation rules (P12, P42, P44). Waivers explicitly signal that a violation is an **intentional design choice**.

---

## 1. Syntax & Attributes

Waivers are defined using the `waivers` attribute in `add` or `pour` statements.

### Component Waivers
```hw
add MOSFET named M1_Bulk at [0, 0, substrate.min_z]:
    waivers:
        merge: true       // Allow overlap with substrate (P12)
        floating: true    // Allow placement without physical support (P44)
        isolated: true    // Suppress air-gap isolation warnings (P43)
```

### Pour Waivers
```hw
pour P_Well on Layer_1:
    boundary: [0, 0] to [10um, 10um]
    material: "P-Dope"
    waivers:
        merge: true       // Allow overlap with other pours/substrate (P42)
```

---

## 2. Waiver Types

| Attribute | Logic | Description | Target |
|-----------|-------|-------------|--------|
| `merge` | Boolean / List | Allows physical interpenetration with substrate or other elements. | P12, P42 |
| `floating` | Boolean | Disables the "Vacuum Rule" check for physical support. | P44 |
| `isolated` | Boolean | Suppresses connectivity checks for intentionally floating nodes. | P41 |
| `virtual` | Boolean | Component exists for logic synthesis but has no physical voxel footprint. | All |

### Granular Merging (v0.1.6)
You can waive specific pins rather than the whole component:
```hw
add IC named U1:
    waivers:
        merge: [thermal_pad, gnd_pin]
```

---

## 3. Engine Implementation

### Decoupled Reporting
The engine (`hwc-engine`) is decoupled from the diagnostics system via the `DiagnosticReporter` trait. When a waiver is successfully applied, the engine reports it through a bridge to the compiler's `DiagnosticCollector`.

### The Vacuum Rule (P44)
Components are normally required to be "supported" (either touching the substrate or another component). `floating: true` waives this check, allowing elements like antenna arrays or suspended MEMS structures.

---

## 4. Diagnostic Output

Waivers are no longer printed to `stdout` via `println!`. They use the native diagnostic system with the **W-prefix**.

**Standard Waiver Header**: `waiver[W001]`

Example Output:
```text
waiver[W001]: Waiver applied: Component 'M1_Waived' allowed to overlap substrate
   --> tests/sprint8_waivers/test_waivers.hw:45:5
    |
 45 |     waivers:
    |     ^^^^^^^^ this intentional deviation was permitted
```

---

## 5. Safety Warnings

> [!WARNING]
> Waivers should be used sparingly. Misusing `merge: true` can hide genuine short circuits or layout errors. Always prefer structured assembly (anchors) over waivers when possible.

> [!IMPORTANT]
> A waiver only suppresses the **warning/error** at compile time. It does **not** change the physical reality of the voxels in the final export.
