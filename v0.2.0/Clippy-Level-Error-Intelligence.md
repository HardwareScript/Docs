# HardwareScript Clippy-Level Error Intelligence System

**Document Type:** Compiler Architecture & Feature Specification  
**Status:** 🚧 **PLANNED** - Comprehensive Error System Upgrade  
**Target:** v0.3.0 (6-month roadmap)  
**Author:** HardwareScript Core Team  
**Date:** 2024

---

## Executive Summary

This document outlines the complete transformation of HardwareScript's error system from **30% Rust-quality** to **100% Clippy-level intelligence** while maintaining the compiler's exceptional speed (<100ms for typical designs).

**Current State (v0.2.0):**
- Basic error codes (C11, R15, etc.)
- Single-location error reporting
- Generic help messages
- No suggestions or typo correction
- No static analysis beyond syntax
- No lints or warnings

**Target State (v0.3.0):**
- Multi-span causality chains
- Context-aware error messages
- Intelligent suggestions with examples
- Fuzzy matching for typos
- Full static analysis (symbol resolution, exhaustiveness, dead code)
- 50+ hardware-specific lints
- Sub-200ms compilation even with full analysis

**Key Insight:** HardwareScript is a *description language*, not a Turing-complete programming language. This means we can achieve Clippy-level intelligence *faster* than Rust because:
- No control flow analysis needed (no if/while/loops at runtime)
- No lifetime tracking
- No borrowing/ownership
- Declarative constraints are easier to validate than imperative code

---

## Section 1: Architecture Overview

### 1.1 The Five-Pass Analysis Pipeline

```
┌──────────────────────────────────────────────────────────────┐
│                    LEXER + PARSER                             │
│  Token stream → AST (10-30ms for typical designs)            │
└────────────────────────┬─────────────────────────────────────┘
                         │
                         v
┌──────────────────────────────────────────────────────────────┐
│              PASS 1: SYMBOL RESOLUTION                        │
│  Build symbol table, check all references exist (5-15ms)     │
│  • Materials, profiles, components, regions                  │
│  • Detect undefined references BEFORE compilation            │
│  • Build dependency graph for exhaustiveness checking        │
└────────────────────────┬─────────────────────────────────────┘
                         │
                         v
┌──────────────────────────────────────────────────────────────┐
│           PASS 2: CONSTRAINT VALIDATION                       │
│  Validate all relational constraints are satisfiable (10ms)  │
│  • Check cyclic dependencies in regions                      │
│  • Validate spacing constraints don't conflict               │
│  • Detect unreachable placements                             │
└────────────────────────┬─────────────────────────────────────┘
                         │
                         v
┌──────────────────────────────────────────────────────────────┐
│            PASS 3: PHYSICAL VALIDATION                        │
│  Check physical geometry rules (15-30ms)                     │
│  • Detect overlapping geometries                             │
│  • Validate material transitions (bridge rules)              │
│  • Check trace width vs. layer capabilities                  │
│  • Verify net continuity                                     │
└────────────────────────┬─────────────────────────────────────┘
                         │
                         v
┌──────────────────────────────────────────────────────────────┐
│              PASS 4: LINT ANALYSIS                            │
│  Run 50+ hardware-specific lints (20-40ms)                   │
│  • Unused nets, disconnected components                      │
│  • Suboptimal routing patterns                               │
│  • Design rule violations (DRC-lite)                         │
│  • Performance anti-patterns                                 │
└────────────────────────┬─────────────────────────────────────┘
                         │
                         v
┌──────────────────────────────────────────────────────────────┐
│           PASS 5: IR COMPILATION (existing)                   │
│  Generate physical geometry (30-50ms)                        │
└──────────────────────────────────────────────────────────────┘

TOTAL: 80-165ms (current: 40-80ms)
TARGET: <200ms with full analysis enabled
```

### 1.2 Parallel Analysis Architecture

**Key Optimization:** Run independent passes in parallel using Rayon.

```rust
// Pseudo-code for parallel analysis
let (symbol_errors, constraint_errors, physical_errors, lint_warnings) = rayon::join(
    || analyze_symbols(&ast, &collector),
    || analyze_constraints(&ast, &collector),
    || analyze_physical(&ast, &profile, &collector),
    || run_lints(&ast, &profile, &collector),
);
```

**Performance Target:**
- Sequential: 80ms + 10ms + 30ms + 40ms = 160ms
- Parallel (4 cores): max(80ms, 10ms, 30ms, 40ms) = 80ms

**Reality Check:** Most designs <1000 components finish in <100ms even with sequential analysis.



---

## Section 2: Multi-Span Error Causality

### 2.1 Current vs. Target Error Quality

**Current (v0.2.0):**
```
error[C36]: Material 'Titanium_Silicide' missing required 'process' field
  --> materials.hw:45:1
   |
45 | export material Titanium_Silicide:
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = help: Add 'process: deposited', 'process: etched', or 'process: drilled_plated'
```

**Target (v0.3.0):**
```
error[C36]: Material 'Titanium_Silicide' is missing required 'process' field
  --> materials.hw:45:1
   |
45 | export material Titanium_Silicide:
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ material defined here without process
46 |     category: conductor
47 |     symbol: "TiSi2"
   |
   = note: Manufacturing process must be explicitly declared - no defaults permitted
   = note: This material is used by contact 'Via_Source'
  --> pmos_transistor.hw:67:5
   |
67 |     add contact(Titanium_Silicide) named Via_Source at [x: 650nm, y: 1000nm]:
   |                 ^^^^^^^^^^^^^^^^^ used here
   |
   = help: Titanium_Silicide is a conductor typically used in ASIC contacts
   = help: Add one of the following to materials.hw:45:
   |
   |     process: deposited     # For CMOS/ASIC thin-film deposition
   |     process: etched        # For MEMS subtractive processes
   |     process: drilled_plated # For PCB-style vias (uncommon for silicides)
   |
   = note: See https://docs.hw-script.org/materials/process-field for details
```

### 2.2 Related Spans Implementation

**New Error Structure:**
```rust
#[derive(Error, Diagnostic, Debug)]
pub enum IrError {
    #[error("Material '{material}' is missing required 'process' field")]
    #[diagnostic(
        code(C36),
        url("https://docs.hw-script.org/errors/C36"),
    )]
    MaterialMissingProcess {
        material: CompactString,
        
        #[label("material defined here without 'process'")]
        definition_span: SourceSpan,
        
        #[related]
        usages: Vec<UsageSpan>,  // All places using this material
        
        #[help]
        suggestions: Vec<String>,
        
        #[note]
        context: Option<String>,
    },
}

#[derive(Debug)]
pub struct UsageSpan {
    #[label = "{usage_type}"]
    pub span: SourceSpan,
    pub usage_type: String,  // "used in contact", "used in plane", etc.
}
```

### 2.3 Causality Chain Tracking

**Problem:** User sees error at use site, doesn't know root cause is missing field in definition.

**Solution:** Track dependency chains during compilation.

```rust
pub struct ErrorContext {
    pub chain: Vec<CausalityLink>,
}

pub struct CausalityLink {
    pub span: SourceSpan,
    pub message: String,
    pub file: CompactString,
}

impl IrError {
    pub fn with_chain(mut self, chain: Vec<CausalityLink>) -> Self {
        // Augment error with causality information
        self
    }
}
```

**Example Chain:**
1. Material defined without process (materials.hw:45)
2. → Contact uses material (pmos_transistor.hw:67)
3. → Contact placement fails (compiler IR phase)



---

## Section 3: Intelligent Suggestions & Typo Correction

### 3.1 Fuzzy Matching for Undefined Symbols

**Current:**
```
error[C57]: Undefined material 'Aluminium'
  --> design.hw:12:15
   |
12 |     add plane(Aluminium) named Pad1:
   |               ^^^^^^^^^
```

**Target:**
```
error[C57]: Undefined material 'Aluminium'
  --> design.hw:12:15
   |
12 |     add plane(Aluminium) named Pad1:
   |               ^^^^^^^^^ not found in this scope
   |
   = note: No material named 'Aluminium' is defined or imported
   = help: Did you mean 'Aluminum'?
   |
   | Defined at: materials.hw:23:17
   |
23 | export material Aluminum:
   |                 ^^^^^^^^
   |
   = note: Other similar materials in scope:
   |   • Tungsten (defined at materials.hw:45)
   |   • Titanium_Silicide (defined at materials.hw:58)
```

### 3.2 Levenshtein Distance Implementation

**Crate:** `strsim = "0.11"`

```rust
use strsim::levenshtein;

pub fn find_similar_symbols(
    typo: &str,
    available: &[&str],
    max_distance: usize,
) -> Vec<(String, usize)> {
    let mut candidates: Vec<_> = available
        .iter()
        .map(|&name| (name.to_string(), levenshtein(typo, name)))
        .filter(|(_, distance)| *distance <= max_distance)
        .collect();
    
    candidates.sort_by_key(|(_, distance)| *distance);
    candidates.truncate(3);  // Show top 3 matches
    candidates
}
```

**Usage:**
```rust
if let Err(IrError::UndeclaredMaterial { material }) = result {
    let available_materials: Vec<&str> = symbol_table
        .materials()
        .keys()
        .map(|s| s.as_str())
        .collect();
    
    let suggestions = find_similar_symbols(&material, &available_materials, 3);
    
    if let Some((suggestion, distance)) = suggestions.first() {
        if *distance == 1 {
            eprintln!("help: Did you mean '{}'?", suggestion);
        } else {
            eprintln!("help: Similar materials: {}", 
                suggestions.iter().map(|(s, _)| s).join(", "));
        }
    }
}
```

### 3.3 Context-Aware Suggestions

**Example 1: Missing Layer**
```
error[R18]: Layer 'metal2' not defined in profile
  --> design.hw:34:20
   |
34 |     add plane(Copper) on layer: metal2:
   |                                 ^^^^^^ unknown layer
   |
   = note: Profile 'ASIC_Flattened' only defines these layers:
   |   • active (Silicon_N, 0-10nm)
   |   • poly (Polysilicon, 15-25nm)
   |   • metal1 (Aluminum, 35-45nm)
   |
   = help: Did you mean 'metal1'?
   = help: To add a new layer, update your profile's stackup:
   |
   |     stackup:
   |         metal2: [material: Aluminum, thickness: 10nm, routable: true]
```

**Example 2: Wrong Property Type**
```
error[C41]: Invalid value for 'diameter' property
  --> design.hw:45:20
   |
45 |         diameter: "200nm"
   |                   ^^^^^^^ expected measurement, found string
   |
   = note: The 'diameter' property expects a measurement value, not a string
   = help: Remove the quotes:
   |
45 |         diameter: 200nm
   |                   ^^^^^
```

### 3.4 Example-Based Error Messages

**Pattern:** Show correct usage alongside error.

```
error[C42]: Missing required property 'net'
  --> design.hw:23:5
   |
23 |     add plane(Copper) named Trace1 on layer: metal1:
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ missing 'net' assignment
24 |         shape: Trace(width: 20um)
25 |         at: [x: 100um, y: 100um, z: 40nm]
   |
   = note: All conductive placements must declare their net assignment
   = help: Add 'net:' property to specify electrical connectivity:
   |
   |     add plane(Copper) named Trace1 on layer: metal1:
   |         shape: Trace(width: 20um)
   |         at: [x: 100um, y: 100um, z: 40nm]
   |         net: Signal_A  // ← Add this line
```



---

## Section 4: Static Analysis - Symbol Resolution Pass

### 4.1 Pre-Compilation Symbol Validation

**Goal:** Catch all undefined references BEFORE attempting IR compilation.

**Current Problem:** Undefined material causes crash deep in IR phase.

**Solution:** Build complete symbol table in Pass 1, validate all references.

```rust
pub struct SymbolResolver {
    materials: HashMap<CompactString, MaterialDef>,
    profiles: HashMap<CompactString, ProfileDef>,
    components: HashMap<CompactString, ComponentDef>,
    regions: HashMap<CompactString, RegionDef>,
    nets: HashMap<CompactString, NetDef>,
}

impl SymbolResolver {
    pub fn resolve_all_references(
        &self,
        ast: &Program,
        collector: &DiagnosticCollector,
    ) -> Result<(), ()> {
        // Validate every symbol reference in the entire AST
        for space in &ast.spaces {
            self.validate_space_references(space, collector)?;
        }
        
        // Return Err if any errors were collected
        if collector.has_errors() {
            Err(())
        } else {
            Ok(())
        }
    }
    
    fn validate_space_references(
        &self,
        space: &SpaceDef,
        collector: &DiagnosticCollector,
    ) -> Result<(), ()> {
        // Check profile exists
        if let Some(profile_name) = &space.profile {
            if !self.profiles.contains_key(profile_name) {
                collector.report(IrError::ProfileNotFound {
                    name: profile_name.clone(),
                    span: space.profile_span,
                    available: self.profiles.keys().cloned().collect(),
                });
            }
        }
        
        // Check all placements reference valid materials
        for placement in &space.placements {
            if !self.materials.contains_key(&placement.material) {
                let suggestions = find_similar_symbols(
                    &placement.material,
                    &self.materials.keys().map(|s| s.as_str()).collect::<Vec<_>>(),
                    2,
                );
                
                collector.report(IrError::UndeclaredMaterial {
                    material: placement.material.clone(),
                    span: placement.material_span,
                    suggestion: suggestions.first().map(|(s, _)| s.clone()),
                    available: self.materials.keys().cloned().collect(),
                });
            }
        }
        
        // Check all regions referenced in placements exist
        for placement in &space.placements {
            if let Some(region_ref) = &placement.inside_region {
                if !self.regions.contains_key(region_ref) {
                    collector.report(IrError::RegionNotFound {
                        region: region_ref.clone(),
                        span: placement.span,
                        defined_regions: self.regions.keys().cloned().collect(),
                    });
                }
            }
        }
        
        Ok(())
    }
}
```

### 4.2 Dependency Graph Construction

**Purpose:** Detect cyclic dependencies and unreachable definitions.

```rust
pub struct DependencyGraph {
    nodes: HashMap<Symbol, Node>,
    edges: Vec<(Symbol, Symbol)>,  // (from, to)
}

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub enum Symbol {
    Material(CompactString),
    Profile(CompactString),
    Region(CompactString),
    Component(CompactString),
}

impl DependencyGraph {
    pub fn detect_cycles(&self) -> Vec<Vec<Symbol>> {
        // Tarjan's algorithm for strongly connected components
        // Return list of dependency cycles
        unimplemented!("Use petgraph crate")
    }
    
    pub fn find_unreachable(&self, roots: &[Symbol]) -> Vec<Symbol> {
        // Find symbols that are defined but never used
        let mut reachable = HashSet::new();
        for root in roots {
            self.dfs_mark_reachable(root, &mut reachable);
        }
        
        self.nodes
            .keys()
            .filter(|sym| !reachable.contains(sym))
            .cloned()
            .collect()
    }
}
```

**Example Error:**
```
error[C22]: Circular dependency detected
  --> design.hw:12:1
   |
12 | region RegionA:
   | ^^^^^^^^^^^^^^^ defined here
13 |     right_of: RegionB with spacing: 100um
   |               ------- depends on RegionB
   |
  --> design.hw:18:1
   |
18 | region RegionB:
   | ^^^^^^^^^^^^^^^ defined here
19 |     right_of: RegionA with spacing: 100um
   |               ------- depends on RegionA
   |
   = note: Dependency cycle: RegionA → RegionB → RegionA
   = help: Break the cycle by using absolute coordinates for one region:
   |
   | region RegionA:
   |     at: [x: 100um, y: 100um, z: 0nm]  // Use absolute instead of relative
   |     boundary: [width: 50um, height: 50um]
```

### 4.3 Exhaustiveness Checking

**Concept:** Ensure all cases are handled (like Rust's match exhaustiveness).

**Example 1: Bridge Rules**
```rust
// Check that all material transitions have bridge rules
pub fn check_bridge_exhaustiveness(
    materials: &HashMap<CompactString, MaterialDef>,
    bridges: &[BridgeDef],
    profile: &ProfileDef,
) -> Vec<MissingBridge> {
    let mut missing = Vec::new();
    
    // Get all conductive materials used in stackup
    let conductors: Vec<_> = profile.stackup
        .layers
        .iter()
        .filter(|layer| {
            materials.get(&layer.material)
                .map(|m| m.category.is_conductive())
                .unwrap_or(false)
        })
        .map(|layer| &layer.material)
        .collect();
    
    // Check all pairwise transitions
    for (i, from_mat) in conductors.iter().enumerate() {
        for to_mat in &conductors[i+1..] {
            let has_bridge = bridges.iter().any(|b| {
                (&b.from_material == *from_mat && &b.to_material == *to_mat) ||
                (&b.from_material == *to_mat && &b.to_material == *from_mat)
            });
            
            if !has_bridge {
                missing.push(MissingBridge {
                    from: (*from_mat).clone(),
                    to: (*to_mat).clone(),
                });
            }
        }
    }
    
    missing
}
```

**Example Error:**
```
warning[W12]: Incomplete bridge rules
  --> profile.hw:45:1
   |
45 | profile ASIC_Standard:
   | ^^^^^^^^^^^^^^^^^^^^^^ profile defined here
   |
   = note: This profile's stackup uses 3 conductive materials:
   |   • Silicon_N (active layer)
   |   • Polysilicon (poly layer)
   |   • Aluminum (metal1 layer)
   |
   = note: Only 2 of 3 required bridge rules are defined:
   |   ✓ Silicon_N → Aluminum (defined at bridges.hw:12)
   |   ✓ Polysilicon → Aluminum (defined at bridges.hw:18)
   |   ✗ Silicon_N → Polysilicon (MISSING)
   |
   = help: Add missing bridge rule:
   |
   | bridge Silicon_N to Polysilicon:
   |     interface: Titanium_Silicide
   |     thickness: 5nm
   |     fill: Tungsten
```



---

## Section 5: Hardware-Specific Lints (Clippy for Silicon)

### 5.1 Lint Categories

**Performance Lints:**
- Suboptimal trace routing (unnecessary detours)
- Excessive via usage
- Long signal paths (high resistance/capacitance)

**Design Rule Lints:**
- Spacing violations (too close, manufacturability issues)
- Aspect ratio warnings (tall thin traces)
- Clearance violations

**Electrical Lints:**
- Unconnected nets (floating nodes)
- Missing power/ground connections
- Impedance mismatches

**Best Practice Lints:**
- Inconsistent naming conventions
- Unused definitions
- Magic numbers (hardcoded values without constants)

### 5.2 Lint Implementation Architecture

```rust
pub trait Lint {
    fn name(&self) -> &'static str;
    fn level(&self) -> LintLevel;
    fn check(&self, ast: &Program, ctx: &LintContext) -> Vec<LintViolation>;
}

pub enum LintLevel {
    Allow,
    Warn,
    Deny,
    Forbid,
}

pub struct LintViolation {
    pub span: SourceSpan,
    pub message: String,
    pub suggestion: Option<String>,
    pub related: Vec<(SourceSpan, String)>,
}

pub struct LintContext<'a> {
    pub symbol_table: &'a SymbolTable,
    pub profile: Option<&'a ProfileDef>,
    pub config: &'a LintConfig,
}
```

### 5.3 Example Lints

#### Lint 1: `unused_material`
```rust
pub struct UnusedMaterial;

impl Lint for UnusedMaterial {
    fn name(&self) -> &'static str {
        "unused_material"
    }
    
    fn level(&self) -> LintLevel {
        LintLevel::Warn
    }
    
    fn check(&self, ast: &Program, ctx: &LintContext) -> Vec<LintViolation> {
        let mut violations = Vec::new();
        
        // Find all material definitions
        let defined: HashMap<_, _> = ast.materials
            .iter()
            .map(|m| (&m.name, m.span))
            .collect();
        
        // Find all material usages
        let mut used = HashSet::new();
        for space in &ast.spaces {
            for placement in &space.placements {
                used.insert(&placement.material);
            }
        }
        
        // Report unused materials
        for (name, span) in defined {
            if !used.contains(name) {
                violations.push(LintViolation {
                    span,
                    message: format!("Material '{}' is defined but never used", name),
                    suggestion: Some(format!("Remove unused material or mark as #[allow(unused)]")),
                    related: vec![],
                });
            }
        }
        
        violations
    }
}
```

**Output:**
```
warning[unused_material]: Material 'Gold' is defined but never used
  --> materials.hw:67:17
   |
67 | export material Gold:
   |                 ^^^^ defined here but never referenced
   |
   = help: If this material is intentionally unused (for future use), suppress this warning:
   |
   | #[allow(unused)]
   | export material Gold:
   |
   = note: To remove this warning, delete the material definition or use it in a placement
```

#### Lint 2: `long_trace_path`
```rust
pub struct LongTracePath {
    max_length_um: f64,
}

impl Lint for LongTracePath {
    fn name(&self) -> &'static str {
        "long_trace_path"
    }
    
    fn level(&self) -> LintLevel {
        LintLevel::Warn
    }
    
    fn check(&self, ast: &Program, ctx: &LintContext) -> Vec<LintViolation> {
        let mut violations = Vec::new();
        
        for space in &ast.spaces {
            for route in &space.routes {
                let path_length = self.calculate_path_length(&route.waypoints);
                
                if path_length > self.max_length_um * 1000.0 {  // Convert to nm
                    let length_um = path_length / 1000.0;
                    
                    violations.push(LintViolation {
                        span: route.span,
                        message: format!(
                            "Route path is {:.1}um long (exceeds recommended {:.1}um)",
                            length_um, self.max_length_um
                        ),
                        suggestion: Some("Consider adding intermediate vias or optimizing placement".into()),
                        related: vec![],
                    });
                }
            }
        }
        
        violations
    }
    
    fn calculate_path_length(&self, waypoints: &[Point3D]) -> f64 {
        waypoints
            .windows(2)
            .map(|w| {
                let dx = (w[1].x - w[0].x) as f64;
                let dy = (w[1].y - w[0].y) as f64;
                (dx * dx + dy * dy).sqrt()
            })
            .sum()
    }
}
```

**Output:**
```
warning[long_trace_path]: Route path is 8.5mm long (exceeds recommended 5.0mm)
  --> design.hw:123:5
   |
123 |     route PadA to PadB:
    |     ^^^^^^^^^^^^^^^^^^ long signal path
124 |         layer: metal1
125 |         width: 20um
    |
    = note: Long traces increase resistance and capacitance
    = note: Path length: 8.5mm
    = note: Recommended maximum: 5.0mm for this technology
    = help: Consider:
    |   • Moving components closer together
    |   • Using wider traces to reduce resistance
    |   • Adding buffering for high-speed signals
```

#### Lint 3: `magic_number`
```rust
pub struct MagicNumber;

impl Lint for MagicNumber {
    fn name(&self) -> &'static str {
        "magic_number"
    }
    
    fn level(&self) -> LintLevel {
        LintLevel::Warn
    }
    
    fn check(&self, ast: &Program, ctx: &LintContext) -> Vec<LintViolation> {
        let mut violations = Vec::new();
        
        // Exempt list: 0, 1, common measurements
        let exempt = HashSet::from([0, 1, 100, 1000]);
        
        for space in &ast.spaces {
            for placement in &space.placements {
                if let Some(coord) = &placement.position {
                    // Check for hardcoded values (not from let bindings)
                    if !self.is_symbolic(&coord.x) {
                        let value = self.extract_literal(&coord.x);
                        if !exempt.contains(&value.abs()) {
                            violations.push(LintViolation {
                                span: coord.x_span,
                                message: format!("Hardcoded coordinate value: {}nm", value),
                                suggestion: Some("Extract to named constant".into()),
                                related: vec![],
                            });
                        }
                    }
                }
            }
        }
        
        violations
    }
}
```

**Output:**
```
warning[magic_number]: Hardcoded coordinate value: 2573nm
  --> design.hw:45:15
   |
45 |         at: [x: 2573nm, y: 1847nm, z: 40nm]
   |                 ^^^^^^ magic number
   |
   = help: Extract to named constant for maintainability:
   |
   | let pad_x_offset = 2573nm
   | let pad_y_offset = 1847nm
   |
   | add plane(Copper) named Pad1:
   |     at: [x: pad_x_offset, y: pad_y_offset, z: 40nm]
```

#### Lint 4: `unconnected_net`
```rust
pub struct UnconnectedNet;

impl Lint for UnconnectedNet {
    fn name(&self) -> &'static str {
        "unconnected_net"
    }
    
    fn level(&self) -> LintLevel {
        LintLevel::Warn
    }
    
    fn check(&self, ast: &Program, ctx: &LintContext) -> Vec<LintViolation> {
        let mut violations = Vec::new();
        
        // Build net connectivity graph
        let mut net_connections: HashMap<String, Vec<SourceSpan>> = HashMap::new();
        
        for space in &ast.spaces {
            for placement in &space.placements {
                if let Some(net) = &placement.net {
                    net_connections
                        .entry(net.to_string())
                        .or_default()
                        .push(placement.span);
                }
            }
        }
        
        // Check for nets with only one connection (floating)
        for (net_name, connections) in net_connections {
            if connections.len() == 1 {
                violations.push(LintViolation {
                    span: connections[0],
                    message: format!("Net '{}' has only one connection (floating)", net_name),
                    suggestion: Some("Add at least one more component to this net".into()),
                    related: vec![],
                });
            }
        }
        
        violations
    }
}
```

**Output:**
```
warning[unconnected_net]: Net 'TestSignal' has only one connection (floating)
  --> design.hw:78:9
   |
78 |         net: TestSignal
   |              ^^^^^^^^^^ net declared here
   |
   = note: This net is only connected to 'Pad_Test' at line 75
   = note: Floating nets have no current path and may indicate a design error
   = help: Add at least one more component or route to complete the circuit
```

### 5.4 Lint Configuration

**File:** `.hwlint.toml` (project root)

```toml
[lints]
# Default lint levels
unused_material = "warn"
long_trace_path = "warn"
magic_number = "warn"
unconnected_net = "warn"
spacing_violation = "deny"
missing_bridge_rule = "deny"

[lints.long_trace_path]
# Customize lint parameters
max_length_um = 5.0

[lints.magic_number]
# Exempt specific values
exempt_values = [0, 1, 100, 1000, 50, 200]

[allow]
# Suppress specific lints for certain files
"tests/**/*.hw" = ["magic_number", "unused_material"]
```

### 5.5 Inline Lint Control

**Syntax:** `#[allow(lint_name)]` or `#[deny(lint_name)]`

```hardware
# Suppress unused warning for intentionally defined material
#[allow(unused_material)]
export material ExperimentalAlloy:
    category: conductor
    process: deposited
    properties:
        resistivity: 1e-7ohm_m

# Deny long traces for critical signals
#[deny(long_trace_path)]
space HighSpeedDesign:
    # This space will error (not warn) on long traces
    route ClockSignal to AllChips:
        layer: metal1
```



---

## Section 6: Performance Optimization Strategies

### 6.1 Incremental Analysis

**Problem:** Re-analyzing entire file on every change is wasteful.

**Solution:** Track which definitions changed, only re-analyze affected symbols.

```rust
pub struct IncrementalCache {
    file_hashes: HashMap<PathBuf, u64>,
    symbol_dependencies: DependencyGraph,
    analysis_results: HashMap<Symbol, AnalysisResult>,
}

impl IncrementalCache {
    pub fn analyze_incremental(
        &mut self,
        changed_files: &[PathBuf],
        ast: &Program,
    ) -> Vec<Diagnostic> {
        let mut invalidated = HashSet::new();
        
        // Find all symbols in changed files
        for file in changed_files {
            let symbols = self.extract_symbols_from_file(file, ast);
            invalidated.extend(symbols);
        }
        
        // Transitively invalidate dependencies
        for symbol in &invalidated.clone() {
            let deps = self.symbol_dependencies.get_dependents(symbol);
            invalidated.extend(deps);
        }
        
        // Re-analyze only invalidated symbols
        let mut diagnostics = Vec::new();
        for symbol in invalidated {
            let result = self.analyze_symbol(&symbol, ast);
            diagnostics.extend(result);
            self.analysis_results.insert(symbol.clone(), result);
        }
        
        diagnostics
    }
}
```

**Performance Impact:**
- Full analysis: 100ms
- Incremental (10% change): 15ms

### 6.2 Parallel Lint Execution

**Use Rayon for lint parallelization:**

```rust
pub fn run_all_lints(
    ast: &Program,
    ctx: &LintContext,
    enabled_lints: &[Box<dyn Lint + Sync>],
) -> Vec<LintViolation> {
    use rayon::prelude::*;
    
    enabled_lints
        .par_iter()  // Parallel iterator
        .flat_map(|lint| lint.check(ast, ctx))
        .collect()
}
```

**Performance Impact:**
- Sequential: 40ms (10 lints × 4ms each)
- Parallel (4 cores): 12ms

### 6.3 Lazy Symbol Table Construction

**Idea:** Don't build entire symbol table upfront - build on demand.

```rust
pub struct LazySymbolTable {
    materials: OnceCell<HashMap<CompactString, MaterialDef>>,
    profiles: OnceCell<HashMap<CompactString, ProfileDef>>,
    // ... other caches
}

impl LazySymbolTable {
    pub fn get_material(&self, name: &str) -> Option<&MaterialDef> {
        self.materials
            .get_or_init(|| self.build_material_table())
            .get(name)
    }
    
    fn build_material_table(&self) -> HashMap<CompactString, MaterialDef> {
        // Expensive operation, only runs once
        unimplemented!()
    }
}
```

### 6.4 Smart Error Limits

**Problem:** Cascading errors spam the output.

**Solution:** Stop analysis early if error density is too high.

```rust
pub struct SmartErrorCollector {
    errors: Vec<Diagnostic>,
    max_errors: usize,
    error_density_threshold: f64,  // errors per 100 LOC
}

impl SmartErrorCollector {
    pub fn should_continue(&self, current_line: usize) -> bool {
        if self.errors.len() >= self.max_errors {
            return false;
        }
        
        let density = (self.errors.len() as f64) / (current_line as f64) * 100.0;
        if density > self.error_density_threshold {
            eprintln!("⚠️  High error density detected. Stopping early.");
            eprintln!("   Fix the most critical errors first.");
            return false;
        }
        
        true
    }
}
```

**Example:**
```
error[C57]: Undefined material 'Copper'
  --> design.hw:12:15
   |
12 |     add plane(Copper) named Pad1:
   |               ^^^^^^

error[C57]: Undefined material 'Copper'
  --> design.hw:18:15
   |
18 |     add plane(Copper) named Pad2:
   |               ^^^^^^

⚠️  High error density detected (2 errors in 20 lines). Stopping early.
   Fix missing material import at design.hw:1 and re-compile.
   
   = help: Add to top of file:
   |
   | import Copper from "./materials"
```

### 6.5 Benchmark Targets

**Target Compilation Times:**

| Design Size | Lines | Components | Analysis Time | Full Build |
|-------------|-------|------------|---------------|------------|
| Tiny | <100 | <10 | <50ms | <150ms |
| Small | 100-500 | 10-50 | <100ms | <300ms |
| Medium | 500-2000 | 50-200 | <150ms | <600ms |
| Large | 2000-5000 | 200-1000 | <300ms | <1200ms |
| Huge | >5000 | >1000 | <500ms | <2000ms |

**Measured on:**
- CPU: AMD Ryzen 7 or equivalent
- RAM: 8GB
- Storage: SSD

**Clippy Comparison:**
- Rust crate (1000 LOC): ~500ms with Clippy
- HardwareScript (1000 LOC): <150ms with full analysis
- **Target:** 3× faster than Clippy

