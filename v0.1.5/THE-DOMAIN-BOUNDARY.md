# Hardware Script v0.1.5 - The Domain Boundary & Soft IP

**Document Type**: Architectural Boundary Definition  
**Status**: Official Release  
**Version**: 0.1.5  

---

## The Mastery of the Electron

To prevent feature-creep and maintain the blazing sub-millisecond performance of the compiler, Hardware Script operates on a strict, philosophically absolute boundary:

> **Hardware Script is the absolute master of the Electron, the Photon, and the Thermal Gradient in a stationary medium. Everything outside of this is explicitly ignored.**

Hardware Script does not attempt to be AutoCAD or SolidWorks. It does not simulate the aerodynamics of a drone propeller, the friction of a rubber tire, or the fluid dynamics of an airplane wing. 

If a technology involves routing energy, data, or light through a *stationary physical medium*, Hardware Script owns it.

### Inside the Boundary (What We Compile)
*   **Silicon VLSI:** Routing electrons through P-N junctions.
*   **PCBs:** Routing power and data through copper traces.
*   **Silicon Photonics:** Routing light (photons) through glass waveguides.
*   **Solid-State Batteries:** Routing ions through electrolytes.
*   **RF Antennas:** Radiating electromagnetic waves into the air.

### Outside the Boundary (Macro-Kinematics)
Once physical objects start spinning, walking, flying, or colliding, the domain shifts to **Mechanical CAD (MCAD)**. 
When designing a Hard Drive, Hardware Script designs the motherboard, the data cables, and the electromagnetic coils in the read/write head. It explicitly ignores the aerodynamic lift of the spinning platter.

**The Clean Hand-Off:** Hardware Script builds the brain, the nervous system, and the heart. Mechanical CAD builds the skeleton and muscle. The two interface perfectly via export targets (`.glb`, `.step`).

---

## "Soft IP" - The Software of Hardware

Because Hardware Script strictly separates the `logic:` block from the physical `space:` block, it enables a massive open-source ecosystem identical to `npm` or `crates.io`.

In the hardware industry, pure mathematical algorithms that have no physical shape yet are called **Soft IP (Intellectual Property)**.

### The Pure Logic Package
A mathematician can write a pure PWM (Pulse Width Modulation) motor controller in Hardware Script without ever defining a physical PCB, coordinate, or material. 

```hw
# Published to the HPM registry as: @robotics/motor_control
define module "PWM_Generator":
    pins: Clk, Rst, Speed[8], PwmOut
    logic:
        let counter = Reg(clock: Clk, reset: Rst, init: 0)
        counter.next = if counter == 255: 0 else: counter + 1
        PwmOut = if counter < Speed: 1 else: 0
```

### The Physical Application
A physical hardware engineer building a drone can simply `import` this Soft IP and drop it onto their physical board.

1. **If building a custom Silicon ASIC:** The compiler physically etches microscopic 8-bit Comparators and Adders into the silicon layout to satisfy the logic.
2. **If building a fiberglass PCB:** The compiler automatically allocates a 50-cent FPGA/GreenPAK chip to the Bill of Materials, flashes the logic inside it, and routes the copper traces to its pins.

By cleanly dividing the domain, mathematicians write the logic, layout engineers write the physical spaces, and the compiler mathematically binds them together.
