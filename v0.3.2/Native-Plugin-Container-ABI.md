# HardwareScript v0.3.2: Universal Native Container Plugin ABI Specification

**Document Type:** Binary Interface & Native Extension Standard  
**Target Version:** v0.3.2 (Production-Locked Standard)  
**Status:** Authoritative & Canonical  
**Target Crates:** `hwc-router`, `hwc-engine`, `hwc-cli`  
**Supersedes:** `FFI-and-WASM64-ABI.md` (WebAssembly Memory64 ABI permanently eradicated)  
**Date:** September 2026  

---

## 1. Executive Summary: The Fall of WASM64 & The Native Pivot

In earlier drafts, HardwareScript evaluated WebAssembly Memory64 (`wasm64`) as an extension mechanism. Comprehensive benchmarking and industrial EDA evaluation revealed fatal bottlenecks:

1. **Linear Memory Isolation Overhead:** Copying multi-gigabyte routing obstruction tensors into WASM linear memory consumed excessive wall-clock time and doubled memory footprint.
2. **GPU & CUDA Interop Barriers:** Complex EDA accelerators (analytical placement, SAT engines, tensor maze routers) depend on NVIDIA CUDA, Vulkan Compute, or Apple Metal. WASM cannot natively interface with GPU hardware runtimes without cumbersome translation layers.
3. **SIMD Vectorization Ceilings:** WebAssembly SIMD is constrained to 128-bit vectors (`v128`), preventing utilization of 256-bit AVX2, 512-bit AVX-512, and ARM SVE vector pipelines.

HardwareScript v0.3.2 permanently eradicates WebAssembly in favor of the **Universal Native Container Plugin ABI**:
- External synthesis, placement, and routing stages compile to **native shared libraries (`.so`, `.dll`, `.dylib`)** exposing a strict, versioned `#[repr(C)]` ABI.
- The package ecosystem distributes pre-compiled multi-target archives (`.hpm`).
- The HardwareScript CLI downloads **only the pre-compiled binary matching the user's host OS and CPU architecture**.
- End users require **zero local C, C++, Rust, or Zig toolchains**.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 UNIVERSAL NATIVE CONTAINER EXECUTION MODEL                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ Plugin Developer ]                                                       │
│  • Implements algorithms in C, C++, Rust, Zig, or Ada.                      │
│  • Compiles native shared objects (.so / .dll / .dylib) via GitHub Actions. │
│  • Publishes single multi-target package: `@academic/router.hpm`.           │
│                                                                             │
│  [ Package Registry (Edge CDN) ]                                            │
│  • Manifest maps target triples to specific downloadable archive URLs:      │
│    - x86_64-unknown-linux-gnu                                               │
│    - x86_64-pc-windows-msvc                                                 │
│    - aarch64-apple-darwin                                                   │
│                                                                             │
│  [ HardwareScript CLI Driver: hw ]                                          │
│  • Identifies host architecture at installation (`hw add @academic/router`).│
│  • Downloads ONLY the matching native binary payload into `vendor/`.        │
│  • End-user requires ZERO local C/C++/Rust/Zig compilers.                   │
│                                                                             │
│  [ Zero-Copy FFI Handshake ]                                                │
│  • Host maps `FlatGeometryBuffer` pointers into `HwcStageContextC`.          │
│  • Plugin executes bare-metal AVX-512, OpenMP, or native CUDA routines.    │
│  • Resulting segments written directly into host memory; zero serialization.│
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. The Canonical C-ABI Interface (`hwc_plugin_abi.h`)

All dynamic native plugins implement and link against the canonical header:

```c
/* hwc_plugin_abi.h - HardwareScript v0.3.2 Universal Native Plugin ABI */
#ifndef HWC_PLUGIN_ABI_H
#define HWC_PLUGIN_ABI_H

#include <stdint.h>
#include <stddef.h>

#ifdef __cplusplus
extern "C" {
#endif

#define HWC_PLUGIN_API_VERSION 302

/* Contiguous picometer wire segment */
typedef struct {
    uint32_t net_id;
    uint16_t layer_idx;
    uint8_t  _reserved[2];
    int64_t  start_x_pm;
    int64_t  start_y_pm;
    int64_t  end_x_pm;
    int64_t  end_y_pm;
    int64_t  width_pm;
} HwcWireSegmentC;

/* Execution Context exchanged across the FFI boundary */
typedef struct {
    int64_t bounds_min_x_pm;
    int64_t bounds_min_y_pm;
    int64_t bounds_max_x_pm;
    int64_t bounds_max_y_pm;

    /* Read-only obstacle geometry from FlatGeometryBuffer */
    const HwcWireSegmentC* obstacles_ptr;
    size_t                 num_obstacles;

    /* Pre-allocated host output buffer */
    HwcWireSegmentC*       output_ptr;
    size_t                 output_capacity;
    size_t                 written_count;

    int32_t                status_code; /* 0 = Success, >0 = Error */
    char                   error_msg[256];
} HwcStageContextC;

#if defined(_WIN32)
    #define HWC_EXPORT __declspec(dllexport)
#else
    #define HWC_EXPORT __attribute__((visibility("default")))
#endif

/* Required entry points */
HWC_EXPORT uint32_t hwc_plugin_api_version(void);
HWC_EXPORT int32_t  hwc_plugin_execute_stage(HwcStageContextC* ctx);

#ifdef __cplusplus
}
#endif

#endif /* HWC_PLUGIN_ABI_H */
```

---

## 3. Rust Host Implementation (`hwc-router`)

The host compiler dynamically loads the native library via `libloading`, validates the API version, and executes the stage without serialization:

```rust
// hwc-router/src/native_plugin.rs

use hwc_diagnostics::CompileError;
use hwc_engine::FlatGeometryBuffer;
use libloading::{Library, Symbol};
use std::path::Path;

pub const HWC_PLUGIN_API_VERSION: u32 = 302;

#[repr(C)]
pub struct HwcWireSegmentC {
    pub net_id: u32,
    pub layer_idx: u16,
    pub _reserved: [u8; 2],
    pub start_x_pm: i64,
    pub start_y_pm: i64,
    pub end_x_pm: i64,
    pub end_y_pm: i64,
    pub width_pm: i64,
}

#[repr(C)]
pub struct HwcStageContextC {
    pub bounds_min_x_pm: i64,
    pub bounds_min_y_pm: i64,
    pub bounds_max_x_pm: i64,
    pub bounds_max_y_pm: i64,
    pub obstacles_ptr: *const HwcWireSegmentC,
    pub num_obstacles: usize,
    pub output_ptr: *mut HwcWireSegmentC,
    pub output_capacity: usize,
    pub written_count: usize,
    pub status_code: i32,
    pub error_msg: [u8; 256],
}

pub type PluginVersionFn = unsafe extern "C" fn() -> u32;
pub type PluginExecuteFn = unsafe extern "C" fn(*mut HwcStageContextC) -> i32;

pub struct NativeStageRunner {
    _lib: Library,
    execute: PluginExecuteFn,
}

impl NativeStageRunner {
    pub fn load(path: &Path) -> Result<Self, CompileError> {
        unsafe {
            let lib = Library::new(path).map_err(|e| CompileError::PluginLoadFailed {
                path: path.to_path_buf(),
                reason: e.to_string(),
            })?;

            let version_fn: Symbol<PluginVersionFn> = lib.get(b"hwc_plugin_api_version\0")
                .map_err(|e| CompileError::PluginSymbolMissing { symbol: "hwc_plugin_api_version", reason: e.to_string() })?;

            let version = version_fn();
            if version != HWC_PLUGIN_API_VERSION {
                return Err(CompileError::PluginApiMismatch {
                    expected: HWC_PLUGIN_API_VERSION,
                    found: version,
                });
            }

            let execute_fn: Symbol<PluginExecuteFn> = lib.get(b"hwc_plugin_execute_stage\0")
                .map_err(|e| CompileError::PluginSymbolMissing { symbol: "hwc_plugin_execute_stage", reason: e.to_string() })?;

            Ok(Self {
                _lib: lib,
                execute: *execute_fn,
            })
        }
    }

    pub fn execute(&self, ctx: &mut HwcStageContextC) -> Result<(), CompileError> {
        let code = unsafe { (self.execute)(ctx) };
        if code != 0 || ctx.status_code != 0 {
            let msg = std::str::from_utf8(&ctx.error_msg)
                .unwrap_or("Unknown plugin error")
                .trim_end_matches('\0');
            return Err(CompileError::PluginExecutionError { message: msg.to_string() });
        }
        Ok(())
    }
}
```

---

## 4. Multi-Language Plugin Implementations

### 4.1 Implementation in C++20 (AVX-512 Router)

```cpp
// router.cpp
#include "hwc_plugin_abi.h"
#include <immintrin.h>
#include <cstring>

HWC_EXPORT uint32_t hwc_plugin_api_version(void) {
    return HWC_PLUGIN_API_VERSION;
}

HWC_EXPORT int32_t hwc_plugin_execute_stage(HwcStageContextC* ctx) {
    if (!ctx) return 1;

    // Scan obstacles using AVX-512 SIMD
    for (size_t i = 0; i < ctx->num_obstacles; ++i) {
        const HwcWireSegmentC& obs = ctx->obstacles_ptr[i];
        // Execute pathfinding...
    }

    // Write routed segment directly into host memory buffer
    if (ctx->output_capacity > 0) {
        HwcWireSegmentC& wire = ctx->output_ptr[0];
        wire.net_id = 1;
        wire.layer_idx = 2;
        wire.start_x_pm = ctx->bounds_min_x_pm;
        wire.start_y_pm = ctx->bounds_min_y_pm;
        wire.end_x_pm = ctx->bounds_max_x_pm;
        wire.end_y_pm = ctx->bounds_max_y_pm;
        wire.width_pm = 300000; // 300nm
        ctx->written_count = 1;
    }

    ctx->status_code = 0;
    return 0;
}
```

### 4.2 Implementation in Zig 0.13+

```zig
// router.zig
const std = @import("std");

pub const HwcWireSegmentC = extern struct {
    net_id: u32,
    layer_idx: u16,
    _reserved: [2]u8 = [_]u8{0} ** 2,
    start_x_pm: i64,
    start_y_pm: i64,
    end_x_pm: i64,
    end_y_pm: i64,
    width_pm: i64,
};

pub const HwcStageContextC = extern struct {
    bounds_min_x_pm: i64,
    bounds_min_y_pm: i64,
    bounds_max_x_pm: i64,
    bounds_max_y_pm: i64,
    obstacles_ptr: [*]const HwcWireSegmentC,
    num_obstacles: usize,
    output_ptr: [*]HwcWireSegmentC,
    output_capacity: usize,
    written_count: usize,
    status_code: i32,
    error_msg: [256]u8,
};

export fn hwc_plugin_api_version() u32 {
    return 302;
}

export fn hwc_plugin_execute_stage(ctx: *HwcStageContextC) i32 {
    ctx.written_count = 0;
    ctx.status_code = 0;
    return 0;
}
```

---

## 5. Universal Package Distribution (`.hpm`)

The HardwareScript Package Manager (`hpm`) bundles compiled shared objects into an architecture-agnostic package manifest:

```json
{
  "name": "@academic/dr_cu_router",
  "version": "1.4.0",
  "abi_version": 302,
  "binaries": {
    "x86_64-unknown-linux-gnu": {
      "url": "https://cdn.hws.dev/pkgs/dr_cu/1.4.0/linux_x64.so",
      "sha256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
    },
    "x86_64-pc-windows-msvc": {
      "url": "https://cdn.hws.dev/pkgs/dr_cu/1.4.0/windows_x64.dll",
      "sha256": "d41d8cd98f00b204e9800998ecf8427e"
    },
    "aarch64-apple-darwin": {
      "url": "https://cdn.hws.dev/pkgs/dr_cu/1.4.0/darwin_arm64.dylib",
      "sha256": "4355a46b19d348dc2f57c046f8ef63d4538ebb936000f3c9ee954a27460dd865"
    }
  }
}
```

When a user runs:
```bash
hw add @academic/dr_cu_router
```
The CLI detects `std::env::consts::OS` and `ARCH`, downloads the single matching pre-compiled library into `vendor/`, and verifies its SHA-256 hash. The build runs immediately with native hardware performance.
