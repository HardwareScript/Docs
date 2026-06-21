# Core Architectural Specification: HardwareScript Digital Simulation Engine (v0.1.8)

**Document Type:** Technical Architecture Specification  
**Status:** Approved for Implementation (v0.1.8-alpha)  
**Focus:** Natively Built Digital Simulation Engine, Cycle-Based SIMD Vectorization, and Cross-Context Operations  

---

## Section 1: Evolution of the Simulation Paradigm (Before vs. Now)

To design a high-performance digital logic simulator, we had to re-evaluate our original design assumptions and compare them against legacy Electronic Design Automation (EDA) systems.

### What Was Originally Planned

In early design drafts, we planned to implement a real-time, event-driven digital simulator that synchronized the simulated clock directly with the host CPU's physical wall clock (using high-precision hardware timers and the host's quartz crystal heartbeat). 

The goal was to map simulated time directly to real-world time to make timing calculations intuitive.

### Why We Rejected This Approach

While real-time wall-clock synchronization is highly valuable for Hardware-in-the-Loop (HIL) environments—where a simulation must interact with physical components like microcontrollers or test equipment—it introduces severe bottlenecks for local software development:

*   **The Simulation Speed Trap:** If a designer is simulating a ten-megahertz clock, forcing it to sync to a physical clock means the simulation cannot execute faster than real-time. Conversely, if simulating a complex four-gigahertz processor, a single simulated clock cycle requires executing thousands of host instructions. The physical CPU cannot maintain a real-time lock, causing the simulation to lag behind.
*   **The Pointer-Chasing Bottle-neck:** Standard digital simulators use single-threaded, event-driven engines. They jump from pointer to pointer in memory to evaluate logic gates one-by-one. This destroys the CPU’s L1 and L2 cache locality, making simulation times scale poorly on modern multi-core processors.

### The New v0.1.8 Architecture: SIMD Bit-Parallel Cycle-Based Emulation

We have replaced the real-time event-driven model with an asynchronous, cycle-based, vectorized emulation engine. 

Instead of evaluating gates one-by-one, the compiler compiles your logical module netlists into flat, contiguous arrays of identical standard logic cells (gates). The engine then leverages the host CPU's vector registers (using AVX-512 or ARM NEON instructions) to evaluate the logic of hundreds of gates simultaneously in single register sweeps. 

Time is quantized into discrete "Ticks" representing propagation delays, completely decoupling simulation speed from the host's physical clock while ensuring that verification runs at maximum hardware-accelerated speeds.

---

## Section 2: Core Algorithmic Engine

The digital simulation engine operates in the Netlist Arena, performing logical and functional verification of your design before physical geometry is stamped into the voxel grid.

### Cycle-Based Execution
Rather than calculating gate outputs at every infinitesimal step, the engine evaluates the circuit state at discrete clock edges (cycle-based simulation). This matches how real digital circuits operate. 

Within each cycle, the execution engine follows a strict multi-step sequence:

1.  **Topological Graph Sorting:** On compilation, the compiler parses your module's logical connections into a Directed Acyclic Graph, or DAG. It performs a topological sort on this graph to establish a stable, deterministic evaluation order from inputs to outputs, ensuring there are no combinational loops or race conditions.
2.  **Vectorized Gate Evaluation:** Gates of the same type (such as AND, OR, or XOR gates) are grouped into contiguous, cache-aligned memory blocks.
3.  **SIMD Register Packing:** The engine packs the state values of multiple independent gates into wide SIMD registers.
4.  **Bit-Parallel Computation:** The CPU executes a single bitwise instruction to calculate the logical results for all packed gates simultaneously.
5.  **State Promotion:** Calculated wire values are promoted to register state variables (using the next property) on the simulated clock edge, moving the system state forward by one tick.

### The Lowercase Register Primitive
To eliminate the blocking and non-blocking race conditions that plague traditional Verilog designs, HardwareScript strictly separates instantaneous wire calculations from clock-driven register updates. 

Wires use standard equals assignments, indicating immediate propagation. Registers are defined using the lowercase `reg` primitive and updated using the `next` property, ensuring that state transitions only occur on the active edge of the declared clock.

---

## Section 3: The Three-Context Execution Pipeline

To deliver a cohesive developer loop, the digital simulation engine is designed to operate seamlessly across three distinct environments: the CLI compiler, the background Language Server, and the Hardware Script Monitor.

```
                     [ Source Code (.hw) ]
                               │
                               ▼
               [ CLI Compiler / Language Server ]
               - Parses and builds Netlist Arena
               - Runs headless check and logic tests
                               │
                               ▼ Generates Unified Binary (.hwsb)
                               │
                               ▼
               [ Hardware Script Monitor ]
               - Loads binary via zero-copy mmap
               - Evaluates logic states in Web Worker
               - Plots waveforms in uPlot viewport
```

### Context One: The CLI Compiler (`hwc`)
When run from the command line, the simulator operates in a completely headless, high-performance mode:

*   **The Command:** Running `hwc simulate --digital` triggers the engine.
*   **The Process:** The compiler parses your design, builds the Netlist Arena, and executes your test vectors.
*   **The Output:** It outputs clean, text-based pass or fail results directly to the terminal.
*   **Use Case:** This headless mode is highly optimized for server-side continuous integration pipelines, automated regression testing, and verification scripts where no graphical environment is available.

### Context Two: The Language Server / Background Daemon
When you are writing code in your editor (such as VS Code), a lightweight language server runs in the background:

*   **The Process:** It executes the syntax and symbol passes in milliseconds without allocating any physical voxel space.
*   **The Validation:** It verifies that signal bit-widths match, constant-folding expressions are evaluated, and all device pins are logically connected.
*   **The Feedback:** It highlights logical and syntax errors with red squiggles directly in your editor, providing immediate feedback before you run a full simulation.

### Context Three: The Hardware Script Monitor (`hsm`)
The Monitor serves as your visual web-style browser companion. It does not stream real-time compilation data over network sockets; instead, it operates through a clean, file-based handshake:

*   **The Compilation Handoff:** On a successful build, the compiler serializes the compiled netlist, component instances, and standard cell stamps into a single, unified binary file called the **Hardware Simulation Binary** (with the `.hwsb` or `.hsx` extension).
*   **The Zero-Copy Load:** The Monitor detects that the binary has changed, memory-maps the file instantly, and loads the data into its SolidJS frontend.
*   **The Interactive View:** The Monitor houses the digital simulation's interactive user interface. It runs the simulation engine inside a browser Web Worker using the compiled WebAssembly or JavaScript-native logical graph.
*   **The Waveform Plot:** As the simulator runs, the Monitor captures the signal transitions and plots them instantly on an interactive waveform tab using a hardware-accelerated charting library.

---

## Section 4: Pure Rust Library Stack

To maintain our goal of a self-contained, lightweight compiler binary under twenty megabytes, we strictly reject heavy external C++ runtimes or non-deterministic libraries. The digital simulation engine is built using a curated selection of five pure, native Rust libraries.

### Library One: `std::simd` (Portable SIMD)
*   **The Role:** This is the standard library's portable SIMD API.
*   **Why We Use It:** It provides safe, compiler-intrinsic access to the host CPU's vector registers (such as AVX-512 on AMD/Intel and NEON on Apple Silicon/ARM). It allows the compiler to pack and execute gate evaluations in parallel across hundreds of bits without requiring platform-specific assembly code.

### Library Two: `petgraph`
*   **The Role:** A highly optimized, native Rust graph library.
*   **Why We Use It:** We use it to represent the electrical netlist and logical connections as a Directed Acyclic Graph. It handles topological sorting and cycle detection, allowing us to find the correct, conflict-free evaluation order for all gates in your module before simulation begins.

### Library Three: `rayon`
*   **The Role:** A data-parallelism library for Rust.
*   **Why We Use It:** When simulating large, multi-core digital systems, different sectors of the chip can be simulated independently. Rayon automatically partitions independent logical sectors and schedules their execution across all available threads of your host CPU, achieving linear performance scaling.

### Library Four: `bitvec`
*   **The Role:** A crate for managing memory-compact bit vectors and bit slices.
*   **Why We Use It:** In digital simulation, representing a single bit (one or zero) as a standard eight-bit byte or thirty-two-bit integer is highly wasteful. The `bitvec` library allows us to pack signal states tightly into individual bits, maximizing CPU cache line utilization and keeping our memory footprint under fifty megabytes for massive designs.

### Library Five: `rkyv` and `memmap2`
*   **The Role:** Zero-copy binary serialization and memory-mapped file input/output.
*   **Why We Use It:** The compiler uses `rkyv` to serialize the intermediate compiled hardware structures into the `.hwsb` binary file. The Monitor uses `memmap2` to memory-map this file, casting the raw bytes directly into usable memory in microseconds with zero parsing overhead and zero heap allocations.

---

## Section 5: Waveform Storage and the Handshake

The output of the digital simulation is stored as a series of signal state changes over time.

### The Value Change Dump (VCD) Standard
To ensure HardwareScript remains fully compatible with the global hardware engineering ecosystem, the simulation engine can export its output into the industry-standard **Value Change Dump** format (with the `.vcd` extension). 

VCD is a highly compressed, text-based format that records the exact timestamps when a signal changes value. Because it is an open standard, you can open your HardwareScript VCD files in any standard waveform viewer (such as GTKWave) to verify your design's timing and state transitions.

### The Handshake Workflow
When designing locally, you experience a clean, integrated development loop:

1.  **Code:** You write your modules and logic in VS Code.
2.  **Save:** Saving the file triggers the CLI watcher.
3.  **Compile & Run:** The compiler executes `hwc simulate --digital` in the background, verifying your logic and outputting the digital waveform transitions into the unified `.hwsb` file.
4.  **Visualize:** The Hardware Script Monitor detects the file update, reloads the binary, and displays the exact digital wave transitions in its interactive SPICE/Waveform tab. 

This architecture keeps your tools decoupled, highly modular, and exceptionally fast, proving that professional-grade digital hardware verification can be accomplished with the speed and elegance of modern software compilation.