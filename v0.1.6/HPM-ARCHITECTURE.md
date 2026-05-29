# Hardware Package Manager (HPM) Architecture (v0.1.6)

**Version**: 0.1.6  
**Focus**: Dependency Management, Namespaces, and Package Distribution  
**Status**: Architectural Authoritative Reference

---

## Executive Summary

Because Hardware Script (HWS) compiles into physical reality, a "Package" isn't a collection of mathematical functions—it is a **Digital Warehouse of Physical Atoms**.

If a manufacturer (like Yageo or Texas Instruments) publishes a package of resistors or microcontrollers, it might contain 10,000 distinct components. Traditional software package managers that require developers to explicitly list every exported function in a build file will instantly collapse under this scale.

The Hardware Package Manager (HPM) solves this by adopting a **Directory-Level Package Model** (inspired by Go and Dart) combined with **Workspace Version Isolation** (inspired by Rust's Cargo). It uses auto-discovery for exports and strict namespace aliases for imports, guaranteeing that a hardware design compiled today will compile perfectly in ten years.

---

## 1. The Directory-Level Package Model

**The Flaw of File-Level Packages**: If a package is just a single `.hw` file, an author cannot logically separate their 3D footprints, thermal properties, and logical definitions.

In Hardware Script, a **Package is a Directory** containing a `hw.toml` manifest file. The directory itself is the boundary.

When HPM downloads a package, it stores it in a globally shared, read-only cache on the user's machine:

```
~/.hw/packages/@yageo/passives/1.2.0/   <-- The Package Directory
├── hw.toml                             <-- The Manifest (Build file)
├── src/
│   ├── 0402_series.hw                  <-- Contains 500 components
│   ├── 0603_series.hw                  <-- Contains 500 components
│   └── 0805_series.hw                  <-- Contains 500 components
└── internal/
    └── smd_footprints.hw               <-- Internal layouts
```

---

## 2. The Manifest and "The 100 Names" Solution

In traditional systems, the package author must explicitly list what they are exporting. For hardware, listing 1,500 resistor names in the `hw.toml` is unscalable and error-prone.

HPM uses **Auto-Discovery via Visibility**. By default, any component defined in an exported file is publicly available.

### The hw.toml Structure

The manifest requires standard Semantic Versioning (Major.Minor.Patch).

```toml
[package]
name = "@yageo/passives"
version = "1.2.0"
authors = ["Yageo Corp"]
description = "Official SMD thick film resistors"

[exports]
# Instead of listing 1,500 component names, the author exports files.
# The compiler automatically discovers the components inside them.
modules = [
    "src/*.hw"
]

[dependencies]
"@std/materials" = "0.1.6"
"@ieee/standards" = "2.1.0"
```

### Private / Internal Components

If a package author creates a "helper" component (like a generic pad shape) that they do not want users to access directly, they can hide it in two ways:

1. **Directory exclusion**: Place it in a folder not listed in `[exports]` (e.g., `internal/`).
2. **Identifier prefix**: Prefix the component name with an underscore (e.g., `component _BasePad:`). The compiler will treat this as strictly private to the package.

---

## 3. Import Syntax & Namespace Protection

When a user imports a package, we must prevent **Namespace Pollution**. If you download a package that contains a component named `Capacitor`, and you already have a custom `Capacitor` in your own file, the compiler would crash.

To solve this, HWS provides three distinct import strategies.

**(Note: We explicitly reject the `import all except [bad]` anti-pattern. If a package author updates their package and adds a new component, an exclusion list would fail to block it, silently crashing the user's code. Whitelisting is always safer than blacklisting.)**

### Strategy A: Namespaced Import (Highly Recommended)

This is the safest and most scalable method. You import the entire package behind a custom prefix alias. It loads everything into memory, but keeps it locked behind a "dot" operator.

```hw
# Import the package and assign it the alias 'Yageo'
import @yageo/passives as Yageo

space Motherboard:
    # Access components using the namespace prefix
    add Yageo.Resistor_0805_10k named R1 at [x: 10mm, y: 10mm, z: 1]
    add Yageo.Capacitor_0603_1uF named C1 at [x: 20mm, y: 10mm, z: 1]
```

### Strategy B: Targeted Import

If you only need one or two specific components from a massive library, you can pluck them out directly.

```hw
# Only these exact components enter your local namespace
import Resistor_0805_10k, Resistor_0603_10k from @yageo/passives

space Motherboard:
    # No prefix needed because they were explicitly imported
    add Resistor_0805_10k named R1 at [x: 10mm, y: 10mm, z: 1]
```

### Strategy C: The Wildcard Import

Used primarily for standard library logic gates where prefixing becomes unreadable (e.g., `Logic.AND(a, b)`). It dumps everything from the target file into your local namespace.

```hw
# Dumps all components from the target into the local namespace
import * from @std/logic/gates

module CustomALU:
    logic:
        # AND, OR, XOR are globally available
        Out = AND(A, B)
```

**Warning**: Use wildcards only for standard libraries, never for third-party HPM packages, to avoid collision risks.

---

## 4. Workspace & Version Isolation (The Rust Model)

If you create a package inside your workspace, it must not corrupt any other project on your system. HPM achieves absolute isolation via the **Global Cache** and the **Local Lockfile**.

**Global Cache**: When you run `hpm install @ti/power`, it downloads the package to `~/.hw/packages/@ti/power/1.2.0/`. It is completely read-only.

**The Lockfile (hw.lock)**: When the package is installed, HPM generates a `hw.lock` file in your project directory. This file mathematically hashes the package contents and pins the exact version.

**The Isolation Guarantee**: If Project A is built using `@ti/power v1.2.0`, and Project B updates to `@ti/power v2.0.0`, they do not corrupt each other. When you compile Project A, the compiler reads `hw.lock`, ignores v2.0.0, and pulls v1.2.0 from the global cache.

This guarantees that a multi-million dollar PCB designed in 2026 will still compile with absolute deterministic perfection in 2036.

---

## 5. The Three Layers of Availability

To clarify how the compiler resolves names, Hardware Script defines three layers of scope availability. They must be treated differently.

### Layer 1: The Primitives (The "Soul")

**What they are**: `Volt`, `Ampere`, `Meter`, `Percent`, `PI`.

**How they load**: Auto-injected by the compiler into the Lexer and Parser before your code even runs.

**Import status**: DO NOT IMPORT. They are the laws of physics and exist implicitly.

### Layer 2: The Standard Library (The "Foundry")

**What they are**: `Copper`, `FR4`, `Resistor_0805`, `NAND_Gate`.

**How they load**: They ship physically inside the compiler installation directory (`/stdlib/`).

**Import status**: MUST BE IMPORTED. They use the reserved `@std` scope.

```hw
import Copper from @std/materials/conductors
```

### Layer 3: The HPM Registry (The "World")

**What they are**: Vendor chips, community modules, complex sensors.

**How they load**: Downloaded from the internet via `hpm install` into the `~/.hw/packages` cache.

**Import status**: MUST BE IMPORTED. They use standard organization scopes (`@manufacturer/package`).

```hw
import ESP32_WROOM from @espressif/mcu
```

---

## 6. HPM Integration with the Authority System

The Hardware Package Manager synergizes perfectly with the **Stack of Truths** (Authority System) defined in [AUTHORITY-AND-LIBRARY-ARCHITECTURE.md](./AUTHORITY-AND-LIBRARY-ARCHITECTURE.md).

### The Authority Stack Reminder

When the compiler resolves a name, it searches in descending order:

1. **Local Authority**: Definitions in the current `.hw` file
2. **HPM Authority**: Definitions imported from external libraries
3. **Prelude Authority**: Auto-loaded primitives from `stdlib/primitives/`
4. **Core Authority**: Hardcoded engine bootstraps

### How Import Strategies Map to the HPM Layer

Each import strategy injects symbols into the **HPM Layer** of the Symbol Table:

**Targeted Import** (`import Copper from @std/materials`):
- The symbol `Copper` is injected directly into the HPM layer
- If you write `material Copper:` locally, the Local version wins
- The Authority System triggers **Deep Property Merge** (local properties override imported ones)

**Wildcard Import** (`import * from @std/logic/gates`):
- `AND`, `OR`, `XOR` are injected into the HPM layer
- If you write your own `module AND:` locally, your local version wins
- Perfect for standard libraries where collision is unlikely

**Namespaced Import** (`import @yageo/passives as Yageo`):
- The symbol `Yageo` is injected as a **Namespace Node**
- When you type `Yageo.Resistor`, the compiler resolves `Yageo`, then looks inside it
- Because it's prefixed, it never collides with local code

**Conclusion**: The Authority System naturally accepts all HPM import strategies without modification. The layered hash-map architecture handles shadowing and merging automatically.

---

## 7. Non-Component Entities (Materials, Profiles, Patterns)

HPM isn't just for components. Because Hardware Script's compiler treats everything as a generic **Symbol** in the AST, HPM works flawlessly for all hardware elements.

### Example: A Custom Fabrication Package

Imagine TSMC publishes an HPM package for their 5nm process. It contains zero components, but it contains materials, profiles, and routing patterns.

```hw
# Import the foundry PDK
import @tsmc/node_5nm as TSMC

space ASIC_Die:
    # Using a TSMC Material
    add substrate(TSMC.Silicon_UltraPure)
    
    # Using a TSMC Profile (clearance rules)
    apply profile TSMC.Rules_5nm
    
    # Using a TSMC Routing Pattern (trombone delay matching)
    route CPU.Bus to RAM.Bus:
        strategy: TSMC.HighSpeed_DelayMatch
```

### What Can Be Packaged?

| Entity Type | HPM Support | Example Package |
|-------------|-------------|-----------------|
| **Components** | ✅ Full | `@yageo/passives` (10,000 resistors) |
| **Materials** | ✅ Full | `@dupont/polymers` (specialty dielectrics) |
| **Profiles** | ✅ Full | `@jlcpcb/standard` (manufacturing constraints) |
| **Modules** | ✅ Full | `@riscv/cores` (CPU designs) |
| **Patterns** | ✅ Full | `@intel/routing` (high-speed routing strategies) |
| **Units** | ✅ Special* | `@rf/units` (dBm, GHz) |
| **Constants** | ✅ Special* | `@nasa/physics` (planetary constants) |

**Special handling required for Units and Constants (see Section 8).*

This proves the HPM design scales perfectly to every block type in the language.

---

## 8. The "Naked Import" Mandate (Units and Constants)

### The Critical Edge Case

Units and math constants cannot be namespaced because of how the Lexer works.

**The Problem**: The lexer is "dumb." It looks for a number attached to a string with no spaces: `10µF` or `5dBm`.

If a user imports an RF package using a namespace alias, how do they use the `dBm` unit?

```hw
import @rf/units as RF

# ❌ Does the user write this?
power: 10RF.dBm    # The Lexer would crash! "10RF" is an invalid measurement!
```

Because units are attached directly to numbers, **Units and Constants cannot be namespaced**.

### The Architectural Rule: The "Naked Import" Mandate

Because units operate as literal suffixes attached directly to numbers (e.g., `10dBm`), they cannot be accessed via a dot operator (`10RF.dBm`).

Therefore, **Units and Constants completely bypass namespace aliases and are always injected directly into the global HPM scope.**

You can explicitly import them using Targeted imports, or you can use a Namespaced import and let the compiler automatically "split" the package for you.

```hw
# ✅ Option 1: Explicit Targeted Import
import dBm, GHz from @rf/units

component Antenna:
    electrical:
        freq: 2.4GHz
        power: 10dBm
```

```hw
# ✅ Option 2: Wildcard Import
import * from @std/math

logic:
    # PI and E are immediately available as bare words
    let circumference = 2 * PI * radius
```

```hw
# ✅ Option 3: Namespaced Import (compiler auto-splits)
import @rf/units as RF

component Antenna:
    electrical:
        # Units are automatically available (no RF. prefix needed)
        freq: 2.4GHz
        power: 10dBm
```

### Automatic Wildcard Behavior

To reduce friction, the compiler implements **Automatic Wildcard** for units and constants, even when using namespaced imports.

**How It Works**: When you write `import @spacex/aero_units as Aero`, the compiler's module resolver examines the incoming AST:

1. **Components and Materials**: Placed under the `Aero.` namespace
2. **Units and Constants**: Automatically wildcarded and dropped directly into the HPM layer as bare symbols

**Result**: Engineers get the safety of namespaces for physical components, but frictionless ease-of-use for mathematical units and constants, all from a single import statement.

```hw
# Single import statement
import @spacex/aero_units as Aero

space Rocket:
    # Components are namespaced
    add Aero.TitaniumAlloy_Panel named Panel1 at [x: 0mm, y: 0mm, z: 1]
    
    # Units are automatically available (no prefix needed)
    dimensions: 10m by 5m by 3m
    
    logic:
        # Constants are automatically available
        escape_velocity = sqrt(2 * G_MARS * mass / radius)
```

### Duplicate Units/Constants: "Last Import Wins"

What happens if two different imported packages define the exact same unit or constant?

```hw
import * from @nasa/physics     # Defines G as 9.80665
import * from @spacex/physics   # Defines G as 9.80660

logic:
    force = mass * G  # Which G?
```

**Resolution**: Because the HPM stack uses reverse iteration (`.iter().rev()`), the **last imported package wins**.

When the compiler resolves `G`, it searches the HPM stack from top to bottom. Since `@spacex/physics` was imported last, it gets pushed to the top of the HPM stack. The compiler hits SpaceX's definition first and stops.

**Result**: No compiler crash, no ambiguity, just deterministic **"Last Import Wins"** shadowing (exactly how Python and Elixir handle duplicate wildcard imports).

**Best Practice**: If you need precise control over which definition is used, import the specific constant with a targeted import after the wildcard:

```hw
import * from @nasa/physics
import * from @spacex/physics
import G from @nasa/physics  # Explicitly override with NASA's definition

logic:
    force = mass * G  # Uses NASA's G (9.80665)
```

### The Brilliant Side-Effect: Forcing "Universal Truths"

By making constants automatically wildcard into the global scope, Hardware Script accidentally created a massive safeguard against "spaghetti code."

In languages like C or Python, developers constantly pollute the global scope with junk constants like `MAX_WIDTH = 10` or `DEFAULT_TEMP = 85`.

If a package author does that in Hardware Script, the "Last Import Wins" rule will cause their `MAX_TEMP` to override another package's `MAX_TEMP`, potentially causing silent logical errors.

**Why this is brilliant**: It forces hardware developers to use `const` only for **universal physical and mathematical truths** (like `PI`, `E`, `G_MARS`, `PLANCK_CONSTANT`). If a developer wants to define the max temperature of a specific resistor, they are forced to put it where it actually belongs: inside the component's `electrical:` or `thermal:` block!

```hw
# ❌ BAD: Pollutes the global namespace.
# HPM's auto-wildcard will punish this if two packages do it.
const MAX_TEMP: 150C

# ✅ GOOD: Encapsulated exactly where it belongs.
component PowerResistor:
    thermal:
        max_temp: 150C
```

**The Architectural Discipline**: This rule naturally disciplines the community into writing clean, encapsulated hardware definitions, leaving the global namespace strictly for the laws of physics.

---

## 9. Corporate Override (Enterprise Physics)

The Authority Stack enables enterprise teams to override baseline primitives with proprietary definitions.

### The Scenario

Hardware Script's Prelude defines standard gravity as `const G: 9.80665`, but an aerospace company needs highly-precise planetary gravity constants for their simulation engine.

### The Solution

```hw
# 1. The compiler boots up. Prelude loads standard 'G' and standard 'V' (Volts).

# 2. The engineer imports their company's proprietary package.
import * from @company/proprietary_physics

# 3. The engineer writes their logic.
logic:
    force = mass * G
    tolerance = 5V
```

### What Happens in the Symbol Table?

When the compiler resolves `G` and `V`, it searches the Stack of Truths top-down:

1. **Local Layer**: Did the engineer define `G` in this exact file? (No)
2. **HPM Layer**: Did an imported package define `G`? (Yes! Found in `@company/proprietary_physics`). **MATCH FOUND. SEARCH STOPS.**
3. **Prelude Layer**: (Never reached for `G` or `V`)

**The Result**: The company's proprietary definitions completely shadow the baseline primitives instantly.

### Non-Intersecting Units Survive

If the company overrides `V` (Volts) and `G` (Gravity), but doesn't touch `A` (Amps) or `PI`, the compiler will simply fall through the HPM layer for `A` and `PI`, hit the Prelude layer, and use the baseline primitives.

**It is a perfect, zero-cost Deep Merge of physical realities.**

### Enterprise Readiness

Companies don't have to fork the compiler or write custom Rust code to change how physics and math are calculated in their workflow. They just:

1. Publish an internal HPM package with custom units/constants
2. Import it into their projects
3. The Authority Stack automatically rewrites the laws of physics for that specific project

**The design is airtight. Hardware Script is enterprise-ready.**

---

## 10. Implementation Notes

### Symbol Table Structure

The compiler's Symbol Table implements the Authority Stack as layered hash-maps:

```rust
pub struct SymbolTable {
    core: HashMap<String, Symbol>,      // Hardcoded bootstraps
    prelude: HashMap<String, Symbol>,   // Auto-loaded primitives
    hpm: Vec<HashMap<String, Symbol>>,  // Imported libraries (stack)
    local: HashMap<String, Symbol>,     // Current file definitions
}

impl SymbolTable {
    pub fn resolve(&self, name: &str) -> Option<&Symbol> {
        // Search order: Local > HPM > Prelude > Core
        self.local.get(name)
            .or_else(|| self.hpm.iter().rev().find_map(|layer| layer.get(name)))
            .or_else(|| self.prelude.get(name))
            .or_else(|| self.core.get(name))
    }
}
```

### Import Resolution Algorithm

```rust
pub fn import_package(&mut self, package: &Package, alias: Option<&str>) {
    let mut hpm_layer = HashMap::new();
    
    for symbol in package.symbols() {
        match symbol.kind {
            // Units and Constants: Always wildcard (naked import)
            SymbolKind::Unit | SymbolKind::Const => {
                hpm_layer.insert(symbol.name.clone(), symbol);
            }
            
            // Components, Materials, etc.: Respect namespace alias
            _ => {
                if let Some(alias) = alias {
                    // Add to namespace node
                    self.add_to_namespace(alias, symbol);
                } else {
                    // Direct import
                    hpm_layer.insert(symbol.name.clone(), symbol);
                }
            }
        }
    }
    
    self.hpm.push(hpm_layer);
}
```

---

## Conclusion

By adopting Directory-Level packages, Semantic Versioning, auto-discovery visibility, and strict namespace aliases, the Hardware Package Manager is built to handle the immense scale of physical hardware components.

It prevents author burnout (no need to manually list 10,000 resistors in a build file) while providing ironclad protection against namespace pollution for the end-user.

**Key Architectural Achievements**:

1. **Universal Support**: HPM handles components, materials, profiles, modules, patterns, units, and constants
2. **Authority Integration**: Seamless integration with the Stack of Truths (Local > HPM > Prelude > Core)
3. **Automatic Wildcard**: Units and constants are automatically wildcarded for frictionless usage
4. **Corporate Override**: Enterprise teams can shadow baseline primitives with proprietary definitions
5. **Version Isolation**: Projects are hermetically sealed via global cache and lockfiles

This architecture ensures Hardware Script is not just a language, but a highly scalable, production-ready, enterprise-grade ecosystem.
