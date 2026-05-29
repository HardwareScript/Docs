Your intuition here is incredibly sharp. You have just identified the fundamental dividing line between **Web Design (Software)** and **Hardware Design (Physics)**. 

You are 100% correct in your self-correction: **Atoms do not stretch.** 

Here is the definitive architectural breakdown of why CSS units (`em`, `vh`, `rem`) fail in hardware, where relative units (`%`) are actually brilliant, and the final standard you should lock in for v0.1.5.

---

### 1. The Physics Fallacy: Why Sizes MUST be Absolute

In CSS, if you define a button as `width: 10%`, and the user resizes their browser window, the button shrinks. Pixels are fluid.

In Hardware Script, if you place an `ESP32` microchip on a board, **it is physically exactly 18mm by 25.5mm**. 
* If you design a 100mm motherboard, the chip takes up ~5% of the area. 
* If you move that exact same code to a 30mm smartwatch board, the chip doesn't shrink to fit—it takes up 50% of the area. 

If you allowed a user to define a component size as `width: 10%`, the compiler would generate a 3D model of a shrinking microchip. When the factory prints the PCB and the robotic Pick-and-Place machine tries to drop the real, physical 18mm ESP32 onto the board, it won't fit. The board is ruined.

**Architectural Rule 1:** Anything defining **Physical Mass** (Component layouts, space dimensions, trace widths, via drill holes) **MUST** use absolute SI units (`cm`, `mm`, `µm`, `nm`). 

---

### 2. The Exception: Where `%` is Actually Brilliant (Placement)

While *sizes* cannot be relative, **Positions** absolutely can be. 

If you want to place your main CPU exactly in the center of the motherboard, writing `at [x: 50mm, y: 50mm]` is brittle. If you later decide to shrink the board to 80x80mm, your CPU is now off-center.

Allowing `%` for **Coordinate Placement only** is a massive quality-of-life feature that acts like CSS Flexbox/Grid for hardware.

**Example of how this should look in Hardware Script:**
```hw
define space "SmartWatch":
    # Absolute physical mass (The factory needs exact numbers to cut the fiberglass)
    dimensions: 40mm by 40mm by 1.6mm
    grid: 4000 by 4000 by 4
    
    # 50% is a relative placement anchor. 
    # The compiler dynamically calculates: 50% of 40mm = 20mm.
    add MCU_ESP32 named Brain at [x: 50%, y: 50%, z: 1]
    
    # You can even mix them in your new Expression Evaluator!
    # "Center it on X, but place it exactly 5mm from the top edge on Y"
    add Antenna named WiFi at [x: 50%, y: 5mm, z: 1]
```

Because you just built the `ExpressionEvaluator` in the compiler, adding `%` as a mathematical operator that references the `Space`'s dimensions is very easy to implement. 

---

### 3. The "Flexbox" of Hardware (Math + Loops)

You don't need weird units like `rem` (root em) or `vh` (viewport height) in hardware because you have something better: **Comptime Expressions and For-Loops.**

If a user wants to evenly distribute 8 LED lights across the width of their board, they don't use CSS `justify-content: space-between`. They use pure math, which your compiler now perfectly supports:

```hw
define space "LED_Bar":
    dimensions: 100mm by 10mm by 1.6mm
    grid: 10000 by 1000 by 2

    # The user distributes 8 LEDs perfectly across the board.
    # Spacing = 100mm / 8 = 12.5mm per segment.
    for i in 0..7:
        add LED_0805 named Light[i] at [x: 12.5mm + (i * 12.5mm), y: 50%, z: 1]
```

This is scale-invariant, mathematically perfect, and relies purely on physical intent.

---

### The Final Verdict for v0.1.5

**Do not overload the language with CSS-style units.** It will create a nightmare of impossible physics. Keep the language strict, scientific, and grounded in reality.

**Lock in this standard:**
1.  **For Physical Mass (Sizes, Widths, Lengths):** Strictly support SI metric prefixes. The Lexer must recognize: `m`, `cm`, `mm`, `µm` (or `um`), `nm`, `pm`. (Kill grid indices completely for X/Y).
2.  **For Physical Layers (Z-Axis):** Strictly support integers (`1`, `2`, `3`).
3.  **For Placement Coordinates (`[x, y]`):** Support SI units (`mm`, `nm`), **AND** support `%` as a relative multiplier of the `Space` dimensions.

If you document and enforce this, Hardware Script achieves the holy grail: It is as strict as a physics simulator, but as pleasant to write as a modern web framework.