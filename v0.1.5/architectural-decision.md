This is the most critical architectural decision you will make regarding the scope of Hardware Script. You are facing the classic "Scope Creep vs. Value Proposition" dilemma. 

Looking at your Day 1 Whitepaper, your original vision (Engine B) was highly focused on simulation and Blender visualization. However, now that you have built a 5-layer MLIR compiler with logic synthesis and constraint-driven Manhattan routing, the project has matured.

Here is exactly where you should draw the line to maintain your "blazing fast, lightweight" ideology, without losing the massive value proposition.

---

### 1. 3D Asset Mapping: KEEP IT (But strictly as static `.glb` files)
**The Verdict:** You *must* keep 3D asset mapping, but you should not overcomplicate it.

**Why it is essential:** 
Hardware does not exist in a vacuum; it goes inside a plastic or metal enclosure. Mechanical Engineers (using SolidWorks or AutoCAD) need a 3D model of your finished motherboard to ensure the capacitors don't physically collide with the plastic casing. If Hardware Script cannot export a 3D model of the board, it cannot compete with KiCad or Altium.

**Why it is actually very easy to build in Rust:**
Since you are planning a Native Rust IDE using `wgpu` or `egui`, rendering 3D is trivial. 
*   You don't need to invent a 3D engine. 
*   In the component's package, the creator simply includes a standard `.glb` (GLTF binary) file. 
*   In the `.hw` file, they map it: `render: "assets/chip.glb"`.
*   Because your compiler already calculates the exact `[X, Y, Z]` anchor point of every component, the IDE just loads the `.glb` and sets its translation matrix to `[X, Y, Z]`. 
*   The copper traces are just drawn as flat 3D rectangles based on your Voxel Grid.

**Where to draw the line:** The compiler does *not* do 3D mesh collisions. The compiler only does *voxel* collisions. The `.glb` file is strictly for the visual viewport and mechanical export (`.step` or `.obj`). It is a "dumb" visual wrapper.

---

### 2. Electron Flow / Animation: CUT IT (Delegate to external tools)
**The Verdict:** You must **ruthlessly cut** native electron flow animation from the core compiler and the native IDE. 

**Why it is a trap:**
1.  **Computational Nightmare:** Calculating visual propagation delay and rendering moving particles across 2 million voxels at 60 Frames Per Second will turn your lightning-fast compiler into a sluggish video game engine.
2.  **Professional Irrelevance:** Professional hardware engineers do not debug circuits by watching animations of electrons. They look at **Waveform Graphs** (Timing diagrams). 
3.  **The Domain Boundary Rule:** As established in your documentation, Hardware Script is the master of the *stationary medium*. You define the copper and the logic. You do not simulate time-domain physics visually.

**How you solve this without losing the feature (The "Unix Philosophy"):**
You do not kill the idea; you just move it out of the compiler. You use your **Export Targets** (Layer 5 of your architecture).

If a student or a YouTuber wants to see electrons flowing through the board for a cool video or tutorial:
*   They run: `hwc build main.hw --target blender`
*   The compiler spits out a Python script (`sim.py`).
*   They open Blender, run the script, and *Blender* does the heavy lifting of animating the electrons flowing through the 3D meshes. 
*   **Your compiler remains 5MB and finishes in sub-milliseconds.** You let a 2GB software like Blender handle the movie-making.

---

### 3. Electrical Simulation (SPICE): DELEGATE IT
Similarly, if an engineer wants to simulate the voltage drops, logic states, or analog frequencies:
*   They run: `hwc build main.hw --target spice`
*   Hardware Script translates the `logic:` blocks and routing into an industry-standard SPICE Netlist.
*   They open it in LTspice or Ngspice to view the waveform graphs.

---

### The New Architectural Boundary (What to document)

To close this loop, you should update your **"The Domain Boundary"** documentation with this exact philosophy. It protects your team from infinite feature requests.

> #### The Visualization Boundary
> Hardware Script guarantees absolute spatial and logical reality. It does not natively render time-domain animations.
> 
> **1. Static 3D Reality (Included):** 
> The compiler calculates exact `[X, Y, Z]` coordinates and supports attaching industry-standard `.glb` meshes to components. This guarantees seamless handoffs to Mechanical CAD engineers (SolidWorks, AutoCAD) and provides instant photorealistic rendering in the Hardware Script IDE.
> 
> **2. Time-Domain Animation (Delegated):** 
> Simulating the visual flow of electrons or generating waveform timing graphs requires heavy, stateful, time-stepping game engines. To keep the compiler blazing fast, Hardware Script delegates this. The compiler exports to targets like `--target blender` (for visual animation scripts) or `--target spice` (for professional electrical waveform simulation). **We compile reality; we leave the movie-making to the rendering engines.**

### Summary

By drawing the line here, you achieve the massive value proposition of Day 1 (people can still get their Blender animations and 3D models), but your core Rust compiler stays incredibly lean, focused solely on parsing text, synthesizing logic, routing voxels, and exporting files.