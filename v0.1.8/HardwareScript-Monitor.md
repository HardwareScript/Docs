# Core Architectural Specification: HardwareScript Monitor (hsm) Companion (v0.1.8)

**Document Type:** Technical Architecture Specification  
**Status:** Approved for Implementation (v0.1.8-alpha)  
**Focus:** Unified Desktop Viewport Companion, Tauri v2 and SolidJS Frontend, Interactive 3D Probing, and Ghost View Visual Debugging

---

## Section 1: The "Visual Browser" Paradigm

Traditional hardware tools require developers to manually open, inspect, and close multiple separate CAD packages just to verify their layouts. The HardwareScript Monitor is designed to serve as the unified, live-refreshing visual companion for your design process.

Under the web-development paradigm, the design loop operates with high speed and zero friction:

*   **The Code Editor:** You write your declarative HardwareScript code in your preferred text editor (such as VS Code), managing pins, netlists, and physical paths.
*   **The Compiler:** When you save, the compiler executes in the background. If compilation succeeds, it serializes your physical, visual, and electrical data into a single, unified binary file called the **Hardware Script eXchange Binary** (with the `.hsx` or `.hwsb` extension).
*   **The Monitor:** The Monitor acts exactly like a web browser. It does not run live compiler processes itself. Instead, a native file watcher in the Monitor’s backend monitors your project directory. The moment the `.hsx` binary updates, the Monitor memory-maps the file and refreshes all viewports instantly. 

This completely eliminates manual file-loading loops, keeping your development feedback loop instantaneous.

---

## Section 2: Unified Desktop Architecture (Tauri v2 + SolidJS)

To maintain a lightweight, standalone footprint of under twenty megabytes while delivering sixty frames per second rendering speeds, the Monitor uses a hybrid, data-parallel desktop architecture.

```
                           [ Tauri v2 Desktop Shell ]
                               /               \
                              /                 \
                             v                   v
                [ Rust Data Factory Backend ]   [ SolidJS Frontend HUD ]
                - memmap2 zero-copy loading     - Kobalte Accessible UI
                - gltf-json PourID injection    - WebGL Viewport Canvas
                - 120Hz coordinate stream       - Interactive Sidebars
```

### The Rust Data Factory (The Backend)
The backend of the Monitor is written in pure Rust and acts as a high-performance data pipeline:

*   **Zero-Copy Ingestion:** It loads the `.hsx` binary using memory-mapping (`memmap2`) and performs direct, pointer-based casts (`bytemuck` and `rkyv`) to extract embedded GLB meshes, DXF geometries, and SPICE netlists in microseconds with zero parsing overhead.
*   **GLB Augmentation:** When extracting the 3D GLB file, the backend programmatically edits the GLTF JSON data in memory. It injects a unique, hex-encoded physical identifier called a **PourID** into the "extras" metadata field of every single mesh node. This enables the frontend to map user clicks on the 3D model directly to logical net names.
*   **High-Frequency Telemetry:** To prevent lag in real-time coordinates, the backend opens a persistent, asynchronous Tauri Channel. It streams your mouse's exact nanometer position in the 3D space directly to the status bar at a locked one-hundred-and-twenty hertz.

### The SolidJS HUD (The Frontend)
The frontend of the Monitor is built on SolidJS, providing a premium, ultra-responsive dark obsidian interface:

*   **No Bloat:** By using SolidJS instead of heavy web frameworks, the application maintains a near-zero memory footprint and runs with near-zero latency.
*   **Surgical CAD Lines:** The interface uses Tailwind CSS to style thin, crisp borders, recessed menus, and modern depth layering.
*   **Accessible Components:** It incorporates Kobalte accessible components to handle tab navigation, collapsible sidebars, and nested dialogs.

---

## Section 3: The Viewport Engines

The Monitor provides several tabbed viewports, each powered by a specialized, professional rendering engine integrated natively into the SolidJS canvas.

### The 3D Viewport: `@babylonjs/viewer`
The 3D board and packages are rendered using a highly customized, stripped-down import of the Babylon.js core engine. 

*   **How it works:** We strip away all pre-built loading screens, default menus, and generic camera controllers.
*   **Visual Quality:** It provides professional, hardware-accelerated Physically Based Rendering, image-based lighting, soft shadows, and multi-sample anti-aliasing.
*   **Performance:** It exposes real-time frame rates and handles the automatic framing of custom component footprints smoothly.

### The 2D Trace Viewport: PixiJS
To visualize dense multi-layer PCB traces and silicon mask lines, the Monitor uses PixiJS.

*   **How it works:** Standard browsers cannot render millions of individual vector lines smoothly in the main thread. We run PixiJS inside a dedicated **Web Worker** using `OffscreenCanvas`.
*   **Performance:** All vector lines, pads, and polygons are batched together and rendered directly in WebGL. This allows you to pan and zoom through millions of trace segments at a locked sixty frames per second without ever lagging the SolidJS user interface.

### The DXF Viewport: `dxf-viewer`
To verify industrial CAD alignments, a dedicated tab integrates a high-level Three.js-based DXF viewer. 

*   **Unified Controls:** The background color and grid toggle are synchronized with the 3D viewport, providing a seamless visual transition when switching views.

### The Waveform Viewport: `uPlot`
To plot simulation results, the Monitor incorporates uPlot, which is the fastest open-source time-series charting library available. It can render up to ten million data points in under one millisecond, enabling you to inspect high-frequency digital transitions and continuous analog waves instantly.

---

## Section 4: The Interactive Simulation and Probing Handshake

By compiling the analog simulator to a headless WebAssembly binary (`ngspice.wasm`) and running it inside a background Web Worker, the Monitor provides an interactive, visual "oscilloscope" environment.

The interactive handshake operates as a lightning-fast, four-step pipeline:

```
User clicks 3D pin in Babylon.js
       │
       ▼
Babylon.js returns mesh name "pour_0xAF31"
       │
       ▼
Monitor Backend runs fxhash lookup on PourID
       │
       ▼
Rust returns DeviceBinding: { device: "M1", net: "VDD" }
       │
       ▼
WASM SPICE Solver calculates VDD waveform
       │
       ▼
uPlot Viewport plots voltage over time in milliseconds
```

1.  **The Click:** The user looks at the 3D board and clicks on a physical component pin or copper trace in the Babylon.js viewport.
2.  **The Mesh Resolution:** Babylon.js's ray-casting engine registers the click and returns the name of the picked mesh node. Because the compiler programmatically named this node (for example, `pour_0xAF31` where `0xAF31` is the hex-encoded PourID), the Monitor immediately has the unique identifier.
3.  **The Fast Index Lookup:** The Monitor's Rust backend receives this identifier over a lightweight IPC command. It runs an instaneous lookup on its pre-calculated, `fxhash`-based `PourMetadata` index. It resolves the PourID to its exact logical **DeviceBinding** (for example, stating: "This physical metal belongs to net VDD and connects to the drain of transistor M1").
4.  **The Interactive Simulation:** SolidJS receives this device binding. It highlights the selected net across all viewports and passes the net name to your background `ngspice.wasm` Web Worker. The WebAssembly solver calculates the non-linear voltages and currents for that specific net and returns a clean numerical array, which is instantly plotted in the Waveform viewport.

This allows you to physically probe your 3D layout to inspect its electrical behavior on the fly, completely avoiding the latency and mess of native FFI bindings.

---

## Section 5: The "Commit Gate" Visual Debugger (Ghost View)

To prevent physically unviable or broken designs from ever reaching manufacturing, the compiler acts as a strict gateway. If a build fails physical verification, the compiler halts the pipeline and outputs a specialized file called `integrity_report.glb`.

The Monitor automatically detects this file and triggers the **PHYSICAL INTEGRITY ALERT** interface:

```
                        [ PHYSICAL INTEGRITY ALERT ]
                     (SolidJS Viewport Overlay Theme)
                                     │
           ┌─────────────────────────┴─────────────────────────┐
           ▼                                                   ▼
  [ Transparent Board Mesh ]                         [ Glowing Wireframes ]
  - Board opacity: 0.05                             - Red Glow: Short Circuits (P41)
  - Components become translucent                   - Yellow Glow: Floating Nodes (P44)
  - Allows you to see internal layers               - Instantly highlights fault coordinate
```

*   **The Visual Theme:** The Monitor’s dark obsidian interface transitions to deep red and amber accents. A translucent, blurred HUD overlay appears on the screen, warning you that physical integrity has been compromised.
*   **The Ghost View Activation:** The 3D viewport automatically loads the `integrity_report.glb` and enters **Ghost View**:
    *   The physical board substrate and component packages are rendered as highly transparent, translucent wireframes (opacity set to five percent). This lets you see directly through the solid structure to inspect internal copper layers.
    *   **Glowing Yellow Wireframes** are overlaid on the 3D model to pinpoint the exact coordinates of floating, unsupported components (violating physical support rules).
    *   **Glowing Red Wireframes** are overlaid on the 3D model to highlight the exact locations of short circuits, overlapping nets, or dielectric breakdown risks.
*   **The Diagnostic Sync:** A collapsible sidebar (using Kobalte Collapsible) displays the compiler's diagnostic report. Clicking a specific error in this list automatically pans the 3D camera and focuses the zoom directly on the glowing red or yellow violation in the layout.

This turns physical debugging into a highly visual, intuitive experience, allowing developers to locate and resolve complex layout and connectivity issues in seconds.

---

## Section 6: Complete Companion Specifications

To ensure the Monitor remains lightweight, fast, and modular, its design is governed by three strict architectural laws:

### Law One: The Silent Atom Law
Geometry rendered in the 3D or 2D viewports is completely inert unless explicitly bound to a device or net via the `device:` property. If a designer places decorative physical shapes that carry no logical bindings, they are treated as non-interactive static assets, preventing them from bloating the simulation or verification pipelines.

### Law Two: The Isolation of Calculations
The Monitor never compiles source code, calculates routing paths, or runs design rule checks itself. All physical, mathematical, and synthesis logic resides exclusively in the high-performance CLI compiler (`hwc`). The Monitor is strictly a high-performance visual browser that displays the results of those calculations.

### Law Three: Platform Independence
The application must compile and run on Windows, Mac, and Linux without requiring the user to install any native C++ compilers, external SPICE software, or visual CAD tools. All libraries (Babylon.js, PixiJS, uPlot, and `ngspice.wasm`) are imported directly via npm or compiled into standalone WebAssembly, ensuring a frictionless, single-binary desktop installation.