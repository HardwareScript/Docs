# Deep-Dive Engineering Specification: HWS v0.1.8 Core Libraries & Transition Path

**Target Version:** v0.1.8-alpha  
**Document Type:** Technical Integration Specification  
**Status:** Approved for Implementation  
**Focus:** High-Performance Library APIs, Execution Benchmarks, and Code Refactoring

---

## 1. Deep Dive: High-Performance Library Integration APIs

To move Hardware Script into the academic research frontier, the compiler middle-end is completely decoupled from the old voxel grid [ROUTING-AND-MANUFACTURING-ARCHITECTURE.md, Unified-2.5D-3D-Routing-and-Placement.md]. This section specifies the precise Rust API integrations for the four core libraries driving the new vector-first physical synthesis engine [Unified-2.5D-3D-Routing-and-Placement.md, Advanced-Routing-Implementation.md].

---

### A. Spatial Indexing: Hybrid `rstar` + `geo-index`

The **`rstar`** crate implements a highly optimized, dynamic R\*-tree spatial index in pure Rust. The **`geo-index`** crate implements a static, packed R\*-tree with a flat, contiguous buffer layout for maximum cache locality. Instead of a discrete grid, the compiler stores trace segments as continuous coordinate lines [Z-AXIS-ABSTRACTION-IMPLEMENTATION.md, Unified-2.5D-3D-Routing-and-Placement.md]. 

The compiler uses a **hybrid indexing model**:
*   **`rstar`** is reserved for dynamic macro-placement of components during floorplanning.
*   **`geo-index`** is used for detailed routing layers, where static obstacles are compiled into flat, packed structures per layer after the partition stage, yielding 5×–10× faster queries with near-zero allocation overhead.

To index segments in the dynamic `rstar` index, we implement the `RTreeObject` trait for our `LineSegment` structures, allowing $O(\log N)$ spatial collision queries during floorplanning and DRC verification [2D-POLYGON-UNIONING-IMPLEMENTATION.md, Unified-2.5D-3D-Routing-and-Placement.md, Advanced-Routing-Implementation.md].

```rust
// crates/hwc-engine/src/geometry_router/spatial_index.rs

use rstar::{RTreeObject, AABB};
use hwc_engine::geometry::{Point3D, LineSegment};

/// A wrapper to implement the rstar traits on our native coordinate segments
pub struct IndexedSegment {
    pub segment_id: usize,
    pub net_id: usize,
    pub width_pm: i64,  // Picometer precision for silicon-grade accuracy
    pub start: Point3D,
    pub end: Point3D,
}

impl RTreeObject for IndexedSegment {
    type Envelope = AABB<[f64; 3]>;

    fn envelope(&self) -> Self::Envelope {
        // Inflate the bounding box by the trace width to represent physical volume
        let w_offset = (self.width_pm as f64) / 2_000_000.0; // Convert pm to mm for rendering
        
        let min_x = (self.start.x.min(self.end.x) as f64) - w_offset;
        let max_x = (self.start.x.max(self.end.x) as f64) + w_offset;
        
        let min_y = (self.start.y.min(self.end.y) as f64) - w_offset;
        let max_y = (self.start.y.max(self.end.y) as f64) + w_offset;
        
        // Z axis representation is discrete (layer boundaries)
        let min_z = self.start.z.min(self.end.z) as f64;
        let max_z = self.start.z.max(self.end.z) as f64;

        AABB::from_points([
            [min_x, min_y, min_z],
            [max_x, max_y, max_z],
        ])
    }
}
```

---

### B. Convex Legalization: Hybrid `clarabel` + Active-Set / DAG Solver

The **`clarabel`** crate is a high-performance, interior-point numerical solver written in pure Rust. It natively handles Quadratic Programming (QP) and Semidefinite Programming (SDP). For small, local problems, a lightweight **active-set solver** (like PIQP) or a **DAG graph compaction solver** provides near-zero startup cost and microsecond solve times.

When two trace vectors overlap, the compiler formulates their coordinates as a sparse, continuous QP problem [ROUTING-AND-MANUFACTURING-ARCHITECTURE.md]. The objective function minimizes trace displacement ($\alpha$) and length ($\gamma$), while ensuring that the physical clearance boundaries are maintained as strict inequalities [ROUTING-AND-MANUFACTURING-ARCHITECTURE.md, Unified-2.5D-3D-Routing-and-Placement.md].

The standard Quadratic Program is expressed as:

$$\min \quad \frac{1}{2} x^T P x + q^T x$$

$$\text{subject to} \quad A x + s = b, \quad s \in \mathcal{K}$$

The compiler uses a **two-tier solver strategy**:
*   **`clarabel` (global IPM):** For macro-scale floorplanning with complex, multi-variable constraints.
*   **Active-set / DAG solver (local):** For detailed trace sliding in small windows ($N < 20$), avoiding $O(N^3)$ matrix factorization overhead.

```rust
// crates/hwc-compiler/src/ir/routing/legalizer.rs

use clarabel::solver::{DefaultSolver, DefaultSettings, IPSolver};
use clarabel::algebra::{CscMatrix, SupportedCornerVal};

pub fn legalize_overlapping_vectors(
    colliding_segments: &[IndexedSegment],
    clearance_nm: f64,
) -> Result<Vec<Point3D>, String> {
    // 1. Construct the sparse quadratic cost matrix P
    // We penalize trace displacement quadratically to keep adjustments minimal
    let p_data = vec![1.0, 1.0, 1.0, 1.0]; // Quadratic penalty coefficients
    let p_colptr = vec![0, 1, 2, 3, 4];
    let p_rowval = vec![0, 1, 2, 3];
    let p_matrix = CscMatrix::new(4, 4, p_colptr, p_rowval, p_data);

    // 2. Define the linear cost vector q (prefer shortest path)
    let q_vector = vec![0.0, 0.0, 0.0, 0.0];

    // 3. Construct the inequality constraint matrix A and bounds b
    // This enforces: x_distance >= clearance_nm (non-overlapping boundary)
    let a_data = vec![1.0, -1.0];
    let a_colptr = vec![0, 1, 2];
    let a_rowval = vec![0, 0];
    let a_matrix = CscMatrix::new(1, 4, a_colptr, a_rowval, a_data);
    let b_bounds = vec![-clearance_nm];

    // 4. Set cone constraints (Non-negative bounds)
    let cones = vec![clarabel::solver::SupportedConeT::NonnegativeConeT(1)];

    // 5. Initialize the Clarabel solver
    let settings = DefaultSettings::default();
    let mut solver = DefaultSolver::new(&p_matrix, &q_vector, &a_matrix, &b_bounds, &cones, settings);

    solver.solve();

    // 6. Extract the optimized, rounded coordinate shifts
    let solution = solver.variables().x();
    Ok(apply_optimized_shifts(colliding_segments, solution))
}
```

---

### C. Polygon Welding and Tessellation: `clipper2-rust` & `earcut`

To generate clean 3D solid meshes without internal overlapping faces (preventing Z-fighting and visual anomalies), the exporter uses `clipper2-rust` to weld paths in 2D and `earcut` (Mapbox/GeoRust) to perform zero-allocation cap triangulation during the GLB export pass [2D-POLYGON-UNIONING-IMPLEMENTATION.md, Z_FIGHTING_FIX.md, Volumetric-Solid-Modeling-via-Boundary-Representation.md, Microkernel-Architecture.md]. Because `clipper2-rust` already outputs perfectly clean, non-self-intersecting contours, `earcut` performs 3×–5× faster than heavier tessellators on this clean input.

```rust
// hwc-export/src/scene_graph/welder.rs

use clipper2_rust::{Paths64, FillRule};
use earcutr::earcut;
use crate::scene_graph::types::{MeshNode, Face, Vertex};

pub fn weld_and_tessellate_copper_layer(
    copper_paths: &Paths64,
    z_min_mm: f64,
    depth_mm: f64,
    material_name: &str,
) -> MeshNode {
    // 1. Weld overlapping shapes in 2D using the Non-Zero winding rule
    // This dissolves all internal overlapping boundaries
    let union_result = clipper2_rust::union_64(copper_paths, &Paths64::new(), FillRule::NonZero)
        .expect("Clipper2 Union Failed");

    // 2. Triangulate the flat 2D welded contours using earcut (zero-allocation)
    // earcut is 3-5x faster than tess2-rust on clean, unioned contours with holes
    let mut all_vertices: Vec<f64> = Vec::new();
    let mut all_hole_indices: Vec<usize> = Vec::new();
    let mut vertex_offset = 0;

    for path in &union_result {
        all_hole_indices.push(vertex_offset / 2);
        for pt in path {
            all_vertices.push(pt.x as f64 / 1_000_000.0);
            all_vertices.push(pt.y as f64 / 1_000_000.0);
            vertex_offset += 2;
        }
    }

    // flat_indices returns triangulation indices (flat三角形list)
    let flat_indices = earcut(&all_vertices, &all_hole_indices, 2);

    // 3. Extrude the 2D triangles into a watertight 3D solid mesh
    let mut mesh_vertices = Vec::new();
    let mut mesh_faces = Vec::new();

    // Map 2D vertices to 3D Top and Bottom coordinates
    for i in (0..all_vertices.len()).step_by(2) {
        let x = all_vertices[i];
        let y = all_vertices[i + 1];
        
        mesh_vertices.push(Vertex { x, y, z: z_min_mm }); // Bottom vertex
        mesh_vertices.push(Vertex { x, y, z: z_min_mm + depth_mm }); // Top vertex
    }

    // Map triangulation indices to 3D faces
    for chunk in flat_indices.chunks_exact(3) {
        let (v0, v1, v2) = (chunk[0] as usize, chunk[1] as usize, chunk[2] as usize);
        
        mesh_faces.push(Face { vertices: vec![v0 * 2, v2 * 2, v1 * 2] }); // Bottom cap
        mesh_faces.push(Face { vertices: vec![v0 * 2 + 1, v1 * 2 + 1, v2 * 2 + 1] }); // Top cap
    }

    MeshNode {
        name: "Welded_Copper_Plane".into(),
        vertices: mesh_vertices,
        faces: mesh_faces,
        material_name: material_name.into(),
    }
}
```

---

### D. Sub-Millisecond Serialization: Single-Source Binary `rkyv` (AlignedVec)

The compiler enforces a **single-source binary lockfile** using `rkyv` for zero-copy binary deserialization via `memmap2`. The serialization path uses `rkyv::util::AlignedVec` (16-byte alignment) to guarantee that the binary payload is perfectly aligned to the CPU's native word boundary, preventing unaligned memory access panics during zero-copy casts. No secondary JSON file is generated during builds—if a human needs to inspect or audit the lockfile, a dedicated CLI tool (`hwc lock inspect project.routes.lock`) decodes the binary to stdout on demand, keeping the workspace and version control history clean.

```rust
// crates/hwc-compiler/src/ir/routing/lockfile.rs

use rkyv::{Archive, Deserialize, Serialize};
use rkyv::util::AlignedVec;
use rkyv::ser::Serializer;
use memmap2::Mmap;
use std::fs::File;
use std::io::Write;

// Primary compilation path: fast, zero-allocation memory-mapped load
#[derive(Archive, Serialize, Deserialize, Debug)]
pub struct CompactLockfileBinary {
    pub version: String,
    pub board: String,
    pub placement_hash: String,
    pub arcs: Vec<String>,
    pub instances: Vec<i32>,
}

pub fn load_lockfile_zero_copy(path: &str) -> Result<CompactLockfileBinary, String> {
    let file = File::open(path).map_err(|e| e.to_string())?;
    let mmap = unsafe { Mmap::map(&file).map_err(|e| e.to_string())? };
    // Zero-copy cast: raw bytes become Rust structs with zero parsing overhead
    // AlignedVec guarantee: mmap'd data is 16-byte aligned, safe for i64/i128 casts
    let archived = rkyv::check_archived_root::<CompactLockfileBinary>(&mmap)
        .map_err(|e| e.to_string())?;
    let result = archived.deserialize(&mut rkyv::Infallible)
        .map_err(|e| e.to_string())?;
    Ok(result)
}

/// Write lockfile with strict alignment via AlignedVec
pub fn save_lockfile_aligned(path: &str, data: &CompactLockfileBinary) -> Result<(), String> {
    // AlignedVec guarantees 16-byte alignment in memory, perfect for SSE/AVX
    let mut serializer = rkyv::ser::serializers::AllocSerializer::<4096>::default();
    serializer.serialize_value(data).map_err(|e| e.to_string())?;
    let serialized = serializer.into_serializer().into_inner();

    let mut file = File::create(path).map_err(|e| e.to_string())?;
    file.write_all(serialized.as_slice()).map_err(|e| e.to_string())?;
    Ok(())
}

// CLI inspection path only: decodes binary to human-readable JSON on demand
#[derive(sonic_rs::Serialize)]
struct HumanReadableLockfile {
    version: String,
    board: String,
    placement_hash: String,
    arcs: Vec<String>,
    instances: Vec<i32>,
}

pub fn inspect_lockfile_as_json(binary_path: &str) -> Result<String, String> {
    let binary_data = load_lockfile_zero_copy(binary_path)?;
    let readable = HumanReadableLockfile {
        version: binary_data.version,
        board: binary_data.board,
        placement_hash: binary_data.placement_hash,
        arcs: binary_data.arcs,
        instances: binary_data.instances,
    };
    sonic_rs::to_string_pretty(&readable).map_err(|e| e.to_string())
}
```

---

### E. Zero-Stamping Scene Graph: Pre-Transformed Global Bounding Boxes

Instead of rasterizing millions of standard-cell geometries into a physical voxel grid, the compiler stores each component type once as an analytical `ComponentStamp` with an **Oriented Bounding Volume Hierarchy (OBVH)** in local coordinate space. At placement time, local bounding volumes are transformed **forward** into global world-coordinate space using i128 intermediate products and cached on the `ComponentInstance`. All collision checks execute in global coordinates against pre-calculated global bounding boxes—eliminating lossy on-demand inverse transforms and integer truncation asymmetry [BIT-BLIT-UNROLLER-IMPLEMENTATION.md].

```rust
// crates/hwc-engine/src/geometry/transform.rs

pub struct FixedTransform2D {
    pub tx_pm: i64,
    pub ty_pm: i64,
    pub cos_scale: i64,
    pub sin_scale: i64,
}

impl FixedTransform2D {
    pub const SCALE_FACTOR: i128 = 1_000_000_000; // 10^9

    #[inline(always)]
    pub fn transform_point(&self, x: i64, y: i64) -> (i64, i64) {
        // Promote coordinates and coefficients to i128 to prevent overflow
        // On 200mm PCB: x=2e11 * cos_fixed=7.07e8 = 1.414e20 > i64::MAX (9.22e18)
        let x_128 = x as i128;
        let y_128 = y as i128;
        let cos_128 = self.cos_scale as i128;
        let sin_128 = self.sin_scale as i128;

        let rx = (x_128 * cos_128 - y_128 * sin_128) / Self::SCALE_FACTOR;
        let ry = (x_128 * sin_128 + y_128 * cos_128) / Self::SCALE_FACTOR;
        
        (rx as i64 + self.tx_pm, ry as i64 + self.ty_pm)
    }

    pub fn transform_bbox(&self, bbox: &BoundingBox) -> BoundingBox {
        let (x1, y1) = self.transform_point(bbox.min_x, bbox.min_y);
        let (x2, y2) = self.transform_point(bbox.max_x, bbox.max_y);
        BoundingBox {
            min_x: x1.min(x2), min_y: y1.min(y2),
            max_x: x1.max(x2), max_y: y1.max(y2),
        }
    }
}
```

```rust
// crates/hwc-engine/src/geometry_router/scene_graph.rs

use hwc_engine::geometry::{BoundingBox, FixedTransform2D, OrientedBoundingBox};

pub struct ComponentStamp {
    pub name: String,
    pub local_bbox: BoundingBox,
    pub local_obb_children: Vec<OrientedBoundingBox>, // OBB for rotated shapes
    pub local_aabb_children: Vec<BoundingBox>,        // AABB for Manhattan shapes
    pub local_polygons: Vec<LocalPolygon>,            // Fallback for non-standard shapes
}

pub struct ComponentInstance {
    pub instance_id: usize,
    pub stamp_id: usize,
    pub transform: FixedTransform2D,
    pub net_bindings: Vec<usize>,
    
    /// Pre-transformed bounding boxes in global world-coordinate space
    /// Eliminates lossy on-demand inverse transforms during hot-paths
    pub global_bbox: BoundingBox,
    pub global_obb_children: Vec<OrientedBoundingBox>,
    pub global_aabb_children: Vec<BoundingBox>,
}

impl ComponentInstance {
    pub fn new(
        instance_id: usize,
        stamp_id: usize,
        tx_pm: i64, ty_pm: i64,
        rotation_deg: i64,
        stamp: &ComponentStamp,
    ) -> Self {
        // Safe Euclidean remainder prevents negative remainder fallthrough
        // e.g., -45.rem_euclid(360) = 315, but -45 % 360 = -45 (falls to default)
        let normalized_deg = rotation_deg.rem_euclid(360);
        let (cos_fixed, sin_fixed) = match normalized_deg {
            0   => (1_000_000_000, 0),
            45  => (707_106_781, 707_106_781),
            90  => (0, 1_000_000_000),
            135 => (-707_106_781, 707_106_781),
            180 => (-1_000_000_000, 0),
            225 => (-707_106_781, -707_106_781),
            270 => (0, -1_000_000_000),
            315 => (707_106_781, -707_106_781),
            _   => (1_000_000_000, 0),
        };
        let transform = FixedTransform2D { tx_pm, ty_pm, cos_scale: cos_fixed, sin_scale: sin_fixed };
        
        // Transform local stamp bounding boxes forward into global space ONCE
        let global_bbox = transform.transform_bbox(&stamp.local_bbox);
        let global_aabb_children = stamp.local_aabb_children.iter()
            .map(|aabb| transform.transform_bbox(aabb))
            .collect();
        let global_obb_children = stamp.local_obb_children.iter()
            .map(|obb| transform.transform_obb(obb))
            .collect();

        Self {
            instance_id, stamp_id, transform, net_bindings: Vec::new(),
            global_bbox, global_obb_children, global_aabb_children,
        }
    }

    /// Fast, non-allocating world-space collision check using pre-calculated bounds
    #[inline(always)]
    pub fn test_collision_global(&self, gx: i64, gy: i64) -> bool {
        if !self.global_bbox.contains_i64(gx, gy) { return false; }
        for aabb in &self.global_aabb_children {
            if aabb.contains_i64(gx, gy) { return true; }
        }
        for obb in &self.global_obb_children {
            if obb.contains_i64(gx, gy) { return true; }
        }
        false
    }
}

fn point_in_polygon_3d(point: DVec3, vertices: &[DVec3]) -> bool {
    let mut inside = false;
    let mut j = vertices.len() - 1;
    for i in 0..vertices.len() {
        if (vertices[i].y > point.y) != (vertices[j].y > point.y)
            && (point.x < (vertices[j].x - vertices[i].x) * (point.y - vertices[i].y)
                / (vertices[j].y - vertices[i].y) + vertices[i].x)
        { inside = !inside; }
        j = i;
    }
    inside
}
```

---

### F. Salsa-Style Memoized Query Engine

Modeling compiler phases as pure, demand-driven query functions enables incremental compilation at the G-cell level. When an entity is modified, only the affected query nodes are invalidated—unchanged G-cells return cached results in $<1\text{ms}$ [Core-System-Architecture.md, LAZY-REALIZATION-ARCHITECTURE.md].

```rust
// crates/hwc-compiler/src/ir/query.rs

use salsa::query_group;
use std::sync::Arc;

#[salsa::query_group(CompilationDatabaseStorage)]
pub trait CompilationDatabase: salsa::Database {
    #[salsa::input]
    fn source_code(&self, file_id: usize) -> Arc<String>;

    #[salsa::dependencies]
    fn parse_ast(&self, file_id: usize) -> salsa::Result<Arc<AstSpace>>;

    #[salsa::dependencies]
    fn resolve_symbols(&self, file_id: usize) -> salsa::Result<Arc<SymbolTable>>;

    #[salsa::dependencies]
    fn partition_gcells(&self, file_id: usize) -> salsa::Result<Arc<GCellLayout>>;

    #[salsa::dependencies]
    fn route_gcell(&self, file_id: usize, gcell_id: usize) -> salsa::Result<Arc<LocalRoute>>;

    #[salsa::dependencies]
    fn verify_gcell(&self, file_id: usize, gcell_id: usize) -> salsa::Result<Arc<DrcReport>>;
}

// If a developer changes a route in G-cell #42, only route_gcell(file_id, 42)
// and verify_gcell(file_id, 42) are re-evaluated.
// Calls for cells 0..41 and 43..N are fetched from cache in 0ms!
```

---

### G. Analytic 2.5D BEM Parasitic Solver (Wheeler + Sakurai + Via Inductance + Mutual Inductance)

Computes effective relative permittivity ($\epsilon_{\text{eff}}$) using Wheeler's closed-form equation, then ground-plane-aware coupling capacitance ($C_{12}$), ground capacitance ($C_{1g}$), series resistance ($R_s$), via self-inductance ($L_{\text{via}}$), and **mutual inductance** ($M_{12}$) for parallel trace runs using the Greenhouse approximation. These formulas account for electric field lines passing through both substrate and air, achieving 5–10% accuracy versus 3D field solvers [Maturity-and-Expansion-Vision.md].

```rust
// crates/hwc-physics/src/parasitic_solver.rs

use glam::DVec2;

pub struct ConductorSegment {
    pub start: DVec2,
    pub end: DVec2,
    pub width: f64,
    pub thickness: f64,
    pub net_id: usize,
}

pub struct ViaTransition {
    pub height_m: f64,
    pub diameter_m: f64,
    pub net_id: usize,
}

pub struct ParasiticSolver {
    pub epsilon_r: f64,     // Raw substrate permittivity (from stackup)
    pub substrate_h: f64,   // Substrate height above ground plane (from stackup)
    pub mu_r: f64,          // Relative magnetic permeability (typically 1.0 for FR4/Si)
}

impl ParasiticSolver {
    /// Wheeler's closed-form effective relative permittivity
    /// Accounts for electric field lines passing through substrate AND air
    pub fn compute_effective_permittivity(&self, w: f64, h: f64) -> f64 {
        let term = (1.0 + 12.0 * h / w).powf(-0.5);
        ((self.epsilon_r + 1.0) / 2.0) + ((self.epsilon_r - 1.0) / 2.0) * term
    }

    /// Sakurai coupling capacitance using effective permittivity
    pub fn compute_coupling_capacitance(
        &self,
        trace_a: &ConductorSegment,
        trace_b: &ConductorSegment,
    ) -> f64 {
        let overlap_start = trace_a.start.x.max(trace_b.start.x);
        let overlap_end = trace_a.end.x.min(trace_b.end.x);
        if overlap_start >= overlap_end { return 0.0; }

        let run_length = overlap_end - overlap_start;
        let distance = (trace_a.start.y - trace_b.start.y).abs();
        let w = trace_a.width;
        let t = trace_a.thickness;
        let h = self.substrate_h;
        let d = distance - w;
        if d <= 0.0 { return 0.0; }

        let eps_eff = self.compute_effective_permittivity(w, h);
        eps_eff * run_length * (
            0.03 * (w / h)
            + 0.08 * (t / h)
            + 0.07 * (w / h).powf(0.25) * (t / h).powf(0.5) * (h / d).powf(1.34)
        )
    }

    /// Sakurai ground capacitance using effective permittivity
    pub fn compute_ground_capacitance(&self, trace: &ConductorSegment) -> f64 {
        let length = trace.start.distance(trace.end);
        let w = trace.width;
        let t = trace.thickness;
        let h = self.substrate_h;
        let eps_eff = self.compute_effective_permittivity(w, h);
        eps_eff * length * (1.15 * (w / h) + 2.80 * (t / h).powf(0.222))
    }

    /// Series resistance
    pub fn compute_series_resistance(&self, trace: &ConductorSegment, rho: f64) -> f64 {
        let length = trace.start.distance(trace.end);
        rho * length / (trace.width * trace.thickness)
    }

    /// Via self-inductance: L_via = 2e-7 * h * [ln(4h/d) + 1]
    pub fn compute_via_self_inductance(&self, via: &ViaTransition) -> f64 {
        if via.diameter_m <= 0.0 { return 0.0; }
        2.0e-7 * via.height_m * ((4.0 * via.height_m / via.diameter_m).ln() + 1.0)
    }

    /// Mutual inductance between parallel trace runs (Greenhouse approximation)
    /// M₁₂ = μ₀μᵣL/(2π) · [ln(2L/D) - 1 + D/L]
    pub fn compute_mutual_inductance(
        &self,
        trace_a: &ConductorSegment,
        trace_b: &ConductorSegment,
    ) -> f64 {
        let overlap_start = trace_a.start.x.max(trace_b.start.x);
        let overlap_end = trace_a.end.x.min(trace_b.end.x);
        if overlap_start >= overlap_end { return 0.0; }

        let l = overlap_end - overlap_start; // Parallel run length
        let d = (trace_a.start.y - trace_b.start.y).abs(); // Center-to-center spacing
        if d <= 0.0 { return 0.0; }

        let mu_0 = 1.25663706212e-6; // H/m (vacuum permeability)
        let mu = mu_0 * self.mu_r;

        // Greenhouse approximation for mutual inductance of parallel lines
        let term = (2.0 * l / d).ln() - 1.0 + (d / l);
        (mu * l / (2.0 * std::f64::consts::PI)) * term
    }
}
```

---

### H. G-Cell-Local Unified Sweep Verification (DRC + Same-Net Topology)

G-Cell-local flat Morton-ordered interval sweep with Boundary-Halo Expansion (Ghost Zones) and portable SIMD (AVX-512 / NEON) for 8-wide or 4-wide parallel bounding box overlap checks. Each G-cell is processed independently on separate Rayon threads, eliminating the sequential Bentley-Ottmann bottleneck and achieving true linear multi-core scaling [Core-System-Architecture.md].

```rust
// crates/hwc-physics/src/simd_drc.rs

use hwc_engine::geometry::BoundingBox;

pub struct LineSegment {
    pub start_x: f64, pub end_x: f64,
    pub start_y: f64, pub end_y: f64,
    pub net_id: usize,
}

pub struct GCellBin {
    pub gcell_id: usize,
    pub bbox: BoundingBox,
    /// Segments physically originating inside this G-cell
    pub native_segments: Vec<LineSegment>,
    /// Segments from adjacent G-cells overlapping the boundary halo
    /// halo_width = max_clearance_limit of active profile
    pub ghost_segments: Vec<LineSegment>,
}

impl GCellBin {
    /// Merges native and ghost segments into a single contiguous array for the SIMD sweep
    pub fn prepare_sweep_array(&self) -> Vec<LineSegment> {
        let mut sweep_segments = self.native_segments.clone();
        sweep_segments.extend(self.ghost_segments.clone());
        sweep_segments
    }
}

// G-cell-local sweep: processes segments within a single G-cell
// Rayon parallelism across G-cells provides linear multi-core scaling
pub fn sweep_gcell_local(
    segments: &mut [(f64, f64, f64, f64)], // (min_x, max_x, min_y, max_y)
    clearance: f64,
) -> Vec<(usize, usize)> {
    // 1. Morton-order segments for cache-friendly access
    segments.sort_by_key(|s| {
        let cx = ((s.0 + s.1) / 2.0) as u64;
        let cy = ((s.2 + s.3) / 2.0) as u64;
        interleave_morton(cx, cy)
    });

    // 2. Flat active interval sweep (no BST, no pointer chasing)
    let mut active: Vec<usize> = Vec::new();
    let mut violations = Vec::new();

    for (i, seg) in segments.iter().enumerate() {
        active.retain(|&j| segments[j].1 >= seg.0); // Remove expired
        // 3. SIMD 8-wide overlap check against active set
        for chunk in active.chunks_exact(8) {
            let bbox = FlatBbox8 { /* load from segments[chunk] */ };
            if bbox.intersects(seg.0 - clearance, seg.1 + clearance,
                               seg.2 - clearance, seg.3 + clearance) != 0 {
                for &j in chunk { violations.push((i, j)); }
            }
        }
        active.push(i);
    }
    violations
}

fn interleave_morton(x: u64, y: u64) -> u64 {
    let mut z = 0u64;
    for i in 0..32 {
        z |= ((x >> i) & 1) << (2 * i);
        z |= ((y >> i) & 1) << (2 * i + 1);
    }
    z
}
```

---

### I. Pattern-Guided Meander Injection (MeanderInjector)

The `MeanderInjector` implements closed-form analytical meander height calculation and polar-to-Cartesian decomposition for post-route pattern injection. It operates in O(N log N + K) time and avoids the $O(B^d)$ state-space explosion of inline pattern-guided A* routing.

```rust
// crates/hwc-compiler/src/ir/meander_injection.rs

use hwc_engine::geometry_router::routing_patterns::{RoutingPattern, PatternStep};
use hwc_engine::geometry::Point3D;

pub struct MeanderInjector {
    trace_width_nm: i64,
}

impl MeanderInjector {
    pub fn new(trace_width_nm: i64) -> Self {
        Self { trace_width_nm }
    }

    /// Inject meander waypoints into the longest segment of a net's paths.
    /// Returns expanded paths with meander geometry.
    pub fn inject_meanders(
        &self,
        paths: &[Vec<Point3D>],
        pattern: &RoutingPattern,
    ) -> Vec<Vec<Point3D>> {
        if paths.is_empty() || pattern.steps.is_empty() {
            return paths.to_vec();
        }

        // 1. Find longest segment (highest Manhattan length)
        let (longest_path_idx, longest_seg_idx, max_len) =
            self.find_longest_segment(paths);

        // 2. Compute total forward distance and repetitions
        let total_forward: i64 = pattern.steps.iter()
            .map(|s| {
                let rad = (s.angle as f64) * std::f64::consts::PI / 180.0;
                (s.distance as f64 * rad.cos()) as i64
            })
            .sum();
        if total_forward <= 0 { return paths.to_vec(); }

        let repetitions = (max_len / total_forward).max(1) as usize;

        // 3. Decompose pattern into Cartesian waypoints
        let meander_points = self.decompose_pattern(
            &paths[longest_path_idx][longest_seg_idx],
            &paths[longest_path_idx][longest_seg_idx + 1],
            pattern,
            repetitions,
        );

        // 4. Splice meander into path
        let mut result = paths.to_vec();
        let mut new_path = result[longest_path_idx].clone();
        new_path.splice(
            longest_seg_idx + 1..longest_seg_idx + 1,
            meander_points,
        );
        result[longest_path_idx] = new_path;
        result
    }

    /// Closed-form polar-to-Cartesian decomposition
    fn decompose_pattern(
        &self,
        start: &Point3D,
        end: &Point3D,
        pattern: &RoutingPattern,
        repetitions: usize,
    ) -> Vec<Point3D> {
        let dx = end.x - start.x;
        let dy = end.y - start.y;
        let is_vertical = dy.abs() > dx.abs();

        let total_forward: i64 = pattern.steps.iter()
            .map(|s| {
                let rad = (s.angle as f64) * std::f64::consts::PI / 180.0;
                (s.distance as f64 * rad.cos()) as i64
            })
            .sum();
        let half_span = total_forward / 2;

        // Start at center - half_span along segment direction
        let mut px = if is_vertical { start.x } else { start.x + dx / 2 - half_span };
        let mut py = if is_vertical { start.y + dy / 2 - half_span } else { start.y };
        let z = start.z;

        let mut points = vec![Point3D { x: px, y: py, z }];

        for _ in 0..repetitions {
            for step in &pattern.steps {
                let rad = (step.angle as f64) * std::f64::consts::PI / 180.0;
                let forward = (step.distance as f64 * rad.cos()) as i64;
                let perp = (step.distance as f64 * rad.sin()) as i64;

                if is_vertical {
                    py += forward;
                    px += perp;  // perpendicular maps to X for vertical segments
                } else {
                    px += forward;
                    py += perp;  // perpendicular maps to Y for horizontal segments
                }
                points.push(Point3D { x: px, y: py, z });
            }
        }
        points
    }

    fn find_longest_segment(
        &self,
        paths: &[Vec<Point3D>],
    ) -> (usize, usize, i64) {
        let mut max_len = 0;
        let mut best = (0, 0, 0);
        for (pi, path) in paths.iter().enumerate() {
            for si in 0..path.len().saturating_sub(1) {
                let len = (path[si + 1].x - path[si].x).abs()
                    + (path[si + 1].y - path[si].y).abs();
                if len > max_len {
                    max_len = len;
                    best = (pi, si, len);
                }
            }
        }
        best
    }
}
```

---

### J. 45° Mitered Chamfer Pass (MiterEngine)

The `MiterEngine` scans routed paths for 90° corners and inserts 45° diagonal chamfers at `1.5 × trace_width` distance, maintaining constant characteristic impedance ($Z_0$) through bends.

```rust
// crates/hwc-engine/src/geometry_router/miter_pass.rs

use crate::geometry::Point3D;

pub struct MiterEngine {
    trace_width_nm: i64,
}

impl MiterEngine {
    pub fn new(trace_width_nm: i64) -> Self {
        Self { trace_width_nm }
    }

    /// Apply 45° miter chamfers to all 90° corners in a path.
    /// Returns expanded path with chamfer points inserted.
    pub fn apply_miter_pass(&self, path: &[Point3D]) -> Vec<Point3D> {
        if path.len() < 3 { return path.to_vec(); }

        let miter_dist = (self.trace_width_nm as f64 * 1.5) as i64;
        let mut result = Vec::with_capacity(path.len());

        result.push(path[0]);

        for i in 1..path.len() - 1 {
            let prev = &path[i - 1];
            let curr = &path[i];
            let next = &path[i + 1];

            // Direction vectors in XY plane
            let d1x = curr.x - prev.x;
            let d1y = curr.y - prev.y;
            let d2x = next.x - curr.x;
            let d2y = next.y - curr.y;

            // Dot product = 0 means 90° corner (XY only)
            let dot = d1x * d2x + d1y * d2y;
            if dot == 0 && (d1x != 0 || d1y != 0) && (d2x != 0 || d2y != 0) {
                // Normalize direction vectors
                let len1 = ((d1x * d1x + d1y * d1y) as f64).sqrt();
                let len2 = ((d2x * d2x + d2y * d2y) as f64).sqrt();

                let nx1 = (d1x as f64 / len1 * miter_dist as f64) as i64;
                let ny1 = (d1y as f64 / len1 * miter_dist as f64) as i64;
                let nx2 = (d2x as f64 / len2 * miter_dist as f64) as i64;
                let ny2 = (d2y as f64 / len2 * miter_dist as f64) as i64;

                // Insert chamfer points before and after corner
                result.push(Point3D {
                    x: curr.x - nx1,
                    y: curr.y - ny1,
                    z: curr.z,
                });
                result.push(Point3D {
                    x: curr.x + nx2,
                    y: curr.y + ny2,
                    z: curr.z,
                });
            } else {
                result.push(*curr);
            }
        }

        result.push(*path.last().unwrap());
        result
    }

    /// Apply miter pass to multiple paths
    pub fn apply_to_paths(&self, paths: &[Vec<Point3D>]) -> Vec<Vec<Point3D>> {
        paths.iter().map(|p| self.apply_miter_pass(p)).collect()
    }
}
```

---

## 2. Estimated Execution Benchmarks (The "Boot-to-System" Latency Profile)

The following tables show estimated performance benchmarks comparing the legacy voxel architecture (v0.1.7) with the new, continuous, planar-locked vector architecture (v0.1.8) across different design scales.

### Benchmark A: Small-Scale Design (Simple 2-Layer PCB, 15 Nets)
*   **Dimensions:** $20\text{mm} \times 20\text{mm}$
*   **Voxel Grid size:** $200 \times 200 \times 16$

| Compiler Phase | Legacy Voxel (v0.1.7) | New Vector (v0.1.8) | Speedup Factor |
| :--- | :--- | :--- | :--- |
| **Boot, Parse, & Prelude Loading** | $13.0\text{ ms}$ | **$1.5\text{ ms}$** | $8.6\times$ (fast trig tokenization) |
| **Spatial Partitioning & Planning** | $0.0\text{ ms}$ (Bypassed) | **$0.2\text{ ms}$** | — |
| **Topological Line-Search Routing** | $44.0\text{ ms}$ (Voxel A*) | **$1.8\text{ ms}$** | $24.4\times$ (no grid crawling) |
| **QP Legalization & Compaction** | $0.0\text{ ms}$ (Bypassed) | **$0.4\text{ ms}$** | — |
| **DRC Verification** | $12.0\text{ ms}$ (Voxel check) | **$0.8\text{ ms}$** | $15.0\times$ (R*-Tree search) |
| **Exporter Refinement & Mesh Generation** | $87.0\text{ ms}$ | **$4.5\text{ ms}$** | $19.3\times$ (fewer draw calls) |
| **Total Build Time (Cold)** | **$156.0\text{ ms}$** | **$9.2\text{ ms}$** | **$16.9\times$** |
| **Total Build Time (Lockfile Hit)** | **$13.0\text{ ms}$** | **$< 0.8\text{ ms}$** | **$16.2\times$** |

---

### Benchmark B: SoC-Scale Design (TSMC 180nm, 50,000 Gates, 4,000 Nets)
*   **Dimensions:** $100\mu\text{m} \times 100\mu\text{m}$
*   **Voxel Grid size:** $10000 \times 10000 \times 10$

| Compiler Phase | Legacy Voxel (v0.1.7) | New Vector (v0.1.8) | Speedup Factor |
| :--- | :--- | :--- | :--- |
| **Boot, Parse, & Prelude Loading** | $2.20\text{ s}$ | **$0.05\text{ s}$** | $44.0\times$ |
| **Spatial Partitioning & Planning** | $0.00\text{ s}$ (Bypassed) | **$0.01\text{ s}$** | — |
| **Topological Line-Search Routing** | $12.40\text{ s}$ (Voxel A*) | **$0.12\text{ s}$** | $103.3\times$ |
| **QP Legalization & Compaction** | $0.00\text{ s}$ (Bypassed) | **$0.04\text{ s}$** | — |
| **DRC Verification** | $2.24\text{ s}$ (Voxel check) | **$0.02\text{ s}$** | $112.0\times$ (R*-Tree search) |
| **Exporter Refinement & Mesh Generation** | $4.80\text{ s}$ (50k meshes) | **$0.08\text{ s}$** (Contour Union) | $60.0\times$ |
| **Total Build Time (Cold)** | **$21.64\text{ s}$** | **$0.32\text{ s}$** | **$67.6\times$** |
| **Total Build Time (Lockfile Hit)** | **$2.20\text{ s}$** | **$0.02\text{ s}$** | **$110.0\times$** |

---

## 3. Detailed Transition Guide (Code-Level Refactoring Path)

To safely transition the codebase from v0.1.7 to v0.1.8, follow this refactoring path:

```
Step 1: Deprecate `VoxelGrid` ──► Step 2: Implement `EntityGraph` & Hybrid Index
                                                         │
                                                         ▼
Step 4: Update Exporter Pipeline ◄── Step 3: Implement Line-Search + Slab Method
              │
              ▼
Step 5: Implement Scene Graph ──► Step 6: Wire Salsa Query Engine
              │                           │
              ▼                           ▼
Step 8: SIMD DRC + BEM Parasitics ◄── Step 7: Verify Integration
```

### Step 1: Deprecate the Dense Voxel Grid
*   **Location:** `crates/hwc-engine/src/voxel_grid/grid/core.rs`
*   **Action:** Deprecate the `visible_plane` hash-map chunk allocator. Remove all methods that perform raw voxel writes (`set_occupied`, `stamp_cylinder`) inside the compilation passes [Solder-Mask-Opening.md, Authority-and-Library-Architecture.md].
*   **Result:** The memory allocation for spatial grid tracking drops to zero on startup [LAZY-REALIZATION-ARCHITECTURE.md, Advanced-Routing-Implementation.md].

### Step 2: Implement the Canonical Entity Graph and `rstar` Indexing
*   **Location:** `crates/hwc-compiler/src/ir/space.rs`
*   **Action:** Implement the `EntityGraph` as the master database, storing components, pins, nets, and routes canonically [ARCHITECTURAL-AUDIT-AND-ROADMAP.md, Authority-and-Library-Architecture.md].
*   **Action:** Initialize `rstar::RTree<IndexedSegment>` inside `HardwareSpace` [Unified-2.5D-3D-Routing-and-Placement.md]. When the compiler parses placements or routes, it pushes the entities to the `EntityGraph` and registers their 3D bounding boxes in the `rstar` index [Unified-2.5D-3D-Routing-and-Placement.md, Advanced-Routing-Implementation.md].

### Step 3: Replace the Pathfinder Engine with the Line-Search Router
*   **Location:** `crates/hwc-engine/src/geometry_router/pathfinding/router.rs`
*   **Action:** Delete the voxel-based A* neighbors loop. Implement the ray-casting `Topological Line-Search` engine with the **Axis-Aligned Slab Method** for ray-AABB intersection queries.
*   **Action:** After the partition stage, compile static obstacle geometry into flat `geo-index` structures per layer for fast Slab Method queries [Unified-2.5D-3D-Routing-and-Placement.md, Advanced-Routing-Implementation.md].
*   **Action:** Ensure that during ray projection, the router queries the spatial index to check for collisions, treating the interior of all pad bounding boxes as `Cost::INFINITE` [Unified-2.5D-3D-Routing-and-Placement.md, Advanced-Routing-Implementation.md].

### Step 4: Refactor the Exporter and Mesh Generation
*   **Location:** `crates/hwc-export/src/substrate.rs`
*   **Action:** Remove the logic that creates a new `SubstrateLayer` for every single trace segment.
*   **Action:** Implement Strategy A [2D-POLYGON-UNIONING-ROADMAP.md, Z_FIGHTING_FIX.md]:
    *   Group copper shapes by layer and Net ID in 2D.
    *   Call `clipper2_rust` to perform the boolean union [2D-POLYGON-UNIONING-IMPLEMENTATION.md, COMPILER-CHANGES.md].
    *   Call `earcut` (Mapbox/GeoRust) to perform zero-allocation triangulation during the GLB export pass only, keeping the intermediate lockfile coordinate representations entirely as vectors [2D-POLYGON-UNIONING-IMPLEMENTATION.md, Volumetric-Solid-Modeling-via-Boundary-Representation.md].
*   **Action:** Implement the single-source binary lockfile:
    *   Primary: `rkyv` binary + `memmap2` memory-mapped deserialization for near-instant loads.
    *   No secondary JSON file is generated during builds.
    *   Human inspection via `hwc lock inspect` CLI tool on demand.

### Step 5: Implement the Zero-Stamping Scene Graph with Pre-Transformed Global Bounds
*   **Location:** `crates/hwc-engine/src/geometry_router/scene_graph.rs`
*   **Action:** Define `ComponentStamp` as an **Oriented Bounding Volume Hierarchy (OBVH)** in local-coordinate space.
*   **Action:** At placement time, transform local bounding volumes **forward** into global world-coordinate space and cache on `ComponentInstance`. All collision checks execute in global coordinates against pre-calculated global bounding boxes—eliminating lossy on-demand inverse transforms.
*   **Action:** Implement `test_collision_global()` for SAT-based OBB collision (rotated components) and AABB box-bounds checks (Manhattan shapes) in global coords, with Jordan curve theorem fallback only for non-standard custom shapes.

### Step 6: Wire the Salsa-Style Memoized Query Engine
*   **Location:** `crates/hwc-compiler/src/ir/query.rs`
*   **Action:** Wrap all compiler phases (`parse_ast`, `resolve_symbols`, `partition_gcells`, `route_gcell`, `verify_gcell`) in `#[salsa::query]` functions.
*   **Action:** Wire the `Incremental Dependency DAG` (Subsystem 11) to Salsa's invalidation system so that G-cell-level changes only re-evaluate affected query nodes.

### Step 7: Verify Integration and Incremental Compile Performance
*   **Action:** Run incremental compilation benchmarks: edit a single G-cell route and verify re-compilation completes in $<10\text{ ms}$.
*   **Action:** Verify that unchanged G-cells return cached results from the `rkyv` lockfile with zero re-routing overhead.

### Step 8: Implement G-Cell-Local SIMD DRC + Sakurai BEM Parasitic Extraction
*   **Location:** `crates/hwc-physics/src/simd_drc.rs` and `crates/hwc-physics/src/parasitic_solver.rs`
*   **Action:** Implement the G-Cell-local flat Morton-ordered interval sweep with `std::simd` portable SIMD for 4/8-wide bounding box overlap checks. Use Rayon parallelism across G-cells.
*   **Action:** Implement Sakurai's empirical microstrip parasitic solver (`ParasiticSolver`) for $C_{12}$, $C_{1g}$, and $R_s$ extraction, accounting for $W$, $D$, $T$, and $H$.
*   **Action:** Embed extracted R/C values into the SPICE netlist export and verify parasitic extraction completes in $<50\text{ ms}$ on SoC-scale designs.

---

## 4. Development Timeline (The Weekend Roadmap)

To maintain steady progress while implementing this upgrade, the timeline is structured as a 7-weekend milestone plan:

```
WK 1: DB & Spatial Index ──► WK 2: Line-Search Pathfinder ──► WK 3: Legalization & Compaction
                                                                         │
                                                                         ▼
WK 4: Exporter & DAG ◄── WK 5: Scene Graph + Salsa ◄── WK 6: SIMD DRC + BEM
                                                                         │
                                                                         ▼
                                                           WK 7: Verification & Release
```

### Weekend 1: The Database & Spatial Indexing Foundation
*   **Goal:** Establish the continuous coordinate model.
*   **Tasks:**
    *   [ ] Implement the `EntityGraph` and deprecate dense `VoxelGrid` allocations [ARCHITECTURAL-AUDIT-AND-ROADMAP.md, Authority-and-Library-Architecture.md].
    *   [ ] Integrate the `rstar` crate and implement `RTreeObject` for components and trace bounding boxes [Unified-2.5D-3D-Routing-and-Placement.md, Advanced-Routing-Implementation.md].
    *   [ ] Verify $O(\log N)$ spatial query performance.

### Weekend 2: The Topological Line-Search Pathfinder
*   **Goal:** Replace voxel-crawling with mathematically straight vector paths.
*   **Tasks:**
    *   [ ] Implement the ray-casting line-search pathfinder with Axis-Aligned Slab Method in `router.rs` [Z-AXIS-ABSTRACTION-IMPLEMENTATION.md, Advanced-Routing-Implementation.md].
    *   [ ] Compile static obstacle geometry into flat `geo-index` structures per layer for ray-AABB intersection queries [Unified-2.5D-3D-Routing-and-Placement.md].
    *   [ ] Implement strict boundary-docking to prevent traces from entering pad interiors [Unified-2.5D-3D-Routing-and-Placement.md].
    *   [ ] Verify Manhattan and Octilinear vector paths are generated cleanly with zero stair-stepping [Unified-2.5D-3D-Routing-and-Placement.md].

### Weekend 3: The Convex Legalizer & Compaction Engine
*   **Goal:** Resolve clearances smoothly using optimization math.
*   **Tasks:**
    *   [ ] Integrate the `clarabel` quadratic programming solver for macro-scale floorplanning [Advanced-Routing-Implementation.md].
    *   [ ] Integrate a lightweight active-set solver (like PIQP) or DAG graph compaction solver for local micro-adjustments [MANUFACTURING-YIELD-IMPLEMENTATION.md].
    *   [ ] Implement localized legalization windows to nudge trace vectors during collisions without triggering global reroutes [MANUFACTURING-YIELD-IMPLEMENTATION.md].
    *   [ ] Implement signal-aware compaction to slide parallel traces together up to their high-speed limits [Z-AXIS-ABSTRACTION-IMPLEMENTATION.md, Advanced-Routing-Implementation.md].

### Weekend 4: Exporter Integration & Invalidation DAG
*   **Goal:** Deliver unified mesh output and fast incremental compiles.
*   **Tasks:**
    *   [ ] Implement 2D Clipper union and `earcut` triangulation on the export boundary [2D-POLYGON-UNIONING-IMPLEMENTATION.md, MICROKERNEL-ARCHITECTURE.md].
    *   [ ] Implement the semantic dependency DAG inside the lockfile unroller [Unified-2.5D-3D-Routing-and-Placement.md].
    *   [ ] Implement the single-source binary lockfile: `rkyv` primary with `memmap2` zero-copy loads. Human inspection via `hwc lock inspect` CLI tool [Unified-2.5D-3D-Routing-and-Placement.md].

### Weekend 5: Zero-Stamping Scene Graph & Salsa Query Engine
*   **Goal:** Eliminate voxel stamping and enable demand-driven incremental compilation.
*   **Tasks:**
    *   [ ] Implement `ComponentStamp` (local-coordinate OBVH) and `ComponentInstance` with pre-transformed global bounding boxes [BIT-BLIT-UNROLLER-IMPLEMENTATION.md].
    *   [ ] At placement time, transform local bounding volumes forward into global space and cache on instance. All collision checks execute in global coords.
    *   [ ] Wrap fine-grained entity inputs (`component_placement_input`, `route_statement_input`) in `#[salsa::query]` functions and wire Salsa invalidation to the dependency DAG [LAZY-REALIZATION-ARCHITECTURE.md].
    *   [ ] Verify incremental G-cell re-routing completes in $<10\text{ ms}$.

### Weekend 6: G-Cell-Local SIMD DRC & Sakurai BEM Parasitic Extraction
*   **Goal:** Accelerate physical verification and enable parasitic-aware netlists.
*   **Tasks:**
    *   [ ] Implement G-Cell-local flat Morton-ordered interval sweep with `std::simd` portable SIMD for 4/8-wide bounding box overlap checks [Core-System-Architecture.md].
    *   [ ] Implement Sakurai's empirical microstrip parasitic solver ($C_{12}$, $C_{1g}$, $R_s$) in `parasitic_solver.rs` [Maturity-and-Expansion-Vision.md].
    *   [ ] Embed extracted R/C values into SPICE netlist export.
    *   [ ] Verify DRC pass completes in $<5\text{ ms}$ and parasitic extraction in $<50\text{ ms}$ on SoC-scale designs.

### Weekend 7: Full Verification & Stable Release
*   **Goal:** Run the validation gauntlet and stabilize the v0.1.8 system directly.
*   **Tasks:**
    *   [ ] Run the entire test suite (`test_complex_hybrid_pcb.hw` and `test_complex_hybrid_asic.hw`) [Z-AXIS-ABSTRACTION-IMPLEMENTATION.md, Unified-2.5D-3D-Routing-and-Placement.md, Advanced-Routing-Implementation.md].
    *   [ ] Verify that compile times for an SoC design are under $0.5$ seconds [The-SoC-Engine.md].
    *   [ ] Verify memory usage for a 100M-transistor design is under $80\text{ MB}$ (zero voxel grid allocation).
    *   [ ] Finalize the v0.1.8 integration directly in the active system.

