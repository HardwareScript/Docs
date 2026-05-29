# HPM Bootstrap Architecture (v0.1.6)

**Version**: 0.1.6  
**Focus**: Serverless Package Registry, GitHub Automation, and Distribution  
**Status**: Architectural Authoritative Reference

---

## Executive Summary

Hardware Script's ultimate "moat" is not just the language—it is the **Hardware Package Manager (HPM)**. If we make it frictionless for manufacturers, hobbyists, and chip designers to publish and consume packages, HWS becomes the undisputed industry standard.

To bootstrap this ecosystem without requiring expensive cloud infrastructure or complex backend servers, HPM will use a **Serverless Registry Index Model** backed entirely by GitHub infrastructure and GitHub Actions.

Because Hardware Script compiles to physical blueprints (not CPU-executable binaries), traditional cybersecurity risks (Remote Code Execution) are minimized. Instead, our security focus is on **Physical Viability**—ensuring published packages don't contain short circuits or broken physics. The HWS Compiler itself acts as the ultimate security bouncer.

---

## 1. The Core Dilemma: Monorepo vs. Multi-Repo

### The Problem

If we put all package files (`.hw`, `.glb`, etc.) into a single GitHub repository organized by folders, users cannot easily download just one package. Git requires cloning the entire repository, which will eventually grow to hundreds of gigabytes.

### The Solution: The Registry Index Model

We do not store the actual package code in our central repository. Instead, we store **Metadata and Pointers**.

### The Architecture Breakdown

**The Author's Repo**: Package authors (e.g., Texas Instruments) host their actual `.hw` code in their own GitHub repositories.

**The hpm-index Repo**: We own a single central repository called `hpm-index`. It contains tiny JSON/TOML files that act as a phonebook. It maps a package name (`@ti/power`) to the URL of the author's repository.

**The Tarball Download**: When a user runs `hpm install`, the CLI looks up the URL in the index, and uses GitHub's native API to download a highly-compressed `.tar.gz` of just that specific package version.

---

## 2. The hpm-index Structure

The central registry repository is purely a directory of text files. To ensure git remains lightning fast, it is structured alphabetically (like crates.io).

```
hpm-index/ (GitHub Repository)
├── config.json
├── t/
│   ├── i/
│   │   └── ti-power.toml      <-- Metadata for @ti/power
│   └── s/
│       └── tsmc-5nm.toml      <-- Metadata for @tsmc/5nm
└── a/
    ├── r/
    │   └── arduino-nano.toml  <-- Metadata for @arduino/nano
```

### What is inside the metadata file?

A tiny, easily searchable record of versions, hashes, and download links.

**Example: t/i/ti-power.toml**

```toml
name = "@ti/power"
description = "Texas Instruments Power Management ICs"
tags = ["power", "regulator", "buck", "smd"]

[versions."1.0.0"]
tarball_url = "https://github.com/texas-instruments/hws-power/archive/refs/tags/v1.0.0.tar.gz"
sha256 = "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"

[versions."1.1.0"]
tarball_url = "https://github.com/texas-instruments/hws-power/archive/refs/tags/v1.1.0.tar.gz"
sha256 = "8f434346648f6b96df89dda901c5176b10a6d83961dd3c1ac88b59b2dc327aa4"
```

---

## 3. The Security Paradigm: 4 Pillars of Defense

Because users will be downloading files authored by strangers on the internet, HPM enforces a draconian, four-pillar security model. Hardware Script compiles to physical blueprints, not CPU-executable binaries, so the attack surface is already small. These pillars close the remaining gaps.

### Pillar 1: The Cryptographic Padlock (Anti-Bait-and-Switch)

**The Threat**: An author publishes a valid package, passes the checks, and then quietly alters their GitHub release tomorrow to include malicious files.

**The Defense**:

1. When a package is published, the GitHub Action calculates the exact **SHA256 hash** of the `.tar.gz` file and hardcodes it into the `hpm-index`.
2. When a user runs `hpm install`, the CLI downloads the file and hashes it locally before unpacking it.
3. If the author altered even a single byte of the file after publishing, the hashes will mismatch. The CLI immediately deletes the file, aborts the installation, and throws a fatal security error:

```
Error: Cryptographic Hash Mismatch
Package: @evil/package v1.0.0
Expected: e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
Received: 8f434346648f6b96df89dda901c5176b10a6d83961dd3c1ac88b59b2dc327aa4

This package has been tampered with after publication.
Installation aborted for your safety.
```

**How Industry Leaders Handle This**:

- **Homebrew**: Uses SHA256 hashes for every formula. If the download doesn't match the hash perfectly, Homebrew refuses to install it.
- **Rust/Cargo**: Hosts tarballs on `crates.io` with immutable storage, making tampering impossible.

**Note**: When HPM grows large enough and gets funding, we will transition to the Rust model (hosting tarballs ourselves on S3 or Cloudflare R2). But for the bootstrap phase, the Homebrew SHA256 method is **100% secure**.

### Pillar 2: The Strict File Allowlist (Anti-Malware)

**The Threat**: An author hides a malicious bash script (`.sh`), a Python script (`.py`), or an executable inside their hardware package.

**The Defense**: During the publish phase, the GitHub Action iterates through every file in the package. It enforces a strict **Default-Deny** policy. If it finds a single file that does not match the following extension allowlist, the package is instantly rejected:

| Extension | Purpose | Security Status |
|-----------|---------|-----------------|
| `.hw` | Hardware Script source files | Parsed into AST, never executed |
| `hw.toml` | Package manifest | Strictly parsed, no code execution |
| `.md` / `.txt` | Documentation and licenses | Inert text |
| `.glb` / `.step` | Static 3D meshes for component footprints | Inert geometry data |

**That is it. Nothing else gets in.**

Because HWS parses `.hw` and `.toml` files into an Abstract Syntax Tree (AST) rather than executing them dynamically, text-based code injection is impossible.

**Example Rejection**:

```
Error: Unauthorized File Type Detected
Package: @suspicious/widget
Rejected File: scripts/install.sh

HPM only allows the following file types:
  - .hw (Hardware Script source)
  - hw.toml (Package manifest)
  - .md, .txt (Documentation)
  - .glb, .step (3D geometry)

Remove all unauthorized files and try again.
```

### Pillar 3: Physical Viability (Anti-Short-Circuit)

**The Threat**: An author intentionally swaps the VCC and GND pins on a published component footprint, causing physical PCBs to catch fire when manufactured.

**The Defense**: The GitHub Action runs `hwc check --foundry` on the submitted package. If the compiler detects:

- Multiple drivers on a single net
- Short circuits between power rails
- Missing thermal properties required for MPV (Minimum Physical Viability)
- Dielectric breakdown violations

...the package is rejected with the exact compiler error message.

**Example Rejection**:

```
Error: Physical Viability Check Failed
Package: @malicious/component

Compiler Error L05: Multiple Drivers
  --> src/power_chip.hw:45:12
   |
45 |     route VCC to GND
   |           ^^^ Short circuit detected
   |
   = This would cause physical damage to manufactured hardware

Package rejected. Fix the design and resubmit.
```

### Pillar 4: The MIT License Mandate (Legal Frictionless)

**The Threat**: Corporate legal departments refusing to allow engineers to use Hardware Script because third-party packages might contain copyleft (GPL) or restrictive licenses.

**The Defense**: To publish to HPM, the package must contain a standard MIT License. The GitHub Action scans the `LICENSE` file; if it is not MIT, it is rejected.

**Example Rejection**:

```
Error: Invalid License
Package: @company/widget
Required: MIT License
Found: GPL-3.0

HPM requires all published packages to use the MIT License.
Please add a LICENSE file with the standard MIT License text.

For reference: https://opensource.org/licenses/MIT
```

**The Legal Guarantee**: Furthermore, the `hpm-index` Terms of Service stipulate:

> "By submitting a Pull Request to the HPM Index, you irrevocably agree that the targeted source code is distributed under the MIT License, regardless of any conflicting licenses in the source repository."

This creates an ironclad legal boundary that protects both package consumers and the HPM ecosystem.

---

## 4. The Publishing Pipeline (The Bouncer)

To publish a package, an author does not need an account on a special HPM website. They just use the CLI, which interacts with GitHub.

### Step 1: Submission

The author runs `hpm publish` inside their local package folder. The HPM CLI:

1. Creates a `.tar.gz` archive of the package
2. Uploads it as a GitHub Release to the author's own repository
3. Automatically creates a **Pull Request (PR)** on our central `hpm-index` repository to add their new version to the TOML file

### Step 2: The Automated Bouncer (GitHub Actions)

When the PR opens on `hpm-index`, a GitHub Action automatically validates the submitted package through all four security pillars:

**The Action Executes**:

```bash
# Pillar 2: File Allowlist Check
hpm validate-files .

# Pillar 4: MIT License Check
hpm validate-license .

# Pillar 3: Syntax and Logic Check
hwc check .

# Pillar 3: Physical Viability Check
hwc check --foundry

# Pillar 1: Calculate SHA256 Hash
sha256sum package.tar.gz
```

### Step 3: Auto-Merge or Rejection

If **any** check fails, the GitHub Action:
- Closes the PR with a detailed error message
- Denies the package entry to the registry
- Provides actionable feedback for the author to fix and resubmit

If **all** checks pass, the GitHub Action:
- Automatically merges the PR
- The package is now live in the global registry
- **Zero human intervention required by the core team**

---

## 5. The Installation Pipeline (Frictionless Consumer)

When a hardware engineer wants to use a package, the process is instantaneous and decentralized.

**Command**:

```bash
hpm install @ti/power
```

**What the CLI does behind the scenes**:

1. Updates its local copy of the `hpm-index` (a simple `git fetch` of text files, takes < 1 second).
2. Looks inside `t/i/ti-power.toml` to find the latest version (e.g., `1.1.0`).
3. Reads the `tarball_url` and `sha256` hash.
4. Downloads the `.tar.gz` file directly from Texas Instruments' GitHub release page.
5. Verifies the SHA256 hash to prevent Man-in-the-Middle (MitM) attacks or silent alterations by the author.
6. Unpacks it into `~/.hw/packages/@ti/power/1.1.0/`.
7. Updates the user's local `hw.lock` file.

**Result**: The user downloaded the package perfectly, and we paid **$0.00** in server bandwidth costs, because GitHub served the tarball.

---

## 6. The Web Frontend (hpm.dev / pkg.hw-script.org)

Users need a place to search for packages, read documentation, and copy install commands.

Because the `hpm-index` is just a repository of TOML files, building the website is trivial.

### The Static Site Generation (SSG) Model

We do not need a complex database or backend server.

1. We set up a simple **Next.js, Svelte, or Hugo** website hosted for free on **GitHub Pages** or **Vercel**.
2. Every time a PR is merged into `hpm-index` (a new package is published), a GitHub Action triggers the website to rebuild.
3. The website parses all the TOML files in the repository and generates a beautiful, searchable HTML catalog.
4. It reads the `tags`, `description`, and `name` to populate a lightning-fast client-side search bar (e.g., using Algolia or plain JSON indexing).

### The User Experience

1. The user searches "0805 Resistor".
2. The website shows `@yageo/passives` with the description and versions.
3. The user clicks a button to copy `hpm install @yageo/passives`.

---

## 7. The "Left-Pad" Problem and Future Evolution

### The Risk: Deleted Repositories

The Serverless Registry Index Model is perfect for bootstrapping the ecosystem for free.

However, it carries one known risk: **The "Left-Pad" Problem**. If an author completely deletes their GitHub repository, the URL in our `hpm-index` goes dead. Users who already have it in their local `~/.hw/packages/` cache will be fine, but new users will fail to compile old projects.

This famously happened to Node.js (npm) when an author deleted the `left-pad` package, breaking millions of builds worldwide.

### The Bootstrap Acceptance

For the bootstrap phase, we accept this risk. If an author deletes their repo, the package breaks for new installations.

### The Migration Path: The Immutable Registry

When Hardware Script secures funding and scales, we will transition HPM to a **Hosted Model** (like Rust's `crates.io`):

1. We will host the `.tar.gz` files ourselves on an S3 bucket or Cloudflare R2.
2. We will enforce an **Immutable Registry** rule: Once published, a package can never be altered or deleted from HPM.
3. This guarantees that a hardware design compiled in 2026 will still compile with absolute deterministic perfection in 2036.

Until that day, the GitHub-backed Registry Index provides a mathematically secure, infinitely scalable, and zero-cost package manager to ignite the Hardware Script revolution.

---

## 8. The Complete Security Summary

| Attack Vector | Threat | Defense Mechanism | Status |
|---------------|--------|-------------------|--------|
| **Bait-and-Switch** | Author swaps tarball after approval | SHA256 cryptographic hash validation | ✅ Mathematically secure |
| **Malware Injection** | Hidden scripts or executables | Strict file allowlist (`.hw`, `.toml`, `.md`, `.glb`, `.step` only) | ✅ Default-deny policy |
| **Code Execution** | Malicious package runs virus on user's machine | HWS compiler is hermetic (AST parsing, no `eval()`) | ✅ Architecturally impossible |
| **Physical Malice** | Component with swapped VCC/GND pins | HWS Compiler runs DRC/Physics checks during publishing | ✅ Automated rejection |
| **Legal Issues** | Non-MIT licensed code causes corporate friction | Automated MIT License enforcement in publishing pipeline | ✅ Ironclad legal boundary |
| **Deleted Repos** | Author deletes GitHub repo, breaking builds | Bootstrap: Accepted risk. Long-term: Immutable hosted registry | ⚠️ Bootstrap limitation |

---

## Summary of the Bootstrapping Moat

By adopting the Registry Index Model with the **4 Pillars of Defense**:

1. **Zero Hosting Costs**: GitHub stores the code, serves the downloads, and runs the validation servers.
2. **Infinite Scaling**: We bypass the monorepo download problem. Users only download the exact megabytes they need.
3. **Cryptographic Security**: SHA256 hash validation prevents bait-and-switch attacks. Math, not trust.
4. **Strict File Allowlist**: Only `.hw`, `.toml`, `.md`, `.txt`, `.glb`, and `.step` files are permitted. Default-deny policy.
5. **Automated Moderation**: The HWS Compiler acts as a robotic bouncer via GitHub Actions, rejecting physically broken hardware.
6. **Legal Clarity**: Strict MIT License enforcement eliminates corporate legal friction.
7. **Frictionless Search**: A statically generated frontend provides a beautiful, instant search experience.

This is the exact blueprint that allowed Rust (Cargo) and Homebrew to scale to millions of packages. It positions Hardware Script to effortlessly become the default package manager for the global hardware industry.

**The architecture is paranoid, mathematically sound, and enterprise-ready. It costs $0 to bootstrap and scales infinitely.**
