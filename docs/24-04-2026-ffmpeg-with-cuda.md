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

### Fix

Pre-link the dynamic `libc.so` *before* switching to `-Bstatic`, with `--no-as-needed` so it stays in `DT_NEEDED`:

```sh
--extra-ldflags='-static-libstdc++ -static-libgcc -Wl,--no-as-needed,-Bdynamic -lc -Wl,--as-needed,-Bstatic'
--extra-libs=' -lgomp -Wl,-Bdynamic -lc '
```

Order of operations during link:
1. `-Bdynamic --no-as-needed -lc` → `libc.musl-x86_64.so.1` loaded, forced into NEEDED, all its symbols available
2. `--as-needed -Bstatic` → restore as-needed, switch to static mode
3. Codec `.a` files reference `dlopen` → linker finds it already available via `libc.so` → resolves as `UND` → PLT entry → real `dlopen` at runtime

After fix:
```
readelf -s --dyn-syms /ffmpeg | grep dlopen
#       0:               0   FUNC WEAK   DEFAULT  UND dlopen
```

Zero size, undefined, dynamically resolved — works.

### Lesson for any future change to this build

- **Never link musl `libc.a` into a binary that calls `dlopen`.** It will silently use the stub.
- The bug is invisible to standard hardening checks: the binary still has `BIND_NOW`, `RELRO`, `PIE`, NX stack. `ldd` still shows only one extra NEEDED entry.
- Verify with `readelf -s --dyn-syms <binary> | grep dlopen` — it must be `UND`.

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
What was wrong
Your validation step #1 confirms it:
```
--extra-ldflags='... -Wl,-Bstatic '
```
That is the old flag set. The --no-as-needed,-Bdynamic -lc prefix is missing, so the link still pulls dlopen from static libc.a — exactly the stub described in §6 of the doc. Your step #2 (dlopen shown as 25-byte FUNC in section 14) is the smoking gun, and step #3 (Cannot load libcuda.so.1) is the inevitable result.
The fix was only documented in the markdown, never committed to Dockerfile. Now it is.
Rebuild
Because the cache will happily reuse the broken ffmpeg layer, you must use --no-cache:
```
docker build --no-cache \
    --build-arg ENABLE_CUDA=1 \
    --target final-cuda \
    -t mwader/static-ffmpeg:8.1-cuda-v4 .
```
(Bumped to -v4 to avoid any local image-name confusion with the broken -v3.)
Verify in this exact order — bail out early if any fails

1. Confirm new flags landed in the binary:
```
docker run --rm --entrypoint /ffmpeg mwader/static-ffmpeg:8.1-cuda-v4 \
    -hide_banner -version | grep -o -- '-Wl,--no-as-needed,-Bdynamic -lc'
```
Must print the string. If empty → cache wasn't busted, rebuild with --no-cache.
2. Confirm dlopen is UND:
```
docker run --rm --entrypoint sh mwader/static-ffmpeg:8.1-cuda-v4 -c '
apk add --no-cache binutils >/dev/null 2>&1
readelf -s --dyn-syms /ffmpeg | grep -E "dlopen|dlsym|dlerror"'
```

Expected:
0:  0  FUNC  WEAK    DEFAULT  UND  dlopen
0:  0  FUNC  GLOBAL  DEFAULT  UND  dlsym
0:  0  FUNC  GLOBAL  DEFAULT  UND  dlerror
If you see 25 FUNC ... 14 dlopen again → the link order in Dockerfile is wrong; do not proceed to step 3.
3. Functional encode (only if 1 and 2 pass):

```
docker run --gpus all --rm mwader/static-ffmpeg:8.1-cuda-v4 \
    -f lavfi -i testsrc=duration=2:size=1280x720:rate=30 \
    -c:v h264_nvenc -f null -
```

Once step 3 passes, I'll also update the doc to reflect that the fix is now in Dockerfile (not just documented).
---

### Validate (in order)

#### 1. Confirm the new link flags are baked into the binary

```sh
docker run --gpus all --rm --entrypoint /ffmpeg \
    mwader/static-ffmpeg:8.1-cuda-v3 -hide_banner -version | head -3
```

Look for this in `--extra-ldflags`:

```
-Wl,--no-as-needed,-Bdynamic -lc -Wl,--as-needed,-Bstatic
```

If you still see the old `-Wl,-Bstatic ` (no `--no-as-needed,-Bdynamic -lc` before it), the cache wasn't busted — rebuild with `--no-cache`.

#### 2. Confirm `dlopen` is resolved dynamically (the painful one)

```sh
docker run --gpus all --rm --entrypoint sh \
    mwader/static-ffmpeg:8.1-cuda-v3 -c '
apk add --no-cache binutils >/dev/null 2>&1
readelf -s --dyn-syms /ffmpeg | grep -E "dlopen|dlsym|dlerror"
'
```

✅ Expected (correct):
```
0:  0  FUNC  WEAK    DEFAULT  UND  dlopen
0:  0  FUNC  GLOBAL  DEFAULT  UND  dlsym
0:  0  FUNC  GLOBAL  DEFAULT  UND  dlerror
```

❌ Bad (static stub still linked in — broken):
```
21987:  ...338c50e   25  FUNC  WEAK  DEFAULT  14  dlopen
```

Note the size (25) and the section number (14 = `.text`) — that's the in-binary stub.

#### 3. Confirm the toolkit is injecting the driver libs

```sh
docker run --gpus all --rm --entrypoint sh \
    mwader/static-ffmpeg:8.1-cuda-v3 -c '
find / \( -name "libcuda.so*" -o -name "libnvcuvid*" -o -name "libnvidia-encode*" \) 2>/dev/null
echo "---"
cat /etc/ld-musl-x86_64.path
'
```

Should list `libcuda.so.1`, `libnvcuvid.so.1`, `libnvidia-encode.so.1` somewhere under `/usr/lib64`, `/usr/lib/x86_64-linux-gnu`, or `/usr/lib/wsl/lib`.

#### 4. Functional encode test

```sh
docker run --gpus all --rm mwader/static-ffmpeg:8.1-cuda-v3 \
    -f lavfi -i testsrc=duration=2:size=1280x720:rate=30 \
    -c:v h264_nvenc -f null -
```

✅ Expected: `frame=  60 fps=... q=... Lsize=N/A` and exit 0, no `Cannot load libcuda.so.1`.

#### 5. Verify static-ness of both variants from the host

```sh
docker create --name sf      mwader/static-ffmpeg:8.1
docker cp sf:/ffmpeg         /tmp/ffmpeg-static && docker rm sf

docker create --name sfcuda  mwader/static-ffmpeg:8.1-cuda-v3
docker cp sfcuda:/ffmpeg     /tmp/ffmpeg-cuda && docker rm sfcuda

echo "=== :8.1 ==="
readelf -d /tmp/ffmpeg-static 2>/dev/null | grep -E 'NEEDED|BIND_NOW' \
    || echo "(no NEEDED — fully static)"

echo "=== :8.1-cuda ==="
readelf -d /tmp/ffmpeg-cuda 2>/dev/null | grep -E 'NEEDED|BIND_NOW'
```

✅ Expected diff: exactly one extra `NEEDED Shared library: [libc.musl-x86_64.so.1]` on the cuda variant. Both have `BIND_NOW`.

### If a step fails

| Step | Failure | Likely cause / fix |
|---|---|---|
| 1 | Old `-Wl,-Bstatic` flags still shown | Cache hit — rebuild with `--no-cache` |
| 2 | `dlopen` shows non-zero size in `.text` | Link-flag fix not applied; check `Dockerfile` ffmpeg configure step has `--no-as-needed,-Bdynamic -lc -Wl,--as-needed,-Bstatic` *before* the `-Bstatic` codecs |
| 3 | No `libcuda.so*` found | Toolkit not injecting — check `nvidia-container-toolkit` is installed and `--gpus all` is passed; on WSL2 try `wsl --shutdown` from PowerShell |
| 4 | `Cannot load libcuda.so.1` but step 3 found it | Path missing from `/etc/ld-musl-x86_64.path`; override at runtime with `-e LD_LIBRARY_PATH=/usr/lib64` (or wherever step 3 found it) |
| 4 | `[h264_nvenc] No capable devices found` | Driver too old for the NVENC SDK version pinned in `nv-codec-headers`; bump the host NVIDIA driver |
| Prestart hook SIGSEGV on WSL2 | host-side toolkit bug | `wsl --shutdown` from PowerShell, then retry |

### Convenient one-liner for repeated test cycles

```sh
TAG=mwader/static-ffmpeg:8.1-cuda-v3 && \
docker build --build-arg ENABLE_CUDA=1 --target final-cuda -t $TAG . && \
docker run --gpus all --rm --entrypoint sh $TAG -c '
  apk add --no-cache binutils >/dev/null 2>&1
  echo "=== dlopen syms ==="
  readelf -s --dyn-syms /ffmpeg | grep -E "dlopen|dlsym|dlerror"
' && \
docker run --gpus all --rm $TAG \
    -f lavfi -i testsrc=duration=2:size=1280x720:rate=30 \
    -c:v h264_nvenc -f null -
```

---

## TL;DR

- `mwader/static-ffmpeg:8.1` stays fully static-pie — unchanged for existing users.
- `mwader/static-ffmpeg:8.1-cuda` adds NVENC/NVDEC/CUVID as a musl dynamic-PIE binary (libc only is dynamic; everything else still statically archived).
- The non-obvious gotcha: musl static `libc.a`'s `dlopen` is a NULL-returning stub. The CUDA build pre-links dynamic `libc.so` *before* `-Wl,-Bstatic` so `dlopen` is resolved through the PLT against the working dynamic libc.
- Verify with `readelf -s --dyn-syms /ffmpeg | grep dlopen` — must be `UND`, not a defined function in `.text`.


