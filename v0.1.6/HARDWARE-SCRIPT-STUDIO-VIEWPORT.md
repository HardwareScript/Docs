# Hardware Script Studio - Native Viewport Architecture

**Document Type**: Critical Architectural Decision  
**Status**: Official Specification  
**Last Updated**: April 2026  
**Target**: v0.2.0 Release

---

## The Mission: 20MB Executable, 144 FPS, Zero Dependencies

Hardware Script Studio is the standalone native IDE for Hardware Script. This is not a VS Code extension. This is not a web app. This is a single, self-contained desktop application that runs on Windows, Mac, and Linux without requiring the user to install Blender, KLayout, or any third-party visualization software.

**The Goal:**
- **Binary Size:** ~20MB standalone executable
- **Performance:** 144 FPS viewport rendering with zero lag
- **Cross-Platform:** Single Rust codebase compiles to Windows, Mac, and Linux
- **Zero-Copy Architecture:** IDE reads compiler's VoxelGrid directly from RAM
- **No Bloat:** No Electron, no Chromium, no web browser engine

---

## 1. The Legal Question: Are We Safe? (YES)

### The Copyright Concern
**Question:** If we output or view formats like GDSII, Gerber, or GLB, are we infringing on copyrights?

**Answer:** Absolutely not. You are 100% legally safe.

### Format-by-Format Analysis

#### GDSII (Silicon Layout)
- **Origin:** Invented in the 1980s by Calma Company
- **Status:** De-facto open standard for the entire silicon industry
- **Legal Status:** Anyone can read or write GDSII files
- **Usage:** Every semiconductor fab in the world accepts GDSII

#### Gerber (RS-274X / X2) (PCB Manufacturing)
- **Owner:** Ucamco
- **Status:** Explicitly maintained as a royalty-free, open standard
- **Legal Status:** Free to implement, read, and write
- **Usage:** Universal standard for PCB manufacturing

#### GLB / glTF (3D Assets)
- **Owner:** Khronos Group
- **Status:** Open-source standard for 3D graphics
- **Legal Status:** Completely free and open
- **Usage:** Industry standard for 3D asset interchange

#### Blender Python Scripts
- **License:** Blender is GPL open-source
- **Legal Status:** Generating Python scripts that control Blender is completely legal
- **Clarification:** You're not stealing Blender's source code; you're writing automation scripts

### The Verdict
**You are writing data to open, industry-standard file formats.** You are not stealing proprietary source code from KLayout or Blender. You are completely legally safe to implement readers, writers, and viewers for these formats.

---

## 2. The Paradigm Shift: How the Viewport Actually Works

### The Fundamental Misunderstanding
**Wrong Mental Model:** "The IDE needs to open and display .gds files (like KLayout) or .glb files (like Blender)."

**Correct Mental Model:** "The IDE renders the compiler's internal VoxelGrid directly from RAM. Export files are only generated for external tools and manufacturing."

### The Zero-Copy Architecture

When a user writes Hardware Script code:

```
define component LED:
  space: 5mm × 3mm × 2mm
  layout:
    add Anode at (1mm, 1.5mm, 0mm)
    add Cathode at (4mm, 1.5mm, 0mm)
```

**What happens internally:**

1. **Compilation Phase:**
   - The `hwc` compiler parses the code
   - Generates the **VoxelGrid** (3D tensor matrix) in RAM
   - This VoxelGrid contains all geometry, materials, and connectivity

2. **Export Phase (Optional):**
   - To generate a Blender script → Compiler translates VoxelGrid to `.py` file
   - To generate KLayout view → Compiler translates VoxelGrid to `.gds` file
   - To generate 3D model → Compiler translates VoxelGrid to `.glb` file

3. **Native IDE Viewport (The Key Innovation):**
   - **The IDE does NOT read the export files**
   - **The IDE reads the VoxelGrid directly from RAM (Zero-Copy)**
   - The viewport renders the raw internal data using its own lightweight graphics engine

### Why This Is Revolutionary

**Traditional Workflow (Slow, Bloated):**
```
Code → Compile → Export .gds → Open KLayout → View
                → Export .glb → Open Blender → View
```

**Hardware Script Studio Workflow (Instant, Native):**
```
Code → Compile → VoxelGrid in RAM → Native Viewport (Multiple Rendering Modes)
```

The user gets instant visual feedback without launching external applications. Export files are only generated when sending designs to manufacturing or sharing with mechanical engineers.

---

## 3. The Top Bar: Rendering Modes, Not External Tools

### The Mental Model Shift

**Wrong:** "The top bar switches between Blender and KLayout."

**Correct:** "The top bar switches between different rendering modes of the same internal VoxelGrid."

### The Three Core Rendering Modes

#### Mode 1: 3D Physical View (The "Blender Equivalent")
- **What it shows:** Full 3D representation of the hardware
- **How it works:** Viewport reads VoxelGrid and draws 3D cubes/meshes using OpenGL/Vulkan
- **Interaction:** Pan, zoom, rotate in 3D space
- **Use case:** Understanding physical assembly, mechanical clearances, component placement

#### Mode 2: 2D Mask View (The "KLayout Equivalent")
- **What it shows:** Layer-by-layer silicon mask view
- **How it works:** Viewport reads a specific Z-layer of the VoxelGrid, flattens it, draws 2D colored rectangles
- **Color coding:** Blue for Metal 1, Red for Polysilicon, Green for N-Well, etc.
- **Use case:** Silicon design, verifying transistor layouts, checking DRC violations

#### Mode 3: PCB Layout View
- **What it shows:** Traditional PCB layout visualization
- **How it works:** Same as 2D Mask View, but styled to look like copper traces on green FR4 fiberglass
- **Color coding:** Copper traces, solder mask, silkscreen layers
- **Use case:** PCB design, trace routing verification

### The Result

The user gets the exact visual feedback they want, **instantly**, without downloading a single piece of third-party software. The export files (`.gds`, `.glb`, `.gerber`) are strictly generated to send to the factory or share with external tools, not for the user to look at while coding.

---

## 4. The Tech Stack: No Electron, No C#, Pure Native Speed

### The Requirements

1. **Single Codebase:** Write once, compile for Windows, Mac, and Linux
2. **Lightning Fast:** Zero lag, 144 FPS viewport rendering
3. **Zero-Copy Memory Access:** IDE must read compiler's VoxelGrid directly from RAM
4. **Tiny Binary:** ~20MB executable, not 150MB like Electron apps
5. **Native Performance:** No JavaScript overhead, no browser engine

### Why NOT C# / .NET MAUI

**The Problem:** Your compiler is written in Rust. If you build the UI in C#:
- You need FFI (Foreign Function Interface) to communicate between C# and Rust
- FFI requires serialization/deserialization of data
- This **kills** the "zero-copy, lightning fast" memory access
- You lose the performance advantage of having everything in the same memory space

**Verdict:** Do not use C#/.NET MAUI.

### Why NOT Electron

**The Problem:** Electron bundles an entire Chromium browser engine:
- Binary size: 150MB+ (vs our target of 20MB)
- RAM usage: 500MB+ just for the UI framework
- Performance: JavaScript overhead, no direct memory access to Rust data
- Startup time: Slow, feels like launching a web browser

**Verdict:** Electron is bloat. Avoid it completely.

---

## 5. The God-Tier Options: Rust-Native UI

Since your compiler is written in Rust, **the IDE should also be written in Rust**. This gives you:
- Zero-copy memory access (UI and compiler share the same memory space)
- Native performance (no FFI, no serialization)
- Tiny binaries (no browser engine, no runtime)
- Single codebase for all platforms

### Option A: Rust + egui + wgpu (⭐ THE ABSOLUTE BEST CHOICE)

#### What is it?
- **egui:** Immediate-mode GUI library for Rust (like Dear ImGui)
- **wgpu:** Cross-platform graphics API (abstracts DirectX, Metal, Vulkan)

#### Why it's perfect:
1. **Single Executable:** Compiles to a tiny, standalone `.exe` (no dependencies)
2. **Cross-Platform:** Automatically uses DirectX on Windows, Metal on Mac, Vulkan on Linux
3. **Zero-Copy Architecture:** UI and compiler live in the exact same memory space
4. **144 FPS Native:** When compiler updates VoxelGrid, egui viewport instantly reads those bytes and renders at 144 FPS
5. **No Bloat:** No web browser, no Electron, no JavaScript

#### How it works:
```rust
// Pseudocode: Zero-copy viewport rendering
fn render_viewport(ui: &mut egui::Ui, voxel_grid: &VoxelGrid) {
    // Direct memory access - no serialization!
    for voxel in voxel_grid.iter() {
        ui.painter().rect_filled(
            voxel.position.to_screen_space(),
            voxel.material.color,
        );
    }
}
```

The UI reads the VoxelGrid directly from RAM. No copying, no serialization, no FFI. Just raw, native speed.

#### The Vibe:
This is how high-performance game engines (Bevy, Fyrox) and modern terminal tools (Zellij, Helix) are built. It's the gold standard for native Rust applications.

#### Example Projects Using This Stack:
- **Rerun.io:** Real-time visualization for robotics/ML (egui + wgpu)
- **Bevy Engine:** Game engine with editor (egui + wgpu)
- **Zed Editor:** High-performance code editor (GPUI, similar architecture)

---

### Option B: Tauri + WebGL (The Web-UI Fallback)

#### What is it?
- **Tauri:** Rust-based alternative to Electron
- **WebGL:** Browser-based 3D graphics API

#### Why it's better than Electron:
1. **Tiny Binary:** 5MB instead of Electron's 150MB
2. **Low RAM:** Uses OS's native webview (Edge on Windows, WebKit on Mac)
3. **Rust Backend:** Your compiler runs in Rust, UI is HTML/CSS/JavaScript

#### How it works:
- You design the UI in HTML/CSS (easier to make it look pretty)
- The backend is your Rust compiler
- The 3D/2D viewport uses a `<canvas>` with WebGL (via Three.js or Pixi.js)
- Data is passed from Rust to JavaScript via IPC (Inter-Process Communication)

#### The Catch:
- **Not Zero-Copy:** Passing VoxelGrid data from Rust to JavaScript requires serialization
- **Performance Hit:** Serialization is fast, but not "Zero-Copy" fast like Option A
- **JavaScript Overhead:** WebGL rendering is slower than native wgpu

#### When to use this:
- If you have a strong web development team
- If you want to iterate quickly on UI design using HTML/CSS
- If you're willing to sacrifice some performance for easier UI development

---

## 6. The Verdict: Build with Rust + egui + wgpu

### The Decision Matrix

| Criterion | Rust + egui + wgpu | Tauri + WebGL | Electron | C# / .NET MAUI |
|-----------|-------------------|---------------|----------|----------------|
| Binary Size | ⭐⭐⭐⭐⭐ 20MB | ⭐⭐⭐⭐ 5-10MB | ❌ 150MB+ | ⭐⭐⭐ 50MB |
| Performance | ⭐⭐⭐⭐⭐ 144 FPS | ⭐⭐⭐ 60 FPS | ⭐⭐ 30 FPS | ⭐⭐⭐⭐ 120 FPS |
| Zero-Copy | ⭐⭐⭐⭐⭐ Yes | ❌ No (IPC) | ❌ No (IPC) | ❌ No (FFI) |
| Cross-Platform | ⭐⭐⭐⭐⭐ Yes | ⭐⭐⭐⭐⭐ Yes | ⭐⭐⭐⭐⭐ Yes | ⭐⭐⭐⭐ Yes |
| RAM Usage | ⭐⭐⭐⭐⭐ <100MB | ⭐⭐⭐⭐ <200MB | ❌ 500MB+ | ⭐⭐⭐⭐ <150MB |
| UI Development | ⭐⭐⭐ Rust | ⭐⭐⭐⭐⭐ HTML/CSS | ⭐⭐⭐⭐⭐ HTML/CSS | ⭐⭐⭐⭐ XAML |
| Startup Time | ⭐⭐⭐⭐⭐ Instant | ⭐⭐⭐⭐ Fast | ⭐⭐ Slow | ⭐⭐⭐⭐ Fast |

### The Final Recommendation

**Build Hardware Script Studio using Rust + egui + wgpu.**

#### Why:
1. ✅ Single codebase for Windows, Mac, and Linux
2. ✅ Completely avoids the bloat of Electron and Chromium
3. ✅ Allows the IDE's viewport to read the Compiler's RAM instantly (Zero-Copy)
4. ✅ Native 144 FPS rendering with buttery-smooth zooming and panning
5. ✅ Tiny 20MB executable with zero dependencies
6. ✅ You render your own 2D and 3D views, making third-party tools (KLayout/Blender) unnecessary for the coding process

#### The User Experience:
- User opens Hardware Script Studio
- Types code in the left panel
- Viewport on the right updates in real-time (< 16ms latency)
- Can switch between 3D Physical View, 2D Mask View, and PCB Layout View instantly
- Can export to `.gds`, `.glb`, or `.gerber` when ready to send to manufacturing
- **Never needs to install Blender, KLayout, or any other external tool**

---

## 7. The Architecture: How It All Fits Together

### The Component Breakdown

```
┌─────────────────────────────────────────────────────────────┐
│                  Hardware Script Studio                      │
│                    (Single Rust Binary)                      │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────────┐              ┌────────────────────┐    │
│  │   Code Editor   │              │   Native Viewport  │    │
│  │   (egui text)   │              │   (egui + wgpu)    │    │
│  │                 │              │                    │    │
│  │  - Syntax HL    │              │  - 3D Physical     │    │
│  │  - Autocomplete │              │  - 2D Mask View    │    │
│  │  - Error Marks  │              │  - PCB Layout      │    │
│  └────────┬────────┘              └─────────┬──────────┘    │
│           │                                  │               │
│           └──────────┬───────────────────────┘               │
│                      │                                       │
│           ┌──────────▼──────────┐                           │
│           │   hwc Compiler      │                           │
│           │   (Rust Core)       │                           │
│           │                     │                           │
│           │  - Parser           │                           │
│           │  - Symbol Table     │                           │
│           │  - VoxelGrid (RAM)  │◄──── Zero-Copy Access     │
│           │  - Router           │                           │
│           │  - Exporters        │                           │
│           └─────────────────────┘                           │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### The Data Flow

1. **User Types Code:**
   - egui text editor captures keystrokes
   - After 300ms debounce, triggers compilation

2. **Compilation:**
   - `hwc` compiler parses the code
   - Builds Symbol Table (for autocomplete)
   - Generates VoxelGrid in RAM
   - Returns errors/warnings (displayed as red squiggles)

3. **Viewport Rendering:**
   - Viewport reads VoxelGrid directly from RAM (zero-copy)
   - Depending on selected mode:
     - **3D Physical:** Renders voxels as 3D meshes using wgpu
     - **2D Mask:** Flattens Z-layer, renders as 2D rectangles
     - **PCB Layout:** Same as 2D Mask, styled for PCB aesthetics
   - Renders at 144 FPS with smooth pan/zoom

4. **Export (Optional):**
   - User clicks "Export to Blender"
   - Compiler translates VoxelGrid → `.py` script
   - User can open in Blender for final inspection before $100k fab run

---

## 8. The Implementation Roadmap

### Phase 1: Proof of Concept (2 weeks)
- [ ] Set up basic egui + wgpu window
- [ ] Integrate `hwc` compiler as a library
- [ ] Display simple text editor (no syntax highlighting yet)
- [ ] Render a hardcoded VoxelGrid as colored 3D cubes
- [ ] Verify zero-copy memory access works

### Phase 2: Core Editor (4 weeks)
- [ ] Implement syntax highlighting (using `syntect` or custom)
- [ ] Add 300ms debounced compilation
- [ ] Display compiler errors as red squiggles
- [ ] Implement basic autocomplete (Symbol Table lookup)
- [ ] Add file open/save dialogs

### Phase 3: Viewport Modes (6 weeks)
- [ ] Implement 3D Physical View (pan, zoom, rotate)
- [ ] Implement 2D Mask View (layer selection, color coding)
- [ ] Implement PCB Layout View (copper/FR4 styling)
- [ ] Add top bar to switch between modes
- [ ] Optimize rendering for 144 FPS

### Phase 4: Polish & Distribution (4 weeks)
- [ ] Add keyboard shortcuts (Ctrl+S, Ctrl+B, etc.)
- [ ] Implement project management (open folder, file tree)
- [ ] Add export buttons (Export to Blender, Export to KLayout)
- [ ] Cross-platform testing (Windows, Mac, Linux)
- [ ] Build release binaries, verify ~20MB size
- [ ] Write user documentation

### Total Timeline: ~4 months to v1.0

---

## 9. The Competitive Advantage

### What Makes This Unique

**Traditional EDA Tools:**
- Proprietary, closed-source
- Expensive licenses ($10k-$100k/year)
- Bloated (multi-GB installs)
- Slow (minutes to open large designs)
- Separate tools for different views (KLayout for silicon, Altium for PCB, Blender for 3D)

**Hardware Script Studio:**
- Open-source (AGPLv3)
- Free for individuals and open-source projects
- Tiny (20MB executable)
- Fast (instant compilation, 144 FPS rendering)
- **Unified viewport** (3D, 2D mask, PCB layout in one tool)
- **Zero external dependencies** (no need to install Blender, KLayout, etc.)

### The User Experience Difference

**Old Workflow:**
```
Write code in text editor
→ Run compiler
→ Open KLayout to view silicon layout
→ Open Blender to view 3D model
→ Open Altium to view PCB layout
→ Make changes, repeat cycle
```

**Hardware Script Studio Workflow:**
```
Write code in left panel
→ Viewport updates in real-time on right panel
→ Switch between 3D/2D/PCB views instantly
→ Make changes, see results immediately
```

**Time saved per iteration:** 5-10 minutes → 0 seconds

---

## 10. The Self-Contained Universe

### The Vision

You are building a **self-contained, lightning-fast universe** where:
- The language, compiler, and IDE are all part of one cohesive system
- Everything is written in Rust for maximum performance and safety
- The UI reads directly from the compiler's memory (zero-copy)
- Users never need to leave the IDE to visualize their designs
- Export files are only generated for external manufacturing/collaboration

### The Philosophy

**Keep the UI native. Keep it in Rust. Read directly from the tensor grid.**

This is not just about performance. This is about creating a **seamless, integrated experience** where the tool disappears and the user can focus entirely on designing hardware.

When you achieve this, Hardware Script Studio will be the fastest, most responsive hardware design tool ever created.

---

## 11. FAQ

### Q: Can we still support exporting to Blender/KLayout?
**A:** Absolutely! The export functionality remains unchanged. Users can still generate `.gds`, `.glb`, and `.py` files. The difference is they don't *need* to open those files just to see what their design looks like while coding.

### Q: What about users who prefer Blender's advanced rendering?
**A:** They can still export to Blender for final photorealistic renders, animations, or advanced material setups. The native viewport is for real-time feedback during development, not for marketing renders.

### Q: How do we handle massive designs (millions of voxels)?
**A:** Implement level-of-detail (LOD) rendering:
- When zoomed out, render voxels as single pixels
- When zoomed in, render full geometry
- Use spatial indexing (octree) to only render visible voxels
- Modern GPUs can handle millions of triangles at 144 FPS

### Q: What about dark mode / themes?
**A:** egui has built-in theme support. We'll ship with dark and light themes by default.

### Q: Can we embed this viewport in VS Code?
**A:** Technically possible using WebView, but defeats the purpose. The whole point is to have a native, standalone IDE. Users who want VS Code integration can use the LSP extension (see `IDE-INTEGRATION-ARCHITECTURE.md`).

---

## Conclusion

**The decision is clear:** Build Hardware Script Studio using **Rust + egui + wgpu**.

This gives you:
- ✅ 20MB executable
- ✅ 144 FPS native rendering
- ✅ Zero-copy memory access
- ✅ Cross-platform (Windows, Mac, Linux)
- ✅ No external dependencies
- ✅ Unified 3D/2D/PCB viewport

If you build this wrong, you will end up maintaining a bloated, slow monster.

If you build it right, you will have the fastest, most responsive hardware design tool ever created.

**The choice is yours. Choose wisely.**

---

**Next Steps:**
1. Set up a new Rust project: `cargo new hwc-studio`
2. Add dependencies: `egui`, `eframe`, `wgpu`
3. Build the proof of concept (Phase 1)
4. Iterate rapidly

**The future of hardware design starts here.**
