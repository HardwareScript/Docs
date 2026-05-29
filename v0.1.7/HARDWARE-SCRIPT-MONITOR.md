# Hardware Script Monitor

**Document Type**: Engineering & Ecosystem Architecture
**Status**: Official Specification (Revised — Aggregate Engine Strategy)
**Target Version**: v0.1.7

---

## 1. Introduction: The Live-Preview Companion

Hardware Script Monitor (`hsm`) is a native live-preview companion application for Hardware Script.

While Hardware Script Studio is the ultimate end-goal for a full native IDE, **Hardware Script Monitor** is the immediate "mini version" solution. It is **not** an editor, not a PCB CAD tool, and not a replacement for KiCad. Its job is focused and simple:

**Open a Hardware Script eXchange Binary (`.hsx`) and instantly visualize everything whenever the Hardware Script project recompiles.**

### The Workflow Problem

Instead of this tedious manual loop:
1. Edit script
2. Save
3. Compile
4. Open DXF in KiCad
5. Open GLB in a 3D viewer
6. Open SPICE in a simulation tool
7. Repeat manually...

### The Native Solution

You get an instant, unified feedback loop:
1. Edit script
2. Save
3. Compile -> Hardware Script eXchange Binary updates
4. **Monitor refreshes instantly**

Everything updates automatically in one lightning-fast window.

---

## 2. Standard Exports vs. Native Binary Output

### Standard Interoperability Exports ("Golden Files")

Hardware Script strictly maintains its "Open Ecosystem" promise. The compiler still generates industry-standard files for interoperability:
*   `.dxf` (2D Geometry)
*   `.glb` (3D Models)
*   `.cir` (SPICE Netlists)
*   Netlists and Drill/Manufacturing files

These files can still be opened in external, specialized tools like KiCad, mesh viewers, analog simulation tools, and fabrication tools.

### Native Binary Output (`project.hsx`)

In addition to standard exports, Hardware Script generates the **Hardware Script eXchange Binary** (`project.hsx`) — a unified, zero-copy-ready binary containing all compiled hardware data.

This unified file contains:
*   Embedded 3D data (as `AnalyticTrace` primitives — LineSegments and Rectangular Prisms)
*   Embedded 2D data (as analytic vector geometry)
*   Embedded netlist (as `NetlistArena` with `DeviceBinding` metadata)
*   Embedded SPICE definitions
*   `PourMetadata` & Indexes (with `PourID` → `DeviceBinding` lookup table)
*   Preview and Runtime structures

**This `.hsx` file is what Hardware Script Monitor opens and watches.**

---

## 3. Core Features & Capabilities

Hardware Script Monitor serves as a unified lens into the compiled hardware.

### Feature 1: Live Watch & Hot Reload

When a user saves their code (`script.hw`), the compiler updates `project.hsx`.
Hardware Script Monitor automatically detects that the binary has changed and **refreshes instantly**.
*   No reopening files.
*   No manual reload clicks.
*   Just instant visual feedback.

### Feature 2: Multiple Live Views

The Monitor provides several tabs/panels to inspect the multi-dimensional output of the compiler:

*   **3D View:** Displays the 3D board and packages using the high-level `@babylonjs/viewer` (professional PBR, IBL, shadows, auto-framing, real engine.getFps()).
*   **2D View:** Displays layout traces, geometry, and board outlines (rendered via PixiJS for 1M+ segment vector performance).
*   **Netlist View:** Shows logical connections, component hierarchies, and structure with `DeviceBinding` truth display.
*   **SPICE / Simulation View:** Displays simulation waveforms and runtime inspection data (rendered via uPlot for 10M-point performance).
*   **DXF View:** Displays DXF vector primitives using the high-level `dxf-viewer` library (robust industrial support). Background color and optional grid are unified with the 3D viewport. Temporary direct file open available for testing.
*   **Drill / Manufacturing View:** Previews drill layers and manufacturing preparations.
*   **Metadata / Inspector:** Exposes project information, binary structure, build data, and diagnostics.

### Feature 3: Silicon Inspection Mode (Optional Deep-Zoom)

When the user activates "Silicon Inspection Mode," HSM invokes **Handshake C (Geometric Realization)** — the only case where analytic primitives are realized into full voxel geometry. This allows nanometer-scale inspection of transistor gates. In all other modes, HSM renders pure analytic primitives, achieving 11,200× faster visualization.

---

## 4. What Hardware Script Monitor is NOT

To maintain focus and prevent tool bloat, it is important to define what `hsm` is not.

It is **NOT**:
*   A code editor
*   A complete Hardware Script IDE (that is Hardware Script Studio)
*   A schematic editor
*   A PCB editor
*   A Gerber authoring tool
*   A replacement for specialized tools like KiCad

If a user requires manual graphical editing or specific export workflows, they should still utilize external specialized software.

---

## 5. Technical Architecture

### 5.1 Stack Overview: The "Aggregate Engine" Strategy

Hardware Script Monitor uses a **Hybrid Data Factory** architecture — Rust handles all heavy data processing and file management, while professional JavaScript rendering engines handle the visual presentation.

In this model, Rust is the **Data Factory** that extracts, parses, and serves binary data. JavaScript is the **Beauty Layer** that renders it using battle-tested professional graphics engines.

| Layer | Technology | Role |
|-------|-----------|------|
| **Desktop Shell** | Tauri v2 | Native windowing, file system, IPC bridge |
| **Data Factory** | Rust | Opens `.hsx` binary, extracts embedded `.glb`/`.obj` bytes, parses netlist/SPICE data, sends raw bytes to JS via Binary IPC |
 | **3D Rendering** | `@babylonjs/viewer` (high-level) | Professional auto-framed 3D with PBR/IBL/shadows. Real `engine.getFps()` exposed. Background color control. |
 | **2D Vector Rendering** | PixiJS (OffscreenCanvas + Web Worker) | Massive speed for 2D vector traces. Can handle 1M segments easily. Runs in a Web Worker so the UI stays responsive. |
 | **DXF Rendering** | `dxf-viewer` (high-level Three.js) | Mature industrial DXF support. Background color + grid toggle synced with 3D. |
| **Waveform Rendering** | uPlot | The fastest waveform/time-series renderer (10M points in 1ms). |
| **The Bridge** | Binary IPC + Tauri Events + Channels | Rust sends raw `.glb`/`.obj` bytes + lightweight state (selection, telemetry) to JS. Tauri v2 Channels enable persistent 120Hz telemetry streaming. |

#### Why the "Aggregate Engine" Wins

| Bottleneck | Old WGPU-Native Approach | New Aggregate Engine Approach |
|-----------|-------------------------|------------------------------|
| Visual Quality | Raw unlit geometry. Must write PBR shaders from scratch. | **Professional PBR, IBL, shadows, tone mapping, MSAA** — all built into Babylon.js/Three.js |
| GLB/GLTF Support | Must write custom parser for every format | **Native GLB/GLTF loading** — Babylon.js and Three.js load directly from bytes |
| 2D Segment Performance | Must implement vector rendering in WGPU | **PixiJS handles 1M+ segments** with WebGL batched rendering in a Web Worker |
| Development Speed | Months to implement professional visuals | **Days** to integrate proven libraries |
| Memory Footprint | Duplicated data in Rust GPU + WebView | Rust holds binary data, JS engines hold GPU buffers — minimal duplication |
| Input Latency | Zero (bypassed browser) | **Still excellent** — Tauri WebView uses the system's GPU drivers directly (Metal/Vulkan/DirectX) |

### 5.2 The "Rust Data Factory" Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                        Tauri Window                             │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                   SOLIDJS SHELL                          │    │
│  │  ┌──────────┐  ┌─────────────┐  ┌───────────────────┐  │    │
│  │  │   MENU   │  │  TAB BAR    │  │    SIDEBAR        │  │    │
│  │  │   BAR    │  │ [3D][2D]    │  │    Inspector      │  │    │
│  │  │          │  │ [SPICE]     │  │                   │  │    │
│  │  └──────────┘  └─────────────┘  └───────────────────┘  │    │
│  │                                                         │    │
│  │  ┌─────────────────────────────────────────────────┐    │    │
│  │  │              VIEWPORT AREA                       │    │    │
│  │  │  ┌───────────────────────────────────────────┐  │    │    │
│  │  │  │  Babylon.js / PixiJS / Three.js / uPlot   │  │    │    │
│  │  │  │  (Professional Engine renders in WebGPU)  │  │    │    │
│  │  │  │  - PBR Materials & IBL Lighting            │  │    │    │
│  │  │  │  - Anti-Aliasing & Shadows                 │  │    │    │
│  │  │  │  - 60 FPS guaranteed                       │  │    │    │
│  │  │  └───────────────────────────────────────────┘  │    │    │
│  │  └─────────────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                     STATUSBAR                              │   │
│  │   FPS: 60  |  Vertices: 1.2M  |  Violations: 0           │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
         ▲                          ▲                 ▲
         │  Binary IPC              │  Tauri Events   │  Rust Watcher
         │  (Raw .glb bytes,        │  (item-selected │  (notify crate
         │   selection state,        │   telemetry,    │   watches .hsx)
         │   120Hz telemetry stream) │   build-failed) │
```

### 5.3 The Rust Data Factory (What Rust handles)

Rust is no longer the "sole master of the viewport." Instead, Rust is the **high-performance backend** that handles all the heavy lifting:

**File Ingestion:**
- Opens `.hsx` binary using `memmap2` (zero-copy memory mapping)
- Extracts embedded `.glb`/`.obj` 3D geometry data
- Parses `NetlistArena`, `PourMetadata`, `AnalyticTrace` arrays
- Extracts SPICE netlists and simulation data

**GLB Augmentation (GLTF Edit):**
- Uses `gltf` / `gltf-json` crate to programmatically edit GLB files before sending to JS
- Injects `PourID` into the "extras" field of each GLTF node
- This enables Babylon.js picking to return the precise PourID of any clicked mesh
- Without this, picking would return generic mesh names with no connection to the logical data

**Data Serialization (rkyv for Zero-Copy):**
- Uses `rkyv` instead of JSON for complex structured data (netlist trees with 50k+ nodes)
- `rkyv` allows JavaScript to read Rust-native structures as if they were local memory
- No serialization/deserialization overhead for large structured payloads

**Parallel Mesh Generation (Rayon):**
- Uses `rayon` for the "Visual Realization" pass
- Turning 100M traces into GLB meshes is a "Pleasingly Parallel" problem
- Critical for Silicon Inspection Mode where analytic primitives are realized into geometry

**Data Serving via Binary IPC:**
- Sends raw `.glb` bytes → JavaScript (Babylon.js loads from byte array)
- Sends raw `.obj` bytes → JavaScript (Three.js loads from byte array)
- Sends 2D trace geometry → JavaScript (PixiJS renders as vector graphics in Web Worker)
- Sends SPICE waveform data → JavaScript (uPlot renders as time-series)
- Sends selection state (`PourID` → `DeviceBinding`) → JavaScript (sidebar display)

**File Watching & Hot Reload:**
- Uses `notify` crate to watch `.hsx` for changes
- On file change, re-maps via `memmap2`, re-extracts data, re-sends to JavaScript
- **Result:** Saving `.hw` script instantly updates all viewports

### 5.4 Viewport-Specific Engine Selection

| Viewport | Engine | Why |
|----------|--------|-----|
 | **3D View** | `@babylonjs/viewer` (high-level) | Dramatically better auto-framing, environment, camera behavior with minimal custom code. Real FPS reporting. |
 | **2D Trace View** | PixiJS (OffscreenCanvas + Web Worker) | WebGL batched rendering. Handles 1M+ vector segments at 60 FPS. Runs in a Web Worker via OffscreenCanvas to keep the SolidJS sidebar responsive even during heavy renders. |
 | **DXF View** | `dxf-viewer` (high-level Three.js) | Robust support for complex industrial DXF files. Unified background/grid controls with 3D viewport. |
| **SPICE / Waveform** | uPlot | The fastest time-series renderer. 10M points in <1ms. Minimal bundle size. |
| **Netlist Tree** | SolidJS (DOM) + Kobalte Collapsible | Hierarchical data renders natively in SolidJS reactive components. Kobalte provides accessible collapsible tree nodes. |
| **Drill / Manufacturing** | SolidJS (Canvas) | Lightweight canvas rendering for drill data. |

### 5.5 The Binary IPC Contract

Rust communicates with JavaScript through two channels:

**Channel 1: Binary Data (invoke commands)**

```typescript
// JavaScript asks Rust for binary 3D data
const glbBytes: number[] = await invoke("get_hsx_3d_layer");
// Babylon.js loads from byte array
const blob = new Blob([new Uint8Array(glbBytes)]);
SceneLoader.LoadAsync("", URL.createObjectURL(blob), scene);
```

```rust
// Rust responds with raw bytes from .hsx
#[tauri::command]
fn get_hsx_3d_layer(state: State<AppState>) -> Result<Vec<u8>, String> {
    let data = state.hsx_data.lock().map_err(|e| e.to_string())?;
    // Extract and augment .glb bytes from .hsx
    data.extract_augmented_glb().ok_or("No 3D data in .hsx".to_string())
}
```

**Channel 2: Lightweight Events (Tauri event system)**

```typescript
// JavaScript listens for selection events
await listen("item-selected", (event: SelectionPayload) => {
    setSelection(event.payload);
    sidebar.setBinding(event.payload.device_binding);
});
```

```rust
// Rust emits selection state after picking
app_handle.emit("item-selected", SelectionPayload {
    pour_id: 0x00A3_F1C2,
    device_binding: Some(DeviceBinding { device: "M1.gate", net: "VDD" }),
});
```

### 5.6 Resize Synchronization

Since all rendering now happens in JavaScript via Tauri's WebView, resize is handled automatically by the WebView's CSS layout system. No manual resize synchronization is needed:

1. User drags window corner
2. Tauri WebView reflows CSS
3. Canvas elements (Babylon.js, PixiJS, etc.) detect resize via their own observers
4. **No IPC needed** — the WebView engine handles it natively

### 5.7 Spatial Math Alignment: The i64 Nanometer Law

HSM's math system is directly inherited from the Hardware Script compiler core:

| Context | Math System | Library | Purpose |
|---------|-----------|---------|---------|
| **Viewport Matrices** | f32 SIMD | Babylon.js / Three.js | Projection, model, and view matrices for 60 FPS rendering |
| **Picking / Hit-Testing** | i64 fixed-point | Native i64 | Inject `PourID` into GLTF extras as i64 hex string. No floating point jitter when zooming from 200mm PCB to 180nm transistor gate. |
| **SDF / Distance Fields** | i64 fixed-point | Native i64 | All spatial queries use the same i64 nanometer system as the compiler. Eliminates "Floating Point Jitter" entirely. |
| **Coordinate Conversion** | i64 → f32 | Rust engine | Conversion only at the final stage when sending data to JavaScript. All internal logic remains in i64. |

### 5.8 Memory Alignment: The .hsx Zero-Copy Handshake

HSM loads the `.hsx` file using zero-copy memory mapping:

```
.hsx file on disk
       │
       ▼
┌──────────────────────┐
│  memmap2             │  Memory-map the .hsx file directly
│  (Rust crate)        │  No file reads. No allocation. Just a pointer.
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│  Rust Engine         │  bytemuck zero-copy cast into structs
│                      │  gltf crate: inject PourID into GLB node extras
│                      │  rkyv: zero-copy serialize netlist trees
│                      │  rayon: parallel GLB mesh generation
└──────┬───────────────┘
       │ Binary IPC (raw bytes) + Tauri Channels (120Hz telemetry)
       ▼
┌──────────────────────┐
│  JavaScript          │  Babylon.js loads .glb from byte array
│  Viewports           │  PixiJS loads 2D trace data in Web Worker
│                      │  uPlot loads SPICE waveform data
└──────────────────────┘
```

**Performance Win:** Loading a 500MB design happens in **<5ms**. The user saves the code, the binary updates, and HSM refreshes all viewports before the user can even blink.

### 5.9 Logic & Selection Alignment: The Binding Law

HSM's selection engine implements the **Silent Atom Architecture** — geometry is inert unless bound via the `device:` property.

**Selection Pipeline:**
1. **User clicks** on a 3D model in Babylon.js.
2. **Babylon.js ray-casting** returns the picked mesh with its name (e.g., `pour_0xAF31`).
3. **Rust** receives only the mesh name string via IPC, then uses `fxhash` to instantly look up the `PourID` → `DeviceBinding`.
4. **Binary IPC** sends the `DeviceBinding` to SolidJS: `{ pour_id, device, net, is_bound }`.
5. **SolidJS Sidebar** displays either:
   - **Bound**: `"This is M1.gate → net: VDD, device: transistor_0032"` — the Logical Truth.
   - **Unbound**: `"Artist Geometry (Inert)"` — if no binding exists, it's purely decorative.

**UX Win:** The Monitor becomes a visual validator for the "Triple-Check Architecture." Users can visually confirm that every trace has a valid device binding.

### 5.10 Modular Architecture

```
hsm/
├── src-tauri/               # The Rust Core (The "Data Factory")
│   ├── src/
│   │   ├── main.rs          # Tauri entrypoint & Command registration
│   │   ├── lib.rs           # Shared library exports
│   │   ├── engine/          # .hsx Zero-Copy Ingestion
│   │   │   ├── loader.rs    # memmap2-based .hsx memory mapper
│   │   │   ├── parser.rs    # bytemuck zero-copy cast + data extraction
│   │   │   └── watcher.rs   # Hot-reload file watcher (notify)
│   │   ├── telemetry/       # Real-time data streams
│   │   │   └── stats.rs     # FPS, memory, frame timing, 120Hz streaming
│   │   └── extractors/      # Data extractors for each viewport
│   │       ├── mod.rs           # Shared extraction traits
│   │       ├── glb_extractor.rs # Extract + augment GLB (inject PourID via gltf crate)
│   │       ├── trace_2d.rs      # Extract 2D trace geometry
│   │       ├── netlist.rs       # Extract NetlistArena via rkyv (zero-copy)
│   │       ├── spice.rs         # Extract SPICE waveforms
│   │       ├── drill.rs         # Extract drill data
│   │       └── diagnostics.rs   # integrity_report.glb ingestion
│   └── Cargo.toml           # Dependencies: memmap2, bytemuck, notify, serde_json, gltf, gltf-json, rkyv, rayon
├── src/                     # SolidJS Frontend (The "Shell")
│   ├── index.tsx            # Entrypoint
│   ├── App.tsx              # Core layout, tab router, menu bar (Kobalte Tabs)
│   ├── App.css              # Obsidian dark theme (Tailwind utility classes)
│   ├── components/
│   │   ├── ThreeDViewport.tsx      # @babylonjs/viewer high-level 3D viewport
│   │   ├── TwoDViewport.tsx        # PixiJS 2D vector viewport (OffscreenCanvas + Web Worker)
│   │   ├── DxfViewport.tsx         # dxf-viewer (high-level) DXF viewport
│   │   ├── SpiceViewport.tsx       # uPlot waveform viewer
│   │   ├── NetlistView.tsx         # SolidJS hierarchical netlist tree (Kobalte Collapsible)
│   │   ├── DrillView.tsx           # SolidJS canvas drill viewer
│   │   ├── Sidebar.tsx             # Property inspector with DeviceBinding (Kobalte Collapsible)
│   │   ├── StatusBar.tsx           # Telemetry display + 120Hz coordinate stream
│   │   └── GhostView.tsx           # PHYSICAL INTEGRITY ALERT overlay (backdrop-blur-md)
│   ├── workers/
│   │   └── pixi-worker.ts     # Web Worker for OffscreenCanvas PixiJS rendering
│   ├── store/
│   │   ├── telemetry.ts     # Reactive telemetry signals
│   │   └── selection.ts     # Reactive PourID / DeviceBinding state
│   └── bridge/
│       └── ipc.ts           # Binary IPC helpers + event listeners + Channel listeners
├── public/
│   └── assets/
└── package.json             # SolidJS + @babylonjs/viewer + dxf-viewer + pixi.js + three + uplot + @kobalte/core
```

### 5.11 Hot-Reload Mechanism

The file watcher in Rust uses the `notify` crate:

```rust
// src-tauri/src/engine/watcher.rs
use std::sync::Arc;
use std::sync::atomic::{AtomicBool, Ordering};

pub fn start_watcher(path: &str, refresh_flag: Arc<AtomicBool>) {
    use notify::{Event, RecursiveMode, Watcher};
    let flag = refresh_flag.clone();
    let mut watcher = notify::recommended_watcher(move |res: Result<Event, _>| {
        if let Ok(_event) = res {
            flag.store(true, Ordering::Relaxed);
        }
    }).unwrap();
    watcher.watch(path.as_ref(), RecursiveMode::NonRecursive).unwrap();
}
```

When the file changes:
1. `notify` fires → Atom flag set
2. Tauri command polling detects flag, re-maps `.hsx`
3. Rust re-extracts all data (GLB with PourID injection, 2D traces, netlist via rkyv, SPICE)
4. Rust emits `hsx-refreshed` event via Tauri event system
5. JavaScript viewports re-load data from Rust via invoke commands
6. **All viewports update in <50ms**

### 5.12 The Picking Handshake

This is the critical bridge that connects Rust's logical data to Babylon.js's visual meshes:

1. **Rust (gltf crate):** When generating the GLB from `.hsx` data, Rust iterates every mesh node and assigns a name like `pour_0xAF31` where `0xAF31` is the hex-encoded `PourID`. This is injected into the GLTF "extras" field using the `gltf-json` crate.

2. **Babylon.js:** When the user clicks on a mesh, Babylon.js returns `pickedMesh.name` (e.g., `pour_0xAF31`). Only this string is sent back over IPC.

3. **Tauri IPC:** A lightweight `invoke("resolve_device_binding", { meshName: "pour_0xAF31" })` call returns the binding data.

4. **Rust (fxhash):** Rust parses the hex `PourID` from the mesh name, looks it up in the `fxhash`-based `PourMetadata` index, and instantly returns the `DeviceBinding`.

5. **SolidJS:** The sidebar updates with the binding truth.

```typescript
// ThreeDViewport.tsx — Babylon.js click handler
canvas.addEventListener("pointerdown", async (evt) => {
  const pickResult = scene.pick(evt.clientX, evt.clientY);
  if (pickResult?.pickedMesh) {
    const meshName = pickResult.pickedMesh.name; // "pour_0xAF31"
    const binding = await invoke("resolve_device_binding", { meshName });
    setSelection(binding);
  }
});
```

**Performance:** Only a tiny string crosses the IPC bridge. Rust does the heavy lookup. The sidebar updates instantly.

### 5.13 High-Frequency Telemetry Streaming

For SPICE probe monitoring and real-time cursor coordinates, HSM uses **Tauri v2 Channels**.

Instead of making a discrete IPC call for every mouse move (which would be slow), Rust opens a persistent Channel stream:

```rust
// Rust side: open a channel that pushes i64 coordinates at 120Hz
let (tx, rx) = tauri::channel::channel();
state.coordinate_stream.lock().unwrap().replace(tx);

// Push updates at 120Hz
loop {
    let coord = get_cursor_nm_position();
    tx.send(TelemetryPayload { x: coord.x, y: coord.y, z: coord.z });
    std::thread::sleep(std::time::Duration::from_millis(8));
}
```

```typescript
// SolidJS side: listen to the Channel stream
const stream = await invoke("subscribe_coordinate_stream");
for await (const payload of stream) {
  setCursorCoord(`${payload.x} nm, ${payload.y} nm, ${payload.z} nm`);
}
```

**Result:** The status bar coordinate display never lags behind the mouse. The stream operates independently of the render loop and does not block any viewport operations.

### 5.14 HUD Events from Rust to SolidJS

```rust
// Rust emits PourID selection + DeviceBinding after picking
app_handle.emit("item-selected", SelectionPayload {
    pour_id: 0x00A3_F1C2,
    device_binding: Some(DeviceBinding {
        device: "M1.gate",
        net: "VDD",
        component: "transistor_0032",
    }),
    is_artist_geometry: false,
});
```

```rust
// Rust emits build failure events for Ghost View
app_handle.emit("build-failed", BuildFailurePayload {
    violation_count: 3,
    floating_components: 2,
    disconnected_nets: 1,
});
```

```rust
// Rust emits refresh signal when .hsx changes
app_handle.emit("hsx-refreshed", RefreshPayload {
    has_3d: true,
    has_2d: true,
    has_spice: true,
});
```

```typescript
// SolidJS listens and updates reactive components
import { listen } from "@tauri-apps/api/event";

onMount(() => {
  const unlisten = listen<RefreshPayload>("hsx-refreshed", async (event) => {
    // Reload 3D data if available
    if (event.payload.has_3d) {
      const glbBytes = await invoke("get_hsx_3d_layer");
      loadGlbToBabylonScene(glbBytes);
    }
    // Reload 2D traces if available
    if (event.payload.has_2d) {
      const traceData = await invoke("get_hsx_2d_layer");
      updatePixiTraces(traceData);
    }
  });
  onCleanup(() => unlisten.then(fn => fn()));
});
```

### 5.15 The "Commit Gate" Visual Debugger

When a build fails, the compiler produces an `integrity_report.glb`. HSM ingests this file and triggers a **Ghost View** overlay:

| Violation Type | Visual Indicator | Source |
|---------------|------------------|--------|
| **P44 (Floating)** | Glowing Yellow wireframe | `integrity_report.glb` floating component markers |
| **P41 (Disconnected)** | Glowing Red wireframe | `integrity_report.glb` disconnected net markers |
| **Clear** | Normal Obsidian theme | No violations |

**UX Flow:**
1. Build fails → compiler writes `integrity_report.glb`.
2. Rust watcher detects the file → ingests violation data.
3. SolidJS receives a `build-failed` event with violation count.
4. SolidJS triggers the **"PHYSICAL INTEGRITY ALERT"** UI theme (Deep Red/Amber accents replacing normal blue).
5. The viewport renders violations in Ghost View — translucent glowing geometry overlaid on the normal render (handled by Three.js/Babylon.js).

---

## 6. Performance Targets

| Metric | Target | Strategy |
|--------|--------|----------|
| Frame Rate | 60 FPS | Babylon.js/Three.js WebGL/WebGPU rendering. Professional engines handle all optimizations. |
| Binary Load Time | < 5ms for 500MB `.hsx` | `memmap2` zero-copy memory mapping. No file reads, no parsing. |
| 3D Load Time | < 100ms for 100MB GLB | Babylon.js loads GLB directly from Rust's raw bytes via `SceneLoader` |
| 2D Render Speed | 1M segments @ 60 FPS | PixiJS batched WebGL rendering in OffscreenCanvas Web Worker |
| Waveform Render | 10M points in <1ms | uPlot time-series rendering |
| UI Responsiveness | < 16ms frame budget | PixiJS in Web Worker ensures SolidJS sidebar remains smooth even during heavy renders |
| Memory Footprint | < 500MB for 100M-transistor designs | Rust holds `memmap2` pointer (OS page cache). `rkyv` avoids JSON serialization overhead. |
| Hot-Reload Latency | < 50ms from save to visual refresh | Rust file watcher + event-driven viewport updates |
| Telemetry Stream | 120Hz cursor coordinate updates | Tauri v2 Channel pushes i64 nanometer coordinates without discrete IPC calls |

---

## 7. Visual Design Philosophy

### 7.1 Standard Theme (Obsidian)

The default UI follows a premium obsidian aesthetic inspired by Microsoft's 3D Viewer:

- **Background**: obsidian-950 (`#0a0a0f`) — ultra-dark, deep, recessed backdrop
- **Sidebar**: zinc-800 to obsidian-950 gradient — creates a "recessed" panel effect
- **Viewport**: The only illuminated area — makes the 3D model pop
- **Accents**: Blue (`#3b82f6`) for interactive elements, Emerald (`#10b981`) for pass indicators
- **Typography**: Monospace for telemetry, sans-serif for labels
- **Status Bar**: Shows current i64 nanometer coordinate at cursor position (120Hz stream)
- **Tailwind CSS Utilities**: Use `backdrop-blur-md` for ghost-view overlays, `border-zinc-800/50` for thin surgical CAD lines, `bg-obsidian-950/30` for depth layering
- **Kobalte**: Accessible `Tabs` component for tab navigation, `Collapsible` for expandable sidebar sections

### 7.2 PHYSICAL INTEGRITY ALERT Theme (Ghost View)

When a build fails, the UI transitions to a Ghost View theme:

- **Accent Colors**: Deep Red (`#dc2626`) for violations, Amber (`#f59e0b`) for warnings
- **Viewport Overlay**: Translucent glowing wireframes
  - **Yellow Glow**: Floating components (P44 violations)
  - **Red Glow**: Short circuits / disconnected nets (P41 violations)
- **Sidebar**: Red-tinted violation summary with expandable detail list (Kobalte Collapsible)
- **Status Bar**: Red pulse animation with violation count

---

## 8. Updated Component Responsibility Matrix

| Component | Responsibility | Performance |
| :--- | :--- | :--- |
| **Rust Engine** | `.hsx` Zero-Copy Ingestion (`memmap2` + `bytemuck`), embedded GLB/OBJ extraction, GLB augmentation (PourID injection via `gltf`), netlist parsing (`rkyv`), SPICE data extraction, file watching, hot-reload signal, 120Hz telemetry streaming (`tauri::channel`) | Native (C++ Speed) |
| **Rust Data Extractors** | Extract per-viewport data from `.hsx` (3D mesh bytes with PourID, 2D traces, netlist via rkyv, SPICE data, drill coordinates) | Native (Rust) |
 | **@babylonjs/viewer 3D View** | High-level Babylon Viewer renders 3D with professional PBR/IBL/shadows. Real `engine.getFps()`. Background color control. | GPU (60 FPS) |
 | **PixiJS 2D View** | Render 2D traces, pads, silkscreen, board outlines. OffscreenCanvas + Web Worker for non-blocking rendering. | GPU (60 FPS) via Web Worker |
 | **dxf-viewer DXF View** | High-level `dxf-viewer` for industrial DXF. Background color + grid synced with 3D. Temporary direct file loading for testing. | GPU (60 FPS) |
| **uPlot SPICE View** | Waveform/time-series rendering with crosshair tooltips. 10M points in <1ms. | Canvas (CPU/GPU) |
| **SolidJS Shell** | Menu bar, tab routing (Kobalte Tabs), sidebar inspector (Kobalte Collapsible), status bar (120Hz coordinate stream), ghost view overlay | Reactive Web (Low CPU) |
| **The IPC Bridge** | Binary data transfer + lightweight event system + 120Hz Channel streaming | Binary IPC (<1ms) + Channels |

---

## 9. Final Architecture Checklist

1. ✅ **Backend**: Rust as Data Factory — ingests `.hsx` via `memmap2`, augments GLB with PourID via `gltf`, serializes netlists via `rkyv`, parallel mesh generation via `rayon`.
2. ✅ **Frontend 3D**: Babylon.js (`@babylonjs/core` + WebGPU Engine) — full PBR rendering with IBL, shadows, MSAA, native GLB support.
3. ✅ **Frontend 2D**: PixiJS (OffscreenCanvas + Web Worker) — batched WebGL vector rendering for 1M+ trace segments, UI remains responsive.
4. ✅ **Frontend DXF**: Three.js orthographic — DXF vector primitive support.
5. ✅ **Frontend SPICE**: uPlot — fastest time-series renderer for waveform data.
6. ✅ **Frontend Netlist**: SolidJS reactive components + Kobalte Collapsible for hierarchical tree display.
7. ✅ **Picking**: Rust injects `pour_0xAF31` into GLB mesh names via `gltf-json`. Babylon.js returns mesh name. Rust `fxhash` lookup resolves `DeviceBinding`. Only string crosses IPC.
8. ✅ **Telemetry**: Tauri v2 Channels push i64 nanometer coordinates at 120Hz to the status bar.
9. ✅ **Math**: Aligned to the compiler's i64 nanometer fixed-point system for picking/SDF.
10. ✅ **Zero-Copy IO**: `memmap2` for file mapping + `bytemuck` for transmute + `rkyv` for structured data. No parsing, no allocation.
11. ✅ **Hot-Reload**: Rust file watcher detects `.hsx` changes, emits refresh event, all viewports reload data.
12. ✅ **Ghost View**: `integrity_report.glb` ingestion for PHYSICAL INTEGRITY ALERT visualization.
13. ✅ **Performance**: 60 FPS in all viewports, <50ms hot-reload latency, <5ms binary load time, 120Hz telemetry streaming.
14. ✅ **UI Framework**: Tailwind CSS for Obsidian theme (backdrop-blur, border-zinc utilities). Kobalte for accessible tabs and collapsible sidebars.

---

## Conclusion

**One Sentence Definition:**
Hardware Script Monitor is a live visualization and inspection tool that automatically reloads Hardware Script eXchange Binary (`.hsx`) outputs, using a **Hybrid Data Factory** architecture where Rust ingests, augments (injecting PourID via `gltf`), and serves binary data (using `rkyv` for zero-copy structured data and `tauri::channel` for 120Hz telemetry streaming), while professional high-level JavaScript engines (`@babylonjs/viewer` for 3D, `dxf-viewer` for DXF, PixiJS OffscreenCanvas, uPlot) render each viewport with best-in-class visual quality and unified background/grid controls — all inside a Tauri v2 + SolidJS desktop shell with Tailwind Obsidian styling and Kobalte accessible components.

It brings the joy of "hot reload" from web development, the professional visual quality from game engines, the zero-copy performance from database systems, and the real-time preview from CAD straight into the Hardware Script ecosystem — **without requiring custom shader development for every visual feature.**