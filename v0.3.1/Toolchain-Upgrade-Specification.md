# Compiler & Toolchain Upgrade Specification

```
Document Type: Authoritative Compiler Toolchain & Runtime Integration Specification
Target Version: v0.3.1 (Production-Locked Standard)
Status: Approved for Implementation
Target Crates: crates/hwc-cli, crates/hwc-package, crates/hwc-compiler, crates/hwc-engine
Date: September 2026
```

---

## 1. Executive Summary & Toolchain Evolution

HardwareScript v0.3.1 completes the transition from disconnected compiler passes and external package scripts into a **Unified Compiler & Hardware Package Platform (`hw`)**. 

The standalone compiler executable (`hwc`) and the experimental package script (`hpm`) are merged into a single native binary: **`hw`**.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       THE UNIFIED `hw` ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  `hw` Unified CLI Toolchain Driver (crates/hwc-cli)                         │
│  ├── Package Management: `new`, `add`, `remove`, `update`, `publish`        │
│  ├── Comptime Compute:   `check`, `eval`                                    │
│  └── Physical Synthesis: `test`, `build`                                    │
│                                                                             │
│  crates/hwc-package (New Dedicated Workspace Crate)                        │
│  ├── MVS Resolver:       Minimal Version Selection dependency engine        │
│  ├── Vendor Engine:      `vendor/` unpacker with Copy-on-Write (CoW)        │
│  ├── Integrity Guard:    Blake3 cryptographic `.vendor_checksum` tracking   │
│  └── Feature Engine:     Cargo-style feature flag evaluator                 │
│                                                                             │
│  crates/hwc-compiler & crates/hwc-engine Integration                        │
│  ├── Salsa Queries:      Pure file caching over `vendor/` module trees      │
│  ├── Symbol Table:       Namespace isolation & single-PDK authority check   │
│  └── WASM Runner:        Hermetic `wasm64` plugin loader for routers/solvers│
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. CLI Architecture & Driver Unification (`crates/hwc-cli`)

The CLI interface adopts modern toolchain ergonomics (modeled after Go, Cargo, and Zig).

### 2.1 Complete Command Registry

```
hw <COMMAND> [OPTIONS]

COMMANDS:
    # ── Workspace & Dependency Commands ──
    new <name>              Scaffold a new HardwareScript workspace with hw.toml
    init                    Initialize a HardwareScript project in current directory
    add <pkg>               Add a package dependency to hw.toml and unpack to vendor/
    remove <pkg>            Remove a package dependency from hw.toml and vendor/
    update                  Re-evaluate MVS dependency tree and update hw.lock
    vendor                  Force re-sync or verify the local vendor/ directory
    publish                 Validate physical DRC, package archive, and publish

    # ── Synthesis, Execution & Verification Commands ──
    check <file.hw>         Run syntax, dimensional unit, and DRC checks (<5ms)
    eval "<expr>" | <file>  Execute compile-time bytecode VM without layout export
    test <file.hw>          Synthesize layout and execute automated SPICE simulation
    build <file.hw>         Run full synthesis (VM -> Router -> DRC -> GDSII/SPICE)
```

---

## 3. Dedicated Package Engine (`crates/hwc-package`)

A new crate, `crates/hwc-package`, is added to the compiler workspace to manage manifests, lockfiles, resolution, and vendoring.

```
crates/hwc-package/
├── Cargo.toml
└── src/
    ├── lib.rs              # Public API for CLI and compiler pipeline
    ├── manifest.rs         # `hw.toml` parser and serializer
    ├── lockfile.rs         # `hw.lock` Blake3-pinned lockfile manager
    ├── mvs.rs              # Minimal Version Selection (MVS) algorithm
    ├── vendor.rs           # `vendor/` directory unpacker and CoW manager
    ├── checksum.rs         # `.vendor_checksum` mutation monitor
    ├── features.rs         # Feature flag dependency resolver
    └── cache.rs            # Global content-addressable storage (`~/.hw/store/`)
```

### 3.1 Manifest Data Structure (`src/manifest.rs`)

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
        path: Option<PathBuf>, // Local override path
        features: Option<Vec<CompactString>>,
        default_features: Option<bool>,
        optional: Option<bool>,
    },
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
pub struct PluginSpec {
    pub r#type: CompactString, // "router", "synthesizer", "solver"
    pub binary: PathBuf,       // Path to .wasm binary within the package
    pub abi_version: CompactString,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
pub struct RegistryOverride {
    pub default: String,
}
```

---

### 3.2 Lockfile Specification (`src/lockfile.rs`)

The `hw.lock` file ensures bit-identical reproducibility.

```toml
# THIS IS AN AUTOGENERATED LOCKFILE. DO NOT EDIT DIRECTLY.
version = 1

[[package]]
name = "@skywater/sky130"
version = "1.4.0"
source = "registry+https://pkg.hardwarescript.org"
checksum = "blake3:4f9a7b2e1c8d0e4a7b3c2d1e0f9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d3e2f10"
features = ["core_1v8", "digital_hd"]
dependencies = ["@std/primitives"]

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

`crates/hwc-package` implements Russ Cox’s Minimal Version Selection (MVS) algorithm to eliminate greedy-range upgrade bugs.

```rust
use compact_str::CompactString;
use rustc_hash::FxHashMap;
use thiserror::Error;
use miette::Diagnostic;

#[derive(Error, Diagnostic, Debug)]
pub enum MvsError {
    #[error("Cycle detected in package dependency graph: {cycle}")]
    #[diagnostic(code(PKG_01), help("HardwareScript package graphs must be strictly acyclic."))]
    CircularDependency { cycle: String },

    #[error("Version conflict for package '{package}': requested versions {v1} and {v2} are incompatible")]
    #[diagnostic(code(PKG_02))]
    IncompatibleVersions { package: String, v1: String, v2: String },
}

pub struct MvsResolver<'a> {
    pub fetch_metadata: &'a dyn Fn(&str) -> Result<PackageManifest, String>,
}

impl<'a> MvsResolver<'a> {
    /// Resolves the minimum sufficient build list for a root manifest.
    pub fn resolve(&self, root: &PackageManifest) -> Result<Vec<(CompactString, CompactString)>, MvsError> {
        let mut build_list: FxHashMap<CompactString, CompactString> = FxHashMap::default();
        let mut visited = Vec::new();

        self.resolve_recursive(root, &mut build_list, &mut visited)?;

        let mut result: Vec<_> = build_list.into_iter().collect();
        result.sort_unstable_by(|a, b| a.0.cmp(&b.0));
        Ok(result)
    }

    fn resolve_recursive(
        &self,
        current: &PackageManifest,
        build_list: &mut FxHashMap<CompactString, CompactString>,
        visited: &mut Vec<CompactString>,
    ) -> Result<(), MvsError> {
        for (pkg_name, dep_spec) in &current.dependencies {
            let req_version = match dep_spec {
                DependencySpec::Simple(v) => v.clone(),
                DependencySpec::Detailed { version: Some(v), .. } => v.clone(),
                DependencySpec::Detailed { path: Some(_), .. } => continue, // Handled as local workspace path
                _ => continue,
            };

            if visited.contains(pkg_name) {
                return Err(MvsError::CircularDependency {
                    cycle: format!("{} -> {}", visited.join(" -> "), pkg_name),
                });
            }

            // MVS Rule: If package is already in build list, choose max(existing, requested)
            // within the same major version.
            let selected_version = if let Some(existing) = build_list.get(pkg_name) {
                Self::max_compatible_version(existing, &req_version)?
            } else {
                req_version.clone()
            };

            build_list.insert(pkg_name.clone(), selected_version.clone());

            visited.push(pkg_name.clone());
            let child_manifest = (self.fetch_metadata)(pkg_name)
                .map_err(|e| MvsError::IncompatibleVersions { package: pkg_name.to_string(), v1: req_version.to_string(), v2: e })?;
            self.resolve_recursive(&child_manifest, build_list, visited)?;
            visited.pop();
        }
        Ok(())
    }

    fn max_compatible_version(v1: &str, v2: &str) -> Result<CompactString, MvsError> {
        // Compares semantic versions, selecting the higher version within the same major version series
        let sem_v1 = semver::Version::parse(v1).unwrap();
        let sem_v2 = semver::Version::parse(v2).unwrap();

        if sem_v1.major != sem_v2.major {
            return Err(MvsError::IncompatibleVersions {
                package: "Major version jump".into(),
                v1: v1.into(),
                v2: v2.into(),
            });
        }

        if sem_v1 >= sem_v2 {
            Ok(CompactString::new(v1))
        } else {
            Ok(CompactString::new(v2))
        }
    }
}
```

---

## 4. Local `vendor/` Engine & Mutation Guard (`src/vendor.rs`)

### 4.1 Reflink & Zero-Copy Unpacking
When packages are installed, `hw` extracts files from the local content store (`~/.hw/store/blake3/<hash>`) into the project's `vendor/` directory using OS Copy-on-Write (CoW) reflinks:
* **macOS (APFS):** `clonefile()`
* **Linux (Btrfs / XFS):** `ioctl(FICLONE)`
* **Windows (ReFS / NTFS):** Duplicate Extents API / Hardlinks

### 4.2 Local Mutation Safety (`src/checksum.rs`)
To protect developers tinkering with PDK files, `hw` computes and records `vendor/.vendor_checksum`.

```rust
use std::path::Path;
use rustc_hash::FxHashMap;

#[derive(Debug, Clone)]
pub struct VendorChecksumMonitor;

impl VendorChecksumMonitor {
    /// Asserts that vendor/ has not been modified before performing destructive updates.
    pub fn assert_clean_vendor_tree(vendor_dir: &Path) -> Result<(), String> {
        let checksum_file = vendor_dir.join(".vendor_checksum");
        if !checksum_file.exists() {
            return Ok(());
        }

        let recorded_hashes: FxHashMap<String, String> = serde_json::from_str(
            &std::fs::read_to_string(&checksum_file).map_err(|e| e.to_string())?
        ).map_err(|e| e.to_string())?;

        for (rel_path, expected_hash) in recorded_hashes {
            let file_path = vendor_dir.join(&rel_path);
            if !file_path.exists() {
                return Err(format!("File deleted in vendor directory: {}", rel_path));
            }
            let actual_bytes = std::fs::read(&file_path).map_err(|e| e.to_string())?;
            let actual_hash = blake3::hash(&actual_bytes).to_hex().to_string();

            if actual_hash != expected_hash {
                return Err(format!(
                    "Local modification detected in 'vendor/{}'.\n\
                     To overwrite changes, run: hw install --force\n\
                     To preserve changes, move directory to './local_pdk' and set: path = \"./local_pdk\"",
                    rel_path
                ));
            }
        }
        Ok(())
    }
}
```

---

## 5. Compiler & Engine Upgrades (`hwc-compiler` & `hwc-engine`)

### 5.1 Salsa Incremental Query Tree Integration
Salsa database queries in `crates/hwc-compiler/src/ir/query.rs` are updated to track files inside `vendor/`:

```rust
#[salsa::query_group(SourceDatabaseStorage)]
pub trait SourceDatabase: salsa::Database {
    #[salsa::input]
    fn file_content(&self, path: PathBuf) -> Arc<String>;

    #[salsa::dependencies]
    fn resolve_import_path(&self, import_target: CompactString, current_file: PathBuf) -> Result<PathBuf, String>;

    #[salsa::dependencies]
    fn parse_package_manifest(&self, project_root: PathBuf) -> Result<Arc<PackageManifest>, String>;
}
```

**Import Resolution Rule:**
1. If path starts with `@std/` $\to$ resolve from internal compiler stdlib directory.
2. If path starts with `@scope/pkg` $\to$ resolve from `vendor/@scope/pkg/mod.hw`.
3. If path is relative (`./` or `../`) $\to$ resolve relative to current source file.

---

### 5.2 Single-PDK Physical Space Authority Enforcement
In `crates/hwc-compiler/src/eval/context.rs`, the symbol table validates that imported standard cells adhere to the active space profile:

```rust
impl<'a> VM<'a> {
    pub fn validate_pcell_profile_compatibility(
        &self,
        pcell_profile: &str,
        space_profile: &str,
    ) -> Result<(), EvalError> {
        if pcell_profile != space_profile {
            return Err(EvalError::TypeMismatch {
                expected: "Matching Physical Profile",
                found: format!("PCell from '{}' placed in space with profile '{}'", pcell_profile, space_profile),
            });
        }
        Ok(())
    }
}
```

---

### 5.3 WebAssembly (`wasm64`) Plugin Loading in `hwc-router` & `hwc-synthesis`
When a package provides a `plugin.binary` (e.g. `router.wasm`), `hwc-router` loads it through the thread-local WASM instance pool:

```rust
// crates/hwc-router/src/ffi/wasm64_runner.rs

impl Wasm64RouterRunner {
    pub fn load_from_vendor(vendor_dir: &Path, plugin_manifest: &PluginSpec) -> Result<Self, RoutingError> {
        let wasm_path = vendor_dir.join(&plugin_manifest.binary);
        let wasm_bytes = std::fs::read(&wasm_path).map_err(|e| RoutingError::PluginFailure {
            message: format!("Failed to read WASM plugin at {:?}: {}", wasm_path, e),
        })?;

        let mut config = wasmtime::Config::new();
        config.wasm_memory64(true);
        config.cranelift_opt_level(wasmtime::OptLevel::Speed);

        let engine = wasmtime::Engine::new(&config).unwrap();
        let module = wasmtime::Module::new(&engine, &wasm_bytes).map_err(|e| RoutingError::PluginFailure {
            message: format!("WASM64 compilation failed: {}", e),
        })?;

        Ok(Self { engine, module })
    }
}
```
