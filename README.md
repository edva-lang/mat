# mat — 2D & 3D Linear Algebra & Graphics Math for Edva

`mat` is a high-performance graphics and spatial mathematics library written in [Edva](https://gitea.gde.to/ike/dva). It provides packed C-ABI `f32` data structures and a **hybrid API** offering both ergonomic **pure value-returning functions** and **zero-allocation `_to` buffer procedures** for WebGL, WebGPU, Vulkan, and native applications.

---

## Features

- **Packed C-ABI Binary Layouts**: Exact alignment matching GPU uniforms and vertex attributes (`Mat4` = 64 bytes column-major `f32 * 16`, `Mat3` = 36 bytes `f32 * 9`, `Vec2`/`Vec3`/`Vec4`/`Quat`).
- **Hybrid Value + Buffer API**:
  - **Pure Value Functions**: Compose transformations naturally as values—e.g. `m = mat4::mul(mat4::translate(x, y, z), mat4::rotate_y(ang))`, `v = mat4::look_at(eye, target, up)`.
  - **In-Place `_to` Buffer Procedures**: Write directly into GPU staging addresses (`mat4::mul_to(out, a, b)`, `mat4::perspective_to(...)`) when managing raw memory blocks.
- **Seamless `std/math/geom` Interop**: Accepts high-precision `geom::Vec3` vectors in `mat4::look_at` alongside packed `f32` `vec3::Vec3` in `mat4::look_at_to`.
- **Zero External Dependencies**: Uses Edva compiler intrinsics (`%sin`, `%cos`, `%tan`, `%sqrt`) directly—requiring no `libm` linkage or JavaScript host math imports when targeting `wasm32-unknown-unknown`.

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

| Module | Core Type | Value API Highlights | Buffer (`_to`) API Highlights |
| :--- | :--- | :--- | :--- |
| [`src/vec2.dva`](src/vec2.dva) | `Vec2: #packed (x, y: f32)` | `vec2`, `zero`, `add`, `sub`, `scale`, `dot`, `len`, `normalize`, `dist`, `lerp` | `write_to(addr, v)` |
| [`src/vec3.dva`](src/vec3.dva) | `Vec3: #packed (x, y, z: f32)` | `vec3`, `zero`, `up`, `add`, `sub`, `scale`, `dot`, `cross`, `len`, `normalize`, `dist`, `lerp` | `write_to(addr, v)` |
| [`src/vec4.dva`](src/vec4.dva) | `Vec4: #packed (x, y, z, w: f32)` | `vec4`, `zero`, `add`, `sub`, `scale`, `dot`, `len`, `normalize`, `lerp` | `write_to(addr, v)` |
| [`src/mat3.dva`](src/mat3.dva) | `Mat3: #packed (m: (f32 * 9))` | `identity`, `zero`, `scale`, `rotate_x`, `rotate_y`, `rotate_z`, `mul`, `transpose`, `det`, `mul_vec3` | `identity_to`, `mul_to`, `transpose_to`, `invert_to`, `from_mat4_to`, `normal_from_mat4_to` |
| [`src/mat4.dva`](src/mat4.dva) | `Mat4: #packed (m: (f32 * 16))` | `identity`, `zero`, `translate`, `scale`, `rotate_x`, `rotate_y`, `rotate_z`, `mul`, `transpose`, `perspective`, `perspective_gl`, `look_at`, `mul_vec4`, `transform_point`, `transform_dir` | `identity_to`, `translate_to`, `scale_to`, `rotate_x_to`, `rotate_y_to`, `rotate_z_to`, `rotate_axis_to`, `mul_to`, `transpose_to`, `invert_to`, `perspective_to`, `perspective_gl_to`, `ortho_to`, `look_at_to` |
| [`src/quat.dva`](src/quat.dva) | `Quat: #packed (x, y, z, w: f32)` | `identity`, `quat`, `from_axis_angle`, `from_euler`, `mul`, `normalize`, `slerp`, `rotate_vec3` | `to_mat4_to(out, q)` |

---

## Usage Examples

### 1. Value-Based Matrix Composition (Recommended)

Because `mat4::Mat4` is a `#packed` 16-`f32` value record, you can compose Model, View, and Projection transforms as values and pass `&mvp` directly to WebGL/WebGPU uniform upload functions:

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

### 2. Zero-Copy In-Place Buffer Procedures (`_to`)

When operating directly on pre-allocated memory arenas or uniform buffers:

```dva
#use @mat/mat4

update_uniform_buffer: Addr, Addr, Float => ()
update_uniform_buffer = scratch_buf, uniform_buf, rot_y =>
   proj  = scratch_buf + 0
   view  = scratch_buf + 64
   model = scratch_buf + 128
   vp    = scratch_buf + 192

   mat4::perspective_to(proj, 1.047197, 16.0 / 9.0, 0.1, 100.0)
   mat4::translate_to(view, 0.0, 0.0, -4.0)
   mat4::rotate_y_to(model, rot_y)

   mat4::mul_to(vp, proj, view)
   mat4::mul_to(uniform_buf, vp, model)
```

### 3. Quaternion Smooth Interpolation (`slerp`)

```dva
#use @mat/vec3
#use @mat/quat

interpolate_orientation = t, out_mat_addr =>
   q0 = quat::from_axis_angle(vec3::vec3(0.0, 1.0, 0.0), 0.0)
   q1 = quat::from_axis_angle(vec3::vec3(0.0, 1.0, 0.0), 1.570796)
   q_cur = quat::slerp(q0, q1, t)
   quat::to_mat4_to(out_mat_addr, q_cur)
```

---

## Running Tests

```bash
cd ~/Z/edva-ware/mat
edva test
```
