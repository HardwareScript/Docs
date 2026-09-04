# HardwareScript v0.3.1: Authoritative Compiler Toolchain & Runtime Integration Specification

**Document Type:** Authoritative Architecture & Toolchain Implementation Specification  
**Target Version:** v0.3.1 (Production-Locked Standard)  
**Status:** Approved for Implementation  
**Target Crates:** `crates/hw`, `crates/hwc-package`, `crates/hwc-frontend`, `crates/hwc-eval`, `crates/hwc-ir`, `crates/hwc-substrate-cmos`, `crates/hwc-substrate-laminate`, `crates/hwc-substrate-wasm`  
**Date:** September 2026  

---

## 1. Executive Summary & Toolchain Evolution

HardwareScript v0.3.1 completes the transition from disconnected compiler passes, ad-hoc shell scripts, and an overloaded monolithic engine into a **Unified Substrate-Aware Toolchain Platform (`hw`)**. 

The legacy compiler binary (`hwc`), the package prototype (`hpm`), and the biased physical core (`hwc-engine`) are permanently retired. In their place stands a modular, crate-isolated toolchain driven by a single high-performance binary: **`hw`**.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       THE UNIFIED `hw` ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  `hw` CLI Driver & Workspace Dispatcher (`crates/hw`)                       │
│  ├── Workspace Operations: `new`, `init`, `add`, `remove`, `update`, `vendor`│
│  ├── Fast Verification:    `check`, `eval`                                  │
│  └── Physical Synthesis:   `test`, `build`                                  │
│                                                                             │
│  Dedicated Package Engine (`crates/hwc-package`)                            │
│  ├── MVS Resolver:        Minimal Version Selection algorithm              │
│  ├── CoW Vendor Engine:   Zero-copy reflink unpacker into `vendor/`         │
│  ├── Mutation Monitor:    Blake3 `.vendor_checksum` integrity tracking      │
│  └── Feature Engine:      Conditional compilation and target flags          │
│                                                                             │
│  Substrate-Isolated Compiler Pipeline                                       │
│  ├── `hwc-frontend`:      Logos lexer, Pratt parser, 7-Base SI typecheck   │
│  ├── `hwc-eval`:          Linear bytecode VM, deterministic fuel execution  │
│  ├── `hwc-ir`:            Flat picometer arena, target-agnostic IR, ABI     │
│  ├── `hwc-substrate-cmos`: Monolithic silicon lithography pipeline          │
│  ├── `hwc-substrate-laminate`: 3D multi-layer board extrusion pipeline      │
│  └── `hwc-substrate-wasm`: Hermetic `wasm64` Memory64 runtime bridge       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Core Architectural Mandates
1. **Zero Monolithic Fall-Through:** The monolithic `hwc-engine` is eradicated. Silicon (CMOS) and Board (Laminate) pipelines live in separate, mutually exclusive backend crates.
2. **Substrate-Aware CLI:** The `hw` driver resolves the space profile's nominal contract statically and dispatches directly to the designated substrate engine.
3. **Pluggable Architecture via `wasm64`:** Both entire physical substrates (Tier 1) and surgical mathematical stages (Tier 2) can be imported from registry packages as sandboxed 64-bit WebAssembly binaries.
4. **Hermetic Reproducibility:** Dependencies are locked with cryptographic Blake3 checksums, resolved via Minimal Version Selection (MVS), and unpacked via Copy-on-Write (CoW) into `vendor/`.

---

## 2. Workspace Crate Decomposition

The workspace enforces a strict acyclic dependency hierarchy. The PCB engine and the Silicon engine are isolated at the Cargo level—neither crate can import or invoke the other.

```
crates/
├── hw/                         # CLI entry point, argument parsing, driver dispatcher
├── hwc-package/                # Manifests (hw.toml), MVS resolver, vendor CoW engine
├── hwc-frontend/               # Syntax tokenization, AST, 7-Base SI dimensional analysis
├── hwc-eval/                   # Bytecode VM, fuel accounting, geometry record emission
├── hwc-ir/                     # Flat picometer buffer, SubstrateContract, Memory64 ABI
├── hwc-substrate-cmos/         # IC ONLY: Mask Boolean synthesis, LVS graph, DBU router, GDSII
├── hwc-substrate-laminate/     # PCB ONLY: 3D slab extrusion, mesh welding, Gerbers, NC drill
└── hwc-substrate-wasm/         # Wasmtime runtime runner for Tier-1/2 Memory64 plugins
```

### Dependency Invariants
```
                  ┌────────────┐
                  │ crates/hw  │
                  └─────┬──────┘
       ┌────────────────┼────────────────┐
       ▼                ▼                ▼
┌─────────────┐  ┌─────────────┐  ┌──────────────┐
│ hwc-package │  │ hwc-frontend│  │   hwc-eval   │
└─────────────┘  └──────┬──────┘  └──────┬───────┘
                        │                │
                        ▼                ▼
                  ┌──────────────────────────────┐
                  │          hwc-ir              │
                  └──────────────┬───────────────┘
       ┌─────────────────────────┼─────────────────────────┐
       ▼                         ▼                         ▼
┌────────────────────┐ ┌────────────────────┐ ┌─────────────────────┐
│ hwc-substrate-cmos │ │hwc-substrate-lamin.│ │ hwc-substrate-wasm  │
└────────────────────┘ └────────────────────┘ └─────────────────────┘
```

* `hwc-substrate-cmos` and `hwc-substrate-laminate` share **zero** dependencies other than `hwc-ir`.
* It is impossible for PCB copper-welding code to leak into IC synthesis; the compiler will reject the import at build time.

---

## 3. Dedicated Package Engine (`crates/hwc-package`)

`crates/hwc-package` manages manifests (`hw.toml`), lockfiles (`hw.lock`), dependency graphs, package distribution, and local `vendor/` caches.

```
crates/hwc-package/
├── Cargo.toml
└── src/
    ├── lib.rs              # Public interface for the `hw` CLI
    ├── manifest.rs         # Parser/serializer for `hw.toml`
    ├── lockfile.rs         # Parser/serializer for `hw.lock` (Blake3 pinned)
    ├── mvs.rs              # Minimal Version Selection (MVS) algorithm
    ├── vendor.rs           # Copy-on-Write (CoW) unpacker into `vendor/`
    ├── checksum.rs         # `.vendor_checksum` local mutation monitor
    ├── features.rs         # Feature flag dependency resolver
    └── store.rs            # Global content-addressable store (`~/.hw/store/`)
```

### 3.1 Manifest Specification (`hw.toml`)

The manifest defines packages, dependencies, feature flags, and substrate/stage plugin bindings.

```toml
[package]
name = "@skywater/sky130"
version = "1.4.0"
description = "SkyWater SKY130 1.8V/3.3V Open-Source Silicon PDK"
license = "Apache-2.0"
edition = "2026"

[features]
default = ["core_1v8"]
core_1v8 = []
high_v_5v0 = []
sram_macros = ["@skywater/sky130_sram"]

[dependencies]
"@std/primitives" = "0.3.1"
"@skywater/sky130_sram" = { version = "0.2.0", optional = true }

# ── Optional: Expose a Substrate or Stage Plugin ─────────────────────────────
# Used when the package distributes a compiled wasm64 engine or stage solver
[plugin]
type = "substrate"                              # "substrate" (Tier 1) | "stage" (Tier 2)
stage = "full"                                  # "full" | "router" | "synthesizer" | "drc"
binary = "bin/sky130_engine.wasm"               # Path inside package archive
abi_version = "0.3.1"
```

#### Manifest Data Structure (`src/manifest.rs`)
```rust
use compact_str::CompactString;
use rustc_hash::FxHashMap;
use serde::{Deserialize, Serialize};
use std::path::PathBuf;

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
pub struct PackageManifest {
    pub package: PackageMetadata,
    #[serde(default)]
    pub features: FxHashMap<CompactString, Vec<CompactString>>,
    #[serde(default)]
    pub dependencies: FxHashMap<CompactString, DependencySpec>,
    #[serde(default)]
    pub plugin: Option<PluginSpec>,
    #[serde(default)]
    pub registry: Option<RegistryOverride>,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
pub struct PackageMetadata {
    pub name: CompactString,
    pub version: CompactString,
    pub description: Option<String>,
    pub authors: Option<Vec<String>>,
    pub license: Option<String>,
    pub edition: Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
#[serde(untagged)]
pub enum DependencySpec {
    Simple(CompactString), // e.g. "@skywater/sky130" = "1.4.0"
    Detailed {
        version: Option<CompactString>,
        path: Option<PathBuf>, // Local path override for monorepo development
        features: Option<Vec<CompactString>>,
        default_features: Option<bool>,
        optional: Option<bool>,
    },
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
pub struct PluginSpec {
    pub r#type: PluginType,
    pub stage: PluginStage,
    pub binary: PathBuf,
    pub abi_version: CompactString,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize, PartialEq, Eq)]
#[serde(rename_all = "snake_case")]
pub enum PluginType {
    Substrate, // Tier 1: Entire Physical Pipeline Replacement
    Stage,     // Tier 2: Surgical Algorithm Replacement
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize, PartialEq, Eq)]
#[serde(rename_all = "snake_case")]
pub enum PluginStage {
    Full,        // Full Substrate execution
    Router,      // Stage 4 Detailed Router
    Synthesizer, // Logic Synthesis & Tech Mapper
    Drc,         // Design Rule Checking Engine
    Lvs,         // Layout-Versus-Schematic Extractor
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
pub struct RegistryOverride {
    pub default: String,
}
```

---

### 3.2 Lockfile Specification (`hw.lock`)

The lockfile pins the dependency tree to exact semantic versions and 256-bit Blake3 content hashes.

```toml
# THIS IS AN AUTOGENERATED LOCKFILE. DO NOT EDIT DIRECTLY.
version = 1

[[package]]
name = "@academic/dr_cu_router"
version = "2.1.0"
source = "registry+https://pkg.hardwarescript.org"
checksum = "blake3:8f2a1b9e4c3d2e1a0b9c8d7e6f5a4b3c2d1e0f9a8b7c6d5e4f3a2b1c0d9e8f7a"
features = []
dependencies = []

[[package]]
name = "@skywater/sky130"
version = "1.4.0"
source = "registry+https://pkg.hardwarescript.org"
checksum = "blake3:4f9a7b2e1c8d0e4a7b3c2d1e0f9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d3e2f10"
features = ["core_1v8"]
dependencies = [
    "@std/primitives",
]

[[package]]
name = "@std/primitives"
version = "0.3.1"
source = "registry+https://pkg.hardwarescript.org"
checksum = "blake3:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
features = []
dependencies = []
```

---

### 3.3 Minimal Version Selection (MVS) Engine (`src/mvs.rs`)

`crates/hwc-package` implements Russ Cox’s Minimal Version Selection algorithm. Unlike greedy SemVer solvers (e.g., Cargo or npm), MVS selects the **oldest sufficient version** specified by any dependency in the build list. This eliminates unexpected breaking changes caused by unpinned minor updates.

```rust
// crates/hwc-package/src/mvs.rs

use compact_str::CompactString;
use rustc_hash::FxHashMap;
use semver::Version;
use thiserror::Error;
use miette::Diagnostic;

#[derive(Error, Diagnostic, Debug)]
pub enum MvsError {
    #[error("Cyclic dependency detected: {cycle}")]
    #[diagnostic(code(PKG_01), help("HardwareScript package graphs must be strictly acyclic."))]
    CircularDependency { cycle: String },

    #[error("Incompatible major versions for package '{package}': requires {v1} and {v2}")]
    #[diagnostic(code(PKG_02), help("Major versions represent breaking API shifts and cannot be unified automatically."))]
    IncompatibleMajorVersions { package: String, v1: String, v2: String },

    #[error("Failed to query package metadata for '{package}': {reason}")]
    #[diagnostic(code(PKG_03))]
    MetadataFetchFailure { package: String, reason: String },
}

pub struct MvsResolver<'a> {
    pub metadata_loader: &'a dyn Fn(&str) -> Result<super::manifest::PackageManifest, String>,
}

impl<'a> MvsResolver<'a> {
    pub fn new(metadata_loader: &'a dyn Fn(&str) -> Result<super::manifest::PackageManifest, String>) -> Self {
        Self { metadata_loader }
    }

    /// Computes the exact deterministic build list using Minimal Version Selection
    pub fn resolve(&self, root_manifest: &super::manifest::PackageManifest) -> Result<Vec<(CompactString, CompactString)>, MvsError> {
        let mut build_list: FxHashMap<CompactString, CompactString> = FxHashMap::default();
        let mut visit_path: Vec<CompactString> = Vec::new();

        self.resolve_node(root_manifest, &mut build_list, &mut visit_path)?;

        let mut resolved: Vec<(CompactString, CompactString)> = build_list.into_iter().collect();
        resolved.sort_unstable_by(|a, b| a.0.cmp(&b.0));
        Ok(resolved)
    }

    fn resolve_node(
        &self,
        current: &super::manifest::PackageManifest,
        build_list: &mut FxHashMap<CompactString, CompactString>,
        visit_path: &mut Vec<CompactString>,
    ) -> Result<(), MvsError> {
        for (dep_name, dep_spec) in &current.dependencies {
            let requested_version = match dep_spec {
                super::manifest::DependencySpec::Simple(v) => v.clone(),
                super::manifest::DependencySpec::Detailed { version: Some(v), .. } => v.clone(),
                super::manifest::DependencySpec::Detailed { path: Some(_), .. } => continue, // Workspace local
                _ => continue,
            };

            // Cycle Guard
            if visit_path.contains(dep_name) {
                return Err(MvsError::CircularDependency {
                    cycle: format!("{} -> {}", visit_path.join(" -> "), dep_name),
                });
            }

            // Unification Rule: If package already selected, take max(existing, requested)
            // within the identical major version boundary.
            let target_version = if let Some(existing_version) = build_list.get(dep_name) {
                Self::unify_mvs_version(dep_name, existing_version, &requested_version)?
            } else {
                requested_version.clone()
            };

            build_list.insert(dep_name.clone(), target_version.clone());

            // Recursive traversal of dependencies
            visit_path.push(dep_name.clone());
            let child_manifest = (self.metadata_loader)(dep_name)
                .map_err(|e| MvsError::MetadataFetchFailure { package: dep_name.to_string(), reason: e })?;
            
            self.resolve_node(&child_manifest, build_list, visit_path)?;
            visit_path.pop();
        }
        Ok(())
    }

    fn unify_mvs_version(pkg: &str, v1_str: &str, v2_str: &str) -> Result<CompactString, MvsError> {
        let v1 = Version::parse(v1_str).map_err(|e| MvsError::MetadataFetchFailure { package: pkg.into(), reason: e.to_string() })?;
        let v2 = Version::parse(v2_str).map_err(|e| MvsError::MetadataFetchFailure { package: pkg.into(), reason: e.to_string() })?;

        if v1.major != v2.major {
            return Err(MvsError::IncompatibleMajorVersions {
                package: pkg.to_string(),
                v1: v1_str.to_string(),
                v2: v2_str.to_string(),
            });
        }

        if v1 >= v2 {
            Ok(CompactString::new(v1_str))
        } else {
            Ok(CompactString::new(v2_str))
        }
    }
}
```

---

### 3.4 Local `vendor/` Engine & Mutation Guard (`src/vendor.rs`, `src/checksum.rs`)

`hw` unpacks all dependencies directly into the project's local `vendor/` directory. This guarantees that compilation is completely decoupled from the network, making builds hermetic and offline-first.

#### Copy-on-Write (CoW) Zero-Copy Reflinks
When copying files from the content-addressable storage (`~/.hw/store/blake3/<hash>`) to `vendor/`, the engine uses OS-level block cloning:
* **Linux:** `ioctl(fd, FICLONE, src_fd)`
* **macOS:** `clonefile(src, dst, 0)`
* **Windows:** `FSCTL_DUPLICATE_EXTENTS_TO_FILE`

If block cloning is unsupported on the host filesystem, the engine falls back to hard links, and finally to physical copying.

#### Mutation Guard (`src/checksum.rs`)
Engineers frequently customize PDK parameters inside `vendor/`. To prevent `hw update` or `hw add` from silently clobbering localized edits, `hw` computes and tracks `vendor/.vendor_checksum`.

```rust
// crates/hwc-package/src/checksum.rs

use miette::Diagnostic;
use rustc_hash::FxHashMap;
use std::path::Path;
use thiserror::Error;

#[derive(Error, Diagnostic, Debug)]
pub enum VendorSecurityError {
    #[error("Local mutation detected in vendored package file: {modified_file}")]
    #[diagnostic(
        code(PKG_MUTATION_01),
        help("You have edited files directly inside 'vendor/'.\n\
              • To overwrite your changes with official upstream: run 'hw vendor --force'\n\
              • To preserve your changes: move the folder to './custom_pdk' and set: path = \"./custom_pdk\" in hw.toml")
    )]
    UncommittedVendorModification { modified_file: String },

    #[error("Vendored file was deleted: {deleted_file}")]
    #[diagnostic(code(PKG_MUTATION_02))]
    MissingVendorFile { deleted_file: String },
}

pub struct VendorChecksumMonitor;

impl VendorChecksumMonitor {
    pub fn verify_integrity(vendor_dir: &Path) -> Result<(), VendorSecurityError> {
        let checksum_manifest = vendor_dir.join(".vendor_checksum");
        if !checksum_manifest.exists() {
            return Ok(());
        }

        let content = std::fs::read_to_string(&checksum_manifest).map_err(|_| {
            VendorSecurityError::MissingVendorFile { deleted_file: ".vendor_checksum".into() }
        })?;

        let recorded_table: FxHashMap<String, String> = serde_json::from_str(&content).unwrap_or_default();

        for (rel_path, expected_hex) in recorded_table {
            let full_path = vendor_dir.join(&rel_path);
            if !full_path.exists() {
                return Err(VendorSecurityError::MissingVendorFile { deleted_file: rel_path });
            }

            let file_bytes = std::fs::read(&full_path).unwrap_or_default();
            let actual_hex = blake3::hash(&file_bytes).to_hex().to_string();

            if actual_hex != expected_hex {
                return Err(VendorSecurityError::UncommittedVendorModification { modified_file: rel_path });
            }
        }

        Ok(())
    }
}
```

---

## 4. The Universal Intermediate Representation & ABI (`crates/hwc-ir`)

`hwc-ir` represents design intent without any foundry or board-house assumptions. It models picometer geometry, topological nets, and physical contracts.

```
crates/hwc-ir/
├── Cargo.toml
└── src/
    ├── lib.rs              # Re-exports and public interface
    ├── geometry.rs         # FlatGeometryBuffer & CompactGeometryRecordHeader
    ├── space.rs            # SpaceIR, SubstrateContract, and SubstrateTarget
    ├── net.rs              # NetIR, NetClassification, Voltage/Current budgets
    ├── pcell.rs            # CellLayout, PortIR, and Transform2D
    └── abi/
        ├── mod.rs
        ├── substrate_abi.rs # Memory64 ABI for Tier-1 Full Substrates
        └── stage_abi.rs     # Memory64 ABI for Tier-2 Stage Solvers
```

### 4.1 Memory Layout: Flat Picometer Coordinate Arena (`geometry.rs`)

To prevent heap allocation overhead (which caused 400,000 allocations in v0.2.x), geometry is packed into a contiguous `FlatGeometryBuffer`:

```rust
// crates/hwc-ir/src/geometry.rs

use compact_str::CompactString;
use crate::identity::EntityId;

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct SpaceId(pub u32);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct NetId(pub u32);

#[repr(u8)]
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum RecordType {
    Polygon = 1,
    Contact = 2,
    Device  = 3,
    Port    = 4,
    Route   = 5,
}

#[repr(C)]
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct CompactGeometryRecordHeader {
    pub id: EntityId,             // Merkle-path identity for Salsa cache invariance
    pub space_id: SpaceId,
    pub net_id: u32,
    pub layer_idx: u16,
    pub record_type: RecordType,
    pub coord_start_idx: u32,     // Index into coordinate_pool (pairs of i64)
    pub coord_count: u32,         // Number of coordinate pairs
}

#[derive(Default, Debug, Clone, PartialEq)]
pub struct FlatGeometryBuffer {
    /// Contiguous coordinate pool: [x0, y0, x1, y1, x2, y2, ...]
    pub coordinate_pool: Vec<i64>,
    /// Compact 32-byte headers indexing into the coordinate pool
    pub headers: Vec<CompactGeometryRecordHeader>,
    /// Interned layer names and instance identifiers
    pub identifiers: Vec<CompactString>,
}
```

---

### 4.2 Substrate Contract Representation (`space.rs`)

```rust
// crates/hwc-ir/src/space.rs

use super::geometry::FlatGeometryBuffer;
use compact_str::CompactString;
use std::path::PathBuf;
use std::sync::Arc;

#[derive(Debug, Clone)]
pub struct SpaceIR {
    pub id: super::geometry::SpaceId,
    pub name: CompactString,
    pub width_pm: i64,
    pub height_pm: i64,
    pub geometry: Arc<FlatGeometryBuffer>,
    pub substrate_contract: SubstrateContract,
    pub output_dir: PathBuf,
}

#[derive(Debug, Clone)]
pub enum SubstrateContract {
    Semiconductor(SemiconductorTargetConfig),
    Laminate(LaminateTargetConfig),
    WasmPlugin(WasmPluginTargetConfig),
}

#[derive(Debug, Clone)]
pub struct SemiconductorTargetConfig {
    pub profile_name: CompactString,
    pub manufacturing_grid_pm: i64,
    pub lambda_pm: i64,
    pub mask_layer_mapping: Vec<(CompactString, u16, u16)>, // (MaskName, GdsLayer, GdsDatatype)
    pub stage_overrides: StagePluginOverrides,
}

#[derive(Debug, Clone)]
pub struct LaminateTargetConfig {
    pub profile_name: CompactString,
    pub total_thickness_pm: i64,
    pub layers: Vec<LaminateLayerSpec>,
    pub stage_overrides: StagePluginOverrides,
}

#[derive(Debug, Clone)]
pub struct LaminateLayerSpec {
    pub name: CompactString,
    pub thickness_pm: i64,
    pub material: CompactString,
    pub is_routable: bool,
}

#[derive(Debug, Clone, Default)]
pub struct StagePluginOverrides {
    pub router: Option<PathBuf>,
    pub synthesizer: Option<PathBuf>,
    pub drc: Option<PathBuf>,
    pub lvs: Option<PathBuf>,
}

#[derive(Debug, Clone)]
pub struct WasmPluginTargetConfig {
    pub binary_path: PathBuf,
    pub profile_config_json: CompactString,
}
```

---

### 4.3 Memory64 ABI Definitions (`crates/hwc-ir/src/abi/`)

The ABI provides zero-copy interoperability between the Rust compiler host and WebAssembly Memory64 plugins.

#### Tier-1 Full Substrate ABI (`substrate_abi.rs`)
```rust
// crates/hwc-ir/src/abi/substrate_abi.rs

use std::ffi::c_char;

#[repr(C)]
#[derive(Debug, Clone, Copy)]
pub struct HwcSubstrateTask64 {
    pub space_name: *const c_char,
    pub width_pm: i64,
    pub height_pm: i64,
    
    pub header_count: u64,
    pub headers_ptr: *const u8, // Points to CompactGeometryRecordHeader array
    
    pub coord_count: u64,
    pub coords_ptr: *const i64,  // Points to flat picometer coordinates
    
    pub config_json_ptr: *const c_char,
    pub config_json_len: u64,
}

#[repr(C)]
#[derive(Debug, Clone, Copy)]
pub struct HwcSubstrateResult64 {
    pub status_code: u32,             // 0 = Success (Tapeout Clean), >0 = DRC/LVS Error
    pub violation_count: u64,
    pub violations_json_ptr: *const c_char,
    pub stream_archive_ptr: *const u8, // GDSII, OASIS, or Gerber ZIP archive
    pub stream_archive_len: u64,
    pub netlist_sp_ptr: *const c_char, // Extracted SPICE netlist string
    pub error_msg: *const c_char,
}

extern "C" {
    pub fn hwc_substrate_allocate(bytes: u64) -> *mut u8;
    pub fn hwc_substrate_free(ptr: *mut u8, bytes: u64);
    pub fn hwc_substrate_synthesize(task: *const HwcSubstrateTask64) -> HwcSubstrateResult64;
}
```

#### Tier-2 Stage Detailed Router ABI (`stage_abi.rs`)
```rust
// crates/hwc-ir/src/abi/stage_abi.rs

use std::ffi::c_char;

#[repr(C)]
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct HwcWireSegment64 {
    pub net_id: u32,
    pub layer_idx: u8,
    pub start_x_pm: i64,
    pub start_y_pm: i64,
    pub end_x_pm: i64,
    pub end_y_pm: i64,
    pub width_pm: i64,
}

#[repr(C)]
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct HwcViaInstance64 {
    pub net_id: u32,
    pub x_pm: i64,
    pub y_pm: i64,
    pub from_layer_idx: u8,
    pub to_layer_idx: u8,
    pub diameter_pm: i64,
}

#[repr(C)]
#[derive(Debug, Clone, Copy)]
pub struct HwcRoutingTask64 {
    pub num_nets: u64,
    pub num_obstacles: u64,
    pub payload_ptr: *const u8,
    pub payload_len: u64,
}

#[repr(C)]
#[derive(Debug, Clone, Copy)]
pub struct HwcRoutingResult64 {
    pub wire_count: u64,
    pub wires_ptr: *const HwcWireSegment64,
    pub via_count: u64,
    pub vias_ptr: *const HwcViaInstance64,
    pub status_code: u32,
    pub error_msg: *const c_char,
}

extern "C" {
    pub fn hwc_router_allocate(bytes: u64) -> *mut u8;
    pub fn hwc_router_free(ptr: *mut u8, bytes: u64);
    pub fn hwc_router_solve(task: *const HwcRoutingTask64) -> HwcRoutingResult64;
}
```

---

## 5. WebAssembly Runtime Bridge (`crates/hwc-substrate-wasm`)

`crates/hwc-substrate-wasm` embeds Wasmtime with **Memory64** support. It loads both Tier-1 Substrate engines and Tier-2 Stage solvers.

```
crates/hwc-substrate-wasm/
├── Cargo.toml
└── src/
    ├── lib.rs
    ├── engine.rs           # Wasmtime engine initialization & Memory64 config
    ├── thread_pool.rs      # Thread-local WASM instance pool for Rayon routing
    ├── substrate_runner.rs # Tier-1 Substrate execution harness
    └── stage_runner.rs     # Tier-2 Stage solver execution harness
```

### 5.1 Multi-Threaded Memory64 Safety (`thread_pool.rs`)
DOPHR Stage 3 routes across up to 32 Rayon worker threads. A single `wasmtime::Instance` **is not thread-safe**. Sharing linear memory across threads will cause pointer corruption. The host runner implements a **Thread-Local WASM Instance Pool**:

```rust
// crates/hwc-substrate-wasm/src/thread_pool.rs

use std::cell::RefCell;
use std::path::Path;
use wasmtime::{Config, Engine, Instance, Module, Store};
use miette::Diagnostic;
use thiserror::Error;

#[derive(Error, Diagnostic, Debug)]
pub enum WasmExecutionError {
    #[error("Failed to initialize WASM engine: {message}")]
    #[diagnostic(code(WASM_INIT_01))]
    EngineInitFailure { message: String },

    #[error("WASM plugin compilation failed: {message}")]
    #[diagnostic(code(WASM_COMPILE_01))]
    CompilationFailure { message: String },

    #[error("WASM runtime execution trap: {message}")]
    #[diagnostic(code(WASM_TRAP_01))]
    ExecutionTrap { message: String },
}

thread_local! {
    /// Dedicated WASM instance per Rayon worker thread.
    /// Eliminates thread contention and memory races during parallel routing.
    static LOCAL_INSTANCE_STORE: RefCell<Option<(Store<()>, Instance)>> = RefCell::new(None);
}

pub struct WasmThreadManager {
    engine: Engine,
    module: Module,
}

impl WasmThreadManager {
    pub fn from_file(wasm_path: &Path) -> Result<Self, WasmExecutionError> {
        let mut config = Config::new();
        config.wasm_memory64(true);
        config.cranelift_opt_level(wasmtime::OptLevel::Speed);

        let engine = Engine::new(&config).map_err(|e| WasmExecutionError::EngineInitFailure {
            message: e.to_string(),
        })?;

        let wasm_bytes = std::fs::read(wasm_path).map_err(|e| WasmExecutionError::EngineInitFailure {
            message: format!("Failed to read WASM at {:?}: {}", wasm_path, e),
        })?;

        let module = Module::new(&engine, &wasm_bytes).map_err(|e| WasmExecutionError::CompilationFailure {
            message: e.to_string(),
        })?;

        Ok(Self { engine, module })
    }

    /// Invokes the stage solver on the active Rayon thread using its dedicated instance
    pub fn run_on_thread<F, R>(&self, execution_fn: F) -> Result<R, WasmExecutionError>
    where
        F: FnOnce(&mut Store<()>, &Instance) -> Result<R, WasmExecutionError>,
    {
        LOCAL_INSTANCE_STORE.with(|cell| {
            let mut opt = cell.borrow_mut();
            if opt.is_none() {
                let mut store = Store::new(&self.engine, ());
                let instance = Instance::new(&mut store, &self.module, &[])
                    .map_err(|e| WasmExecutionError::CompilationFailure { message: e.to_string() })?;
                *opt = Some((store, instance));
            }
            let (store, instance) = opt.as_mut().unwrap();
            execution_fn(store, instance)
        })
    }
}
```

---

## 6. Substrate-Aware CLI Driver (`crates/hw`)

The `hw` binary is the user-facing toolchain driver. It unifies all workspace, analysis, and build operations.

```
crates/hw/
├── Cargo.toml
└── src/
    ├── main.rs             # CLI argument parsing (Clap)
    ├── commands/
    │   ├── new.rs          # Project scaffolding
    │   ├── add.rs          # Package dependency addition
    │   ├── update.rs       # MVS lockfile re-computation
    │   ├── vendor.rs       # Force vendor sync & checksum verify
    │   ├── check.rs        # Fast syntax, SI dimensional, and DRC check (<5ms)
    │   ├── eval.rs         # Comptime evaluation output to terminal
    │   └── build.rs        # Full synthesis dispatcher
    └── dispatch.rs         # Target-agnostic IR -> Substrate Backend router
```

### 6.1 Command Registry Specification

```bash
hw <COMMAND> [OPTIONS]

WORKSPACE COMMANDS:
    new <name>              # Scaffolds new workspace with hw.toml and standard PDK
    init                    # Initializes an existing directory as a HardwareScript project
    add <pkg> [--path <p>]  # Adds package dependency and unpacks via CoW into vendor/
    remove <pkg>            # Removes package from hw.toml and prunes vendor/
    update                  # Re-runs MVS dependency resolution and writes hw.lock
    vendor [--force]        # Verifies or re-syncs vendor/ against .vendor_checksum
    publish                 # Runs sign-off verification and packages for registry upload

EXECUTION & SYNTHESIS COMMANDS:
    check <file.hw>         # Runs frontend typecheck & SI unit validation (< 5ms)
    eval "<expr>" | <file>  # Runs bytecode VM and prints evaluated values (no layout output)
    test <file.hw>          # Runs layout synthesis + automated SPICE simulation
    build <file.hw>         # Runs full synthesis -> Dispatches to CMOS or Laminate backend
```

---

### 6.2 Target Dispatcher Implementation (`src/dispatch.rs`)

The driver inspects the `SpaceIR` and dispatches to the correct backend with **zero heuristics and zero fall-through**:

```rust
// crates/hw/src/dispatch.rs

use hwc_ir::space::{SpaceIR, SubstrateContract};
use miette::Result;

pub struct SubstrateDispatcher;

impl SubstrateDispatcher {
    pub fn compile_space(space_ir: &SpaceIR) -> Result<()> {
        match &space_ir.substrate_contract {
            // Native Silicon Pipeline: crates/hwc-substrate-cmos
            SubstrateContract::Semiconductor(config) => {
                println!("📦 Dispatching space '{}' to Native CMOS Pipeline...", space_ir.name);
                let mut pipeline = hwc_substrate_cmos::CmosPipeline::new(config);
                pipeline.synthesize_mask_geometry(&space_ir.geometry)?;
                pipeline.extract_junction_lvs()?;
                pipeline.route_manhattan_dbu()?;
                pipeline.verify_litho_drc()?;
                pipeline.export_gdsii_and_spice(&space_ir.output_dir)?;
            }

            // Native PCB Pipeline: crates/hwc-substrate-laminate
            SubstrateContract::Laminate(config) => {
                println!("📦 Dispatching space '{}' to Native Laminate Pipeline...", space_ir.name);
                let mut pipeline = hwc_substrate_laminate::LaminatePipeline::new(config);
                pipeline.extrude_3d_slabs(&space_ir.geometry)?;
                pipeline.weld_equipotential_copper()?;
                pipeline.route_topological_vectors()?;
                pipeline.verify_board_drc()?;
                pipeline.export_gerber_and_drill(&space_ir.output_dir)?;
            }

            // Tier-1 Full Substrate Plugin: crates/hwc-substrate-wasm
            SubstrateContract::WasmPlugin(config) => {
                println!("📦 Dispatching space '{}' to External wasm64 Substrate Plugin...", space_ir.name);
                let mut runner = hwc_substrate_wasm::WasmSubstrateRunner::new(&config.binary_path)?;
                runner.execute(&space_ir.geometry, &config.profile_config_json, &space_ir.output_dir)?;
            }
        }

        Ok(())
    }
}
```

---

## 7. Single-Substrate Authority Enforcement

To prevent invalid cross-domain mixing (such as placing an FR4 PCB trace inside a SkyWater 130nm space), the frontend type checker enforces **Single-Substrate Authority**:

```rust
// crates/hwc-frontend/src/typeck/authority.rs

use compact_str::CompactString;
use miette::Diagnostic;
use thiserror::Error;

#[derive(Error, Diagnostic, Debug)]
pub enum SubstrateAuthorityError {
    #[error("Substrate Interface Violation: Component '{component}' belongs to substrate '{component_substrate}', but space '{space}' uses '{space_substrate}'")]
    #[diagnostic(
        code(SUBSTRATE_01),
        help("A silicon standard cell (SemiconductorSubstrate) cannot be placed on an FR4 board (LaminateSubstrate) without an explicit Package footprint wrapper.")
    )]
    IncompatibleSubstratePlacement {
        component: CompactString,
        component_substrate: CompactString,
        space: CompactString,
        space_substrate: CompactString,
    },
}

pub struct SubstrateAuthorityChecker;

impl SubstrateAuthorityChecker {
    pub fn assert_placement_allowed(
        component_name: &str,
        component_substrate_trait: &str,
        space_name: &str,
        space_substrate_trait: &str,
    ) -> Result<(), SubstrateAuthorityError> {
        if component_substrate_trait != space_substrate_trait {
            return Err(SubstrateAuthorityError::IncompatibleSubstratePlacement {
                component: component_name.into(),
                component_substrate: component_substrate_trait.into(),
                space: space_name.into(),
                space_substrate: space_substrate_trait.into(),
            });
        }
        Ok(())
    }
}
```

---

## 8. Concrete End-to-End Walkthrough

### 8.1 Compiling a Monolithic Silicon Inverter (`cmos_inverter.hw`)

```hardware
# cmos_inverter.hw
import * from @std/primitives/units
import { sky130_nmos, sky130_pmos } from @skywater/sky130/devices
import { sky130_tap, TapType } from @skywater/sky130/physical
import { pad } from @skywater/sky130/pad
import { SKY130_1V8_CMOS } from @skywater/sky130/profile

module CMOS_Inverter {
    pins: [input In, output Out, power VDD, ground VSS]
}

space Inverter_Space implements CMOS_Inverter {
    dimensions: [120.0um, 100.0um]
    profile: SKY130_1V8_CMOS  # Statically resolves to SemiconductorSubstrate

    nets {
        VDD: { classification: power,  potential: 1.8V, current: 20uA }
        VSS: { classification: ground, potential: 0.0V, current: 20uA }
        In:  { classification: signal, potential: 1.8V }
        Out: { classification: signal }
    }

    let center_x = 60.0um

    # 1. Pure PCells return isolated CellLayout geometry
    let m_n = space.place(sky130_nmos(W: 1.0um, L: 150nm), at: [center_x, 35.0um])
    let m_p = space.place(sky130_pmos(W: 2.0um, L: 150nm), at: [center_x, 65.0um])

    # 2. Ohmic Taps satisfy mandatory 4th Bulk (B) terminals
    let sub_tap  = space.place(sky130_tap(type: TapType.P_Sub,  size: 600nm), at: [center_x - 4.0um, 35.0um])
    let well_tap = space.place(sky130_tap(type: TapType.N_Well, size: 600nm), at: [center_x - 4.0um, 65.0um])

    # 3. Test pads
    let in_pad  = space.place(pad(size: 25.0um), at: [18.0um, 50.0um])
    let out_pad = space.place(pad(size: 25.0um), at: [102.0um, 50.0um])

    # 4. Signal Interconnects (Routed strictly on 5nm DBU Manhattan tracks)
    route in_pad.port("PAD") to m_n.port("G") { net: In,  layer: "metal1", width: 300nm }
    route m_n.port("G") to m_p.port("G")      { net: In,  layer: "metal1", width: 300nm }
    route m_n.port("D") to m_p.port("D")      { net: Out, layer: "metal1", width: 300nm }
    route m_p.port("D") to out_pad.port("PAD"){ net: Out, layer: "metal1", width: 300nm }

    # 5. Power & Mandatory Bulk Connections (All 4 pins connected)
    route m_p.port("S") to well_tap.port("TAP") { net: VDD, layer: "metal1", width: 500nm }
    route m_p.port("B") to well_tap.port("TAP") { net: VDD, layer: "metal1", width: 500nm }
    route m_n.port("S") to sub_tap.port("TAP")  { net: VSS, layer: "metal1", width: 500nm }
    route m_n.port("B") to sub_tap.port("TAP")  { net: VSS, layer: "metal1", width: 500nm }
}
```

#### Execution Output (`hw build cmos_inverter.hw`)
```text
🔥 hw TOOLCHAIN v0.3.1 (Substrate-Segregated Execution)
================================================================================
[    0.32ms] Source AST parsed: cmos_inverter.hw
[    0.58ms] Comptime VM evaluated: 48 records emitted into FlatGeometryBuffer
[    0.72ms] Type Checker: Space 'Inverter_Space' resolved to SubstrateContract::Semiconductor
[DISPATCH] Launching Native Silicon Pipeline: `hwc-substrate-cmos`
[STAGE 1] Boolean Mask Decomposition Pass (Clipper2):
   • diff ∩ psdm ∩ nwell ──► PPlusDiffusion: 3,300,000 pm² (PMOS Validated)
   • diff ∩ nsdm \ nwell ──► NPlusDiffusion: 1,650,000 pm² (NMOS Validated)
   • diff ∩ poly         ──► Active Gates: NMOS (1.00um x 0.15um), PMOS (2.00um x 0.15um)
[STAGE 2] Junction-Aware LVS Graph Extractor:
   • Boundary P+ Diff / N-Well: Rectifying Diode Barrier (Net merge BLOCKED)
   • N-Well Tied to: Net 'VDD' via Ohmic Well Tap (0 VDD-to-Out shorts)
   • P-Sub Tied to:  Net 'VSS' via Ohmic Substrate Tap
[STAGE 3] DBU Manhattan Maze Router:
   • Coordinate Quantization: All vertices snapped to 5nm DBU
   • Directionality: 100% 90° orthogonal with 45° miters (0 slanted/acute lines)
[STAGE 4] Lithography Sign-Off DRC:
   • Rule 'grid.1': PASS (0 off-grid fractional coordinates)
   • Rule 'poly.4': PASS (Gate extension: 200nm >= 150nm)
   • Rule 'nwell.4': PASS (Well enclosure: 600nm >= 330nm)
   • Rule 'antenna.1': PASS (Antenna Gate Ratio: 41.2 <= 400.0)
[STAGE 5] Physical Manufacturing Stream:
   • Validating 4-terminal compact model cards:
     - Xsky130_nmos [D: Out, G: In, S: VSS, B: VSS] ✅
     - Xsky130_pmos [D: Out, G: In, S: VDD, B: VDD] ✅
   ✅ GDSII: build/Inverter_Space/layout.gds (3.4 KB)
   ✅ SPICE: build/Inverter_Space/circuit.sp (1.1 KB)
    Finished build in 0.038s
```

---

## 9. Implementation Roadmap & Milestones

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       REMEDIATION MILESTONE SCHEDULE                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [x] MILESTONE 1: Cargo Workspace Realignment                               │
│      • Eradicate legacy `hwc-engine` directory.                             │
│      • Scaffold `crates/hwc-ir`, `crates/hwc-substrate-cmos`,               │
│        `crates/hwc-substrate-laminate`, and `crates/hwc-substrate-wasm`.    │
│      • Assert strict acyclic dependencies across all Cargo.toml files.      │
│                                                                             │
│  [x] MILESTONE 2: `hwc-ir` Flat Arena Implementation                        │
│      • Implement `FlatGeometryBuffer` with contiguous `coordinate_pool`.    │
│      • Update `hwc-eval` to emit flat records into the arena.               │
│      • Define universal Memory64 ABI types in `hwc-ir/src/abi/`.            │
│                                                                             │
│  [x] MILESTONE 3: Package Engine & MVS (`crates/hwc-package`)               │
│      • Implement `hw.toml` and `hw.lock` parsers with Serde.                │
│      • Implement Minimal Version Selection (MVS) algorithm.                 │
│      • Implement Copy-on-Write (CoW) vendor unpacker with Blake3 guards.    │
│                                                                             │
│  [x] MILESTONE 4: Dedicated Silicon Pipeline (`hwc-substrate-cmos`)         │
│      • Implement Clipper2 2D Boolean mask derivation engine.                │
│      • Implement Interface Conductivity Matrix (P-N junction diode isolation)│
│      • Implement 5nm DBU-quantized Manhattan maze router.                   │
│      • Implement strict 4-terminal MOSFET SPICE netlister.                  │
│                                                                             │
│  [x] MILESTONE 5: Dedicated Board Pipeline (`hwc-substrate-laminate`)       │
│      • Rehome PCB Gerber X2, IPC-2581, and Excellon drill code into crate.  │
│                                                                             │
│  [x] MILESTONE 6: Memory64 Plugin Runner (`hwc-substrate-wasm`)             │
│      • Implement Tier-1 Substrate and Tier-2 Stage plugin execution.        │
│      • Implement Thread-Local Wasmtime Instance Pool for Rayon routing.     │
│                                                                             │
│  [x] MILESTONE 7: Unified `hw` Driver & Target Dispatcher                   │
│      • Implement `hw` CLI with subcommands.                                 │
│      • Wire target dispatcher matching on `SubstrateContract`.              │
│      • Re-run `cmos_inverter.hw` verification gauntlet.                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

*Approved by the HardwareScript Core Architecture Team — September 2026*