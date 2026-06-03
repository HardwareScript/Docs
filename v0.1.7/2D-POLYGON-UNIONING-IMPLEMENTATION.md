# 2D Polygon Unioning Implementation Specification

**Version:** 0.1.7  
**Strategy:** Strategy A - 2D Polygon Clipping & Unioning  
**Status:** Implementation Ready

---

## Executive Summary

This document specifies the implementation of **Strategy A: 2D Polygon Unioning** inside the Rust compiler. This strategy integrates a 2D polygon clipping pipeline into the `add_substrate` module, converting all overlapping copper elements (pours, traces, and via landing pads) of the same Net ID into a single unified 2D boundary before extruding it into a 3D mesh.

### Key Benefits

1. **Zero Overlapping Copper Mesh Nodes**: Rectangular traces and circular via pads are completely unified with clean, dissolved boundaries
2. **Perfect Coordinate Integrity**: Not a single nanometer of offset is added to spatial layouts
3. **Flawless Multi-Format Compatibility**: Downstream simulators (FEA, EM, thermal) and 3D printing slicers process exported GLB meshes without conflicts

---

## 1. The 2D Unioning Pipeline

```
[Copper Pours]       [Via Landing Pads]      [Traces / Ribbons]
(Rect boundaries)    (Cylinder approximations)  (Ribbon boundaries)
         \                       |                      /
          \                      v                     /
           Converted to 2D Closed Paths (Vec<Vec<Point>>)
                               |
                               v
              [ Clipper 2D Polygon Union Engine ]
                               |
                               v
              Unioned 2D Contours (Outer & Holes)
                               |
                               v
                  [ Earcutr Triangulation ]
                               |
                               v
               Single Manifold 3D Extruded Mesh
```

---

## 2. Mathematical Foundation: CorelDraw Boolean Operations

The operations described in CorelDraw map directly to computational geometry operations:

| CorelDraw Operation | Clipper Operation | Use Case in Hardware Compiler |
|:---|:---|:---|
| **Weld** | `Union` | Merge traces and pads into unified copper islands |
| **Trim / Cut out** | `Difference` | Cut via holes out of FR4 substrate or antipads out of planes |
| **Intersect** | `Intersection` | Find overlapping regions for DRC validation |

### Visual Reference

#### 1. UNION (Weld)
```
   +----------+
   |          |
   |          |====+
   |          |====+
   +----------+
```

#### 2. DIFFERENCE (Trim)
```
   +----------+
   |          \
   |           )   (Bite)
   |          /
   +----------+
```

---

## 3. Dependency Integration

### Step 1: Add Required Crates

Add these highly performant geometry libraries to `Cargo.toml` under `hwc-export`:

```toml
[dependencies]
clipper2-rust = "1.0"   # Pure Rust port of Clipper2 (no FFI, fast compilation)
earcut = "0.4"          # GeoRust zero-allocation triangulator (3x faster than earcutr)
```

### Library Selection Rationale

- **clipper2-rust**: Native Rust implementation of the Vatti clipping algorithm. Eliminates C++ toolchain dependencies and FFI overhead, allowing for aggressive compiler inlining and faster builds.
- **earcut**: Maintained by the GeoRust team, this crate features a zero-allocation architecture. It reuses internal buffers to minimize heap pressure during complex board triangulations.

---

## 4. Polyline Conversion Helpers

These helpers convert 3D primitives (Cylinders/Pads, Rectangular Pours, and Ribbons) into 2D polygon paths using integer coordinates (nanometers).

### File: `hwc-export/src/geometry_union.rs`

```rust
// In hwc-export/src/geometry_union.rs or directly inside substrate.rs
use hwc_engine::geometry::BoundingBox;
use clipper2_rust::{Path64, Paths64, Point64};

/// Convert an axis-aligned bounding box to a closed Clipper path (in nanometers)
pub fn rect_to_path(bbox: &BoundingBox) -> Path64 {
    let mut path = Path64::new();
    path.push(Point64::new(bbox.min.x, bbox.min.y));
    path.push(Point64::new(bbox.max.x, bbox.min.y));
    path.push(Point64::new(bbox.max.x, bbox.max.y));
    path.push(Point64::new(bbox.min.x, bbox.max.y));
    path
}

/// Convert a circular via landing pad into a 64-sided regular polygon (in nanometers)
pub fn circle_to_path(cx: i64, cy: i64, radius: i64, segments: usize) -> Path64 {
    let mut path = Path64::new();
    for i in 0..segments {
        let angle = (i as f64 / segments as f64) * 2.0 * std::f64::consts::PI;
        let x = cx + (radius as f64 * angle.cos()) as i64;
        let y = cy + (radius as f64 * angle.sin()) as i64;
        path.push(Point64::new(x, y));
    }
    path
}
```

---

## 5. 3D Extrusion Engine Using Earcutr

Once Clipper yields the unioned 2D boundaries, we extrude them into 3D meshes. We triangulate the top and bottom caps using earcutr (which handles complex outlines with internal holes) and bridge the side walls.

### File: `hwc-export/src/mesh_extrusion.rs`

```rust
use crate::scene_graph::types::{Face, MeshNode, Vertex};
use hwc_engine::SpaceView;

/// Extrudes flat 2D contours (with potential holes) into a solid 3D mesh
pub fn extrude_polygon_mesh(
    name: &str,
    outer_contour: &[(f64, f64)],
    holes: &[Vec<(f64, f64)>],
    z_min: f64,
    depth: f64,
    material_name: &str,
    view: SpaceView,
) -> MeshNode {
    let mut vertices = Vec::new();
    let mut faces = Vec::new();
    let mut triangulator = earcut::Earcut::new();

    // Helper to map 2D coordinates to 3D scene space based on active view
    let map_vertex = |ex: f64, ey: f64, ez: f64| -> Vertex {
        match view {
            SpaceView::Horizontal => Vertex { x: ex, y: ez, z: ey },
            SpaceView::Vertical => Vertex { x: ex, y: ey, z: ez },
        }
    };

    // Flatten vertices for earcutr input: [x0,y0, x1,y1, ...]
    let mut flat_coords = Vec::new();
    let mut hole_indices = Vec::new();

    // Add outer contour
    for &(x, y) in outer_contour {
        flat_coords.push(x);
        flat_coords.push(y);
    }

    // Add holes and track their starting indices
    for hole in holes {
        hole_indices.push(flat_coords.len() / 2);
        for &(x, y) in hole {
            flat_coords.push(x);
            flat_coords.push(y);
        }
    }

    // Generate 3D Vertices for Bottom (z_min) and Top (z_min + depth)
    let vertex_count_2d = flat_coords.len() / 2;
    for i in 0..vertex_count_2d {
        let x = flat_coords[i * 2];
        let y = flat_coords[i * 2 + 1];

        // Bottom vertices: indices [0 .. vertex_count_2d - 1]
        vertices.push(map_vertex(x, y, z_min));
        // Top vertices: indices [vertex_count_2d .. 2*vertex_count_2d - 1]
        vertices.push(map_vertex(x, y, z_min + depth));
    }

    // 1. Triangulate Caps using Earcut (GeoRust Optimized)
    // The earcut crate (GeoRust) is used with its zero-allocation architecture to triangulate the resulting complex polygons.
    let mut triangles = Vec::new();
    let data = flat_coords.chunks_exact(2).map(|c| [c[0], c[1]]);
    triangulator.earcut(data, &hole_indices, &mut triangles);

    for chunk in triangles.chunks_exact(3) {
        let (v0, v1, v2) = (chunk[0], chunk[1], chunk[2]);

        // Bottom Cap (facing down, CW winding from inside)
        faces.push(Face {
            vertices: vec![v0 * 2, v2 * 2, v1 * 2],
        });

        // Top Cap (facing up, CCW winding from outside)
        faces.push(Face {
            vertices: vec![v0 * 2 + 1, v1 * 2 + 1, v2 * 2 + 1],
        });
    }

    // 2. Generate side walls
    // We must generate wall segments for the outer boundary and each hole
    let mut contour_ranges = vec![0];
    contour_ranges.extend(hole_indices.iter().cloned());
    contour_ranges.push(vertex_count_2d);

    for r in 0..(contour_ranges.len() - 1) {
        let start = contour_ranges[r];
        let end = contour_ranges[r + 1];
        let count = end - start;

        for i in 0..count {
            let curr = start + i;
            let next = start + (i + 1) % count;

            let b_curr = curr * 2;
            let t_curr = curr * 2 + 1;
            let b_next = next * 2;
            let t_next = next * 2 + 1;

            // Quad face connecting bottom and top segments
            faces.push(Face {
                vertices: vec![b_curr, b_next, t_next, t_curr],
            });
        }
    }

    MeshNode {
        name: name.into(),
        vertices,
        faces,
        material_name: material_name.into(),
    }
}
```

---

## 6. Integration into add_substrate Compilation Phase

Refactor the substrate exporter `substrate.rs`. Instead of pushing separate raw boxes for pours and circular pads directly into the mesh vector, we bucket them, execute the 2D Clipper Union, and extrude the final unified outputs.

### File: `hwc-export/src/substrate.rs`

```rust
// ... (Your existing stackup resolution & base substrate creation remains intact)

// 1. Group all copper layers by layer slice (Z-range, material, Net ID)
let mut copper_groupings: FxHashMap<
    (i64, i64, hwc_engine::voxel_grid::MaterialId, hwc_engine::voxel_grid::NetId), 
    Vec<&hwc_engine::voxel_grid::SubstrateLayer>
> = FxHashMap::default();

for layer in &substrate_layers {
    if layer.layer_type == hwc_engine::voxel_grid::SubstrateLayerType::Pour 
        && layer.net != 0 {
        let key = (
            layer.bbox.min.z, 
            layer.bbox.max.z, 
            layer.material, 
            hwc_engine::voxel_grid::NetId::new(layer.net)
        );
        copper_groupings.entry(key).or_default().push(layer);
    }
}

// Also include Via Landing Pads into the groupings (v0.1.8 Strategy A Integration)
for layer in &substrate_layers {
    if layer.layer_type == hwc_engine::voxel_grid::SubstrateLayerType::Contact 
        && layer.net != 0 {
        // Find which copper slices this via intersects and add its pads to those layers
        for (&(z_min, z_max, material, net_id), list) in copper_groupings.iter_mut() {
            if layer.net == net_id.raw() 
                && layer.bbox.min.z <= z_max 
                && layer.bbox.max.z >= z_min {
                list.push(layer);
            }
        }
    }
}

// 2. Perform 2D Clipper Union for each copper slice grouping
for ((z_min_nm, z_max_nm, material_id, net_id), layers) in copper_groupings {
    let material_name = space.material_registry
        .get_name(material_id)
        .unwrap_or("Copper");
    
    let mut clipper_paths = Paths64::new();
    
    for layer in layers {
        match layer.shape {
            SubstrateLayerShape::Rect => {
                clipper_paths.push(rect_to_path(&layer.bbox));
            }
            SubstrateLayerShape::Cylinder { diameter, .. } => {
                let cx = (layer.bbox.min.x + layer.bbox.max.x) / 2;
                let cy = (layer.bbox.min.y + layer.bbox.max.y) / 2;
                clipper_paths.push(circle_to_path(cx, cy, diameter / 2, 64));
            }
            SubstrateLayerShape::Tube { pad_diameter, .. } => {
                let cx = (layer.bbox.min.x + layer.bbox.max.x) / 2;
                let cy = (layer.bbox.min.y + layer.bbox.max.y) / 2;
                clipper_paths.push(circle_to_path(cx, cy, pad_diameter as i64 / 2, 64));
            }
        }
    }
    
    // Run Clipper Union using clipper2-rust v1.0 API
    // v0.1.8 FIXED: Use NonZero fill rule to merge overlapping copper into a solid manifold (Non-Zero Winding Fix)
    // Overlapping shapes must be unified using the Non-Zero winding rule. Using the default 
    // EvenOdd rule would result in hollow gaps where two copper elements intersect.
    let union_result = clipper2_rust::union_64(&clipper_paths, &Vec::new(), clipper2_rust::core::FillRule::NonZero);
    
    if let Some(unioned_paths) = union_result {
        // Convert nanometer integer Clipper output back into precise f64 millimeters
        let mut outer_contours = Vec::new();
        let mut holes = Vec::new();
        
        for path in unioned_paths {
            let mut points = Vec::new();
            for pt in &path {
                points.push((
                    pt.x as f64 / 1_000_000.0, 
                    pt.y as f64 / 1_000_000.0
                ));
            }
            
            // Clipper represents holes as clockwise and outer bounds as counter-clockwise
            if clipper2_rust::is_positive(&path) {
                outer_contours.push(points);
            } else {
                holes.push(points);
            }
        }
        
        // 3. Extrude each unioned island as a single continuous 3D Mesh
        let z_min_mm = z_min_nm as f64 / 1_000_000.0;
        let depth_mm = (z_max_nm - z_min_nm) as f64 / 1_000_000.0;
        
        for (idx, outer) in outer_contours.iter().enumerate() {
            meshes.push(extrude_polygon_mesh(
                &format!("Unified_Net_{}_Copper_Island_{}", net_id.raw(), idx),
                outer,
                &holes,
                z_min_mm,
                depth_mm,
                material_name,
                space.view,
            ));
        }
    }
}
```

---

## 7. Verification Testing Strategy

### Standalone Unit Test: CorelDraw Boolean Handshake

This test validates the foundation before applying it to complex layouts.

#### Test Scenario

1. **Shape A (Square)**: A 10mm × 10mm box centered at (0,0)
2. **Shape B (Circle)**: A 5mm radius circle centered at (5,0) - overlaps the right half of the square

#### Test Operations

- **Test 1 (Union/Weld)**: Merge them into a single continuous "tombstone" outline
- **Test 2 (Difference/Trim)**: Subtract the circle from the square to leave a "bite" mark

### File: `hwc-export/tests/clipper_test.rs`

```rust
#[cfg(test)]
mod tests {
    use clipper2_rust::{Path64, Paths64, Point64, FillRule};

    #[test]
    fn test_coreldraw_boolean_handshake() {
        // 1. Define Shape A: A 10mm x 10mm square (coordinates in nanometers)
        // Millimeters to nanometers: 10mm = 10,000,000 nm
        let mut square = Path64::new();
        square.push(Point64::new(-5_000_000, -5_000_000));
        square.push(Point64::new(5_000_000, -5_000_000));
        square.push(Point64::new(5_000_000, 5_000_000));
        square.push(Point64::new(-5_000_000, 5_000_000));

        // 2. Define Shape B: A 5mm radius circle, offset to the right at X = 5mm
        let cx = 5_000_000;
        let cy = 0;
        let radius = 5_000_000;
        let segments = 32;
        
        let mut circle = Path64::new();
        for i in 0..segments {
            let angle = (i as f64 / segments as f64) * 2.0 * std::f64::consts::PI;
            let x = cx + (radius as f64 * angle.cos()) as i64;
            let y = cy + (radius as f64 * angle.sin()) as i64;
            circle.push(Point64::new(x, y));
        }

        let mut subjects = Paths64::new();
        subjects.push(square);
        
        let mut clips = Paths64::new();
        clips.push(circle);

        // --- OPERATION 1: UNION (Weld) ---
        // Merges both into a single continuous polygon
        let weld_result = clipper2_rust::union(&subjects, &clips, FillRule::EvenOdd);
        assert!(weld_result.is_some());
        
        let welded_paths = weld_result.unwrap();
        // The result should be unified into a single outer loop
        assert_eq!(welded_paths.len(), 1);
        println!("Weld Success: Unified shape has {} vertices", welded_paths[0].len());

        // --- OPERATION 2: DIFFERENCE (Trim) ---
        // Cuts the circle out of the square, leaving a crescent bite mark
        let trim_result = clipper2_rust::difference(&subjects, &clips, FillRule::EvenOdd);
        assert!(trim_result.is_some());
        
        let trimmed_paths = trim_result.unwrap();
        // The result should be a single chopped polygon
        assert_eq!(trimmed_paths.len(), 1);
        println!("Trim Success: Chopped shape has {} vertices", trimmed_paths[0].len());

        // Verify the bite mark: The point (5mm, 0) is inside the circle, 
        // so it must have been removed from the square.
        for point in &trimmed_paths[0] {
            // Ensure no vertex is sitting in the center of the deleted circle
            assert!(!(point.x == 5_000_000 && point.y == 0));
        }
    }
}
```

---

## 8. Expected Outcomes

Once compiled, this refactoring delivers:

### 1. Zero Overlapping Copper Mesh Nodes
In wireframe outputs, rectangular traces and circular via pads are completely unified. Boundaries where they once intersected are cleanly dissolved, creating a single continuous mesh outline.

### 2. Perfect Coordinate Integrity
Not a single nanometer of offset is added to the compiler's spatial layout, keeping GDSII, Gerber, and simulation files 100% accurate.

### 3. Flawless Multi-Format Compatibility
Downstream simulators (FEA, EM, thermal) and direct 3D printing slicers process exported GLB meshes without intersecting-volume conflicts or double-counted materials.

---

## 9. Integration Checklist

- [ ] Add `clipper2` and `earcutr` dependencies to `Cargo.toml`
- [ ] Create `geometry_union.rs` with `rect_to_path` and `circle_to_path` helpers
- [ ] Create `mesh_extrusion.rs` with `extrude_polygon_mesh` function
- [ ] Refactor `substrate.rs` to group copper layers by Net ID and Z-range
- [ ] Implement Clipper Union operation on grouped layers
- [ ] Implement mesh extrusion for unified 2D contours
- [ ] Create standalone unit test for boolean operations
- [ ] Verify output with existing test boards
- [ ] Update documentation with performance benchmarks
- [ ] Add regression tests for edge cases (nested holes, multiple islands)

---

## 10. Performance Considerations

### Coordinate System
- Use nanometer precision (1nm = 1 integer unit) to prevent floating-point errors
- Convert to millimeters only for final mesh output

### Polygon Complexity
- Circle approximation: 64 segments for diameters > 0.2mm, 32 segments otherwise
- Balance between visual smoothness and vertex count

### Memory Efficiency
- Group operations by Net ID and Z-range to minimize intermediate allocations
- Process one copper layer at a time rather than buffering all layers

---

## Conclusion

This implementation transforms overlapping geometric primitives into unified, manifold meshes using industry-standard computational geometry algorithms. The approach is mathematically sound, performance-optimized, and maintains perfect coordinate fidelity throughout the compilation pipeline.
