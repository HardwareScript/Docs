
# Book 7: Analog, RF, & Neuromorphic Systems

**Hardware Script v1.0-draft**  
**Target Audience**: Analog IC Designers, RF Engineers, Neuromorphic Researchers  
**Last Updated**: March 2026

---

## 1. The Analog Crisis in EDA

For forty years, Electronic Design Automation (EDA) has been overwhelmingly optimized for digital systems. Languages like Verilog, VHDL, and Chisel abstract away the physical universe into a simple binary: `1` (High) and `0` (Low). 

This "Digital Illusion" allows compilers to automatically place and route billions of transistors, because a wire carrying 3.3V and a wire carrying 2.8V are both mathematically treated as a `1`. The exact physical shape of the wire doesn't matter as long as the threshold is met.

**Analog systems do not have this luxury.**
In analog, the exact physical geometry *is* the circuit. 
* A 0.001V fluctuation is not a rounding error; it is a corrupted audio signal. 
* A wire placed 2 nanometers too close to another wire creates parasitic capacitance, destroying a 10GHz microwave signal.
* A transistor operating 2°C hotter than its neighbor introduces a DC offset that ruins a sensor reading.

Because analog design is fundamentally bound to 3D continuous physics, it could never be abstracted into traditional HDLs. Consequently, analog engineers still use GUI tools (like Cadence Virtuoso) to draw custom shapes by hand, painstakingly matching transistors visually on a screen. 

Hardware Script ends this era. By compiling intent into a **Static 3D Voxel Grid** and running native physics validation, Hardware Script handles analog design with the speed and reproducibility of modern software.

---

## 2. Physics-First Compilation

Hardware Script natively supports analog design because it does not compile down to logical gates; it compiles down to physical atoms.

When the compiler executes Pass 2 (Space Assembly), it places materials (Copper, Silicon, Oxide) into a 3D coordinate system. Because the compiler knows the exact location, dimension, and material property of every feature, it calculates continuous physics automatically.

### Parasitic Extraction at Compile-Time
The ultimate enemy of analog circuits is **Parasitics**—unintended resistance ($R$) and capacitance ($C$) created by physical proximity.

Traditional workflow: Draw layout → Export → Run extraction tool → Discover failure → Redraw layout.

**Hardware Script workflow:**
The Layer 3 Voxel Engine mathematically evaluates the distance between all conductive voxels. Using the dielectric properties defined in `materials.hw`, it automatically computes the capacitance ($C = \epsilon A / d$) and resistance ($R = \rho L / A$) during the routing phase. If a routed trace violates the signal integrity constraint of an analog wave, the routing engine actively pushes the trace further away to reduce capacitance.

---

## 3. Parametric Analog Layout (Eliminating Hand-Drawing)

The most tedious part of analog design is drawing matched transistors. In a differential amplifier, two transistors must be perfectly identical and exposed to the exact same thermal gradients. Engineers solve this using **Interdigitation** or **Common-Centroid Layouts**—splitting the transistors into "fingers" and weaving them together like a checkerboard.

In Hardware Script, this complex 2D geometric puzzle is solved instantly using **Comptime loops and math operators**.

```hw
import "materials.hw"
import NMOS from "@foundry/tsmc_180nm"

define module "Common_Centroid_Pair" (fingers: Number, w: Measurement, l: Measurement):
    pins: V_In_A, V_In_B, V_Out_A, V_Out_B, Source_Common
    
    # Validation constraint
    if fingers % 2 != 0:
        throw Error "Common centroid layouts require an even number of fingers."
        
    # Generate the physical transistors parametrically
    for i in 0..(fingers - 1):
        add NMOS (width: w, length: l) named M_A[i]
        add NMOS (width: w, length: l) named M_B[i]

define space "Analog_Front_End":
    dimensions: 50µm by 50µm by 5µm
    grid: 500 by 500 by 50
    origin: tl by t
    
    add Common_Centroid_Pair (fingers: 8, w: 2µm, l: 180nm) named Diff_Pair
    
    layout Diff_Pair:
        # Programmatic Checkerboard (AB / BA pattern) for perfect thermal matching
        for row in 0..1:
            for col in 0..3:
                if (row + col) % 2 == 0:
                    M_A[(row * 4) + col] at[x: 5µm + (col * 3µm), y: 10µm + (row * 5µm), z: 1]
                else:
                    M_B[(row * 4) + col] at[x: 5µm + (col * 3µm), y: 10µm + (row * 5µm), z: 1]
```

**The Result:** A perfectly symmetrical, thermally matched, parametrically scalable analog layout generated in 25 lines of text, rather than 3 days of manual mouse clicks.

---

## 4. Continuous Geometries for RF and Microwaves

Radio Frequency (RF) and Microwave engineering is the extreme edge of analog design. At 10GHz, a 90-degree corner in a copper trace acts like a brick wall, causing the signal to reflect back on itself. 

RF requires sweeping curves, teardrop pad connections, and custom waveguide shapes. Hardware Script handles this seamlessly via the `polygon` primitive, allowing continuous geometries defined by exact mathematical coordinates.

```hw
define space "5G_Transceiver":
    dimensions: 10mm by 10mm by 1mm
    
    # Define a 50-ohm RF Waveguide with a smooth curve
    add polygon(Gold) named RF_Waveguide at [x:0, y:0, z:1]:
        points:
            -[0, 5mm]
            - [2mm, 5mm]
            # Smooth 1mm radius curve to prevent signal reflection
            - [2.29mm, 4.71mm]
            -[2.5mm, 4mm]
            - [2.5mm, 0]

    # Teardrop pad generation for component transition
    add polygon(Copper) named Teardrop_Transition at[x:5mm, y:5mm, z:1]:
        points:
            - [0, 0]           # Attach to standard trace
            - [0.5mm, 0.5mm]   # Sweep outward
            -[1.0mm, 0.5mm]   # Match pad width
            -[1.0mm, -0.5mm]
            - [0.5mm, -0.5mm]
```

These polygons are rasterized into the high-resolution voxel grid and verified against the speed of light and impedance constraints defined in your `constraints.hw` file.

---

## 5. Neuromorphic Computing & Analog AI (Compute-in-Memory)

The future of Large Language Models (LLMs) depends on escaping the "Von Neumann Bottleneck"—the massive power wasted shuffling data between memory and processors. The bleeding edge of academic research is **Analog Compute-in-Memory (CiM)**, which uses grids of programmable resistors (Memristors) to calculate Matrix Multiplication ($V = I \times R$) instantly using raw analog physics at near-zero power.

Because Hardware Script's architecture separates physical materials from logical intent (The Extension Absorption Principle), researchers can model and layout experimental Analog AI chips without requiring updates to the compiler.

### Step 1: Define the Experimental Physics
```hw
# In materials.hw
define material "Titanium_Dioxide_Memristor":
    category: semiconductor
    properties:
        # Analog properties (continuous spectrum)
        min_resistance: 100Ω
        max_resistance: 10kΩ
        switching_voltage: 1.2V
```

### Step 2: Parametric Generation of the AI Crossbar
Instead of drawing 16,384 intersecting lines by hand, an AI researcher can generate an entire analog neural network crossbar in a few lines.

```hw
# In modules.hw
define module "Analog_Matrix_Multiplier":
    pins: 
        Input_Vector[128]
        Output_Vector[128]
    
    # Create a 128x128 grid of memristors for analog matrix math
    for row in 0..127:
        for col in 0..127:
            add Memristor(Titanium_Dioxide_Memristor) named Weight[row][col]
            
            # Form the physical crossbar connections
            route Input_Vector[row] to Weight[row][col].Top_Electrode
            route Weight[row][col].Bottom_Electrode to Output_Vector[col]
```

### Step 3: Native Validation
When the user runs `hwc test tests.hw`, the compiler's native SPICE engine sweeps through the crossbar, injecting continuous analog voltage arrays and validating the matrix multiplication natively against the physical resistance of the Voxel Grid. 

---

## 6. Conclusion: The Grand Unification of Analog and Digital

For decades, digital teams and analog teams have lived in separate silos, using different software, different languages, and different workflows. 

Hardware Script unifies them. 

By treating the physical layout as code (`define space`), describing the exact physical atoms (`define material`), and parametrically generating the geometries (`for` loops and `polygons`), Hardware Script captures the continuous physical reality of the analog domain using the speed, version control, and LLM-friendliness of modern software.

Whether you are placing a 64-bit digital ALU, or sweeping a 10GHz continuous RF sine wave through a curved gold waveguide, the mathematical pipeline is exactly the same: **Intent → Text → Voxels → Physics → Manufacturing.**