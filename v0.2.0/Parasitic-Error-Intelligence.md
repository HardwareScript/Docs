# Parasitic Validation Error Intelligence System

**Document Type:** Compiler Error System - Parasitic Analysis Module  
**Status:** 🚧 **PLANNED** - Extension to Clippy-Level Error Intelligence  
**Target:** v0.3.0 (6-month roadmap)  
**Companion Doc:** `Clippy-Level-Error-Intelligence.md`  
**Author:** HardwareScript Core Team  
**Date:** 2024

---

## Executive Summary

This document extends the Clippy-Level Error Intelligence system with **detailed parasitic validation error reporting**. The current DRC system detects electromigration, thermal, crosstalk, and forbidden junction violations but reports them with generic messages. This specification defines comprehensive error structures with multi-span causality, material property context, and actionable fix recommendations.

**Current State (v0.2.0):**
```
❌ DRC VIOLATIONS DETECTED:
• Current density violation for In at [1150nm, 2000nm, 50nm]: 5000.00 A/mm² actual, 0.10 A/mm² max
```

**Target State (v0.3.0) - CONCISE FORMAT:**
```
error[P21]: current density 5.0 mA/μm² exceeds material limit 0.1 mA/μm²
  --> test_thermal_violation.hw:125:14
   |
125 |         width: 200nm
    |                ^^^^ trace too narrow for 100μA current (A = 0.01μm²)
    |
  --> materials.hw:45:5
   |
45  |     max_current_density: 0.1
    |     ^^^^^^^^^^^^^^^^^^^^^^^^ Polysilicon limit
    |
    = note: J = I/A = 100μA / 0.01μm² = 5000 A/mm²
    = help: increase width to 1000nm, reduce current to 10μA, or use Copper (20× higher limit)
```

**Comparison with verbose format:**
- ❌ **Verbose:** 35+ lines with multi-level bullets and ASCII art
- ✅ **Concise:** 12 lines with essential info only
- ✅ Rust-style: One-line help with alternatives, compact note with formula

---

## Section 0: Design Philosophy - Rust-Style Conciseness

### 0.1 Core Principles

**WRONG - Verbose and overwhelming:**
```
= help: To fix this electromigration violation, choose one of:
    
    1. Increase trace width (RECOMMENDED):
       width: 1000nm  // Reduces J to 2.0 mA/μm² (20× safety margin)
    
    2. Reduce net current limit:
       current: 10uA  // Reduces J to 1.0 mA/μm² (10× safety margin)
    
    3. Use a different material with higher J_max:
       • Copper (2.0 mA/μm²) - 20× higher limit
       • Aluminum (1.0 mA/μm²) - 10× higher limit
```

**RIGHT - Concise and actionable:**
```
= help: increase width to 1000nm, reduce current to 10μA, or use Copper (20× higher limit)
```

### 0.2 Error Message Guidelines

| Element | Max Length | Example |
|---------|------------|---------|
| **Error title** | 1 line | `current density 5.0 mA/μm² exceeds limit 0.1 mA/μm²` |
| **Label** | 1 line | `trace too narrow for 100μA current (A = 0.01μm²)` |
| **Note** | 1-2 lines | `J = I/A = 100μA / 0.01μm² = 5000 A/mm²` |
| **Help** | 1 line | `increase width to 1000nm or use Copper (20× higher limit)` |
| **Related spans** | 2-4 locations | Material def + profile layer (essential only) |

### 0.3 Information Density Targets

- **Primary span:** Show the violation site with inline context
- **Related spans:** 2-4 max, only direct causality (not every indirect reference)
- **Notes:** Essential calculation only, no explanatory paragraphs
- **Help:** Comma-separated options, not numbered lists
- **Documentation:** Single URL at end, not inline lectures

### 0.4 Verbosity Levels (User Configurable)

```toml
# .hwconfig.toml
[errors]
verbosity = "standard"  # Options: minimal, standard, detailed

# minimal: Error + primary span only
# standard: + 1-2 related spans + 1 note + 1 help (DEFAULT)
# detailed: + all related spans + calculations + URLs
```

---

## Section 1: Parasitic Error Code Architecture


### 1.1 Error Code Taxonomy

All parasitic validation errors use the **P** prefix (Physical/Parasitic):

| Code | Name | Category | Severity | Description |
|------|------|----------|----------|-------------|
| **P21** | Electromigration Violation | Current Density | Error | J = I/A exceeds material max_current_density |
| **P22** | Thermal Rise Violation | Temperature | Error | I²R heating exceeds max_temp_rise |
| **P23** | Crosstalk Violation | Signal Integrity | Error | Coupling capacitance exceeds crosstalk budget |
| **P24** | Voltage Drop Violation | IR Drop | Error | Resistive drop exceeds supply tolerance |
| **P25** | Inductance Violation | Signal Integrity | Warning | Via/trace inductance may affect high-speed signals |
| **P45** | Forbidden Junction | Material Interface | Error | Direct contact between incompatible materials |
| **P46** | Missing Bridge Rule | Profile Completeness | Warning | No bridge defined for conductor pair transition |
| **P47** | Bridge Interface Mismatch | Material Compatibility | Error | Interface material incompatible with conductors |

### 1.2 Error Structure Database Schema

Each parasitic error requires rich contextual data stored during compilation:

```rust
/// Master parasitic error context - attached to every P-series error
pub struct ParasiticErrorContext {
    /// Geometric location of violation
    pub location: Point3D,
    
    /// Net name involved
    pub net_name: CompactString,
    
    /// Material properties at violation site
    pub material_props: MaterialProperties,
    
    /// Profile constraints
    pub profile_limits: ProfileLimits,
    
    /// Calculated parasitic values
    pub parasitics: ParasiticValues,
    
    /// Multi-file span chain
    pub causality_chain: Vec<CausalitySpan>,
}

/// Material properties relevant to parasitic analysis
pub struct MaterialProperties {
    pub name: CompactString,
    pub category: MaterialCategory,
    pub resistivity: Option<f64>,              // Ω·m
    pub thermal_conductivity: Option<f64>,     // W/(m·K)
    pub max_current_density: Option<f64>,      // A/mm²
    pub relative_permittivity: Option<f64>,    // εr (dimensionless)
    pub definition_span: SourceSpan,
}

/// Profile-defined physical limits
pub struct ProfileLimits {
    pub profile_name: CompactString,
    pub layer_name: CompactString,
    pub layer_thickness: f64,                  // nm
    pub min_width: f64,                        // nm
    pub min_spacing: f64,                      // nm
    pub max_temp_rise: f64,                    // °C
    pub max_crosstalk_db: Option<f64>,         // dB
    pub profile_span: SourceSpan,
    pub layer_span: SourceSpan,
}

/// Calculated parasitic values at violation point
pub struct ParasiticValues {
    pub resistance: Option<f64>,               // Ω
    pub capacitance: Option<f64>,              // F
    pub inductance: Option<f64>,               // H
    pub coupling_capacitance: Option<f64>,     // F (for crosstalk)
    pub current_density: Option<f64>,          // A/mm²
    pub power_dissipation: Option<f64>,        // W
    pub temperature_rise: Option<f64>,         // °C
    pub voltage_drop: Option<f64>,             // V
}

/// Causality span for multi-file error reporting
pub struct CausalitySpan {
    pub file: PathBuf,
    pub span: SourceSpan,
    pub label: String,
    pub priority: CausalityPriority,
}

pub enum CausalityPriority {
    Primary,      // The actual violation site
    Contributing, // Definition that influenced the violation
    Contextual,   // Related but not directly causal
}
```

---

## Section 2: Error P21 - Electromigration Violation


### 2.1 Detection Logic

**Trigger Condition:**
```rust
let cross_section = trace_width * layer_thickness;  // nm²
let current_density = peak_current / (cross_section * 1e-12);  // A/mm²

if current_density > material.max_current_density {
    // VIOLATION: P21
}
```

**Required Data Collection:**
1. Trace geometry (width, thickness from route + profile)
2. Net current declaration (from `nets:` block)
3. Material max_current_density (from material definition)
4. All source locations (route statement, net declaration, material def, layer def)

### 2.2 Error Structure Implementation

```rust
#[derive(Error, Diagnostic, Debug)]
#[error("Electromigration violation - current density exceeds material limit")]
#[diagnostic(
    code(P21),
    url("https://docs.hw-script.org/errors/P21"),
    severity(Error),
)]
pub struct ElectromigrationViolation {
    /// Primary violation site - the route statement
    #[label("trace carrying {current_ua}μA")]
    pub route_span: SourceSpan,
    
    /// Width specification that's too narrow
    #[label("cross-section too small for current")]
    pub width_span: SourceSpan,
    
    /// Related spans for causality chain
    #[related]
    pub related_spans: Vec<RelatedSpan>,
    
    /// Calculated values for display
    pub current_ua: f64,
    pub width_nm: f64,
    pub thickness_nm: f64,
    pub area_um2: f64,
    pub current_density_actual: f64,
    pub current_density_limit: f64,
    pub location: Point3D,
    pub net_name: CompactString,
    pub material_name: CompactString,
    
    /// Fix suggestions
    #[help]
    pub suggestions: Vec<String>,
    
    /// Additional context notes
    #[note]
    pub notes: Vec<String>,
}

#[derive(Debug)]
pub struct RelatedSpan {
    #[label = "{message}"]
    pub span: SourceSpan,
    pub message: String,
    pub file: Option<PathBuf>,
}
```

### 2.3 Error Construction Function


```rust
impl ElectromigrationViolation {
    pub fn new(
        route: &RouteDef,
        net: &NetDef,
        material: &MaterialDef,
        layer: &LayerDef,
        violation_point: Point3D,
        trace_geometry: &TraceGeometry,
    ) -> Self {
        let cross_section = trace_geometry.width * layer.thickness;  // nm²
        let area_um2 = cross_section / 1e6;  // Convert to μm²
        let j_actual = net.current / area_um2;  // mA/μm²
        let j_limit = material.max_current_density.unwrap();
        
        // Build causality chain
        let mut related = vec![
            RelatedSpan {
                span: violation_point.to_source_span(),
                message: format!("Detected at position [x: {}nm, y: {}nm, z: {}nm]",
                    violation_point.x, violation_point.y, violation_point.z),
                file: None,
            },
            RelatedSpan {
                span: material.max_current_density_span,
                message: format!("{} limit: {} mA/μm²", material.name, j_limit),
                file: Some(material.file.clone()),
            },
            RelatedSpan {
                span: layer.thickness_span,
                message: "layer thickness defined here".into(),
                file: Some(layer.file.clone()),
            },
            RelatedSpan {
                span: net.current_span,
                message: format!("net current: {}μA", net.current * 1e6),
                file: Some(net.file.clone()),
            },
        ];
        
        // Generate fix suggestions
        let suggestions = Self::generate_suggestions(
            trace_geometry.width,
            layer.thickness,
            net.current,
            j_limit,
            material,
        );
        
        // Generate explanatory notes
        let notes = vec![
            format!("Calculation breakdown:"),
            format!("  • Trace width: {}nm", trace_geometry.width),
            format!("  • Layer thickness: {}nm (from profile {} layer)",
                layer.thickness, layer.name),
            format!("  • Cross-sectional area: A = {}nm × {}nm = {:.2} μm²",
                trace_geometry.width, layer.thickness, area_um2),
            format!("  • Peak current: I = {}μA (declared in net)",
                net.current * 1e6),
            format!("  • Current density: J = I/A = {}μA / {:.2}μm² = {:.2} A/mm² = {:.2} mA/μm²",
                net.current * 1e6, area_um2, j_actual * 1e3, j_actual),
            String::new(),
            "Electromigration causes metal atoms to migrate under high current,".into(),
            "eventually leading to open circuits or shorts. This is a hard physical".into(),
            "limit that cannot be bypassed.".into(),
        ];
        
        Self {
            route_span: route.span,
            width_span: route.width_span,
            related_spans: related,
            current_ua: net.current * 1e6,
            width_nm: trace_geometry.width,
            thickness_nm: layer.thickness,
            area_um2,
            current_density_actual: j_actual,
            current_density_limit: j_limit,
            location: violation_point,
            net_name: net.name.clone(),
            material_name: material.name.clone(),
            suggestions,
            notes,
        }
    }
    
    fn generate_suggestions(
        width_nm: f64,
        thickness_nm: f64,
        current_a: f64,
        j_limit: f64,
        material: &MaterialDef,
    ) -> Vec<String> {
        let mut suggestions = Vec::new();
        
        // Calculate required width for safe operation (with 10× safety margin)
        let cross_section_required = current_a / (j_limit * 0.1 * 1e-12);  // nm²
        let width_required = cross_section_required / thickness_nm;
        let safety_factor = j_limit / (current_a / (width_nm * thickness_nm * 1e-12));
        
        suggestions.push("To fix this electromigration violation, choose one of:".into());
        suggestions.push(String::new());
        suggestions.push(format!(
            "  1. Increase trace width (RECOMMENDED):\n     width: {}nm  // Reduces J to {:.2} mA/μm² ({}× safety margin)",
            (width_required as i64 + 50) / 100 * 100,  // Round up to nearest 100nm
            j_limit * 0.1,
            10
        ));
        suggestions.push(String::new());
        suggestions.push(format!(
            "  2. Reduce net current limit:\n     current: {}uA  // Reduces J to {:.2} mA/μm² ({}× safety margin)",
            (current_a * 1e6 * 0.1) as i64,
            j_limit * 0.1,
            10
        ));
        suggestions.push(String::new());
        suggestions.push("  3. Use a different material with higher J_max:".into());
        suggestions.push("     • Copper (2.0 mA/μm²) - 20× higher limit".into());
        suggestions.push("     • Aluminum (1.0 mA/μm²) - 10× higher limit".into());
        suggestions.push("     • Tungsten (10.0 mA/μm²) - 100× higher limit".into());
        
        suggestions
    }
}
```

---

## Section 3: Error P22 - Thermal Rise Violation


### 3.1 Detection Logic

**Trigger Condition:**
```rust
let resistance = material.resistivity * length / (width * thickness * 1e-18);  // Ω
let power = current_rms * current_rms * resistance;  // W
let temp_rise = Self::calculate_thermal_rise(power, geometry, material.thermal_k);

if temp_rise > profile.max_temp_rise {
    // VIOLATION: P22
}
```

**Thermal Rise Calculation (Simplified 1D Model):**
```rust
fn calculate_thermal_rise(
    power_w: f64,
    geometry: &TraceGeometry,
    thermal_conductivity: f64,
) -> f64 {
    // IPC-2152 approximation for trace heating
    // ΔT ≈ P / (k × A_dissipation)
    let surface_area = 2.0 * geometry.length * geometry.width * 1e-18;  // m²
    let delta_t = power_w / (thermal_conductivity * surface_area);
    delta_t
}
```

### 3.2 Error Structure Implementation

```rust
#[derive(Error, Diagnostic, Debug)]
#[error("Thermal rise violation - I²R heating exceeds temperature budget")]
#[diagnostic(
    code(P22),
    url("https://docs.hw-script.org/errors/P22"),
    severity(Error),
)]
pub struct ThermalRiseViolation {
    #[label("trace with excessive heating")]
    pub route_span: SourceSpan,
    
    #[label("geometry causes high resistance")]
    pub width_span: SourceSpan,
    
    #[related]
    pub related_spans: Vec<RelatedSpan>,
    
    pub net_name: CompactString,
    pub material_name: CompactString,
    pub location: Point3D,
    
    // Thermal calculations
    pub resistance_ohms: f64,
    pub current_rms_ua: f64,
    pub power_uw: f64,
    pub temp_rise_actual: f64,
    pub temp_rise_limit: f64,
    pub thermal_conductivity: f64,
    
    // Geometry
    pub length_um: f64,
    pub width_nm: f64,
    pub thickness_nm: f64,
    
    #[help]
    pub suggestions: Vec<String>,
    
    #[note]
    pub notes: Vec<String>,
}
```

### 3.3 Example Output


```
error[P22]: Thermal rise violation - I²R heating exceeds temperature budget
  --> test_thermal_violation.hw:123:5
   |
123 |     route Left_Terminal to Right_Terminal:
    |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ trace with excessive heating
124 |         net: In
125 |         width: 200nm
    |              ^^^^^^ geometry causes high resistance
    |
    = note: Detected at trace segment [x: 4000nm, y: 2000nm, z: 50nm]
    = note: Temperature rise ΔT = 47°C exceeds profile limit 20°C
    |
    = note: Thermal calculation breakdown:
    |   • Trace geometry: L = 5.0μm, W = 200nm, t = 50nm
    |   • Material: Polysilicon (ρ = 4.0e-6 Ω·m)
    |   • Trace resistance: R = ρ×L/(W×t) = 4.0e-6 × 5e-6 / (200e-9 × 50e-9)
    |                       R = 2000Ω
    |   • RMS current: I_RMS = 100μA (declared in net)
    |   • Power dissipation: P = I²R = (100e-6)² × 2000 = 20μW
    |   • Thermal conductivity: k = 30 W/(m·K) (poor conductor)
    |   • Temperature rise: ΔT = P/(k×A) ≈ 47°C
    |
  --> materials.hw:67:5
   |
67  |     resistivity: 4.0e-6
    |     ^^^^^^^^^^^^^^^^^^^ Polysilicon: high resistivity
    |
68  |     thermal_conductivity: 30.0
    |     ^^^^^^^^^^^^^^^^^^^^^^^^^^ poor thermal conductor
    |
  --> parasitic_pdk.hw:28:9
   |
28  |         max_temp_rise: 20C
    |         ^^^^^^^^^^^^^^^^^^ strict thermal limit
    |
    = help: To fix this thermal violation, choose one of:
    |
    |   1. Increase trace width to reduce resistance (RECOMMENDED):
    |      width: 800nm  // Reduces R to 500Ω, P to 5μW, ΔT to 12°C
    |
    |   2. Reduce RMS current:
    |      current: 50uA  // Reduces P to 5μW, ΔT to 12°C
    |
    |   3. Use material with lower resistivity:
    |      • Aluminum (ρ = 2.82e-8 Ω·m) - 142× lower resistance
    |      • Copper (ρ = 1.68e-8 Ω·m) - 238× lower resistance
    |
    |   4. Use material with higher thermal conductivity:
    |      • Aluminum (k = 237 W/m·K) - 7.9× better heat dissipation
    |      • Copper (k = 400 W/m·K) - 13.3× better heat dissipation
    |
    |   5. Relax profile thermal budget (if thermally safe):
    |      max_temp_rise: 50C  // In profile thermal section
    |
    = note: Excessive heating can cause:
    |       • Dopant diffusion (changes electrical properties)
    |       • Thermal stress and warping
    |       • Accelerated electromigration
    |       • Device parameter drift
    |
    = note: See https://docs.hw-script.org/errors/P22 for thermal design guidelines
```

---

## Section 4: Error P45 - Forbidden Junction


### 4.1 Detection Logic

**Trigger Condition:**
```rust
// Check for direct overlap of incompatible conductive materials
for (pour_a, pour_b) in overlapping_pours {
    if pour_a.material.is_conductive() && pour_b.material.is_conductive() {
        let bridge_exists = profile.bridges.iter().any(|b| {
            (b.from == pour_a.material && b.to == pour_b.material) ||
            (b.from == pour_b.material && b.to == pour_a.material)
        });
        
        if !bridge_exists {
            // VIOLATION: P45 - Forbidden junction
        }
    }
}
```

**Detection Points:**
1. Layer-to-layer transitions (pour on pour)
2. Via contact to incompatible conductor
3. Direct material contact without interface

### 4.2 Error Structure Implementation

```rust
#[derive(Error, Diagnostic, Debug)]
#[error("Forbidden junction - incompatible materials in direct contact")]
#[diagnostic(
    code(P45),
    url("https://docs.hw-script.org/errors/P45"),
    severity(Error),
)]
pub struct ForbiddenJunction {
    #[label("{material_top} placement")]
    pub top_pour_span: SourceSpan,
    
    #[label("directly overlaps {material_bottom}")]
    pub overlap_span: SourceSpan,
    
    #[related]
    pub related_spans: Vec<RelatedSpan>,
    
    pub material_top: CompactString,
    pub material_bottom: CompactString,
    pub location: Point3D,
    pub net_name: CompactString,
    pub profile_name: CompactString,
    
    #[help]
    pub suggestions: Vec<String>,
    
    #[note]
    pub notes: Vec<String>,
}
```

### 4.3 Example Output

```
error[P45]: Forbidden junction - incompatible materials in direct contact
  --> test_forbidden_junction.hw:89:5
   |
89  |     add pour(Copper) named Top_Contact on layer: metal1:
    |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ Copper placement
90  |         net: In
91  |         dimensions: 800nm by 800nm
92  |         align: center with Bottom_Node
    |              ^^^^^^^^^^^^^^^^^^^^^^^^^ directly overlaps Polysilicon
    |
  --> test_forbidden_junction.hw:76:5
   |
76  |     add pour(Polysilicon) named Bottom_Node on layer: polyres:
    |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ Polysilicon defined here
77  |         net: In
78  |         dimensions: 1.0um by 1.0um
    |
    = note: Detected overlapping region at [x: 2000nm, y: 2500nm]
    = note: Overlap area: 800nm × 800nm = 0.64 μm²
    |
  --> Forbidden_Junction_Profile stackup
   |
    = note: Profile 'Forbidden_Junction_Profile' defines these layers:
    |   • polyres: Polysilicon, 100nm thick
    |   • d1: Silicon_Dioxide, 200nm thick (dielectric)
    |   • metal1: Copper, 200nm thick
    |
    = note: No bridge rule exists for Polysilicon → Copper transition
    |
    = note: Direct contact between Copper and Polysilicon without proper interface
    |       material can cause:
    |       • High contact resistance (poor electrical connection)
    |       • Schottky barrier formation (voltage drop)
    |       • Diffusion and intermixing (reliability issues)
    |       • Adhesion failure (mechanical instability)
    |
    = help: Add required bridge definition to your profile:
    |
    | bridge Polysilicon to Copper:
    |     interface: Titanium_Silicide
    |     thickness: 5nm
    |     contact_resistance: 1e-8 ohm_m2
    |     fill: Tungsten
    |
    = note: Common interface materials for Polysilicon-Copper contacts:
    |   • Titanium_Silicide (TiSi2) - Industry standard for ASIC
    |   • Cobalt_Silicide (CoSi2) - Low resistance alternative
    |   • Tungsten (W) - Via plug material
    |
    = note: See reference profile 'ASIC_Standard' at parasitic_pdk.hw:45
    |
    = note: See https://docs.hw-script.org/errors/P45 for material compatibility
```

---

## Section 5: Error P23 - Crosstalk Violation

### 5.1 Detection Logic

**Trigger Condition:**
```rust
// Calculate coupling capacitance using Sakurai's formula
let coupling_cap = calculate_coupling_capacitance(
    trace_a, trace_b, spacing, dielectric_er, geometry
);

let crosstalk_db = 20.0 * (coupling_cap / ground_cap).log10();

if crosstalk_db > profile.max_crosstalk_db {
    // VIOLATION: P23
}
```

### 5.2 Error Structure & Example Output

```
error[P23]: Crosstalk violation - coupling capacitance exceeds budget
  --> test_crosstalk_violation.hw:112:5
   |
112 |     route ClockA to DestA:
    |     ^^^^^^^^^^^^^^^^^^^^^ aggressor net
113 |         layer: metal1
114 |         width: 300nm
    |
  --> test_crosstalk_violation.hw:118:5
   |
118 |     route ClockB to DestB:
    |     ^^^^^^^^^^^^^^^^^^^^^ victim net (parallel run: 8.0μm)
119 |         layer: metal1
120 |         width: 300nm
    |
    = note: Detected excessive coupling at [x: 4000nm, y: 2500nm]
    = note: Crosstalk: -18.2 dB exceeds limit -25.0 dB
    |
    = note: Coupling analysis:
    |   • Parallel length: 8.0μm
    |   • Edge-to-edge spacing: 100nm (very tight)
    |   • Dielectric: High-K (εr = 25.0) - strong coupling
    |   • Trace width: 300nm each
    |   • Coupling capacitance: C12 = 12.5 fF
    |   • Ground capacitance: C0 = 45.0 fF
    |   • Crosstalk ratio: C12/C0 = 0.278 = -11.1 dB
    |   • Voltage crosstalk: ΔV ≈ 278 mV (for 1V swing on aggressor)
    |
  --> materials.hw:89:5
   |
89  |     relative_permittivity: 25.0
    |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^ High-K dielectric increases coupling
    |
  --> crosstalk_profile.hw:34:9
   |
34  |         max_crosstalk_db: -25.0
    |         ^^^^^^^^^^^^^^^^^^^^^^^ signal integrity budget
    |
    = help: To fix this crosstalk violation, choose one of:
    |
    |   1. Increase spacing between traces (RECOMMENDED):
    |      min_spacing: 500nm  // Reduces C12 to 3.8 fF, crosstalk to -30.7 dB
    |
    |   2. Reduce parallel run length:
    |      • Add vias to route on different layers
    |      • Stagger trace paths to minimize overlap
    |
    |   3. Use lower-K dielectric material:
    |      • SiO2 (εr = 3.9) reduces coupling 6.4×
    |
    |   4. Add grounded guard traces between signals:
    |      add pour(Copper) named Guard on layer: metal1:
    |          net: GND
    |          between: [ClockA, ClockB]
    |
    |   5. Relax crosstalk budget if acceptable for design:
    |      max_crosstalk_db: -20.0  // In profile routing section
    |
    = note: High crosstalk can cause:
    |       • Signal integrity issues (jitter, noise)
    |       • Timing violations in high-speed designs
    |       • False triggering on victim nets
    |       • EMI/EMC compliance failures
    |
    = note: See https://docs.hw-script.org/errors/P23 for SI design guidelines
```

---

## Section 6: Error P24 - Voltage Drop (IR Drop) Violation

### 6.1 Detection Logic

```rust
let voltage_drop = current_dc * resistance;

if voltage_drop > supply_voltage * ir_drop_tolerance {
    // VIOLATION: P24
}
```

### 6.2 Example Output

```
error[P24]: Voltage drop violation - resistive drop exceeds supply tolerance
  --> power_grid.hw:67:5
   |
67  |     route VDD_Source to VDD_Load:
    |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ power trace
68  |         net: VDD
69  |         width: 500nm
    |              ^^^^^^ insufficient width for power delivery
    |
    = note: Detected at trace endpoint [x: 8000nm, y: 4000nm]
    = note: Voltage drop: ΔV = 180mV exceeds tolerance 100mV
    |
    = note: IR drop calculation:
    |   • Trace geometry: L = 10μm, W = 500nm, t = 200nm
    |   • Material: Copper (ρ = 1.68e-8 Ω·m)
    |   • Trace resistance: R = ρ×L/(W×t) = 1.68Ω
    |   • DC current: I_DC = 100mA (load requirement)
    |   • Voltage drop: ΔV = I×R = 100mA × 1.68Ω = 168mV
    |   • Supply voltage: VDD = 1.8V
    |   • Drop percentage: 168mV / 1800mV = 9.3%
    |   • Tolerance: 5% (100mV for 1.8V supply)
    |
    = help: To fix this IR drop violation:
    |
    |   1. Increase power trace width (RECOMMENDED):
    |      width: 2000nm  // Reduces R to 0.42Ω, ΔV to 42mV (2.3%)
    |
    |   2. Use multiple parallel traces for power delivery
    |
    |   3. Use thicker metal layer for power routing
    |
    |   4. Add power grid with low-resistance mesh
    |
    = note: Excessive IR drop causes:
    |       • Reduced noise margins
    |       • Timing violations
    |       • Device performance degradation
    |       • Increased ground bounce
```

---

## Section 7: Implementation Database Structure

### 7.1 Parasitic Error Registry

To enable rich error reporting, maintain a registry during DRC passes:

```rust
pub struct ParasiticErrorRegistry {
    /// Map from geometric location to error context
    violations: HashMap<ViolationKey, ParasiticViolation>,
    
    /// Material property cache
    material_cache: HashMap<CompactString, MaterialProperties>,
    
    /// Profile constraint cache
    profile_cache: HashMap<CompactString, ProfileLimits>,
    
    /// Span tracking for multi-file errors
    span_tracker: SpanTracker,
}

#[derive(Hash, Eq, PartialEq)]
pub struct ViolationKey {
    pub error_code: ParasiticErrorCode,
    pub net: CompactString,
    pub location: QuantizedPoint3D,  // Quantized to 10nm grid
}

pub struct ParasiticViolation {
    pub error_type: ParasiticErrorCode,
    pub context: ParasiticErrorContext,
    pub severity: Severity,
    pub fix_suggestions: Vec<FixSuggestion>,
}

pub enum ParasiticErrorCode {
    P21_Electromigration,
    P22_ThermalRise,
    P23_Crosstalk,
    P24_VoltageDrop,
    P25_Inductance,
    P45_ForbiddenJunction,
    P46_MissingBridge,
    P47_BridgeMismatch,
}

pub struct FixSuggestion {
    pub priority: u8,  // 1 = highest
    pub description: String,
    pub code_example: Option<String>,
    pub expected_improvement: Option<String>,
}
```

### 7.2 Span Tracking System

Track source locations during compilation to enable multi-span errors:

```rust
pub struct SpanTracker {
    /// Material definition locations
    material_defs: HashMap<CompactString, MaterialDefSpans>,
    
    /// Net declaration locations
    net_defs: HashMap<CompactString, NetDefSpans>,
    
    /// Route statement locations
    route_defs: HashMap<RouteId, RouteDefSpans>,
    
    /// Profile/layer locations
    profile_defs: HashMap<CompactString, ProfileDefSpans>,
}

pub struct MaterialDefSpans {
    pub name_span: SourceSpan,
    pub resistivity_span: Option<SourceSpan>,
    pub thermal_k_span: Option<SourceSpan>,
    pub max_current_density_span: Option<SourceSpan>,
    pub relative_permittivity_span: Option<SourceSpan>,
    pub file: PathBuf,
}

pub struct RouteDefSpans {
    pub route_span: SourceSpan,
    pub width_span: SourceSpan,
    pub net_span: SourceSpan,
    pub layer_span: SourceSpan,
    pub file: PathBuf,
}
```

### 7.3 Error Collection During DRC

Integrate into existing DRC pass:

```rust
impl DrcChecker {
    pub fn check_parasitics(
        &mut self,
        registry: &mut ParasiticErrorRegistry,
    ) -> Result<(), Vec<ParasiticViolation>> {
        let mut violations = Vec::new();
        
        // Check electromigration
        for segment in &self.trace_segments {
            if let Some(violation) = self.check_electromigration(segment, registry) {
                violations.push(violation);
            }
        }
        
        // Check thermal rise
        for segment in &self.trace_segments {
            if let Some(violation) = self.check_thermal_rise(segment, registry) {
                violations.push(violation);
            }
        }
        
        // Check forbidden junctions
        for (pour_a, pour_b) in self.find_overlapping_pours() {
            if let Some(violation) = self.check_junction(pour_a, pour_b, registry) {
                violations.push(violation);
            }
        }
        
        // Check crosstalk (if enabled)
        if self.profile.routing.max_crosstalk_db.is_some() {
            for (trace_a, trace_b) in self.find_parallel_traces() {
                if let Some(violation) = self.check_crosstalk(trace_a, trace_b, registry) {
                    violations.push(violation);
                }
            }
        }
        
        if violations.is_empty() {
            Ok(())
        } else {
            Err(violations)
        }
    }
    
    fn check_electromigration(
        &self,
        segment: &TraceSegment,
        registry: &ParasiticErrorRegistry,
    ) -> Option<ParasiticViolation> {
        let material = registry.material_cache.get(&segment.material)?;
        let j_max = material.max_current_density?;
        
        let cross_section = segment.width * segment.thickness;  // nm²
        let j_actual = segment.current / (cross_section * 1e-12);  // A/mm²
        
        if j_actual > j_max {
            Some(ParasiticViolation::electromigration(
                segment, material, j_actual, j_max, registry
            ))
        } else {
            None
        }
    }
}
```

---

## Section 8: User-Facing Error Output Format

### 8.1 Terminal Output (Miette Format)

Use `miette` crate for beautiful terminal output:

```rust
use miette::{Diagnostic, SourceSpan};

// Example rendering
let report = miette::Report::new(ElectromigrationViolation { /* ... */ });
eprintln!("{:?}", report);
```

### 8.2 JSON Output for IDE Integration

For VS Code/LSP integration:

```json
{
  "diagnostics": [
    {
      "code": "P21",
      "severity": "error",
      "message": "Electromigration violation - current density exceeds material limit",
      "source": "hwc-drc",
      "range": {
        "start": { "line": 122, "character": 4 },
        "end": { "line": 125, "character": 25 }
      },
      "relatedInformation": [
        {
          "location": {
            "uri": "file:///path/to/materials.hw",
            "range": { "start": { "line": 44, "character": 4 }, "end": { "line": 44, "character": 28 } }
          },
          "message": "Polysilicon limit: 0.1 mA/μm²"
        }
      ],
      "data": {
        "calculated_j": 5000.0,
        "limit_j": 0.1,
        "suggestions": [
          "Increase trace width to 1000nm",
          "Reduce current to 10μA",
          "Use Copper instead (20× higher limit)"
        ]
      }
    }
  ]
}
```

---

## Section 9: Configuration & Customization

### 9.1 Error Verbosity Levels

Allow users to control detail level:

```toml
# .hwconfig.toml
[errors.parasitic]
verbosity = "detailed"  # Options: minimal, standard, detailed, debug

[errors.parasitic.P21]
show_calculation = true
show_material_properties = true
show_fix_suggestions = true
max_suggestions = 3

[errors.parasitic.P22]
thermal_model = "IPC2152"  # Options: simple, IPC2152, FEA
```

### 9.2 Custom Fix Suggestion Templates

```toml
[errors.parasitic.P21.suggestions]
templates = [
  "Increase width to {calculated_width}nm for {safety_factor}× margin",
  "Reduce current to {safe_current}μA",
  "Switch to {better_material} ({improvement}× higher J_max)"
]
```

---

## Section 10: Testing & Validation

### 10.1 Test Coverage Matrix

| Error | Detection | Reporting | Suggestions | Multi-span | Test File |
|-------|-----------|-----------|-------------|------------|-----------|
| P21 | ✅ | ⏳ | ⏳ | ⏳ | test_em_violation.hw |
| P22 | ✅ | ⏳ | ⏳ | ⏳ | test_thermal_violation.hw |
| P23 | ⏳ | ⏳ | ⏳ | ⏳ | test_crosstalk_violation.hw |
| P24 | ⏳ | ⏳ | ⏳ | ⏳ | test_ir_drop.hw |
| P45 | ⏳ | ⏳ | ⏳ | ⏳ | test_forbidden_junction.hw |

### 10.2 Golden Output Tests

Create golden output files for each error type:

```bash
tests/parasitic-errors/
├── golden/
│   ├── P21_electromigration.txt
│   ├── P22_thermal_rise.txt
│   └── P45_forbidden_junction.txt
└── test_error_format.rs
```

---

## Section 11: Implementation Roadmap

### Phase 1: Foundation (Weeks 1-2)
- [ ] Implement `ParasiticErrorContext` struct
- [ ] Build `ParasiticErrorRegistry` system
- [ ] Create `SpanTracker` for multi-file locations
- [ ] Wire into existing DRC pass

### Phase 2: P21 Electromigration (Week 3)
- [ ] Implement `ElectromigrationViolation` struct
- [ ] Add calculation breakdown notes
- [ ] Generate fix suggestions (3 options)
- [ ] Multi-span causality (material def + route + net)
- [ ] Test with existing test_em_violation.hw

### Phase 3: P22 Thermal Rise (Week 4)
- [ ] Implement `ThermalRiseViolation` struct
- [ ] Thermal calculation with material properties
- [ ] Resistance calculation display
- [ ] Multi-span causality chain
- [ ] Test with test_thermal_violation.hw

### Phase 4: P45 Forbidden Junction (Week 5)
- [ ] Implement `ForbiddenJunction` struct
- [ ] Bridge rule suggestion generation
- [ ] Material compatibility matrix
- [ ] Test with test_forbidden_junction.hw

### Phase 5: P23 Crosstalk (Week 6)
- [ ] Implement coupling capacitance calculation
- [ ] `CrosstalkViolation` struct
- [ ] Parallel run detection
- [ ] Guard trace suggestion

### Phase 6: Polish & Integration (Week 7-8)
- [ ] JSON output for IDE integration
- [ ] Configuration system
- [ ] Performance optimization
- [ ] Documentation & examples

---

## Appendix A: Material Property Requirements

For accurate parasitic error reporting, all materials must declare:

**Conductors:**
- `resistivity` (Ω·m) - Required for P21, P22, P24
- `thermal_conductivity` (W/m·K) - Required for P22
- `max_current_density` (mA/μm²) - Required for P21

**Dielectrics:**
- `relative_permittivity` (εr) - Required for P23

**Example:**
```hardware
export material Copper:
    category: conductor
    process: electroplated
    properties:
        resistivity: 1.68e-8
        thermal_conductivity: 400.0
        max_current_density: 2.0
```

---

## Appendix B: Profile Requirements

For comprehensive parasitic checking:

```hardware
profile MyProfile:
    thermal:
        max_temp_rise: 50C
        ambient_temp: 25C
        
    routing:
        max_crosstalk_db: -25.0
        ir_drop_tolerance: 0.05  # 5%
        
    reliability:
        em_safety_factor: 10     # 10× margin
        thermal_safety_factor: 2 # 2× margin
```

---

## Summary

This parasitic error intelligence system transforms generic DRC violations into **actionable, educational error messages** that guide users to correct solutions. By showing:

1. **What went wrong** (violation description)
2. **Why it went wrong** (calculation breakdown with material properties)
3. **Where it was defined** (multi-span causality across files)
4. **How to fix it** (prioritized suggestions with expected results)
5. **Why it matters** (physical consequences)

Users can quickly understand and resolve parasitic violations without deep expertise in semiconductor physics or thermal analysis.

**Target Result:** From generic `Current density violation` to comprehensive, Clippy-level guidance that makes users feel supported rather than blocked.
