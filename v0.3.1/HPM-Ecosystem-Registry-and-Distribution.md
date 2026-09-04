# HPM Ecosystem, Registry & Distribution Specification

**Document Type:** Authoritative Package Registry, Distribution & Plugin Ecosystem Specification  
**Target Version:** v0.3.1 (Production-Locked Standard)  
**Status:** Approved for Implementation  
**Platform:** GitHub (`index`, `vault`, `bouncer-ci`) + Cloudflare Pages (Edge CDN)  
**Infrastructure Cost:** $0.00 / month  
**Target Crates / Repos:** `crates/hwc-package`, `crates/hw`, `github.com/hpm-registry/*`  
**Date:** September 2026  

---

## 1. Executive Architecture: The Serverless Edge Ecosystem

The HardwareScript Package Manager (HPM) ecosystem provides decentralized, reproducible package resolution and plugin distribution without persistent database servers, dynamic API containers, or subscription costs.

By coupling **GitHub** (as an immutable, append-only data store) with **Cloudflare Pages** (as a global, sub-10ms static edge CDN), HPM delivers a zero-maintenance registry supporting code packages, verified foundry PDKs, and pre-compiled 64-bit WebAssembly (`wasm64`) substrate/stage engines.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 HPM THREE-TIER ZERO-COST ARCHITECTURE                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. METADATA EDGE CDN (Cloudflare Pages @ pkg.hardwarescript.org)           │
│     • Static JSON package metadata replicated across 300+ edge locations.   │
│     • Query latency: < 10 ms globally. Unlimited bandwidth. Cost: $0.00.    │
│                                                                             │
│  2. METADATA SOURCE OF TRUTH (GitHub: `hpm-registry/index`)                 │
│     • Two-level prefix tree (`s/k/skywater-sky130.json`).                   │
│     • Pull Request queue driven by automated verification bots.             │
│                                                                             │
│  3. IMMUTABLE PERMANENT STORAGE (GitHub: `hpm-registry/vault`)              │
│     • Release assets store Blake3-hashed `.tar.gz` package payloads.        │
│     • Immutable storage ensures 10-year reproducible silicon builds.        │
│                                                                             │
│  4. AUTOMATED TRUST GATE (GitHub Actions: `hpm-registry/bouncer-ci`)        │
│     • Sandboxed container validating physical viability, Substrate Trait     │
│       conformance, and Memory64 WebAssembly ABI safety before merge.        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Package Taxonomies & Contract Classifications

In HardwareScript v0.3.1, packages are not generic code tarballs. Every package declares its **Architectural Taxonomy** in `hw.toml`, establishing its role in the compilation pipeline:

```
                          HPM PACKAGE TAXONOMIES
                          
 ┌─────────────────────────────────────────────────────────────────────────┐
 │ 1. PURE HARDWARE IP LIBRARY (`taxonomy = "library"`)                    │
 │ • Reusable parameterized cells (PCells), controllers, and math helpers. │
 │ • Written in: 100% Pure HardwareScript (`.hw`).                         │
 │ • Examples: `@std/primitives`, `@generic/passives`, `@riscv/cores`.     │
 └─────────────────────────────────────────────────────────────────────────┘
                                     │
 ┌─────────────────────────────────────────────────────────────────────────┐
 │ 2. CERTIFIED FOUNDRY PDK (`taxonomy = "pdk"`)                           │
 │ • Implements nominal `Substrate` traits (`Semiconductor` / `Laminate`). │
 │ • Optical masks, GDS layer maps, SPICE models, DRC decks.               │
 │ • Examples: `@skywater/sky130`, `@tsmc/n16`, `@jlc/pcb_4layer`.         │
 └─────────────────────────────────────────────────────────────────────────┘
                                     │
 ┌─────────────────────────────────────────────────────────────────────────┐
 │ 3. TIER-1 SUBSTRATE PLUGIN (`taxonomy = "substrate_plugin"`)            │
 │ • Replaces the ENTIRE physical manufacturing pipeline via `wasm64`.     │
 │ • Ships a compiled Memory64 binary exporting the Substrate C-ABI.       │
 │ • Examples: `@aim/silicon_photonics`, `@intel/emib_interposer`.         │
 └─────────────────────────────────────────────────────────────────────────┘
                                     │
 ┌─────────────────────────────────────────────────────────────────────────┐
 │ 4. TIER-2 STAGE SOLVER PLUGIN (`taxonomy = "stage_plugin"`)             │
 │ • Surgical algorithm replacement inside a native substrate engine.      │
 │ • Ships a compiled Memory64 binary targeting a single stage ABI.        │
 │ • Examples: `@academic/dr_cu_router`, `@yosys/synthesis_abc`.           │
 └─────────────────────────────────────────────────────────────────────────┘
```

### 2.1 Manifest Taxonomy Declarations (`hw.toml`)

#### A. Foundry PDK Package
```toml
[package]
name = "@skywater/sky130"
version = "1.4.0"
taxonomy = "pdk"
substrate = "SemiconductorSubstrate"
description = "SkyWater SKY130 1.8V/3.3V Silicon Process Design Kit"
license = "Apache-2.0"
edition = "2026"

[features]
default = ["core_1v8"]
core_1v8 = []
high_v_5v0 = []

[dependencies]
"@std/primitives" = "0.3.1"
```

#### B. Tier-1 Substrate Engine Package (WASM64)
```toml
[package]
name = "@aim/silicon_photonics"
version = "0.9.0"
taxonomy = "substrate_plugin"
substrate = "PhotonicsSubstrate"
description = "Curvilinear Optical Waveguide & Resonator Synthesis Engine"
license = "MIT"
edition = "2026"

[plugin]
type = "substrate"
binary = "bin/photonics_engine.wasm"
abi_version = "0.3.1"
memory64_required = true
```

#### C. Tier-2 Stage Solver Package (WASM64 Router)
```toml
[package]
name = "@academic/dr_cu_router"
version = "2.1.0"
taxonomy = "stage_plugin"
stage = "router"
description = "Dr. CU 2.0 Sparse-Grid Detailed Router for Advanced CMOS"
license = "BSD-3-Clause"
edition = "2026"

[plugin]
type = "stage"
stage = "router"
binary = "bin/dr_cu_v2.wasm"
abi_version = "0.3.1"
compatible_substrates = ["SemiconductorSubstrate"]
```

---

## 3. The Edge Registry Data Model (`hpm-registry/index`)

To achieve instant lookups on Cloudflare Pages without full-text database queries, package metadata is organized in a **Two-Level Prefix Directory Tree**:

```
github.com/hpm-registry/index/
├── config.json                     # Registry global settings and CDN endpoints
├── a/
│   └── i/
│       └── aim-silicon_photonics.json
├── s/
│   └── k/
│       └── skywater-sky130.json
└── a/
    └── c/
        └── academic-dr_cu_router.json
```

### 3.1 Metadata Schema (`s/k/skywater-sky130.json`)

```json
{
  "name": "@skywater/sky130",
  "taxonomy": "pdk",
  "substrate": "SemiconductorSubstrate",
  "description": "SkyWater SKY130 1.8V/3.3V Silicon Process Design Kit",
  "license": "Apache-2.0",
  "repository": "https://github.com/google/skywater-pdk",
  "versions": {
    "1.4.0": {
      "tarball_url": "https://github.com/hpm-registry/vault/releases/download/skywater-sky130-1.4.0/package.tar.gz",
      "checksum": "blake3:4f9a7b2e1c8d0e4a7b3c2d1e0f9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d3e2f10",
      "dependencies": {
        "@std/primitives": "0.3.1"
      },
      "features": {
        "default": ["core_1v8"],
        "core_1v8": [],
        "high_v_5v0": []
      },
      "plugin": null,
      "published_at": "2026-09-01T04:00:00Z"
    }
  }
}
```

### 3.2 Metadata Schema with WASM64 Plugin (`a/c/academic-dr_cu_router.json`)

```json
{
  "name": "@academic/dr_cu_router",
  "taxonomy": "stage_plugin",
  "description": "Dr. CU 2.0 Sparse-Grid Detailed Router for Advanced CMOS",
  "license": "BSD-3-Clause",
  "repository": "https://github.com/cuhk-eda/dr-cu",
  "versions": {
    "2.1.0": {
      "tarball_url": "https://github.com/hpm-registry/vault/releases/download/academic-dr_cu_router-2.1.0/package.tar.gz",
      "checksum": "blake3:8f2a1b9e4c3d2e1a0b9c8d7e6f5a4b3c2d1e0f9a8b7c6d5e4f3a2b1c0d9e8f7a",
      "dependencies": {},
      "features": {},
      "plugin": {
        "type": "stage",
        "stage": "router",
        "abi_version": "0.3.1",
        "binary_rel_path": "bin/dr_cu_v2.wasm",
        "memory64": true,
        "compatible_substrates": ["SemiconductorSubstrate"]
      },
      "published_at": "2026-09-02T11:20:00Z"
    }
  }
}
```

---

## 4. Automated Verification Gate: The Bouncer CI Engine

Every package submission is mediated by an automated GitHub Actions bot (`bouncer-ci`). No human intervention is permitted; admission to the registry is governed by strict mathematical, physical, and binary execution gates.

```
                      BOUNCER-CI VERIFICATION GATES
                      
  Submission PR ──► Gate 1: Namespace & Scope Verification
                         │ (Valid GitHub Org Membership)
                         ▼
                    Gate 2: File Structure & Size Whitelist
                         │ (No .exe, .sh, .py; <= 50 MB)
                         ▼
                    Gate 3: Substrate Contract & Physical Viability
                         │ (`hw check --foundry`, clean DRC/LVS)
                         ▼
                    Gate 4: WASM64 Memory64 Binary Audit
                         │ (Symbol exports, Memory64, sandbox trap check)
                         ▼
                    Gate 5: Blake3 Checksum & Permanent Archival
                         │ (Upload tarball to vault, merge PR)
                         ▼
              Cloudflare Edge Deploy (< 2 seconds globally)
```

### 4.1 Detailed Bouncer Gate Checks

#### Gate 1: Namespace & Cryptographic Identity
* Packages with scopes (e.g., `@skywater/*`, `@tsmc/*`) require verification that the PR author is an authorized member of the matching GitHub organization (`github.com/skywater`).
* Unscoped packages are reserved for core standards.

#### Gate 2: Default-Deny File Structure Whitelist
* The unpacked archive must contain only: `.hw`, `.toml`, `.md`, `.txt`, `.wasm`.
* Any archive containing shell scripts (`.sh`, `.bat`), dynamic libraries (`.so`, `.dll`, `.dylib`), or compiled binaries (`ELF`, `PE`, `Mach-O`) is rejected immediately.
* Maximum uncompressed archive size: $50\text{ MB}$.

#### Gate 3: Substrate Contract & Physical Viability
* The bouncer runs `hw check --verify-pdk` inside an isolated container.
* **For PDK packages:** Validates that active profiles declare optical masks, enforce Database Unit (DBU) grid quantization, declare 4-terminal compact models ($D, G, S, B$), and contain zero single-material assignments on lithography layers.
* **For Library packages:** Validates that PCells evaluate with zero dimensional mismatches ($L \times L = L^2$) and adhere to SI base units.

#### Gate 4: WebAssembly Memory64 Binary Audit (`wasm-inspect`)
If the package includes a `plugin.binary` (`.wasm`), the bouncer executes strict binary analysis:
1. **Memory64 Validation:** Verifies that the WebAssembly binary targets 64-bit linear memory (`wasm64`).
2. **Export Symbol Contract:** 
   - Tier-1 Substrates must export: `hwc_substrate_allocate`, `hwc_substrate_free`, `hwc_substrate_synthesize`.
   - Tier-2 Stage Routers must export: `hwc_router_allocate`, `hwc_router_free`, `hwc_router_solve`.
3. **Hermetic Sandbox Execution:** Instantiates the plugin inside Wasmtime with a test task payload. Verifies zero illegal host imports (no filesystem access, no network calls, no system clocks).

---

### 4.2 The Automated Bouncer Workflow (`.github/workflows/verify-publish.yml`)

```yaml
name: HPM Registry Automated Bouncer
on:
  pull_request_target:
    branches: [ main ]
    paths:
      - '**/*.json'

jobs:
  bouncer_verification:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write

    steps:
      - name: Checkout Index Repository
        uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}

      - name: Install HardwareScript Native Toolchain
        uses: hardwarescript/setup-hw@v1
        with:
          version: '0.3.1'

      - name: Run Multi-Gate Bouncer Audit
        id: bouncer
        run: |
          CHANGED_FILE=$(git diff --name-only origin/main HEAD | grep '\.json$' | head -n 1)
          echo "Auditing package metadata: $CHANGED_FILE"
          
          # Extract tarball URL from proposed metadata
          SUBMISSION_URL=$(jq -r '.versions | to_entries | last | .value.tarball_url' "$CHANGED_FILE")
          curl -sL "$SUBMISSION_URL" -o submission.tar.gz

          # 1. Whitelist & Archive Structure Audit
          hw-bouncer audit-archive --file submission.tar.gz --max-size-mb 50

          # 2. Extract into Isolated Sandbox
          mkdir -p ./sandbox
          tar -xzf submission.tar.gz -C ./sandbox

          # 3. Substrate & Physical Viability Gate
          hw-bouncer audit-physical-contract --dir ./sandbox

          # 4. WASM64 Memory64 Binary Audit (if plugin declared)
          if [ -f "./sandbox/hw.toml" ] && grep -q "\[plugin\]" "./sandbox/hw.toml"; then
            hw-bouncer audit-wasm64-abi --dir ./sandbox --abi-version 0.3.1
          fi

          # 5. Compute Canonical Blake3 Hash
          COMPUTED_HASH="blake3:$(b3sum submission.tar.gz | awk '{print $1}')"
          DECLARED_HASH=$(jq -r '.versions | to_entries | last | .value.checksum' "$CHANGED_FILE")

          if [ "$COMPUTED_HASH" != "$DECLARED_HASH" ]; then
            echo "::error::Blake3 Checksum Mismatch! Declared: $DECLARED_HASH, Computed: $COMPUTED_HASH"
            exit 1
          fi

          echo "checksum=$COMPUTED_HASH" >> $GITHUB_OUTPUT
          TAG_NAME=$(basename "$CHANGED_FILE" .json)-$(jq -r '.versions | to_entries | last | .key' "$CHANGED_FILE")
          echo "tag_name=$TAG_NAME" >> $GITHUB_OUTPUT

      - name: Archive to Permanent Vault Release
        uses: softprops/action-gh-release@v1
        with:
          repository: hpm-registry/vault
          tag_name: "${{ steps.bouncer.outputs.tag_name }}"
          files: submission.tar.gz
          token: ${{ secrets.VAULT_RELEASE_TOKEN }}

      - name: Auto-Merge Verified Pull Request
        run: gh pr merge ${{ github.event.pull_request.number }} --auto --squash --delete-branch
        env:
          GITHUB_TOKEN: ${{ secrets.BOT_PAT }}
```

---

## 5. The Complete Publishing Lifecycle (`hw publish`)

Publishing a package into the global registry requires a single command:

```
  [ DEVELOPER MACHINE ]
  1. Executes `hw publish`
     ├── Evaluates `hw check --verify-pdk`
     ├── Packs source and `.wasm` into `package.tar.gz`
     ├── Computes canonical Blake3 hash
     └── Creates public GitHub release on user's fork
           │
           ▼
  2. Opens PR against `hpm-registry/index`
     └── Adds version record to `s/k/skywater-sky130.json`
           │
           ▼
  [ HPM BOUNCER ENGINE ]
  3. Executes 5 Automated Gates (< 45 seconds)
     ├── Verifies namespace ownership
     ├── Asserts strict file allowlist
     ├── Validates Substrate Trait & physical rules
     └── Audits WASM64 Memory64 ABI exports
           │
           ▼
  4. Vault Archival & Auto-Merge
     ├── Copies tarball to permanent `hpm-registry/vault`
     └── Squash-merges PR into `main`
           │
           ▼
  [ CLOUDFLARE EDGE CDN ]
  5. Webhook triggers Pages deployment (< 2 seconds)
     └── Metadata live globally at `pkg.hardwarescript.org`
```

---

## 6. The Installation & Consumption Lifecycle

When an engineer adds a dependency (`hw add @skywater/sky130`) or builds a project (`hw build`), the client toolchain runs entirely offline-first using local content-addressable storage:

```
  [ LOCAL WORKSPACE ]
  `hw add @skywater/sky130`
        │
        ▼ (HTTPS GET < 10ms)
  [ CLOUDFLARE EDGE CDN ]
  Query: `https://pkg.hardwarescript.org/s/k/skywater-sky130.json`
        │
        ▼
  [ MVS RESOLVER (`crates/hwc-package`) ]
  Resolves minimum sufficient build list across dependencies
        │
        ▼
  [ LOCAL CONTENT STORE (`~/.hw/store/blake3/<hash>`) ]
  Checks if tarball already cached:
    • Cache Hit:  Skip network download
    • Cache Miss: Download from `hpm-registry/vault` & verify Blake3
        │
        ▼ (OS Copy-on-Write Reflink)
  [ VENDOR UNPACK (`vendor/@skywater/sky130/`) ]
  Clones files into workspace via `clonefile()` / `FICLONE` (< 5ms)
        │
        ▼
  [ LOCAL MUTATION MONITOR ]
  Writes `vendor/.vendor_checksum` table
        │
        ▼
  [ SALSA INCREMENTAL COMPILER ]
  `hwc-eval` compiles pure `.hw` PCells into `Arc<FlatGeometryBuffer>`
```

---

## 7. Single-Substrate Authority & Cross-Package Compatibility

HPM enforces **Substrate Compatibility Contracts** at resolution time. A project space bound to a `SemiconductorSubstrate` cannot import plugins or physical cells that belong to an incompatible physical domain.

### Compatibility Matrix

| Package Taxonomy / Type | Supported Space Substrate | Incompatible Placement / Binding |
| :--- | :--- | :--- |
| **CMOS Standard Cell PDK** | `SemiconductorSubstrate` | ❌ Cannot place on `LaminateSubstrate` without package footprint |
| **PCB Component Footprint** | `LaminateSubstrate` | ❌ Cannot place on `SemiconductorSubstrate` (violates litho masks) |
| **Tier-2 Router (`dr_cu_router.wasm`)** | `SemiconductorSubstrate` | ❌ Cannot route `LaminateSubstrate` (lacks drilled via support) |
| **Tier-2 PCB Router (`freeroute.wasm`)** | `LaminateSubstrate` | ❌ Cannot route `SemiconductorSubstrate` (violates 5nm DBU grid) |
| **Tier-1 Substrate (`photonics_engine`)**| `PhotonicsSubstrate` | ❌ Cannot bind to CMOS standard cell libraries |

#### Client-Side Resolution Check (`crates/hwc-package/src/authority.rs`)
```rust
// crates/hwc-package/src/authority.rs

use hwc_ir::space::SubstrateContract;
use miette::Diagnostic;
use thiserror::Error;

#[derive(Error, Diagnostic, Debug)]
pub enum SubstrateCompatibilityError {
    #[error("Plugin Compatibility Error: Plugin '{plugin}' requires substrate '{required_substrate}', but space '{space}' uses '{active_substrate}'")]
    #[diagnostic(
        code(PKG_SUBSTRATE_01),
        help("Stage plugins (routers, placers) are specialized for specific physical domains. You cannot use a semiconductor router on a PCB layout.")
    )]
    IncompatibleSubstrateBinding {
        plugin: String,
        required_substrate: String,
        space: String,
        active_substrate: String,
    },
}

pub struct SubstrateCompatibilityValidator;

impl SubstrateCompatibilityValidator {
    pub fn assert_plugin_compatible(
        plugin_name: &str,
        compatible_substrates: &[String],
        space_name: &str,
        active_contract: &SubstrateContract,
    ) -> Result<(), SubstrateCompatibilityError> {
        let active_name = match active_contract {
            SubstrateContract::Semiconductor(_) => "SemiconductorSubstrate",
            SubstrateContract::Laminate(_) => "LaminateSubstrate",
            SubstrateContract::WasmPlugin(_) => "ExternalWasmSubstrate",
        };

        if !compatible_substrates.iter().any(|s| s == active_name) {
            return Err(SubstrateCompatibilityError::IncompatibleSubstrateBinding {
                plugin: plugin_name.to_string(),
                required_substrate: compatible_substrates.join(" | "),
                space: space_name.to_string(),
                active_substrate: active_name.to_string(),
            });
        }

        Ok(())
    }
}
```

---

## 8. Security Boundaries & Disaster Recovery

HPM operates on a **Zero-Trust Security Model** designed to protect physical silicon and hardware systems from supply-chain attacks and operational failure:

| Threat Vector | Attack Scenario | HPM Defense Policy | Enforcement Point |
| :--- | :--- | :--- | :--- |
| **Namespace Squatting** | Malicious user publishes fake `@skywater/sky130` | **GitHub Org Verification:** Author must prove membership in `github.com/skywater`. | Bouncer Gate 1 |
| **WASM Host Escape** | Plugin attempts to read `/etc/passwd` or query local network | **Wasmtime Hermetic Sandbox:** Memory64 runtime compiles with zero host I/O or WASI capabilities. | Host Runtime |
| **Physical Silicon Arson** | Malicious PDK shorts VDD to VSS to overheat chip | **Static DRC/LVS Gate:** `hw check --verify-pdk` extracts netlist and proves zero power-ground shorts. | Bouncer Gate 3 |
| **Release Tampering** | Attacker modifies `.tar.gz` asset on GitHub Releases | **Blake3 Cryptographic Pinning:** Client verifies computed hash against `hw.lock`. Mismatches abort build. | Client Installer |
| **Platform Outage** | Cloudflare Pages or GitHub experiences downtime | **Local `vendor/` & Content Store:** Builds run 100% offline from local disk cache (`~/.hw/store/`). | Client Architecture |
| **Upstream Deletion** | Package author deletes original repository | **Permanent Vault Archival:** Tarballs are stored in `hpm-registry/vault`. Releases are append-only and never deleted. | Registry Policy |

---

## 9. Implementation Deliverables & Rollout Schedule

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      HPM ROLLOUT MILESTONE SCHEDULE                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [x] MILESTONE 1: Package Engine Core (`crates/hwc-package`)                │
│      • Implement `hw.toml` and `hw.lock` parsers with taxonomy metadata.    │
│      • Implement Minimal Version Selection (MVS) algorithm.                 │
│      • Implement OS Copy-on-Write (CoW) unpacker (`vendor/`).               │
│      • Implement `.vendor_checksum` local mutation monitor.                 │
│                                                                             │
│  [x] MILESTONE 2: Substrate Compatibility Engine                            │
│      • Wire Single-Substrate Authority Validator into `hwc-frontend`.       │
│      • Enforce nominal `SubstrateContract` compatibility on plugin loading. │
│                                                                             │
│  [x] MILESTONE 3: Registry Infrastructure Setup                             │
│      • Create GitHub organization: `hpm-registry`.                          │
│      • Initialize `hpm-registry/index` with two-level prefix structure.     │
│      • Initialize `hpm-registry/vault` for immutable release assets.        │
│      • Deploy `index` repository to Cloudflare Pages.                       │
│                                                                             │
│  [x] MILESTONE 4: Automated Bouncer CI (`hpm-registry/bouncer-ci`)         │
│      • Write `verify-publish.yml` GitHub Actions workflow.                  │
│      • Implement `hw-bouncer` CLI tool:                                     │
│        - File allowlist auditor (reject non-HW/WASM files).                 │
│        - Physical DRC/LVS gate (`hw check --verify-pdk`).                   │
│        - WASM64 Memory64 ABI symbol and sandbox verification.              │
│      • Configure automatic squash-merge bot.                                │
│                                                                             │
│  [x] MILESTONE 5: Baseline Package Publications                             │
│      • Publish `@std/primitives` (Taxonomy: Library).                       │
│      • Publish `@skywater/sky130` (Taxonomy: PDK, Substrate: Semiconductor). │
│      • Publish `@jlc/pcb_4layer` (Taxonomy: PDK, Substrate: Laminate).      │
│      • Publish `@academic/dr_cu_router` (Taxonomy: StagePlugin).            │
│                                                                             │
│  [x] MILESTONE 6: End-to-End Verification Gauntlet                          │
│      • Scaffold new project: `hw new inverter_project`.                     │
│      • Add packages: `hw add @skywater/sky130`.                             │
│      • Synthesize layout: `hw build cmos_inverter.hw`.                      │
│      • Assert cold build time remains under 40 milliseconds.                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

*Approved by the HardwareScript Core Architecture Team — September 2026*