# Core Architectural Specification: HardwareScript Analog Simulation Engine (v0.1.8)

**Document Type:** Technical Architecture Specification  
**Status:** Approved for Implementation (v0.1.8-alpha)  
**Focus:** Natively Built Pulse Quantization Architecture, Headless WebAssembly SPICE Integration, and Automatic Parasitic Extraction

---

## Section 1: Evolution of the Analog Simulation Paradigm

Analog circuit verification requires capturing physical truth, including wave propagation, thermal limits, and transistor-level physics. To design our analog engine, we evaluated three major historical bottlenecks in the industry.

### The Traditional SPICE Bottleneck
Standard SPICE simulators use a single, massive global matrix to solve the continuous differential equations governing an entire circuit. 

To simulate a transient waveform accurately, the simulator must slice time into tiny femtosecond steps and solve non-linear equations iteratively at every step using numerical solvers like the Newton-Raphson method. For circuits with millions of transistors, this continuous matrix math is highly performant but exceptionally slow, requiring server clusters to compute over several days.

### The FFI C-Binding Nightmare
To automate this, early design plans considered compiling a standard SPICE engine like ngspice as a shared library and linking it to the compiler backend using Rust Foreign Function Interface (FFI) bindings. 

However, cross-platform C-bindings are notoriously fragile. They require users to have platform-specific compilers (such as MSVC on Windows and GCC on Linux) installed on their local machines, causing target-architecture mismatches and making open-source compilation brittle and messy.

### The Pure JavaScript Performance Wall
We also evaluated using a pure JavaScript or TypeScript SPICE engine from npm. 

While highly modular, solving non-linear simultaneous equations in a single-threaded, garbage-collected language like JavaScript introduces severe CPU stalls. A circuit with more than a few dozen transistors will instantly freeze the user interface. Furthermore, these pure-JS engines only support simplified, ideal Level 1 transistor models, making them inaccurate for foundry-grade silicon design.

---

## Section 2: The Two-Tiered Analog Simulation Strategy (PQA + WASM Bridge)

To bypass the FFI nightmare and the JavaScript performance wall, HardwareScript implements a two-tiered analog simulation strategy: the native Pulse Quantization Architecture and a headless WebAssembly SPICE bridge.

```
                           [ ANALOG SIMULATION ENGINE ]
                                       │
            ┌──────────────────────────┴──────────────────────────┐
            ▼                                                     ▼
  [ Tier 1: Native Pulse Engine ]                       [ Tier 2: WASM SPICE Bridge ]
  - Quantizes Time into Ticks                           - Headless C-to-WASM (ngspice.wasm)
  - Models physics as delays                            - Runs in browser Web Worker
  - Used for 95% of daily design loops                  - Used for 5% foundry-grade verification
```

### Tier 1: Native Pulse Quantization Architecture (PQA)
The native, built-from-scratch physics engine in HardwareScript is called the **Pulse Quantization Architecture**, or PQA. 

Just as the vector database quantizes physical space, the Pulse engine quantizes temporal change using discrete "Ticks" representing propagation delays. PQA operates asynchronously and is designed to handle ninety-five percent of a designer's daily timing, delay, and thermal iterations:

*   **Electrical Delay:** Resistance is modeled as inertia. The engine calculates how many ticks a signal takes to cross a trace based on the material's resistivity.
*   **Thermal Diffusion:** Thermal conductivity is modeled as diffusion latency. Heat is treated as a slow pulse propagating through adjacent voxels at a rate defined by the material's thermal properties.
*   **Photonic Phase:** Refractive index is modeled as phase lag, allowing the engine to calculate optical interference patterns based on physical channel lengths.

### Tier 2: Headless WebAssembly SPICE (ngspice.wasm)
For the remaining five percent of design work—where foundry-certified, non-linear transistor-level verification is required—we bypass FFI bindings by compiling the C-source code of ngspice into a WebAssembly (WASM) binary.

*   **How it works:** WebAssembly compiles directly to platform-agnostic bytecode that executes at near-native CPU speeds. It runs natively inside your browser or desktop environment, completely eliminating the need for local C++ compilers or operating system dependencies.
*   **WASM Web Workers:** The simulation solver runs entirely inside a standard browser Web Worker on the client side. This keeps your user interface responsive and running at sixty frames per second during long transient analyses.

---

## Section 3: Native Parasitic Extraction (The Compiler's Job)

In traditional EDA, calculating the "accidental" resistance and capacitance of physical traces requires running heavy, slow 3D numerical field solvers that mesh the entire space. HardwareScript completely replaces this manual extraction phase.

During physical verification, the compiler executes an automated **Analytical Boundary Element Method (BEM) Solver** in milliseconds:

1.  **Coordinate Extraction:** The compiler retrieves the pristine, continuous vector coordinates of your routed traces and components from the Entity Graph.
2.  **Effective Permittivity:** It calculates the effective relative permittivity accounting for the electric fields passing through both the substrate and the air using Wheeler's closed-form equation:
    $$\epsilon_{\text{eff}} = \frac{\epsilon_r + 1}{2} + \frac{\epsilon_r - 1}{2} \left( 1 + 12\frac{H}{W} \right)^{-1/2}$$
3.  **Capacitance and Resistance Extraction:** It evaluates the ground-plane-aware coupling capacitance, ground capacitance, and series trace resistance using Sakurai's empirical microstrip formulas on the raw vector segments:
    $$C_{12} = \epsilon_0 \epsilon_{\text{eff}} L \left[ 0.03\left(\frac{W}{H}\right) + 0.08\left(\frac{T}{H}\right) + 0.07\left(\frac{W}{H}\right)^{0.25}\left(\frac{T}{H}\right)^{0.5}\left(\frac{H}{D}\right)^{1.34} \right]$$
4.  **Via and Mutual Inductance:** It computes localized via self-inductance and parallel-run mutual inductance using the Greenhouse approximation.
5.  **SPICE Annotation:** The compiler automatically outputs a fully annotated, parasitic-ready SPICE netlist (with the `.sp` or `.cir` extension) containing all your transistors, plus these calculated resistors, capacitors, and coupled inductors.

---

## Section 4: Cross-Context Workflow (CLI, LS, Monitor)

To align with the "VS Code and Web Browser" paradigm, the analog simulation engine is designed to operate seamlessly across different execution environments.

### Context One: The CLI Compiler (`hwc`)
In headless environments (such as remote servers, automated terminal runs, or CI/CD test runners), the CLI compiler handles the heavy lifting:

*   **The Command:** Running `hwc build --physical` triggers the automatic BEM extractor.
*   **The Execution:** The compiler extracts the parasitics, generates the fully annotated SPICE netlist, and—if requested—executes a background parallel simulation using an installed instance of Xyce or ngspice.
*   **The Output:** It compiles the physical simulation results directly into the unified `.hwsb` binary file, ready for verification.

### Context Two: The Hardware Script Monitor (`hsm`)
The Monitor acts as the graphical, interactive simulation interface. It houses the embedded WebAssembly SPICE engine:

*   **Headless Assembly:** We compile ngspice using its shared-library configuration (`--with-ngshared`). This strips away the command-line interface, the old plotting window, and all original UI assets, producing a pure mathematical engine (`ngspice.wasm`).
*   **SolidJS User Interface:** Because the solver is completely headless, you write one hundred percent of the visual presentation yourself in SolidJS.
*   **Interactive Probing (The Handshake):**
    *   The user looks at the 3D board in the Babylon.js viewport.
    *   They click on a specific trace or pin.
    *   Because the physical GLB mesh nodes are programmatically named after their corresponding net IDs in the SPICE file, the Monitor instantly knows which net was clicked.
    *   The Monitor passes this net ID to the background `ngspice.wasm` Web Worker.
    *   The solver calculates the continuous voltage and current curves and returns a clean array of numbers.
    *   The Monitor plots these waveforms natively in its SPICE/Waveform tab using a fast, hardware-accelerated charting library (like uPlot), styled to match your dark obsidian theme.

---

## Section 5: Summary of the Simulation Stack

By combining the native Pulse engine with a headless WebAssembly SPICE bridge, HardwareScript delivers a highly performant and modular analog verification ecosystem:

*   **For Daily Iteration:** You use the native **Pulse Quantization Architecture** (PQA) for instant timing, signal delay, and thermal checks directly inside the compiler.
*   **For Foundry Verification:** The compiler automatically extracts parasitic resistances and capacitances in milliseconds, generates a standard SPICE netlist, and lets you run it natively inside the Monitor's WebAssembly solver with interactive, visual 3D probing.
*   **Zero FFI Overhead:** Your application remains entirely modular. It compiles with zero native C-dependencies or cross-platform compilation issues, providing a responsive environment that can scale to industrial-grade hardware design.