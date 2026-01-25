# Geometry Module & PyO3 Integration Guide

**Purpose:** Comprehensive guide for understanding the geometry module, exposing it to PyO3, and adding new primitive shapes like sphere and cube.

---

## Table of Contents

1. [Geometry Module Overview](#geometry-module-overview)
2. [Current PyO3 Exposure](#current-pyo3-exposure)
3. [Adding New Primitives](#adding-new-primitives)
4. [Implementation Examples](#implementation-examples)
5. [Best Practices](#best-practices)

---

## Geometry Module Overview

### Architecture

The geometry module (`crates/processing_render/src/geometry/`) provides a **retained-mode** representation of 3D mesh data. Unlike the immediate-mode drawing API (where shapes are drawn and forgotten each frame), geometry objects persist and can be efficiently rendered multiple times.

**Key Components:**

1. **`Geometry` Component** - Stores mesh handle, layout, and current attribute values
2. **`VertexLayout`** - Defines which vertex attributes are present (position, normal, color, UV, custom)
3. **`Attribute`** - Defines a single vertex attribute with name and format
4. **`Topology`** - Primitive topology (PointList, LineList, TriangleList, etc.)

### Core Data Flow

```
User creates Geometry
  → Empty mesh created with specified topology
  → Default layout spawned (position, normal, color, UV)
  → Geometry component stores mesh handle and layout entity
  → User adds vertices/indices via API
  → Mesh data updated in Bevy Assets<Mesh>
  → Geometry can be drawn via DrawCommand::Geometry
```

### Current Implementation

**Location:** `crates/processing_render/src/geometry/mod.rs`

**Key Functions:**
- `create()` - Create empty geometry with default layout
- `create_with_layout()` - Create geometry with custom vertex layout
- `create_box()` - Create box primitive (existing example)
- `vertex()` - Add vertex with current attribute values
- `index()` - Add index for indexed rendering
- Attribute getters/setters (positions, normals, colors, UVs)

**Box Primitive Example:**

```rust
pub fn create_box(
    In((width, height, depth)): In<(f32, f32, f32)>,
    mut commands: Commands,
    mut meshes: ResMut<Assets<Mesh>>,
    builtins: Res<BuiltinAttributes>,
) -> Entity {
    let cuboid = Cuboid::new(width, height, depth);
    let mesh = cuboid.mesh().build();
    let handle = meshes.add(mesh);

    let layout_entity = commands
        .spawn(VertexLayout::with_attributes(vec![
            builtins.position,
            builtins.normal,
            builtins.color,
            builtins.uv,
        ]))
        .id();

    commands.spawn(Geometry::new(handle, layout_entity)).id()
}
```

**Key Pattern:**
1. Create Bevy primitive (e.g., `Cuboid`)
2. Convert to mesh via `.mesh().build()`
3. Add mesh to `Assets<Mesh>`
4. Create layout entity with standard attributes
5. Spawn `Geometry` component with mesh handle and layout

---

## Current PyO3 Exposure

### Python API Structure

**Location:** `crates/processing_pyo3/src/graphics.rs`

### Geometry Class

```rust
#[pyclass(unsendable)]
pub struct Geometry {
    entity: Entity,
}
```

**Why `unsendable`?** Geometry contains an `Entity` which is not `Send` (can't be moved between threads). This is safe because libprocessing is single-threaded.

### Current Methods

**Constructor:**
```rust
#[new]
#[pyo3(signature = (**kwargs))]
pub fn new(kwargs: Option<&Bound<'_, PyDict>>) -> PyResult<Self> {
    let topology = kwargs
        .and_then(|k| k.get_item("topology").ok().flatten())
        .and_then(|t| t.cast_into::<Topology>().ok())
        .and_then(|t| geometry::Topology::from_u8(t.borrow().as_u8()))
        .unwrap_or(geometry::Topology::TriangleList);

    let geometry = geometry_create(topology)
        .map_err(|e| PyRuntimeError::new_err(format!("{e}")))?;
    Ok(Self { entity: geometry })
}
```

**Usage in Python:**
```python
# Create with default topology (TriangleList)
geometry = Geometry()

# Create with specific topology
geometry = Geometry(topology=Topology.TriangleStrip)
```

**Vertex Manipulation:**
- `color(r, g, b, a)` - Set current color for subsequent vertices
- `normal(nx, ny, nz)` - Set current normal for subsequent vertices
- `vertex(x, y, z)` - Add vertex with current attributes
- `index(i)` - Add index for indexed rendering
- `set_vertex(i, x, y, z)` - Update existing vertex position

**Drawing:**
- `draw_geometry(geometry)` - Draw geometry in graphics context

### Module Registration

**Location:** `crates/processing_pyo3/src/lib.rs`

```rust
#[pymodule]
fn processing(m: &Bound<'_, PyModule>) -> PyResult<()> {
    m.add_class::<Geometry>()?;
    m.add_class::<Topology>()?;
    m.add_function(wrap_pyfunction!(draw_geometry, m)?)?;
    // ...
}
```

### Example Usage

**Location:** `crates/processing_pyo3/examples/animated_mesh.py`

```python
from processing import Geometry, Topology, size, run, draw_geometry

geometry = None

def setup():
    global geometry
    size(400, 400)
    mode_3d()
    
    geometry = Geometry()
    # ... add vertices and indices ...
    geometry.color(x/grid_size, 0.5, z/grid_size, 1.0)
    geometry.normal(0.0, 1.0, 0.0)
    geometry.vertex(px, 0.0, pz)

def draw():
    global geometry
    # ... update geometry ...
    draw_geometry(geometry)
```

---

## Adding New Primitives

### Step-by-Step Guide

#### Step 1: Add Primitive Function to `processing_render`

**Location:** `crates/processing_render/src/geometry/mod.rs`

**Pattern:** Follow the `create_box()` function as a template.

**Example: Adding Sphere**

```rust
pub fn create_sphere(
    In(radius): In<f32>,
    mut commands: Commands,
    mut meshes: ResMut<Assets<Mesh>>,
    builtins: Res<BuiltinAttributes>,
) -> Entity {
    // Create Bevy sphere primitive
    let sphere = Sphere::new(radius);
    
    // Convert to mesh
    let mesh = sphere.mesh().build();
    let handle = meshes.add(mesh);

    // Create layout with standard attributes
    let layout_entity = commands
        .spawn(VertexLayout::with_attributes(vec![
            builtins.position,
            builtins.normal,
            builtins.color,
            builtins.uv,
        ]))
        .id();

    // Spawn Geometry component
    commands.spawn(Geometry::new(handle, layout_entity)).id()
}
```

**Example: Adding Cube (alias for box with equal dimensions)**

```rust
pub fn create_cube(
    In(size): In<f32>,
    mut commands: Commands,
    mut meshes: ResMut<Assets<Mesh>>,
    builtins: Res<BuiltinAttributes>,
) -> Entity {
    // Reuse create_box with equal dimensions
    create_box(
        (size, size, size),
        commands,
        meshes,
        builtins,
    )
}
```

**Note:** Bevy provides these primitives via the `Meshable` trait:
- `Sphere` - `Sphere::new(radius)`
- `Cuboid` - `Cuboid::new(width, height, depth)` (already used)
- `Cylinder` - `Cylinder::new(radius, height)`
- `Torus` - `Torus::new(major_radius, minor_radius)`
- `Icosphere` - `Icosphere::new(radius, subdivisions)`
- `Capsule` - `Capsule::new(radius, length)`
- `Plane3d` - `Plane3d::default()`

#### Step 2: Expose Function in `processing_render` Public API

**Location:** `crates/processing_render/src/lib.rs`

Add the function to the public API:

```rust
pub fn geometry_sphere(radius: f32) -> error::Result<Entity> {
    app_mut(|app| {
        Ok(app
            .world_mut()
            .run_system_cached_with(geometry::create_sphere, radius)
            .unwrap())
    })
}
```

**Pattern:** Use `app_mut()` to access thread-local App, then `run_system_cached_with()` to execute the system function.

#### Step 3: Add PyO3 Method to Geometry Class

**Location:** `crates/processing_pyo3/src/graphics.rs`

**Option A: Static Method (Recommended for Primitives)**

Add to the `Geometry` impl block:

```rust
#[pymethods]
impl Geometry {
    // ... existing methods ...

    #[staticmethod]
    pub fn sphere(radius: f32) -> PyResult<Self> {
        let entity = geometry_sphere(radius)
            .map_err(|e| PyRuntimeError::new_err(format!("{e}")))?;
        Ok(Self { entity })
    }

    #[staticmethod]
    pub fn cube(size: f32) -> PyResult<Self> {
        let entity = geometry_cube(size)
            .map_err(|e| PyRuntimeError::new_err(format!("{e}")))?;
        Ok(Self { entity })
    }
}
```

**Usage in Python:**
```python
sphere = Geometry.sphere(radius=50.0)
cube = Geometry.cube(size=100.0)
```

**Option B: Module-Level Function (Alternative)**

Add to `crates/processing_pyo3/src/lib.rs`:

```rust
#[pyfunction]
fn sphere(radius: f32) -> PyResult<Geometry> {
    let entity = geometry_sphere(radius)
        .map_err(|e| PyRuntimeError::new_err(format!("{e}")))?;
    Ok(Geometry { entity })
}

#[pymodule]
fn processing(m: &Bound<'_, PyModule>) -> PyResult<()> {
    // ...
    m.add_function(wrap_pyfunction!(sphere, m)?)?;
    // ...
}
```

**Usage in Python:**
```python
from processing import sphere
geo = sphere(50.0)
```

**Recommendation:** Use static methods (`#[staticmethod]`) for consistency with the existing `Geometry` class API.

#### Step 4: Add FFI Binding (Optional, for C API)

**Location:** `crates/processing_ffi/src/lib.rs`

If you want to expose the primitive to C FFI:

```rust
#[unsafe(no_mangle)]
pub extern "C" fn processing_geometry_sphere(radius: f32) -> u64 {
    error::clear_error();
    error::check(|| geometry_sphere(radius))
        .map(|e| e.to_bits())
        .unwrap_or(0)
}
```

---

## Implementation Examples

### Complete Example: Adding Sphere Primitive

#### 1. Add to `crates/processing_render/src/geometry/mod.rs`

```rust
pub fn create_sphere(
    In(radius): In<f32>,
    mut commands: Commands,
    mut meshes: ResMut<Assets<Mesh>>,
    builtins: Res<BuiltinAttributes>,
) -> Entity {
    use bevy::prelude::Sphere;
    
    let sphere = Sphere::new(radius);
    let mesh = sphere.mesh().build();
    let handle = meshes.add(mesh);

    let layout_entity = commands
        .spawn(VertexLayout::with_attributes(vec![
            builtins.position,
            builtins.normal,
            builtins.color,
            builtins.uv,
        ]))
        .id();

    commands.spawn(Geometry::new(handle, layout_entity)).id()
}
```

#### 2. Add to `crates/processing_render/src/lib.rs`

```rust
pub fn geometry_sphere(radius: f32) -> error::Result<Entity> {
    app_mut(|app| {
        Ok(app
            .world_mut()
            .run_system_cached_with(geometry::create_sphere, radius)
            .unwrap())
    })
}
```

#### 3. Add to `crates/processing_pyo3/src/graphics.rs`

```rust
#[pymethods]
impl Geometry {
    // ... existing methods ...

    #[staticmethod]
    pub fn sphere(radius: f32) -> PyResult<Self> {
        use processing::prelude::geometry_sphere;
        let entity = geometry_sphere(radius)
            .map_err(|e| PyRuntimeError::new_err(format!("{e}")))?;
        Ok(Self { entity })
    }
}
```

#### 4. Python Usage

```python
from processing import Geometry, size, run, draw_geometry, mode_3d, camera_position, camera_look_at

def setup():
    size(400, 400)
    mode_3d()
    camera_position(0, 0, 300)
    camera_look_at(0, 0, 0)

def draw():
    # Create sphere
    sphere = Geometry.sphere(radius=100.0)
    sphere.color(1.0, 0.0, 0.0, 1.0)  # Red
    draw_geometry(sphere)

run()
```

### Complete Example: Adding Cube Primitive

#### 1. Add to `crates/processing_render/src/geometry/mod.rs`

```rust
pub fn create_cube(
    In(size): In<f32>,
    mut commands: Commands,
    mut meshes: ResMut<Assets<Mesh>>,
    builtins: Res<BuiltinAttributes>,
) -> Entity {
    // Reuse create_box with equal dimensions
    create_box(
        (size, size, size),
        commands,
        meshes,
        builtins,
    )
}
```

#### 2. Add to `crates/processing_render/src/lib.rs`

```rust
pub fn geometry_cube(size: f32) -> error::Result<Entity> {
    app_mut(|app| {
        Ok(app
            .world_mut()
            .run_system_cached_with(geometry::create_cube, size)
            .unwrap())
    })
}
```

#### 3. Add to `crates/processing_pyo3/src/graphics.rs`

```rust
#[pymethods]
impl Geometry {
    // ... existing methods ...

    #[staticmethod]
    pub fn cube(size: f32) -> PyResult<Self> {
        use processing::prelude::geometry_cube;
        let entity = geometry_cube(size)
            .map_err(|e| PyRuntimeError::new_err(format!("{e}")))?;
        Ok(Self { entity })
    }
}
```

#### 4. Python Usage

```python
from processing import Geometry, size, run, draw_geometry, mode_3d

def setup():
    size(400, 400)
    mode_3d()

def draw():
    cube = Geometry.cube(size=100.0)
    cube.color(0.0, 1.0, 0.0, 1.0)  # Green
    draw_geometry(cube)

run()
```

### Advanced Example: Parameterized Sphere

For a sphere with configurable subdivisions:

```rust
pub fn create_sphere_icosphere(
    In((radius, subdivisions)): In<(f32, usize)>,
    mut commands: Commands,
    mut meshes: ResMut<Assets<Mesh>>,
    builtins: Res<BuiltinAttributes>,
) -> Entity {
    use bevy::prelude::Icosphere;
    
    let icosphere = Icosphere::new(radius, subdivisions);
    let mesh = icosphere.mesh().build();
    let handle = meshes.add(mesh);

    let layout_entity = commands
        .spawn(VertexLayout::with_attributes(vec![
            builtins.position,
            builtins.normal,
            builtins.color,
            builtins.uv,
        ]))
        .id();

    commands.spawn(Geometry::new(handle, layout_entity)).id()
}
```

**PyO3 exposure:**

```rust
#[staticmethod]
pub fn sphere_icosphere(radius: f32, subdivisions: usize) -> PyResult<Self> {
    let entity = geometry_sphere_icosphere(radius, subdivisions)
        .map_err(|e| PyRuntimeError::new_err(format!("{e}")))?;
    Ok(Self { entity })
}
```

---

## Best Practices

### 1. Error Handling

Always use proper error handling:

```rust
pub fn create_primitive(...) -> Entity {
    // Implementation
    // No Result needed for system functions that can't fail
    // But validate inputs if needed
}
```

For public API:

```rust
pub fn geometry_primitive(...) -> error::Result<Entity> {
    app_mut(|app| {
        Ok(app
            .world_mut()
            .run_system_cached_with(geometry::create_primitive, args)
            .unwrap())
    })
}
```

For PyO3:

```rust
pub fn primitive(...) -> PyResult<Self> {
    let entity = geometry_primitive(...)
        .map_err(|e| PyRuntimeError::new_err(format!("{e}")))?;
    Ok(Self { entity })
}
```

### 2. Naming Conventions

- **Rust functions:** `create_<primitive>()` (system function)
- **Public API:** `geometry_<primitive>()` (public function)
- **PyO3 methods:** `Geometry.<primitive>()` (static method)
- **Python usage:** `Geometry.<primitive>()`

### 3. Layout Consistency

Always use the standard layout for primitives:

```rust
let layout_entity = commands
    .spawn(VertexLayout::with_attributes(vec![
        builtins.position,
        builtins.normal,
        builtins.color,
        builtins.uv,
    ]))
    .id();
```

This ensures:
- Consistent attribute ordering
- Compatibility with existing code
- Proper normal/color/UV support

### 4. System Function Pattern

Use `In<T>` parameter for system functions:

```rust
pub fn create_primitive(
    In(args): In<PrimitiveArgs>,
    mut commands: Commands,
    mut meshes: ResMut<Assets<Mesh>>,
    builtins: Res<BuiltinAttributes>,
) -> Entity {
    // Implementation
}
```

For multiple parameters, use tuples:

```rust
pub fn create_primitive(
    In((arg1, arg2, arg3)): In<(f32, f32, f32)>,
    // ...
) -> Entity {
    // Implementation
}
```

### 5. Documentation

Add doc comments:

```rust
/// Create a sphere geometry primitive.
///
/// # Arguments
/// * `radius` - The radius of the sphere
///
/// # Returns
/// Entity ID of the created geometry
pub fn geometry_sphere(radius: f32) -> error::Result<Entity> {
    // ...
}
```

For PyO3:

```rust
/// Create a sphere geometry primitive.
///
/// Args:
///     radius: The radius of the sphere
///
/// Returns:
///     A new Geometry object representing a sphere
#[staticmethod]
pub fn sphere(radius: f32) -> PyResult<Self> {
    // ...
}
```

### 6. Testing

Add examples in `crates/processing_pyo3/examples/`:

```python
# examples/sphere.py
from processing import Geometry, size, run, draw_geometry, mode_3d

def setup():
    size(400, 400)
    mode_3d()

def draw():
    sphere = Geometry.sphere(radius=100.0)
    sphere.color(1.0, 0.0, 0.0, 1.0)
    draw_geometry(sphere)

run()
```

### 7. Reusability

If a primitive can be built from another, reuse it:

```rust
pub fn create_cube(size: f32, ...) -> Entity {
    // Reuse create_box
    create_box((size, size, size), ...)
}
```

### 8. Parameter Validation

Add validation if needed:

```rust
pub fn create_sphere(
    In(radius): In<f32>,
    // ...
) -> Entity {
    // Validate radius
    assert!(radius > 0.0, "Sphere radius must be positive");
    
    // Or return Result if validation can fail
    // ...
}
```

---

## Common Patterns

### Pattern 1: Simple Primitive (Sphere, Cube)

```rust
// 1. System function
pub fn create_sphere(In(radius): In<f32>, ...) -> Entity {
    let sphere = Sphere::new(radius);
    let mesh = sphere.mesh().build();
    // ... standard layout and spawn
}

// 2. Public API
pub fn geometry_sphere(radius: f32) -> error::Result<Entity> {
    app_mut(|app| {
        Ok(app.world_mut()
            .run_system_cached_with(geometry::create_sphere, radius)
            .unwrap())
    })
}

// 3. PyO3
#[staticmethod]
pub fn sphere(radius: f32) -> PyResult<Self> {
    let entity = geometry_sphere(radius)
        .map_err(|e| PyRuntimeError::new_err(format!("{e}")))?;
    Ok(Self { entity })
}
```

### Pattern 2: Parameterized Primitive (Cylinder, Torus)

```rust
// 1. System function with tuple
pub fn create_cylinder(
    In((radius, height)): In<(f32, f32)>,
    // ...
) -> Entity {
    let cylinder = Cylinder::new(radius, height);
    // ...
}

// 2. Public API
pub fn geometry_cylinder(radius: f32, height: f32) -> error::Result<Entity> {
    app_mut(|app| {
        Ok(app.world_mut()
            .run_system_cached_with(geometry::create_cylinder, (radius, height))
            .unwrap())
    })
}

// 3. PyO3
#[staticmethod]
pub fn cylinder(radius: f32, height: f32) -> PyResult<Self> {
    let entity = geometry_cylinder(radius, height)
        .map_err(|e| PyRuntimeError::new_err(format!("{e}")))?;
    Ok(Self { entity })
}
```

### Pattern 3: Alias (Cube from Box)

```rust
// Reuse existing function
pub fn create_cube(In(size): In<f32>, ...) -> Entity {
    create_box((size, size, size), ...)
}
```

---

## Available Bevy Primitives

Bevy provides these primitives via the `Meshable` trait:

| Primitive | Constructor | Parameters |
|-----------|-------------|------------|
| `Sphere` | `Sphere::new(radius)` | `radius: f32` |
| `Cuboid` | `Cuboid::new(w, h, d)` | `width, height, depth: f32` |
| `Cylinder` | `Cylinder::new(radius, height)` | `radius, height: f32` |
| `Torus` | `Torus::new(major, minor)` | `major_radius, minor_radius: f32` |
| `Icosphere` | `Icosphere::new(radius, subdiv)` | `radius: f32, subdivisions: usize` |
| `Capsule` | `Capsule::new(radius, length)` | `radius, length: f32` |
| `Plane3d` | `Plane3d::default()` | None |

**Note:** Check Bevy documentation for the exact API, as it may vary by version.

---

## Troubleshooting

### Issue: "Geometry not found" Error

**Cause:** Entity was despawned or invalid.

**Solution:** Ensure geometry is not destroyed before use, and check entity validity.

### Issue: Primitive Not Rendering

**Cause:** Missing layout attributes or incorrect mesh setup.

**Solution:** Ensure standard layout is used (position, normal, color, UV).

### Issue: PyO3 Import Error

**Cause:** Function not registered in `#[pymodule]`.

**Solution:** Ensure `Geometry` class is registered and method is in `#[pymethods]` block.

### Issue: System Function Not Found

**Cause:** Function not accessible or wrong signature.

**Solution:** Ensure function is public and uses `In<T>` parameter pattern.

---

## Summary

Adding new geometry primitives follows a consistent pattern:

1. **Add system function** in `geometry/mod.rs` using Bevy primitives
2. **Expose public API** in `lib.rs` using `app_mut()` pattern
3. **Add PyO3 method** in `graphics.rs` as static method
4. **Test** with Python example

The key insight is that Bevy's `Meshable` trait provides a clean way to convert primitives to meshes, and the existing `create_box()` function serves as a perfect template for new primitives.

---

**Last Updated:** 2026-01-25  
**Related Documentation:** [CODEBASE_KNOWLEDGE.md](./CODEBASE_KNOWLEDGE.md)
