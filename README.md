# servo-paint-api (fork)

`servo-paint-api` 0.3.0, copied verbatim from crates.io except for one hunk in
`rendering_context.rs`. Used by [bace](https://github.com/crcrworks/bace) through
`[patch.crates-io]`, pinned by rev.

## Why

`SurfmanRenderingContext::new` created the surfman context and then loaded glow **before**
making the context current:

```rust
let context = device.create_context(&context_descriptor, None)?;
let gleam_gl = ...;                 // loaded against a not-yet-current context
let glow_gl = unsafe { glow::Context::from_loader_function(...) };
// ...
// SoftwareRenderingContext::new calls make_current() only after this returned
```

`glow::Context::from_loader_function` immediately reads `GL_VERSION`, which is `NULL`
when no context is current. On Windows (ANGLE/WGL) that panics inside glow —

```
thread 'main' panicked at glow-0.17.0/src/native.rs:69:13:
Reading GL_VERSION failed. Make sure there is a valid GL context currently active.
```

— and takes the whole application down; on Linux/EGL the call happened to work. Servo
fixed this upstream in 0.8.0 by making the context current right after creating it; this
fork is that fix backported to 0.3.0, whose API bace uses.

## The change

`rendering_context.rs`, `SurfmanRenderingContext::new` — identical to servo 0.8.0's
`components/paint-api/rendering_context.rs`:

- `device.make_context_current(&context)` immediately after `create_context`, before the
  `gleam` and `glow` loaders run; a failure destroys the context and returns the error.
- the context is bound `mut` so it can be destroyed on that error path.

Nothing else differs from the published 0.3.0 crate. When bace moves to a servo release
that carries the fix (0.8.0 or later), this fork and its patch entry can be dropped.

## Using it

```toml
[patch.crates-io]
servo-paint-api = { git = "https://github.com/crcrworks/servo-paint-api", rev = "<rev>" }
```

## License

MPL-2.0, the license of servo-paint-api / servo/servo. See [LICENSE](LICENSE); every source
file keeps its original Mozilla Public License header. Source: crates.io
`servo-paint-api` 0.3.0 (normalized manifest in `Cargo.toml`, original in
`Cargo.toml.orig`), from <https://github.com/servo/servo>.
