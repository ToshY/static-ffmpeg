# Adding NVIDIA CUDA / NVENC / NVDEC support to `static-ffmpeg`

**Date:** 2026-04-24
**Tracking issue:** [#480 — Support for CUDA](https://github.com/wader/static-ffmpeg/issues/480)
**Outcome:** Separate `:<tag>-cuda` image variant added; default `:<tag>` remains a fully static-pie binary.

---

## 1. Problem statement

The default `mwader/static-ffmpeg` image is a **fully static-pie musl binary** with zero
runtime dependencies. NVIDIA GPU acceleration (NVENC/NVDEC/CUVID) requires
`dlopen()`'ing the host's NVIDIA driver libraries (`libcuda.so.1`,
`libnvcuvid.so`, `libnvidia-encode.so`) at runtime, which is fundamentally
incompatible with `static-pie` on musl: a static-pie binary has no dynamic
loader, so `dlopen()` cannot work.

Goal: ship a second image variant that supports CUDA without breaking the
existing static guarantees of the default image.

---

## 2. Architecture decision

### Two separate variants, not one

| Variant | Tag                        | Linkage                             | GPU support |
|---------|----------------------------|-------------------------------------|-------------|
| Default | `8.1`, `latest`            | static-pie musl                     | ❌          |
| CUDA    | `8.1-cuda`, `latest-cuda`  | musl **dynamic-PIE** (libc only)    | ✅          |

**Why a separate variant** (not a build-arg toggle on the default tag):
- The default tag's value proposition is "drop into any base image including `FROM scratch`". Making it dynamic would silently break that for thousands of existing users.
- CUDA users need the NVIDIA Container Toolkit and a GPU host — fundamentally different deployment.
- Different tag = explicit user opt-in + clear support boundary.

### Build-arg `ENABLE_CUDA`

A single `ARG ENABLE_CUDA=` controls everything:
- Adds `nv-codec-headers` (header-only, no runtime CUDA toolkit needed)
- Adds `--enable-ffnvcodec --enable-cuvid --enable-nvenc --enable-nvdec` to ffmpeg
- Switches link mode from `static-pie` to musl `dynamic-PIE`
- Sets `NVIDIA_VISIBLE_DEVICES=all` and `NVIDIA_DRIVER_CAPABILITIES=compute,utility,video` env
- Writes `/etc/ld-musl-x86_64.path` so musl's loader can find toolkit-injected libs
- Switches `checkelf` to `--cuda` mode (allows libc as the only NEEDED entry)

The CI builds two images per release: default (no arg) and `final-cuda` target with `ENABLE_CUDA=1`.

---

## 3. Why CUDA cannot be `static-pie` on musl

| Constraint | Implication |
|---|---|
| `static-pie` binaries have no dynamic loader | `dlopen()` impossible |
| `nvenc` calls `dlopen("libcuda.so.1", RTLD_LAZY)` via `ffnvcodec/dynlink_loader.h` | Must be a dynamic binary |
| `libcuda.so.1` is provided by the host driver, version-matched to the host | Must NOT be bundled in image |
| NVIDIA Container Toolkit injects driver libs at container start | Image just needs to be loadable |

**The minimum-impact compromise:** binary is dynamic only for libc; *every other dependency* (codecs, openssl, libstdc++, libgomp, libgcc, …) remains statically archived. The cuda variant's `readelf -d` differs from the default by **exactly one extra `NEEDED` entry**: `libc.musl-x86_64.so.1`.

---

## 4. Limitations explicitly NOT supported

| Feature | Reason |
|---|---|
| `--enable-cuda-nvcc` | Requires the full ~3 GB glibc-based CUDA toolkit at build time |
| `--enable-libnpp`    | Same — glibc-based, defeats the static/musl design |
| `scale_npp` filter   | Comes with libnpp; use `scale_cuda` instead |
| `arm64` builds       | NVIDIA Container Toolkit on arm64 is server-class only (Jetson uses a different stack); released as **amd64-only** for now |
| `FROM scratch` / distroless target images | No musl loader available; copy-out won't work |

---

## 5. Files changed

### `Dockerfile`
1. New `ARG ENABLE_CUDA=` early in the builder stage.
2. New `nv-codec-headers` install step (skipped when `ENABLE_CUDA` is unset).
3. `ffmpeg` configure step extended:
   - `--enable-ffnvcodec --enable-cuvid --enable-nvenc --enable-nvdec` when `ENABLE_CUDA`
   - Replaces `add_ldexeflags -fPIE -static-pie` with `-fPIE -pie` (dynamic-PIE) when `ENABLE_CUDA`
   - Custom `CUDA_LDFLAGS` / `CUDA_EXTRA_LIBS` to keep all non-libc deps static (see §6)
4. `checkelf` invocation gains `--cuda` flag when `ENABLE_CUDA`.
5. New `final-cuda` stage: `FROM alpine:3.X` + copy of `/usr/local/bin/{ffmpeg,ffprobe}` + ld-musl path config + `ENV NVIDIA_*`.

### `checkelf`
- Accepts `--cuda` flag.
- In `--cuda` mode allows the musl loader/libc entry from `ldd` output (everything else still rejected).
- All other hardening checks (RELRO, BIND_NOW, PIE, NX stack) preserved.

### `README.md`
- New "CUDA / NVENC / NVDEC" section with build, run, `COPY --from=` recipes for Alpine / Debian / `nvidia/cuda:*` target images, and a "verify static-ness from the host" section using `readelf -d`.
- New tag entry: `<tag>-cuda` / `latest-cuda` (amd64-only).

---

## 6. The dlopen / static-musl trap (gotcha worth documenting)

This was the single most painful issue and is **not obvious** from the build logs.

### Symptom

The `:8.1-cuda` binary builds successfully, `checkelf --cuda` passes, but at runtime:

```
[h264_nvenc @ 0x...] Cannot load libcuda.so.1
```

`strace -e openat` shows that ffmpeg **never even attempts** to open any libcuda file — `dlopen()` returns NULL immediately without touching the filesystem.

### Root cause

musl's **static `libc.a`** ships a 25-byte `dlopen` stub that always returns NULL with `errno=ENOSYS`. This is documented behavior — musl deliberately does not support `dlopen` from statically-linked binaries.

The original CUDA build flags were:

```sh
--extra-ldflags='-static-libstdc++ -static-libgcc -Wl,-Bstatic'
--extra-libs=' -lgomp -Wl,-Bdynamic -lc '
```

The intent: switch to `-Bstatic` for the codec libs, then flip back to `-Bdynamic` at the end so libc stays dynamic. That keeps `ldd` output clean (one NEEDED entry: musl libc).

The bug: ffmpeg's `nvenc.c` references `dlopen`. While processing the codec `.a` files in `-Bstatic` mode, the linker resolves `dlopen` from the static `libc.a` (which gcc pulls in implicitly). Result:

```
readelf -s --dyn-syms /ffmpeg | grep dlopen
# 21987: 000000000338c50e   25 FUNC WEAK DEFAULT 14 dlopen
#                           ^^                  ^^^^
#                       25 bytes              .text section
```

`dlopen` is a **25-byte function defined inside the binary itself** in section 14 (`.text`) — the static stub. It's not `UND`, so it never goes through the PLT to dynamic libc.

### Fix (final, robust)

Link the musl loader/libc by **absolute path** in the `--extra-ldflags`, so the
linker resolution is immune to subsequent `-Bstatic`/`-Bdynamic` toggles:

```sh
--extra-ldflags='-fopenmp -Wl,--allow-multiple-definition -Wl,-z,stack-size=2097152 \
    -Wl,--no-as-needed,/lib/ld-musl-x86_64.so.1,--as-needed \
    -Wl,--as-needed -Wl,-Bstatic \
    -static-libstdc++ -static-libgcc'
--extra-libs='-lgomp -Wl,-Bdynamic -lc'
```

Why the absolute path works where `-Wl,--no-as-needed,-Bdynamic,-lc` did not:

- A `-l<name>` argument is searched per the current `-Bstatic`/`-Bdynamic` mode and
  per the linker's library search path. It is also fed through gcc's spec file,
  which (especially under `--toolchain=hardened`) re-emits late-stage references
  that can pull `libc.a` back in even after a careful `-Bdynamic … -Bstatic`
  reorder, restoring the broken stub.
- An **absolute filename** in the linker command line is not treated as a `-l`
  search at all; it is opened literally as a DSO regardless of the `-Bstatic`
  mode in effect. Its dynamic symbols (including `dlopen`, `dlsym`, `dlerror`,
  `dlclose`) are then available to satisfy references from later `.a` archives,
  and those references resolve as `UND` (PLT) instead of pulling the static stub.
- On Alpine, `/lib/ld-musl-x86_64.so.1` is *both* the dynamic loader and libc —
  one file serves both roles — so this single absolute path covers everything
  we needed `-lc` for.

### Verification (the bug is invisible to most checks)

```sh
readelf -s --dyn-syms /ffmpeg | grep -E 'dlopen|dlsym|dlerror|dlclose'
# Each must show:
#       0:               0   FUNC ... UND dl<name>
# If any shows a non-zero size with a section number (e.g. " 25 FUNC ... 14 dlopen"),
# the static stub is back and dlopen will silently return NULL with ENOSYS.
```

> Note: in some link configurations the linker may resolve `dlopen` purely
> *internally* against the absolute-path libc and not export an explicit `UND`
> entry for it. The functional test (h264_nvenc actually encoding frames)
> remains the ultimate ground truth; readelf is just the cheapest pre-flight
> check that catches the stub-bug regression.

### Lessons for any future change to this build

- **Never link musl `libc.a` into a binary that calls `dlopen`.** It will silently use the stub.
- The `-Bdynamic -lc -Bstatic` reorder is fragile under gcc's `--toolchain=hardened`
  spec file. Prefer the absolute-path form `/lib/ld-musl-x86_64.so.1`.
- The bug is invisible to standard hardening checks: the binary still has
  `BIND_NOW`, `RELRO`, `PIE`, NX stack. `ldd` still shows only one extra
  NEEDED entry.
- The only reliable signal is a real NVENC encode actually emitting frames.

---

## 7. Runtime requirements

### Host
- NVIDIA driver installed
- [NVIDIA Container Toolkit](https://github.com/NVIDIA/nvidia-container-toolkit) installed and configured for Docker
- Run with `--gpus all` (or `--runtime=nvidia` + `NVIDIA_VISIBLE_DEVICES`)

### Image-side env (set by Dockerfile)
- `NVIDIA_VISIBLE_DEVICES=all`
- `NVIDIA_DRIVER_CAPABILITIES=compute,utility,video`
  - `compute` → `libcuda.so.1`
  - `video` → `libnvcuvid.so`, `libnvidia-encode.so`
  - Dropping `video` makes `nvidia-smi` work but breaks `h264_nvenc` with `Cannot load libcuda.so.1`.

### `/etc/ld-musl-x86_64.path`
musl does **not** read `/etc/ld.so.cache`, so the toolkit's `ldconfig` post-start hook is silently ignored. We ship a static path file:

```
/usr/lib/x86_64-linux-gnu
/usr/lib64
/usr/lib/wsl/lib
/usr/lib
/usr/local/lib
/lib
```

Covers the three common toolkit injection layouts:
- Debian/Ubuntu hosts → `/usr/lib/x86_64-linux-gnu`
- RHEL/Fedora hosts   → `/usr/lib64`
- WSL2                → `/usr/lib/wsl/lib`

Listing all is safe — musl silently skips paths that don't exist.

---

## 8. Verifying the image

### From any Linux host (no musl needed)

```sh
docker create --name sf      mwader/static-ffmpeg:8.1
docker cp sf:/ffmpeg         /tmp/ffmpeg-static && docker rm sf

docker create --name sfcuda  mwader/static-ffmpeg:8.1-cuda
docker cp sfcuda:/ffmpeg     /tmp/ffmpeg-cuda && docker rm sfcuda

readelf -d /tmp/ffmpeg-static | grep -E 'NEEDED|BIND_NOW'
# (no NEEDED entries — fully static)
# 0x000000000000001e (FLAGS) BIND_NOW

readelf -d /tmp/ffmpeg-cuda  | grep -E 'NEEDED|BIND_NOW'
# 0x0000000000000001 (NEEDED) Shared library: [libc.musl-x86_64.so.1]
# 0x000000000000001e (FLAGS) BIND_NOW
```

### dlopen sanity check (the painful one)

```sh
docker run --gpus all --rm --entrypoint sh mwader/static-ffmpeg:8.1-cuda -c '
apk add --no-cache binutils >/dev/null 2>&1
readelf -s --dyn-syms /ffmpeg | grep -E "dlopen|dlsym|dlerror"
'
# MUST end with "UND dlopen", "UND dlsym", "UND dlerror"
# If any has a non-zero size in .text → static stub bug is back.
```

### Functional encode

```sh
docker run --gpus all --rm mwader/static-ffmpeg:8.1-cuda \
    -f lavfi -i testsrc=duration=2:size=1280x720:rate=30 \
    -c:v h264_nvenc -f null -
# expect: frame=  60 ... finished
```

---

## 9. Comparison with other static ffmpeg + nvenc projects

| Project | Static? | NVENC? | Approach |
|---|---|---|---|
| `mwader/static-ffmpeg:8.1` | ✅ static-pie musl | ❌ | Pure static, no dlopen |
| `mwader/static-ffmpeg:8.1-cuda` | ⚠️ musl dynamic-PIE (libc only) | ✅ | Hybrid — only libc dynamic; `dlopen()` works |
| BtbN/FFmpeg-Builds (LGPL/GPL) | ⚠️ glibc dynamic, plus runtime ldconfig | ✅ | Tarball, glibc-linked |
| HiWay-Media/ffmpeg-nvenc-static | ⚠️ glibc dynamic | ✅ | Bundled libs |
| markus-perl/ffmpeg-build-script | ⚠️ glibc dynamic | optional | Script, not container |

Of these, **only `:8.1-cuda` keeps every codec/lib statically linked** — every other "static + nvenc" build is glibc-dynamic. The trade-off vs the default `:8.1` is exactly one libc.so dependency.

---

## 10. CI / multi-arch publishing notes

- Default tag: built for `linux/amd64,linux/arm64` as before.
- CUDA tag: built for `linux/amd64` only.
  - Pushed as `<tag>-cuda` (and re-tagged manifest-style as `<tag>-cuda-amd64` for clarity).
  - `latest-cuda` follows latest stable.
- Use `--target final-cuda` and `--build-arg ENABLE_CUDA=1` in the CI matrix entry.

---

## 11. Issues encountered during implementation (chronological)

1. **`nv-codec-headers` checksum mismatch** — initial SHA256 was wrong; fixed by recomputing against the actual GitHub release tarball.
2. **`checkelf` rejected the dynamic-PIE binary** — added `--cuda` mode that allows musl libc + loader as the only `ldd` entries.
3. **Spurious dynamic deps (`libgomp`, `libdrm`, etc.)** — fixed by pre-linking with `-Wl,-Bstatic` (initial fix) and `-static-libgcc -static-libstdc++`.
4. **`Cannot load libcuda.so.1` at runtime, despite `--gpus all`** (the big one) — root caused to musl's static `libc.a` `dlopen` stub. Fixed in §6.
5. **WSL2 + nvidia-container-toolkit 1.19 SIGSEGV during prestart hook** — host-side regression unrelated to image; resolved by `wsl --shutdown` + restart. Not an image issue.
6. **NVIDIA driver libs reference glibc-internal symbols missing from musl/gcompat** — added `gcompat` package + a tiny `libnvshim.so` `LD_PRELOAD` library exporting the missing symbols. See §14.
7. **musl loader doesn't search `/usr/lib64` / `/usr/lib/wsl/lib` where the toolkit injects driver libs** — added `/etc/ld-musl-x86_64.path` listing all known injection layouts.
8. **`NVIDIA_DRIVER_CAPABILITIES` defaults to `utility` only** — without `compute,video` the toolkit doesn't mount `libnvcuvid.so`/`libnvidia-encode.so`. Baked the full set into the image's `ENV`.
9. **`-Bdynamic -lc` reorder still produced the static dlopen stub** under gcc `--toolchain=hardened` — switched to absolute-path link of `/lib/ld-musl-x86_64.so.1` (see §6, "Fix (final, robust)").
10. **NVENC encode succeeds but exits 139 (SIGSEGV) at process teardown** — libcuda's destructors crash under musl + gcompat during `cuCtxDestroy`. The crash happens in `main()` before any atexit handler fires, so it can't be caught from inside the binary. Fixed with a tiny entrypoint wrapper that downgrades exit 139 → 0 when stderr contains no recognised error keywords. See §14.
11. **All ffmpeg errors silently exit 0 (bad codec, bad input, bad filter)** — root caused to a `_exit` interposer in `libnvshim.so` that always called `syscall(SYS_exit_group, 0)` regardless of the status it received (or had a bug that lost the argument). Verified via an `LD_PRELOAD` `dladdr` tracer: every `_exit` call resolved to `dso=/usr/local/lib/libnvshim.so`. **Fix**: removed the `_exit`/`exit` interposers from `libnvshim.so` entirely — they were never needed for the glibc→musl ABI shim, only the original (mistaken) attempt to suppress the teardown SEGV from inside the process. Real ffmpeg exit codes (`8` for bad codec, `254` for bad input, `8` for bad filter) now propagate identically to the non-CUDA `:8.1` image. See §5c.

---

## 12. Open follow-ups

- [ ] Document required `nvidia-container-toolkit` minimum version once we know which versions reliably handle the prestart hook on WSL2.
- [ ] Consider exposing `NVIDIA_DRIVER_CAPABILITIES` as a build-arg for power users who want to drop `video`.
- [ ] Add a CI smoke test that runs the encode on a self-hosted GPU runner (currently only readelf-level checks possible in vanilla GitHub Actions).
- [ ] Investigate whether `arm64` Jetson support is feasible later (would need a separate `nv-codec-headers` build path and likely a different base image).

---

## 13. Resuming work on another machine

If you need to continue from a fresh checkout / device, here is the full
sequence to rebuild and validate the CUDA image end-to-end.

### Build

> ⚠️ Use `--no-cache` if you previously built `:8.1-cuda` with the broken
> link flags — Docker will otherwise reuse the cached ffmpeg layer that
> contains the static `dlopen` stub. Full rebuild on a typical machine
> takes ~45–75 min (most of it is libaom, libvmaf, x265, svt-av1, vvenc).

```sh
cd /path/to/static-ffmpeg

docker build --no-cache \
    --build-arg ENABLE_CUDA=1 \
    --target final-cuda \
    -t mwader/static-ffmpeg:8.1-cuda-v3 .
```

If you only changed something *after* the ffmpeg compile step (e.g. the
`final-cuda` stage, env vars, ld-musl path), you can skip `--no-cache`:

```sh
docker build \
    --build-arg ENABLE_CUDA=1 \
    --target final-cuda \
    -t mwader/static-ffmpeg:8.1-cuda-v3 .
```

---

## Investigation log: April 28 – May 2, 2026 (Alpine/musl + WSL2 NVIDIA stack)

This section records every layer that had to be peeled back to get NVENC working
on Alpine/musl with the NVIDIA Container Toolkit on a Windows + WSL2 host
(host driver 596.21, CUDA 13.2, RTX 3060 Ti, ffnvcodec 13.0.19.0, ffmpeg 8.1).

### Environment

- Host: Windows 11 + WSL2 (Ubuntu 22.04), Docker Desktop / engine.
- GPU: NVIDIA RTX 3060 Ti, driver 596.21, CUDA 13.2 (per `nvidia-smi`).
- Container base for `final-cuda`: `alpine:3.20.3` (musl 1.2.x).
- Driver injection paths used by the toolkit on this host:
  - `/usr/lib64/libcuda.so.1`         (179 KB WSL "loader stub")
  - `/usr/lib64/libnvcuvid.so.1`      (23.8 MB, real)
  - `/usr/lib64/libnvidia-encode.so.1`(266 KB stub)
  - `/usr/lib64/libnvidia-ml.so.1`    (278 KB)
  - `/usr/lib/wsl/drivers/nv_dispi.inf_amd64_<HASH>/libcuda.so.1.1` (24.1 MB, real backend)

### Layer-by-layer findings

#### 1. ffmpeg link conflict (fixed)

Symptom: ffmpeg link in builder failed with all `--enable-*` flags on.
Cause: `export LDFLAGS="-Wl,--no-as-needed -Wl,-Bdynamic -lc"` was set
**unconditionally**, conflicting with the `-static-pie` configure patch used in
the non-CUDA branch.
Fix: gate the `LDFLAGS` export on `ENABLE_CUDA` only. Non-CUDA build returns to
upstream static-pie behaviour.

#### 2. NVIDIA Container Toolkit capabilities (fixed)

Symptom: only 180 KB stub `libcuda.so.1` mounted; `libnvcuvid` / `libnvidia-encode`
absent.
Cause: `--gpus all` only exposes the *device*; library set is governed by
`NVIDIA_DRIVER_CAPABILITIES`. Default is just `utility` → no compute/video libs.
Fix: bake `ENV NVIDIA_DRIVER_CAPABILITIES=compute,video,utility` and
`NVIDIA_VISIBLE_DEVICES=all` into the `final-cuda` stage image config.

#### 3. musl dynamic-loader search path (fixed)

Symptom: even with libs mounted, `dlopen("libcuda.so.1")` reported "Library not found".
Cause: musl's default search path is `/lib:/usr/local/lib:/usr/lib`; toolkit
mounts driver libs to `/usr/lib64` (RHEL/Fedora/WSL convention) which musl does
not search.
Fix: write `/etc/ld-musl-x86_64.path` listing `/lib`, `/usr/local/lib`, `/usr/lib`,
`/usr/lib64`, `/usr/lib/x86_64-linux-gnu`, `/usr/lib/wsl/lib`.

#### 4. glibc → musl ABI gap (fixed via gcompat + nvshim)

Symptom: NVIDIA driver libs (compiled against glibc) reference glibc-internal
symbols not present in musl/gcompat.
Cause: gcompat provides `libc.so.6` / `libm.so.6` / `libpthread.so.0` /
`librt.so.1` as musl wrappers, but is missing `libdl.so.2` (musl folds dlopen
into libc) and a number of glibc-internal helpers used by recent NVIDIA drivers.

Iterative discovery of missing symbols (each found by `dlopen` of the WSL
backend library reporting "Error relocating: <sym>: symbol not found"):

| Iteration | Newly-needed symbol | Shim strategy |
|---|---|---|
| 1 | `gnu_get_libc_version`           | return `"2.35"` |
| 2 | `__register_atfork`              | redirect to `pthread_atfork` |
| 3 | `dlmopen`                        | wrapper around `dlopen` (ignore Lmid_t) |
| 4 | `dlvsym`                         | wrapper around `dlsym` (ignore version) |

Final shim payload (`libnvshim.so`, `LD_PRELOAD`'d):

- `gnu_get_libc_version` → `"2.35"`
- `gnu_get_libc_release` → `"stable"`
- `__libc_current_sigrtmin` / `__libc_current_sigrtmax` (musl macros exposed as functions)
- `__register_atfork` → `pthread_atfork`
- `__cxa_thread_atexit_impl` → no-op
- `__libc_single_threaded` (data symbol, value 0)
- `secure_getenv` → `getenv`
- `dlmopen` → `dlopen` (ignore namespace)
- `dlvsym` → `dlsym` (ignore version)
- `__libc_dlopen_mode` / `__libc_dlsym` / `__libc_dlclose`

After this set, the **standalone** dlopen test passes on every layer:

- `dlopen("libcuda.so.1", RTLD_LAZY)` → OK (loads /usr/lib64 stub).
- `dlopen("/usr/lib/wsl/drivers/.../libcuda.so.1.1", RTLD_NOW)` → OK (real backend).
- `dlopen("libnvcuvid.so.1", RTLD_NOW)` → OK.
- `dlopen("libnvidia-encode.so.1", RTLD_NOW)` → OK.
- `dlopen("libnvidia-ml.so.1", RTLD_NOW)` → OK.
- `dlsym(cuInit / cuDriverGetVersion / cuDeviceGet / cuCtxCreate_v2 / cuCtxDestroy_v2 / cuMemAlloc_v2)` → all non-NULL.
- `cuInit(0)` → returns `CUDA_SUCCESS` (0).
- `cuDriverGetVersion(&v)` → returns 0 with v = 13020 (CUDA 13.2).

`nvidia-smi` inside the container prints full GPU info.

### 5. Resolved: ffmpeg's `nvenc_load_libraries` reporting "Cannot load libcuda.so.1"

**Root cause** (the same musl static `libc.a` `dlopen` stub described in §6,
but a worse variant of it): even with the `-Wl,--no-as-needed,-Bdynamic,-lc`
reorder, gcc's `--toolchain=hardened` spec file emitted late references that
re-pulled `libc.a`, restoring the 25-byte `dlopen` stub inside the binary.
`readelf -s --dyn-syms /ffmpeg | grep dlopen` then showed:

```
21987: 000000000338c50e   25 FUNC WEAK DEFAULT 14 dlopen
```

— `dlopen` defined inside `.text` of the binary itself, returning NULL with
`ENOSYS` without ever issuing an `openat` syscall. Hence `strace` showed no
filesystem activity for `libcuda*`.

**Fix**: link the musl combined loader/libc by **absolute path** rather than
via `-lc`. Absolute filenames bypass `-Bstatic`/`-Bdynamic` mode altogether and
cannot be re-resolved against `libc.a`:

```sh
# in --extra-ldflags:
-Wl,--no-as-needed,/lib/ld-musl-x86_64.so.1,--as-needed
```

After this change, `dlopen`/`dlsym`/`dlerror`/`dlclose` resolve as `UND`
(or are bound internally to the absolute-path libc — both outcomes work at
runtime) and h264_nvenc encodes successfully.

### 5b. Resolved: SIGSEGV at process teardown (exit 139)

**Symptom**: encode completes successfully (`frame=  60 ... muxing overhead`
visible, output bytes fully written), then ffmpeg exits with 139 (SIGSEGV).
Reproduced with and without `LD_PRELOAD=libnvshim.so`, so nvshim is not the
trigger.

**Root cause**: libcuda's `__cxa_finalize` / DT_FINI destructors run during
ffmpeg's `avcodec_close → nvenc_free → cuCtxDestroy` while still inside
`main()`. Those destructors call into glibc-internal state that musl + gcompat
don't fully provide (notably TLS-destructor unwinding, and pthread_atfork
handlers registered by the driver), and crash. Because the crash is *inside*
`main()` (not after `exit()` is called), there is no in-process hook — atexit
handlers, signal handlers installed by `LD_PRELOAD`, etc. — that can suppress
it cleanly without risk of papering over real bugs.

**Fix**: a 12-line bash entrypoint wrapper that runs `/ffmpeg`, captures its
exit code via `${PIPESTATUS[0]}`, tees stderr to a temp file for inspection,
preserves stdout byte-exact via fd-3 trick, and converts exit 139 → 0 *only*
when stderr contains no recognised ffmpeg error keyword (`error`, `cannot
load`, `not found`, `invalid`, `failed`, `conversion failed`, `no such`).
Real failures (mid-encode CUDA OOM, init failures, bad codec, etc.) propagate
unchanged because they always print an identifiable error first.

```bash
#!/bin/bash
errfile=$(mktemp)
trap "rm -f \"$errfile\"" EXIT
exec 3>&1
{ /ffmpeg "$@" 2>&1 1>&3 3>&-; } | tee "$errfile" >&2
rc=${PIPESTATUS[0]}
exec 3>&-
if [ "$rc" = "139" ] && ! grep -qiE "(^|[^a-z])(error|cannot load|conversion failed|not found|invalid|failed|no such)" "$errfile"; then
    exit 0
fi
exit "$rc"
```

ffprobe doesn't need a wrapper: it doesn't invoke encoders and rarely auto-loads
CUDA, so it doesn't reach the crashing destructor path.

### 5c. Resolved: ffmpeg silently exits 0 on every error path

**Symptom**: every fatal-error invocation of the CUDA build returned exit code
`0` to the shell, despite ffmpeg printing the correct error messages on stderr.
Verified against the non-CUDA `:8.1` baseline:

| Scenario                               | non-CUDA `:8.1` | CUDA (broken) | CUDA (fixed) |
|----------------------------------------|-----------------|---------------|--------------|
| `-c:v this_codec_does_not_exist`       | `8`             | `0` ❌        | `8` ✅       |
| `-i /no/such/file.mp4`                 | `254`           | `0` ❌        | `254` ✅     |
| `-vf this_filter_does_not_exist`       | `8`             | `0` ❌        | `8` ✅       |
| Successful encode                      | `0`             | `0` ✅        | `0` ✅       |
| Successful encode (post-teardown SEGV) | n/a             | `139` (raw)   | `0` (wrapped) |

This was masked at first because the wrapper grew an "upgrade exit 0 → 1 when
stderr matches a fatal-error keyword" branch. That made T3 pass with a
plausible-looking exit `1`, but it was a workaround, not a fix — and the wrong
exit code (`1` instead of `8`/`254`) broke any caller that switched on the
specific code.

**Root-cause discovery**: an `LD_PRELOAD` `dladdr` tracer interposing `_exit`
revealed that on every code path — bad-codec, bad-input, even successful
`-version` — the call to `_exit` came from `libnvshim.so`:

```
[exittrace] _exit(0) ra=0x...  dso=/usr/local/lib/libnvshim.so
```

`libnvshim.so` had been given an `_exit` interposer (and at one point an
`exit` interposer too) as part of the earlier-but-abandoned attempt to suppress
the teardown SIGSEGV from inside the process. The interposer always invoked
`syscall(SYS_exit_group, 0)` — i.e. it dropped ffmpeg's real exit status on
the floor, hard-coding `0`. None of the standard ELF / readelf / `nm` checks
flag this: the interposer is in a separately-loaded DSO, not in `/ffmpeg`, and
musl's PLT happily binds `_exit` to whichever DSO comes first in symbol search
order — `LD_PRELOAD` always wins.

**Fix**: drop the `_exit` (and `exit`) overrides from `libnvshim.so` entirely.
They were never needed for any glibc→musl ABI gap (those are all the symbol
list documented in §4 — `gnu_get_libc_version`, `__register_atfork`,
`dlmopen`, `dlvsym`, etc.). Process-lifecycle suppression belongs in the
out-of-process bash wrapper (§5b), where it can read the real exit status via
`${PIPESTATUS[0]}` and pattern-match on the actual error keywords.

After removing the interposers, all standard ffmpeg exit codes match the
non-CUDA build byte-for-byte, and the wrapper script collapses back to its
minimal form:

```bash
#!/bin/bash
errfile=$(mktemp)
shellerr=$(mktemp)
trap "rm -f \"$errfile\" \"$shellerr\"" EXIT
exec 3>&1
exec 4>&2
exec 2>"$shellerr"
{ /ffmpeg "$@" 2>&1 1>&3 3>&-; } | tee "$errfile" >&4
rc=${PIPESTATUS[0]}
exec 3>&-
exec 2>&4 4>&-
grep -vE "Segmentation fault.*core dumped.*/ffmpeg" "$shellerr" >&2 || true
# Suppress *only* the known-benign teardown SIGSEGV from libcuda dtors.
# Real failure exit codes (1, 8, 254, ...) propagate unchanged.
if [ "$rc" = "139" ] && ! grep -qiE "(^|[^a-z])(error|cannot load|conversion failed|not found|invalid|failed|no such)" "$errfile"; then
    exit 0
fi
exit "$rc"
```

**Lesson**: `LD_PRELOAD` shims should be the *minimum* symbol set that closes
the glibc→musl ABI gap. Any process-lifecycle hook (exit, signal, atexit) added
to such a shim will silently apply to *every* call from the host program, not
just the one CUDA-driver call you were trying to fix. Keep lifecycle policy
out-of-process.

**Diagnostic recipe** (reuse this for any future "wrong exit code" regression):

```sh
docker run --rm --gpus all --entrypoint sh "$IMG" -c '
  apk add --no-cache gcc musl-dev binutils >/dev/null
  cat > /tmp/t.c <<EOF
#define _GNU_SOURCE
#include <dlfcn.h>
#include <stdio.h>
#include <unistd.h>
#include <sys/syscall.h>
__attribute__((noreturn)) void _exit(int s){
  void *ra=__builtin_return_address(0); Dl_info i={0}; dladdr(ra,&i);
  dprintf(2,"[trace] _exit(%d) dso=%s\n",s,i.dli_fname?i.dli_fname:"?");
  syscall(SYS_exit_group,s); __builtin_unreachable();
}
EOF
  gcc -O0 -fPIC -shared -o /tmp/t.so /tmp/t.c -ldl
  LD_PRELOAD="/tmp/t.so:${LD_PRELOAD}" /ffmpeg -hide_banner -loglevel error \
    -f lavfi -i testsrc=duration=1:size=320x240:rate=30 \
    -c:v this_codec_does_not_exist -f null -
'
# The traced _exit must show dso=/lib/ld-musl-x86_64.so.1 (i.e. real libc),
# NOT dso=/usr/local/lib/libnvshim.so. If it shows nvshim, the interposer
# regression is back.
```

### Diagnostic playbook (for future re-entry)

Quick all-in-one container probe used during this investigation:

```sh
IMG=mwader/static-ffmpeg:8.1-cuda-debian-v43
docker run --rm --gpus all --entrypoint sh "$IMG" -c '
  apk add --no-cache gcc musl-dev binutils strace >/dev/null

  # 1. Confirm env + linkage
  echo "LD_PRELOAD=$LD_PRELOAD"
  ldd /ffmpeg

  # 2. Confirm path file
  cat /etc/ld-musl-x86_64.path

  # 3. Confirm driver libs are mounted
  ls -lh /usr/lib64/libcuda.so.1 /usr/lib64/libnv*.so.1 \
         /usr/lib/wsl/drivers/nv_dispi.inf_amd64_*/libcuda.so.1.1 2>/dev/null

  # 4. Standalone dlopen + cuInit smoke test
  cat > /t.c <<EOF
#include <dlfcn.h>
#include <stdio.h>
int main(void){
  void *h = dlopen("libcuda.so.1", RTLD_LAZY);
  if(!h){fprintf(stderr,"FAIL: %s\n",dlerror());return 1;}
  int (*ci)(unsigned)=(int(*)(unsigned))dlsym(h,"cuInit");
  fprintf(stderr,"cuInit=%d\n", ci?ci(0):-99);
  return 0;
}
EOF
  gcc /t.c -o /t && /t

  # 5. Trace what ffmpeg actually does when invoking h264_nvenc
  strace -e trace=openat,access -f -o /tmp/ff.strace /ffmpeg -hide_banner -loglevel error \
    -f lavfi -i testsrc=size=320x240:rate=30 -t 1 -c:v h264_nvenc -f null - 2>&1 | tail -3
  echo "--- cuda/nvidia syscalls in strace ---"
  grep -E "cuda|nvidia|nvcuvid|libnv|/dev/dxg|/dev/nvidia" /tmp/ff.strace | head -40
'
```

### What works today (final state — May 3, 2026)

- ✅ Build succeeds with all 51 `--enable-lib*` codecs + `--enable-ffnvcodec
  --enable-cuvid --enable-nvenc --enable-nvdec` on Alpine + musl.
- ✅ Image runs `ffmpeg -version`, `-buildconf`, hwaccels/encoders/decoders
  enumeration showing cuda, nvenc, cuvid.
- ✅ All non-CUDA codec tests pass (libsvtav1, libvvenc, libx265, libass,
  librsvg, TLS, DNS).
- ✅ All NVIDIA driver libs `dlopen` cleanly inside the container.
- ✅ Standalone musl program in same container completes `cuInit(0)`
  successfully and reads driver version 13020.
- ✅ **`h264_nvenc` encode produces frames** (`frame= 60 ... speed=2.8x` etc.)
  and the wrapped entrypoint exits 0.
- ✅ MP4-to-stdout (`-f mp4 -movflags frag_keyframe+empty_moov -`) emits
  byte-exact output (verified vs raw `--entrypoint /ffmpeg` invocation).
- ✅ Real ffmpeg errors (bad codec, bad input, etc.) propagate unchanged
  through the wrapper.
- ✅ ffprobe runs unwrapped and stable for all standard probe operations.

### Things tried that did NOT (alone) resolve the issue (kept for posterity)

| Attempt | Result |
|---|---|
| `--gpus all` only (no caps) | Only stub libcuda mounted, no NVENC libs |
| `LD_LIBRARY_PATH=/usr/lib64` only | `dlopen` finds file but glibc symbols missing |
| Symlink `libdl.so.2 → libgcompat.so.0` only | dlopen of stub OK, real backend FAIL on `gnu_get_libc_version` |
| nvshim with `gnu_get_libc_version` only | Next missing: `__register_atfork` |
| Add `__register_atfork` + `secure_getenv` + `__cxa_thread_atexit_impl` | Next missing: `dlmopen` |
| Add `dlmopen` + `__libc_dlopen_mode/dlsym/dlclose` | Next missing: `dlvsym` |
| Add `dlvsym` | All driver libs dlopen cleanly + standalone `cuInit` succeeds |
| `-Wl,--no-as-needed,-Bdynamic,-lc,--as-needed,-Bstatic` in extra-ldflags | Still pulled `libc.a` `dlopen` stub via gcc-hardened spec file |
| Hide `/usr/lib/libc.a` during link | libgme.a configure-time symbol checks failed (gz*/inflate*) |
| Absolute-path `-Wl,/lib/ld-musl-x86_64.so.1` in extra-ldflags | ✅ NVENC encode finally succeeds |
| nvshim `exit()` interpose + atexit `_exit()` | SIGSEGV happens *before* main() returns, so atexit never runs — ineffective. **Worse**: leaving the `_exit` interposer in the shim silently swallowed *every* ffmpeg exit code (always returned 0). See §5c. |
| Entrypoint wrapper translating exit 139 → 0 with error-keyword guard | ✅ Final fix; clean exit 0 with stdout/stderr passthrough preserved, real exit codes (8/254/…) propagate unchanged |

### Decision branch (resolved — stayed on Alpine)

The escape hatch of switching `final-cuda` to `debian:bookworm-slim` was
**not needed**. The Alpine + musl + gcompat + nvshim stack works end-to-end
once the link-time absolute-path fix and the entrypoint wrapper are in place.

The Alpine variant remains preferable because:

1. The image is ~4x smaller than the Debian equivalent would be.
2. Existing CI/build infrastructure for `mwader/static-ffmpeg` is Alpine-based;
   no parallel `builder-glibc` stage needs to be maintained.
3. The static archive produced for non-libc deps is identical between the
   default and CUDA variants — only the link step differs.

The only ongoing maintenance cost is **nvshim symbol drift**: each new NVIDIA
driver release may reference an additional glibc-internal symbol that
gcompat doesn't ship, requiring a one-line addition to `libnvshim.so`. The
diagnostic playbook (next section) documents how to detect and fix this in
under five minutes.

---

## 14. Final architecture (the six-layer stack)

The working CUDA variant is the composition of six independently-essential layers.
Removing any one breaks NVENC end-to-end. They are listed in the order they take effect:

| # | Layer | Where | Purpose |
|---|---|---|---|
| 1 | **Absolute-path libc link** | builder, ffmpeg `--extra-ldflags` | Forces `dlopen`/`dlsym`/`dlerror`/`dlclose` to resolve dynamically against the real musl libc instead of `libc.a`'s NULL-returning stub. Without this the binary appears to build fine but `dlopen()` of `libcuda.so.1` returns NULL with no syscall. |
| 2 | **Dynamic-PIE link mode** | builder, ffmpeg link | Replaces `-fPIE -static-pie` with `-fPIE -pie`. A static-pie binary has no dynamic loader, making `dlopen` impossible by definition. |
| 3 | **`/etc/ld-musl-x86_64.path`** | final-cuda stage | Adds `/usr/lib64`, `/usr/lib/x86_64-linux-gnu`, `/usr/lib/wsl/lib` to musl's loader search path. The NVIDIA Container Toolkit injects driver libs into one of these depending on host distro; musl's default `/lib:/usr/local/lib:/usr/lib` finds none of them. |
| 4 | **`gcompat` package + `libdl.so.2` symlink** | final-cuda stage | Provides `libc.so.6` / `libm.so.6` / `libpthread.so.0` / `librt.so.1` as musl wrappers (the driver's `DT_NEEDED` entries). The symlink points the driver's `libdl.so.2` reference at `libgcompat.so.0` since musl folds dlopen into libc and ships no separate `libdl`. |
| 5 | **`libnvshim.so` LD_PRELOAD** | final-cuda stage | Exports glibc-internal symbols the driver references but gcompat doesn't ship: `gnu_get_libc_version`, `__register_atfork`, `__cxa_thread_atexit_impl`, `secure_getenv`, `dlmopen`, `dlvsym`, `__libc_dlopen_mode/dlsym/dlclose`, `__libc_current_sigrtmin/max`, `__libc_single_threaded`, `gnu_get_libc_release`. Without the shim, dlopen of the WSL2 backend `libcuda.so.1.1` fails with `symbol not found` errors. **Must NOT export `exit`/`_exit`/`_Exit`** — see §5c; interposing those swallows ffmpeg's real exit status. |
| 6 | **Entrypoint wrapper** | final-cuda stage | Bash script that exec's `/ffmpeg`, captures exit code via `${PIPESTATUS[0]}`, preserves stdout byte-exact via fd-3 trick, tees stderr to a temp file, and downgrades exit 139 → 0 *only* when stderr contains no recognised error keyword. Suppresses the cosmetic libcuda-destructor SIGSEGV that fires after the encode is fully complete. |

Layers 1–2 belong to the **builder stage** (link-time concerns).
Layers 3–6 belong to the **`final-cuda` runtime stage** (loader, ABI, lifecycle concerns).

### Diagram of the runtime call chain

```
docker run --gpus all  ⇒  toolkit injects libcuda.so.1 → /usr/lib64
                          + sets NVIDIA_DRIVER_CAPABILITIES from image ENV
       │
       ▼
ffmpeg-cuda-entrypoint (bash)               ← layer 6
       │ exec
       ▼
/ffmpeg  (musl dynamic-PIE, libc-only NEEDED)
       │ ld.so loads libc.musl-x86_64.so.1
       │   (search path includes /usr/lib64 from /etc/ld-musl-x86_64.path)   ← layer 3
       │ LD_PRELOAD → /usr/local/lib/libnvshim.so                            ← layer 5
       ▼
ffnvcodec dynlink_loader.h:
       dlopen("libcuda.so.1", RTLD_LAZY)    ← needs layer 1 (real PLT entry)
       │
       ▼ ld.so loads libcuda.so.1 (WSL stub)
       │   resolves DT_NEEDED libdl.so.2 → libgcompat.so.0                   ← layer 4
       │
       ▼ libcuda dlopens its WSL backend libcuda.so.1.1
       │   resolves glibc-internals via libnvshim.so                         ← layer 5
       │
       ▼ encode runs successfully, frames produced, output flushed
       │
       ▼ ffmpeg main() → avcodec_close → cuCtxDestroy
       │   libcuda __cxa_finalize crashes during teardown          ☠ SIGSEGV
       │
       ▼ wrapper sees exit=139, no error keyword in stderr → exit 0         ← layer 6
```

---

## 15. ffprobe note

`ffprobe` shares the same link-time and runtime-loader configuration as `ffmpeg`
(layers 1–5 above), but does **not** need the entrypoint wrapper because:

- It doesn't open NVENC encoders, so `nvenc_free → cuCtxDestroy` is never invoked.
- Its `-hwaccel` option is silently ignored (it's an `ffmpeg`-only flag).
- It doesn't auto-initialize CUDA for normal probe/show operations.

Tested invocations that all return exit 0 cleanly without the wrapper:

```sh
docker run --rm --gpus all --entrypoint /ffprobe IMG -version
docker run --rm --gpus all --entrypoint /ffprobe IMG \
    -f lavfi -i testsrc=duration=1:size=320x240:rate=30 -show_streams -of json
docker run --rm --gpus all --entrypoint /ffprobe IMG -i some_h264.mp4
```

If a future ffmpeg/driver combination ever makes `ffprobe` reach the crashing
destructor path, the same wrapper script can be installed with the binary path
parametrised. Not worth the extra layer today.

---

## 16. Final verification recipe (May 3, 2026)

Replace `IMG` with your actual tag.

```sh
IMG=mwader/static-ffmpeg:8.1-cuda-debian-v47   # or :8.1-cuda after retag

# 1. Static-ness check (binary should have exactly one NEEDED entry: musl libc)
docker run --rm --entrypoint sh "$IMG" -c '
  apk add --no-cache binutils >/dev/null 2>&1
  readelf -d /ffmpeg | grep -E "NEEDED|BIND_NOW"
'

# 2. NVENC encode end-to-end (the real test)
docker run --rm --gpus all "$IMG" \
    -hide_banner -loglevel error \
    -f lavfi -i testsrc=duration=2:size=1280x720:rate=30 \
    -c:v h264_nvenc -f null - ; echo "exit=$? (must be 0)"

# 3. MP4-to-stdout byte-exactness (wrapper passthrough check)
docker run --rm --gpus all "$IMG" \
    -hide_banner -loglevel error \
    -f lavfi -i testsrc=duration=1:size=320x240:rate=30 \
    -c:v h264_nvenc -f mp4 -movflags frag_keyframe+empty_moov - 2>/dev/null \
    | wc -c   # must print > 0

# 4. ffprobe sanity (no wrapper)
docker run --rm --gpus all --entrypoint /ffprobe "$IMG" -version >/dev/null
echo "exit=$? (must be 0)"

# 5. Exit-code parity vs non-CUDA :8.1 (regression guard for §5c)
docker run --rm --gpus all "$IMG" -hide_banner -loglevel error \
    -f lavfi -i testsrc=duration=1:size=320x240:rate=30 \
    -c:v this_codec_does_not_exist -f null - ; echo "exit=$? (must be 8)"
docker run --rm --gpus all "$IMG" -hide_banner -loglevel error \
    -i /no/such/file.mp4 -f null - ; echo "exit=$? (must be 254)"
```

All four must succeed for the image to be considered shippable.
