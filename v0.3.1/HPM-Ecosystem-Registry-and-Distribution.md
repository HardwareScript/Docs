
---

# HPM Ecosystem, Registry & Distribution Specification

```
Document Type: Authoritative Package Registry & Distribution Specification
Target Version: v0.3.1 (Production-Locked Standard)
Status: Approved for Implementation
Platform: GitHub (Index & Vault) + Cloudflare Pages (Edge CDN)
Infrastructure Cost: $0.00 / month
Date: September 2026
```

---

## 1. Executive Architecture: The Serverless Edge Registry

The HardwareScript Package Manager (HPM) registry operates completely without dynamic database servers, message queues, or persistent virtual machines.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 HPM THREE-TIER ZERO-COST ARCHITECTURE                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. METADATA EDGE CDN (Cloudflare Pages @ pkg.hardwarescript.org)           │
│     • Static JSON/TOML package metadata served from 300+ edge cities.       │
│     • Query response time: < 10 ms globally.                                │
│     • Cost: $0 (Unlimited bandwidth & requests).                            │
│                                                                             │
│  2. METADATA SOURCE OF TRUTH (GitHub: `hpm-registry/index`)                 │
│     • Alphabetical phonebook of package versions and Blake3 hashes.         │
│     • PR-driven queue handles concurrent package publishes.                 │
│                                                                             │
│  3. IMMUTABLE PERMANENT STORAGE (GitHub: `hpm-registry/vault`)              │
│     • Release assets store Blake3-hashed `.tar.gz` packages.                │
│     • Permanent immutability guarantees 10-year reproducible silicon builds.│
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. GitHub Organization Setup (`hpm-registry`)

The dedicated GitHub organization `hpm-registry` contains three primary repositories:

```
github.com/hpm-registry/
├── index/                # The Metadata Phonebook (Deploys to Cloudflare Pages)
│   ├── config.json       # Registry parameters & CDN URLs
│   ├── s/k/skywater-sky130.json
│   └── t/i/ti-power.json
│
├── vault/                # The Immutable Tarball Storage
│   └── releases/         # GitHub Releases holding immutable package archives
│
└── bouncer-ci/           # Automated Verification Engine
    └── src/              # Rust validation tool invoked by GitHub Actions
```

---

## 3. The `hpm-registry/index` Data Format

To ensure rapid resolution, metadata is stored as compact JSON files structured by a 2-level character prefix directory tree (e.g. `@skywater/sky130` $\to$ `s/k/skywater-sky130.json`).

### 3.1 Metadata Schema (`s/k/skywater-sky130.json`)

```json
{
  "name": "@skywater/sky130",
  "description": "SkyWater SKY130 1.8V/3.3V Silicon PDK",
  "license": "Apache-2.0",
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
      "published_at": "2026-09-01T04:00:00Z"
    }
  }
}
```

---

## 4. The Complete Publishing Lifecycle

```
  [ DEVELOPER ]
  Runs `hw publish`
        │
        ▼ (Packs `.hw`, `hw.toml`, `.wasm` into `submission.tar.gz`)
  [ FORK & PR CREATION ]
  Opens PR on `hpm-registry/index` (Adds/Updates JSON file)
        │
        ▼
  [ GITHUB ACTIONS BOUNCER (`verify-publish.yml`) ]
  ├── 1. Organization Scope Gate: Verifies PR author belongs to `@scope` on GitHub
  ├── 2. File Allowlist Gate: Rejects any file not matching `.hw`, `.toml`, `.md`, `.wasm`
  ├── 3. Syntax & Physical Gate: Executes `hw check --foundry`
  ├── 4. Hashing Gate: Computes canonical Blake3 checksum
  └── 5. Archival Step: Uploads `package.tar.gz` to `hpm-registry/vault` Release
        │
        ▼
  [ AUTOMATIC MERGE ]
  Bot squash-merges PR into `hpm-registry/index::main`
        │
        ▼ (Cloudflare Pages Webhook fires)
  [ CLOUDFLARE EDGE DEPLOY ]
  Metadata live worldwide across 300+ cities in < 2 seconds
```

### 4.1 The Bouncer Workflow (`.github/workflows/verify-publish.yml`)

```yaml
name: HPM Registry Automated Bouncer
on:
  pull_request_target:
    branches: [ main ]
    paths:
      - '**/*.json'

jobs:
  verify_and_vault:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write

    steps:
      - name: Checkout Index Repository
        uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}

      - name: Install HardwareScript Toolchain
        uses: hardwarescript/setup-hw@v1
        with:
          version: '0.3.1'

      - name: Run Allowlist & Physical DRC Verification
        id: bouncer
        run: |
          # 1. Download proposed tarball from PR metadata
          SUBMISSION_URL=$(jq -r '.tarball_url' ${{ github.event.pull_request.files[0].filename }})
          curl -sL "$SUBMISSION_URL" -o submission.tar.gz

          # 2. Strict File Extension Allowlist Check
          hw-bouncer check-allowlist \
            --archive submission.tar.gz \
            --allow ".hw,.toml,.md,.txt,.wasm"

          # 3. Physical Silicon/Board Viability Check
          mkdir -p ./sandbox
          tar -xzf submission.tar.gz -C ./sandbox
          hw check --foundry ./sandbox

          # 4. Compute Blake3 Checksum
          CHECKSUM="blake3:$(b3sum submission.tar.gz | awk '{print $1}')"
          echo "checksum=$CHECKSUM" >> $GITHUB_OUTPUT

      - name: Permanent Vault Archival
        uses: softprops/action-gh-release@v1
        with:
          repository: hpm-registry/vault
          tag_name: "${{ steps.bouncer.outputs.tag_name }}"
          files: submission.tar.gz
          token: ${{ secrets.VAULT_RELEASE_TOKEN }}

      - name: Auto-Merge Pull Request
        run: gh pr merge ${{ github.event.pull_request.number }} --auto --squash --delete-branch
        env:
          GITHUB_TOKEN: ${{ secrets.BOT_PAT }}
```

---

## 5. The Complete Installation & Consumption Lifecycle

When an engineer runs `hw add @skywater/sky130` or compiles with `hw build`:

```
  [ USER TERMINAL ]
  `hw add @skywater/sky130`
        │
        ▼ (HTTPS GET < 10ms)
  [ CLOUDFLARE EDGE CDN ] ──► `https://pkg.hardwarescript.org/s/k/skywater-sky130.json`
        │
        ▼
  [ MVS RESOLVER ] ──► Selects minimum compatible version (e.g. 1.4.0)
        │
        ▼ (HTTPS GET)
  [ GITHUB RELEASES VAULT ] ──► Downloads `package.tar.gz` to local cache `~/.hw/store/`
        │
        ▼
  [ BLAKE3 VERIFICATION ] ──► Confirms hash matches lockfile exactly
        │
        ▼ (Zero-Copy Reflink)
  [ VENDOR UNPACK ] ──► Unpacks directly into `vendor/@skywater/sky130/`
        │
        ▼
  [ SALSA COMPILATION ] ──► Bytecode VM evaluates pure `.hw` PCells in < 3ms
```

---

## 6. Security Boundaries & Protection Policies

| Security Threat | Attack Scenario | HPM Defense Policy | Enforcement Point |
| :--- | :--- | :--- | :--- |
| **Namespace Hijacking** | Malicious user publishes fake `@skywater` package | **GitHub Org Membership Verification**: Submitter must be a member of `github.com/scope`. | Bouncer CI Gate 1 |
| **Malicious Code Injection** | Attacker embeds bash script (`.sh`) or `.exe` | **Default-Deny File Allowlist**: Only `.hw`, `hw.toml`, `.md`, `.wasm` admitted. | Bouncer CI Gate 2 |
| **Physical Arson / Shorts**| Broken layout shorting VDD to Ground | **Static Physics DRC Gate**: `hw check --foundry` validates electrical viability. | Bouncer CI Gate 3 |
| **Bait-and-Switch** | Author mutates release archive after approval | **Immutable Vault + Blake3 Lockfile**: Tarballs stored in permanent vault. | Download Verifier |
| **Supply Chain Deletion** | Author deletes original GitHub repository | **Permanent Archival in `hpm-registry/vault`**: Releases never deleted. | Registry Policy |

---

## 7. Implementation Deliverables & Rollout Checklist

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          IMPLEMENTATION CHECKLIST                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  PHASE 1: TOOLCHAIN INTEGRATION (`crates/hwc-package`)                      │
│  • [ ] Rename binary driver to `hw` across all crates and build configs.    │
│  • [ ] Implement `hw.toml` and `hw.lock` parsers with Serde in Rust.        │
│  • [ ] Implement MVS dependency resolver algorithm.                         │
│  • [ ] Implement `vendor/` engine with OS Copy-on-Write (CoW) reflinks.     │
│  • [ ] Implement `.vendor_checksum` local mutation monitor.                 │
│                                                                             │
│  PHASE 2: HPM REGISTRY INFRASTRUCTURE (`hpm-registry`)                     │
│  • [ ] Create GitHub organization: `hpm-registry`.                          │
│  • [ ] Initialize `hpm-registry/index` repository with prefix tree structure│
│  • [ ] Initialize `hpm-registry/vault` repository for permanent releases.   │
│  • [ ] Deploy `index` to Cloudflare Pages (`pkg.hardwarescript.org`).       │
│  • [ ] Write `verify-publish.yml` GitHub Actions automated bouncer.         │
│                                                                             │
│  PHASE 3: END-TO-END VERIFICATION                                           │
│  • [ ] Publish baseline packages: `@std/primitives` and `@skywater/sky130`. │
│  • [ ] Execute `hw new`, `hw add`, and `hw build` on test chip design.      │
│  • [ ] Assert cold build time remains under 30 ms.                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```