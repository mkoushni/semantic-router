# Dockerfiles

This directory contains Dockerfiles used across the project.

- `tools/docker/Dockerfile`: development base image (CentOS Stream) with toolchains (Rust, Go, Envoy, HF CLI).
- `tools/docker/Dockerfile.extproc`: builds the `extproc` (semantic-router external processor) image.
- `tools/docker/Dockerfile.extproc.cross`: cross-compilation optimized `extproc` Dockerfile.
- `tools/docker/Dockerfile.precommit`: pre-commit / lint tooling image for CI and local use.
- `tools/docker/Dockerfile.stack`: single-image “stack” build bundling router + dashboard + observability components.

## Build optimization (CI)

The workflow [.github/workflows/docker-publish.yml](../../.github/workflows/docker-publish.yml) builds multi-arch images (e.g. vllm-sr, extproc) on push to `main`. To keep build times under **60 minutes** per image:

- **Cross-compilation (no QEMU):** On push, **extproc** uses `Dockerfile.extproc.cross` so ARM64 is built by cross-compiling on the amd64 runner instead of QEMU emulation. **vllm-sr** uses `TARGETARCH` in its Dockerfile so Rust and Go stages run on `--platform=linux/amd64` and produce arm64 artifacts natively when building for arm64. This avoids the 5–10× slowdown of QEMU.
- **No `cargo clean`:** `Dockerfile.extproc` and `Dockerfile.extproc.cross` do not run `cargo clean` before the real build, so the dependency cache from the pre-build layer is reused and only application code is recompiled.
- **Job timeout:** The build job has a 90-minute timeout; long or stuck builds fail instead of blocking the pipeline.
- **Rust cache:** vllm-sr and extproc use GitHub Actions cache for `candle-binding/target/` and `ml-binding/target/` (and cargo registry). Docker build uses `cache-from: type=gha` for layer reuse.
- **CARGO_BUILD_JOBS:** Set to 20 on push (8 on PR) for higher parallelism.
- **Build time metrics:** Each run reports **Build time: Xm Ys** in the job step summary and in a notice. Use this to track regressions and confirm builds stay within targets.
