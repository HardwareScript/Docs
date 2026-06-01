# Component Physical Mounting and Elevation Abstraction

**Status**: Official Design Specification  
**Version**: v0.1.7  
**Domain**: Physical Geometry / AST

## 1. The Core Concept: Planar Stackup vs. Mounting Plane

The Z-axis of a printed circuit board is divided into two distinct regions:

1.  **The Board Stackup (Interior)**: A flat, 2.5D sandwich of layers spanning from $z = z_{\min}$ to $z = z_{\max}$, as defined in the active profile's stackup section.
2.  **The Mounting Space (Exterior)**: The 3D space strictly above the board ($z > z_{\max}$) and below the board ($z < z_{\min}$).

```
                              ABOVE-BOARD SPACE (z > Z_max) 
                         Component Body (U1) sits here. 
                         Body Z: [Z_max to Z_max + Component_Height] 
                         ┌─────────────────────────────────┐ 
                         │            IC BODY              │ 
                         └───────────────┬─────────────────┘ 
 ────────────────────────────────────────┼─────────────────────────────────── [z = Z_max (Board Top Surface)] 
                                         │ Pin Contact Point (z = Z_max) 
                         ┌───────────────┴─────────────────┐ 
                         │      U1_VCC_Pad (Copper)        │ 
                         └─────────────────────────────────┘ 
                         Top Copper Layer (z = Z_max - t_copper to Z_max) 
                         
                         BOARD STACKUP (Dielectric Core) 
```

By decoupling these regions, the compiler ensures that the component body never interpenetrates the top copper layer, preventing Z-fighting and rendering artifacts while maintaining physical accuracy.

## 2. Syntax Reference

To declare where a component is mounted relative to the board stackup, we introduce the `mount:` property in the space-level component placement.

### 2.1 Top Mounting (The Standard SMT Layout)

When a component is mounted on the top, its body extends upwards into positive Z-space, while its unrolled copper pads reside in the top copper layer.

```hw
space MultiConnectedPCB implements MultiConnectedModule: 
    dimensions: 10mm by 5mm by 2.0mm 
    grid: 200 by 100 by 40 
    profile: AdvancedPCBStackup 
 
    add IC_Package named U1 at [x: 1.0mm, y: 1.0mm]: 
        mount: top  # Sits on top of the board. Body Z is positive. 
        net: [VCC: VDD, GND: GND, IO: SIG] 
```

### 2.2 Bottom Mounting (The Mirrored SMT Layout)

When a component is mounted on the bottom, its body extends downwards into negative Z-space, while its unrolled copper pads reside in the bottom copper layer. The compiler automatically mirrors the pin X-coordinates to reflect physical flipping.

```hw
space MultiConnectedPCB: 
    # ... 
    add IC_Package named U2 at [x: 5.5mm, y: 1.0mm]: 
        mount: bottom  # Sits on the bottom of the board. Body Z is negative. 
        net: [VCC: VDD, GND: GND, IO: SIG] 
```

### 2.3 Stand-off Height (Physical Air Gap)

To prevent Z-fighting and model physical assembly reality, components support an optional `standoff:` property. This elevates (or lowers) the component body relative to the board surface without affecting pad connectivity.

```hw
component IC_Package:
    layout:
        standoff: 50um # Default for the package

space MultiConnectedPCB:
    add IC_Package named U1 at [...]:
        standoff: 100um # Optional override
```

## 3. AST Schema Updates (hwc-parser)

### 3.1 The MountingSide Enum

```rust
// hwc-parser/src/ast/component.rs 
 
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)] 
pub enum MountingSide { 
    /// Mounted on the top surface of the board (Z-up) 
    Top, 
    /// Mounted on the bottom surface of the board (Z-down) 
    Bottom, 
    /// Embedded inside a dielectric cavity (3D integration) 
    Embedded, 
} 
```

### 3.2 ComponentPlacement Update

```rust
// hwc-parser/src/ast/component.rs 
 
pub struct ComponentPlacement { 
    pub component_type: Identifier, 
    pub parameters: SmallVec<[Parameter; 4]>, 
    pub name: Option<ComponentName>, 
    pub position: Coordinate, 
    pub rotation: Option<Rotation>, 
    pub elevation: Option<Elevation>, 
    
    /// NEW (v0.1.7): Component mounting plane 
    pub mount: Option<MountingSide>, 
    
    pub array_config: Option<ArrayConfig>, 
    pub pin_net_bindings: FxHashMap<CompactString, NetBinding>, 
    pub waivers: Waivers, 
    pub span: Span, 
} 
```
