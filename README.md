# Build LGPL libmpv — Linux, Windows, Android

This repository provides a GitHub Actions workflow (`build-libmpv.yaml`) that produces **LGPL-only** builds of [libmpv](https://github.com/mpv-player/mpv) for use in the Emby media player. Every component is compiled as a **shared library** and every GPL-only code path is disabled at configure time, keeping the entire output under the GNU Lesser General Public License.

## Target platforms

| Platform | Architecture | Rendering path | HW decode | TLS |
|---|---|---|---|---|
| Linux | x86_64 | OpenGL (gpu / gpu-next) | VAAPI, VDPAU | GnuTLS |
| Windows | x86_64 | Vulkan + OpenGL (gpu-next) | D3D11VA | SChannel |
| Android | armeabi-v7a, arm64-v8a | OpenGL ES (gpu / gpu-next) | MediaCodec | GnuTLS |

## How to run

The workflow is triggered manually via `workflow_dispatch`. Every version parameter has a tested default; override any of them at dispatch time.

| Parameter | Default | Purpose |
|---|---|---|
| `ffmpeg_version` | `n6.1.1` | FFmpeg release tag |
| `mpv_version` | `0.38.0` | mpv release (without leading `v`) |
| `libplacebo_version` | `v6.338.2` | libplacebo release tag |
| `libass_version` | `0.17.4` | libass release tag |
| `vcpkg_ref` | *(pinned commit)* | vcpkg tree commit for Android deps |
| `android_api_level` | `24` | Minimum Android API |
| `windows_strip_mode` | `strip-debug` | `strip-debug` / `strip-unneeded` / `none` |
| `build_linux` | `true` | Enable the Linux job |
| `build_windows` | `true` | Enable the Windows job |
| `build_android_armeabi_v7a` | `true` | Enable Android ARMv7 |
| `build_android_arm64_v8a` | `true` | Enable Android AArch64 |

## Build artifacts

Each successful run publishes the following GitHub Actions artifacts.

| Artifact | Contents |
|---|---|
| `source-code-lgpl` | Complete source trees of FFmpeg, mpv, libplacebo, libass, plus the workflow file and a version manifest |
| `vcpkg-src` | Pinned vcpkg source tree (consumed only by the Android job) |
| `libmpv-lgpl-linux-x86_64` | `libmpv.so`, FFmpeg/libass/libplacebo shared objects, and mpv headers |
| `libmpv-lgpl-windows-x86_64` | `libmpv-2.dll`, all runtime DLLs (FFmpeg, libass, libplacebo, MinGW runtime, Vulkan loader, shaderc), and mpv headers |
| `libmpv-lgpl-android-armeabi-v7a` | `libmpv.so`, FFmpeg/libplacebo shared objects, and mpv headers |
| `libmpv-lgpl-android-arm64-v8a` | Same as above for AArch64 |

## Workflow structure

The workflow consists of four jobs.

**prepare-sources** runs on `ubuntu-latest` and clones each upstream repository at its pinned tag, bundles the source trees, and uploads them as artifacts. It also applies a Python 3.14 compatibility patch to libplacebo and, for Android, patches the GnuTLS vcpkg portfile to fix a `CRAU_MAYBE_UNUSED` macro issue with Android clang.

**linux-x86_64** runs on `ubuntu-22.04` (glibc 2.35 for broad compatibility). It installs system dependencies via apt, builds FFmpeg with GnuTLS/VAAPI/VDPAU, then builds libass, libplacebo (OpenGL), and finally libmpv.

**windows-x86_64** runs on `windows-latest` with an MSYS2/MINGW64 toolchain. FFmpeg is built with SChannel for TLS. libplacebo is built with Vulkan and shaderc for SPIR-V compilation. All runtime DLLs (including transitive MinGW dependencies) are collected and optionally stripped.

**android** runs on `ubuntu-latest` and uses the NDK r26d cross-compilation toolchain. A vcpkg-based step builds libass, GnuTLS, and their transitive dependencies as static libraries. FFmpeg, libplacebo (OpenGL ES), and libmpv are then cross-compiled as shared libraries for each selected ABI.

---

## LGPL compliance

This section documents how this build system satisfies the obligations of the **GNU Lesser General Public License, version 2.1** (LGPL-2.1-or-later). It is intended for distributors who ship binaries produced by this workflow.

### Components and licenses

| Component | Version | License | LGPL? | Source |
|---|---|---|---|---|
| FFmpeg | n6.1.1 | LGPL-2.1-or-later ¹ | Yes | https://git.ffmpeg.org/ffmpeg.git |
| mpv (libmpv) | 0.38.0 | LGPL-2.1-or-later ² | Yes | https://github.com/mpv-player/mpv |
| libplacebo | v6.338.2 | LGPL-2.1-or-later | Yes | https://github.com/haasn/libplacebo |
| libass | 0.17.4 | ISC | No | https://github.com/libass/libass |
| GnuTLS | *(vcpkg / system)* | LGPL-2.1-or-later | Yes | https://www.gnutls.org |
| FreeType | *(system)* | FTL or GPL-2.0 | No ³ | https://freetype.org |
| FriBidi | *(system)* | LGPL-2.1-or-later | Yes | https://github.com/fribidi/fribidi |
| HarfBuzz | *(system)* | MIT | No | https://github.com/harfbuzz/harfbuzz |
| shaderc | *(MSYS2, Windows only)* | Apache-2.0 | No | https://github.com/google/shaderc |
| Vulkan Loader | *(MSYS2, Windows only)* | Apache-2.0 | No | https://github.com/KhronosGroup/Vulkan-Loader |

¹ FFmpeg is configured with `--disable-gpl --disable-version3`, which excludes all GPL-only and (L)GPL-3-only modules. The resulting libraries are licensed under LGPL-2.1-or-later.

² mpv is configured with `-Dgpl=false`, which disables all code paths that would require GPL licensing. The resulting libmpv is licensed under LGPL-2.1-or-later.

³ FreeType is used under its permissive FreeType License (FTL), not under GPL-2.0.

### How GPL code is excluded

The following configure flags ensure that no GPL-only code is compiled into any output binary.

**FFmpeg** (all platforms):

```
--disable-gpl --disable-version3
```

These flags cause FFmpeg's configure script to reject any module whose license would require GPL or LGPL-3.0+. The build will fail if a GPL-only dependency is accidentally linked.

**mpv** (all platforms):

```
-Dgpl=false
```

This flag tells meson to exclude all GPL-only source files from the build. Attempting to enable a GPL-dependent feature with this flag set will produce a configuration error.

**FFmpeg Windows-specific exclusions**:

```
--disable-cuda-llvm --disable-cuvid --disable-ffnvcodec --disable-nvdec --disable-nvenc
```

These explicitly disable NVIDIA's proprietary codec SDK headers, which carry their own restrictive license.

### Source code availability (LGPL §6(a) / §6(b))

The LGPL requires that recipients of the binaries can obtain the **complete corresponding source code** for all LGPL-licensed libraries contained in the binary distribution.

This workflow satisfies that obligation in two ways:

1. **Artifact-based distribution.** Every workflow run produces a `source-code-lgpl` artifact containing the exact source trees (at the exact commit) used to build the binaries. Distributors should retain and make this artifact available alongside the binaries for at least three years, or for as long as they offer spare parts or customer support for the product (whichever is longer).

2. **Reproducible build.** The workflow file itself (also included in the source artifact) pins every version, flag, and toolchain. Anyone can re-run it on GitHub Actions — or adapt the commands for a local build — to reproduce the binaries from source.

If you distribute binaries produced by this workflow, you must do **at least one** of the following (per LGPL-2.1 §6):

- **(a)** Include the complete corresponding source code alongside the binaries (the `source-code-lgpl` artifact satisfies this), or
- **(b)** Include a written offer, valid for at least three years, to give any third party the source code for the LGPL portions, for a charge no more than the cost of distribution, or
- **(c)** If you received the binaries with such an offer, pass that same offer along.

### Relinking with modified libraries (LGPL §4)

The LGPL grants users the right to modify the LGPL-licensed libraries and use their modified versions with your application. Because every LGPL library in this build is a **shared library** (`.so` on Linux/Android, `.dll` on Windows), users can exercise this right by simply replacing the shared library file with their own modified build. No relinking step is required — the dynamic linker will load the replacement at runtime.

For this to work, distributors must:

- Ship the LGPL libraries as **separate shared library files** (not statically linked into the application binary). This workflow already ensures this.
- Provide the **mpv C header files** (`include/mpv/client.h`, etc.) so that users can build compatible replacements. These headers are included in every output artifact.
- Not employ any **technical measures** (code signing enforcement, integrity checks, secure boot restrictions) that would prevent the user from running the application with a modified LGPL library, unless the user is also given the means to re-sign or re-authorize the modified library.

### Reverse engineering (LGPL §4(a))

You may not prohibit reverse engineering of the portions of your application that use the LGPL libraries, to the extent that such reverse engineering is needed to debug modifications to those libraries.

### License texts

The full text of the following licenses applies to components in this distribution. Distributors must include these license texts (or a pointer to them) with the binary distribution:

- **GNU Lesser General Public License, version 2.1 or later** — https://www.gnu.org/licenses/old-licenses/lgpl-2.1.html (FFmpeg, mpv, libplacebo, FriBidi, GnuTLS)
- **GNU General Public License, version 2 or later** — https://www.gnu.org/licenses/old-licenses/gpl-2.0.html (referenced by LGPL-2.1 §3)
- **ISC License** — contained in the libass source tree (libass)
- **FreeType License (FTL)** — https://freetype.org/license.html (FreeType)
- **MIT License** — contained in the HarfBuzz source tree (HarfBuzz)
- **Apache License 2.0** — https://www.apache.org/licenses/LICENSE-2.0 (shaderc, Vulkan Loader — Windows only)

### Prominent notice (LGPL §4(a))

Any application that incorporates these libraries should display, in its documentation or About screen, a notice similar to:

> This software uses libraries from the FFmpeg project, mpv, libplacebo, libass, and GnuTLS, licensed under the LGPL-2.1-or-later. Source code is available at [your URL here].

### Recommended compliance checklist for distributors

When shipping binaries produced by this workflow, verify the following:

- [ ] The `source-code-lgpl` artifact for the exact workflow run is retained and made available (download link, hosted archive, or physical medium).
- [ ] All LGPL libraries are distributed as separate shared library files, not statically linked.
- [ ] The mpv header files are included or otherwise available.
- [ ] The LGPL-2.1, GPL-2.0, and other applicable license texts are included in the distribution (e.g., in a `LICENSES/` directory or an About dialog).
- [ ] A prominent notice identifies the use of LGPL libraries and points to where the source code can be obtained.
- [ ] No technical protection measures prevent users from replacing LGPL shared libraries with their own modified versions.

---

## Rebuilding from source

### Prerequisites

The workflow is designed to run on GitHub Actions runners, but the build steps can be executed locally with the following tools.

**Linux (Ubuntu 22.04 or later)**:

```bash
sudo apt-get install build-essential git ninja-build pkg-config nasm python3-venv \
  libdrm-dev libegl1-mesa-dev libfontconfig1-dev libfreetype-dev \
  libfribidi-dev libgl1-mesa-dev libharfbuzz-dev libpulse-dev \
  libgnutls28-dev libva-dev libvdpau-dev \
  libx11-dev libxext-dev libxinerama-dev libxrandr-dev libxv-dev
pip install meson==1.4.2
```

**Windows (MSYS2 MINGW64)**:

```bash
pacman -S base-devel git mingw-w64-x86_64-{gcc,meson,nasm,ninja,pkgconf,python,\
  freetype,fribidi,harfbuzz,shaderc,vulkan-headers,vulkan-loader}
```

**Android**: Requires the Android NDK r26d and a Linux host with cmake, ninja, meson, and vcpkg.

### Build order

All platforms follow the same dependency chain:

```
FFmpeg ─────────────────┐
libass ─────────────────┤
libplacebo ─────────────┤
                        ▼
                      libmpv
```

On Android, libass and GnuTLS (plus their transitive dependencies) are first built via vcpkg, then the rest follows the same order.

### Modifying a single library

Because all libraries are shared, you can rebuild just one and drop it in. For example, to test a patched FFmpeg on Linux:

```bash
git clone --branch n6.1.1 https://git.ffmpeg.org/ffmpeg.git && cd ffmpeg
# Apply your patch
./configure --prefix=/your/prefix --disable-gpl --disable-version3 \
  --enable-shared --disable-static --enable-gnutls --enable-libdrm
make -j$(nproc) && make install
# Replace the .so files in your Emby installation
cp /your/prefix/lib/libav*.so* /path/to/emby/libs/
```

The same approach works for any other LGPL library in the stack.

---

## Notes

**Why OpenGL on Linux instead of Vulkan?** The Linux job runs on `ubuntu-22.04` for glibc 2.35 compatibility (wide distro support). Ubuntu 22.04 does not package `libshaderc-dev`, and libplacebo's Vulkan backend requires a SPIR-V compiler (shaderc or glslang) at build time. Using OpenGL avoids this dependency while keeping mpv's `gpu` and `gpu-next` video output fully functional via the OpenGL path. VAAPI and VDPAU hardware decoding remain available.

**Why OpenGL ES on Android instead of Vulkan?** Cross-compiling shaderc for Android is complex and fragile. The OpenGL ES rendering path through libplacebo is well-tested on Android and is the path used by most mpv-based Android players. MediaCodec hardware decoding is fully supported.

**libplacebo Python 3.14 patch.** The `prepare-sources` job applies a one-line fix to `src/vulkan/utils_gen.py` so that code generation works on hosts with Python 3.14+, where `xml.etree.ElementTree.parse()` returns an `ElementTree` object rather than an `Element`.

**GnuTLS CRAU patch (Android).** The vcpkg portfile for GnuTLS is patched at build time to fix a `CRAU_MAYBE_UNUSED` macro that is left undefined when building with Android clang, which supports `__has_c_attribute` but not `__maybe_unused__`.
