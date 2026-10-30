# mat — 2D & 3D Linear Algebra & Graphics Math for Edva

`mat` is a high-performance graphics and spatial mathematics library written in [Edva](https://gitea.gde.to/ike/dva). It provides C-ABI `f32` data structures and pure value-returning linear algebra functions for WebGL, WebGPU, Vulkan, and native applications.

---

## Features

- **C-ABI Binary Layouts**: Exact memory layout matching GPU uniforms and vertex attributes (`Mat4` = `(f32 * 16)` 64 bytes column-major, `Mat3` = `(f32 * 9)` 36 bytes, `#packed` `Vec2`/`Vec3`/`Vec4`/`Quat`).
- **Pure Value Functions**: Compose transformations naturally as values—e.g. `m = mat4::mul(mat4::translate(x, y, z), mat4::rotate_y(ang))`, `v = mat4::look_at(eye, target, up)`.
- **Seamless `std/math/geom` Interop**: Accepts high-precision `geom::Vec3` vectors in `mat4::look_at`, `mat4::translate_vec`, and `mat4::scale_vec`.
- **Zero External Dependencies**: Uses Edva compiler intrinsics (`%sin`, `%cos`, `%sqrt`) directly—requiring no `libm` linkage or JavaScript host math imports when targeting `wasm32-unknown-unknown`.

---

## Installation

Add `mat` to your `edva.ccl`:

```ccl
dependencies =
  mat =
    path = ../mat
```

---

## Module Overview

| Module | Core Type | Value API Highlights |
| :--- | :--- | :--- |
| [`src/vec2.dva`](src/vec2.dva) | `Vec2: #packed (x: f32, y: f32)` | `vec2`, `zero`, `add`, `sub`, `mul`, `scale`, `dot`, `len_sq`, `len`, `normalize`, `distance`, `lerp`, `negate` |
| [`src/vec3.dva`](src/vec3.dva) | `Vec3: #packed (x: f32, y: f32, z: f32)` | `vec3`, `zero`, `up`, `right`, `forward`, `add`, `sub`, `mul`, `scale`, `dot`, `cross`, `len_sq`, `len`, `normalize`, `distance`, `lerp`, `negate` |
| [`src/vec4.dva`](src/vec4.dva) | `Vec4: #packed (x: f32, y: f32, z: f32, w: f32)` | `vec4`, `from_vec3`, `zero`, `add`, `sub`, `scale`, `dot`, `len_sq`, `len`, `normalize`, `negate` |
| [`src/mat3.dva`](src/mat3.dva) | `Mat3: (f32 * 9)` | `mat3`, `identity`, `mul`, `transpose`, `determinant`, `invert` |
| [`src/mat4.dva`](src/mat4.dva) | `Mat4: (f32 * 16)` | `mat4`, `identity`, `translate`, `translate_vec`, `scale`, `scale_vec`, `rotate_x`, `rotate_y`, `rotate_z`, `mul`, `perspective`, `perspective_gl`, `ortho`, `look_at`, `transpose`, `extract_normal_matrix` |
| [`src/quat.dva`](src/quat.dva) | `Quat: #packed (x: f32, y: f32, z: f32, w: f32)` | `quat`, `identity`, `from_axis_angle`, `from_euler`, `mul`, `len`, `normalize`, `conjugate`, `slerp`, `to_mat4` |

---

## Usage Examples

### 1. Value-Based Matrix Composition

Because `mat4::Mat4` is a 16-`f32` tuple `(f32 * 16)`, you can compose Model, View, and Projection transforms as values and pass `&mvp` directly to WebGL/WebGPU uniform upload functions:

```dva
#use math
#use "math/geom"
#use @mat/mat4

render_object: Float, Float => ()
render_object = aspect, angle =>
   // Camera projection (OpenGL/WebGL NDC [-1, 1]) & view matrix
   proj = mat4::perspective_gl(math::rad(50.0), aspect, 0.1, 100.0)
   view = mat4::look_at(geom::vec3(0.0, 1.5, 6.0), geom::vec3(0.0, 0.0, 0.0), geom::vec3(0.0, 1.0, 0.0))
   vp   = mat4::mul(proj, view)

   // Model transform: translate * rotate_y * scale
   t = mat4::translate(1.5, 0.0, 0.0)
   r = mat4::rotate_y(angle)
   s = mat4::scale(0.8, 0.8, 0.8)
   model = mat4::mul(t, mat4::mul(r, s))
   mvp   = mat4::mul(vp, model)

   // Pass &mvp and &model directly to GPU uniform upload
   gl_draw(&mvp, &model, 0, 36)
```

### 2. Quaternion Smooth Interpolation (`slerp`)

```dva
#use @mat/quat

interpolate_orientation = t =>
   q0 = quat::from_axis_angle(0.0, 1.0, 0.0, 0.0)
   q1 = quat::from_axis_angle(0.0, 1.0, 0.0, 1.570796)
   q_cur = quat::slerp(q0, q1, t)
   quat::to_mat4(q_cur)
```

---

## Running Tests

```bash
cd ~/Z/edva-ware/mat
edva test
```
