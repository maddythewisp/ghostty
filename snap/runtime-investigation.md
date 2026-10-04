# Snap graphics runtime investigation

Investigation date: 2026-10-03. Repository baseline: `f96c9711b`.
This is a prototype and a migration plan, not a hardware-validated release fix.

## Recommendation

Keep `confinement: classic`. Retain an internally consistent, base-matched GTK
runtime. Prefer Canonical-owned GPU integration when it becomes usable with
classic confinement. The currently available `gpu` extension is **not** such an
integration: adding `extensions: [gpu]` to this Snap is unsupported.

For the immediate experiment, keep core24 and separate unpatched GLVND dispatch
libraries from the patched application/GTK/Mesa libraries. This restores the
standard host driver lookup at the vendor boundary without discovering NVIDIA
library names, creating symlinks, or exposing the entire host library directory
through `LD_LIBRARY_PATH`. It is an interim packaging policy, not a replacement
for a shared graphics runtime. Test it on Linux before shipping it.

Core26 is available and worth evaluating separately for newer GTK and a longer
base lifetime. It does not by itself fix the classic GPU-runtime boundary.

## Evidence and root cause

There are two independent lookup steps: GLVND must discover an ICD JSON, then the
dynamic loader must load the vendor library named by that JSON.

- [#10760](https://github.com/ghostty-org/ghostty/issues/10760) identified that
  Ghostty overrode EGL's vendor directories and hid the host ICDs.
  [#11209](https://github.com/ghostty-org/ghostty/pull/11209) restored those
  directories; that fix is already present in this checkout.
- [#14155](https://github.com/ghostty-org/ghostty/discussions/14155) supplies
  evidence for a second failure: the bundled GLVND loader finds the NVIDIA JSON,
  but cannot load its library. Its `DF_1_NODEFLIB` flag prevents lookup in default
  host library directories, including cache entries resolving into those
  directories. The report's driver-only symlink directory bypassed that lookup
  restriction and restored acceleration on its Ubuntu 22.04/NVIDIA 560 system.
- [#12713](https://github.com/ghostty-org/ghostty/discussions/12713) changed GTK,
  libadwaita, and graphics resolution together. Its before/after performance and
  EGL logs support investigating driver selection; they do not isolate an older
  GTK performance defect. OpenGL 4.5 versus 4.6 alone does not identify a renderer.

[Snapcraft's ELF patcher](https://github.com/canonical/snapcraft/blob/9.1.3/snapcraft/elf/_patcher.py)
sets `--no-default-lib` when system-library lookup is disabled; it also constructs
RPATH and patches the interpreter. This behavior is packaging infrastructure,
not a flag added by Ghostty's `-Dsnap` or `-Dpatch-rpath`.
[glibc's loader](https://github.com/bminor/glibc/blob/release/2.39/master/elf/dl-load.c)
checks the requesting object's `DF_1_NODEFLIB` flag. It does not reject every
cache entry: it rejects entries in its default system directories, and skips
the final default-directory search. Explicit paths and `LD_LIBRARY_PATH` are
different lookup mechanisms. Consequently the flags on `libEGL.so.1` matter,
not just those on `bin/ghostty`.

The report's successful load under the core24 loader is evidence of compatibility
on that particular machine. A newer host driver can still require a newer libc
or another dependency absent from the base. This prototype does not remove that
ABI requirement.

This checkout's renderer also differs from the reported released binaries:
`src/renderer/opengl/Device.zig` requests `EGL_PLATFORM_SURFACELESS_MESA`, and its
existing diagnostics distinguish missing drivers from missing surfaceless
support. Loading NVIDIA successfully is therefore necessary but not sufficient:
the selected driver must support the platform and the EGL/DMA-BUF presentation
path used by this revision. Validate this independently of GTK's display
context, especially with older proprietary drivers and hybrid-GPU machines.

## Current packaging audit

`snap/snapcraft.yaml` builds amd64 and arm64, downloads Zig 0.16.0, builds Ghostty
with `-Dsnap`, and stages the following runtime groups:

| Group | Current purpose and ownership |
| --- | --- |
| GTK4, libadwaita, GLib and their dependencies | Consistent desktop UI and object/runtime libraries; generic infrastructure, not Ghostty-specific GPU logic. |
| `ibus-gtk4`, GTK GStreamer media backend | Input-method and generic GTK media integration. They are not NVIDIA-driver solutions. |
| librsvg, gdk-pixbuf and loaders | SVG/icons/image decoding and generic GTK resources. |
| libtinfo, libedit, libunistring | General runtime dependencies; distinct from the graphics driver boundary. |
| libpciaccess, libelf, libsensors, LLVM, per-vendor libdrm | Supporting graphics dependencies. Explicit versions such as `libllvm17` add base-upgrade maintenance. |
| `mesa` part | Mesa EGL/GLX, GLVND EGL dispatch, GLU, DRI/Gallium dependencies; shared objects are patched for classic execution. |
| `mesa-gl1-dri` part | Supplies DRI and Gallium without ELF patching, to preserve driver build IDs. |
| `i965-va-driver` on amd64 | Intel video decode; separate from Ghostty's OpenGL terminal renderer. |
| Locally built gtk4-layer-shell | Wayland quick-terminal/window protocol support. It must link before wayland-client and remain dynamically linked. |

Stage packages pull dependencies as well as the explicitly named packages. Removing
only the `mesa` parts would therefore not guarantee a graphics-free payload: GTK
and other packages can pull GL/EGL/Vulkan dependencies back into the Snap.
Some existing DRI comments call the provider `mesa-unpatched`; its actual part
name is `mesa-gl1-dri`.

The launcher manually configures:

| Setting | Effect | Ownership in a compatible official GPU-runtime design |
| --- | --- | --- |
| `LD_LIBRARY_PATH` | Prefers bundled libraries, includes VDPAU, preserves inherited paths. | GPU wrapper owns GPU paths; application runtime still needs its own dependency policy. |
| `LIBGL_DRIVERS_PATH` | Adds bundled Mesa DRI search directory. | GPU provider. |
| `LIBVA_DRIVERS_PATH` | Adds bundled video-acceleration drivers. | GPU provider; usually unnecessary for the terminal renderer itself. |
| `__EGL_VENDOR_LIBRARY_DIRS` | Replaces EGL JSON defaults with explicit host and bundled directories. | GPU provider, including supported host-driver integration. |
| `__EGL_EXTERNAL_PLATFORM_CONFIG_DIRS` | Previously hid host EGL platform registrations. Prototype restores host directories alongside bundled registrations. | GPU provider. |
| `DRIRC_CONFIGDIR` | Selects Mesa configuration data. | GPU provider. |
| `VK_LAYER_PATH` | Adds Vulkan layer directories. | GPU provider if needed; Ghostty's terminal renderer uses OpenGL. |
| `XLOCALEDIR` | Selects bundled X11 locale data. | Generic desktop/GPU runtime. |
| `XDG_DATA_DIRS` | Adds bundled desktop data. | Generic desktop runtime plus application resources. |
| `GTK_PATH`, `GIO_MODULE_DIR`, unset `GIO_EXTRA_MODULES` | Keep GTK/GIO modules matched to bundled libraries. | GTK runtime; cannot simply be removed while retaining bundled GTK. |
| `GDK_PIXBUF_MODULEDIR`, `GDK_PIXBUF_MODULE_FILE` | Select matched image loaders and a writable cache. | GTK runtime. |
| `HOME`, XDG home defaults | Restore normal host user directories for a classic terminal. | Application launch policy. |
| `TERMINFO_DIRS`, `GHOSTTY_RESOURCES_DIR`, application `PATH` | Find Ghostty's terminal data, resources, and command. | Ghostty-specific. |

The existing PRIME/VA-API conditional has no useful effect: `LIBVA_DRIVERS_PATH`
is already exported above it. It does not implement PRIME offload. Keep user
offload selectors intact and validate the actual selected renderer.

## Current Snapcraft ecosystem and the classic constraint

The inspected release is Snapcraft **9.1.3**, not an installed local executable.
[Snapcraft 9.0's release notes](https://ubuntu.com/docs/snapcraft/latest/release-notes/snapcraft-9-0/)
announce stable core26 support. The
[GPU extension source](https://github.com/canonical/snapcraft/blob/9.1.3/snapcraft/extensions/gpu_extension.py)
supports core22/core24/core26 but allows **only strict and devmode** confinement.
For core26 it adds `gpu-2604`, default provider `mesa-2604`, a command-chain
wrapper, and provider cleanup. Core24 uses `gpu-2404`/`mesa-2404` and also adds
graphics layouts. These are real supported architectures for suitable confined
applications, but not drop-in architectures for this classic terminal.

This is not merely a validator restriction to bypass.
[snap-confine](https://github.com/snapcore/snapd/blob/master/cmd/snap-confine/snap-confine.c)
takes a separate classic execution path rather than constructing the confined
application's filesystem. Canonical's
[explanation of classic content mounting](https://forum.snapcraft.io/t/classic-confinement-mounting-internal-snap-folder/34120/2)
states that layouts and content mounts cannot be used in the host mount namespace.
Manually copying the extension's YAML does not make its content mounts work.
Hardcoding `/snap/mesa-2604/current` is also unsuitable: it bypasses dependency,
connection, refresh and rollback semantics and is not equivalent to using the
official content interface.

The
[GNOME extension source](https://github.com/canonical/snapcraft/blob/9.1.3/snapcraft/extensions/gnome.py)
supports core22 and core24, with `gnome-42-2204` and `gnome-46-2404` runtimes/SDKs;
it allows only strict/devmode. There is no core26 GNOME extension in this release.
Even a future core26 GNOME content snap would need an appropriate classic
integration mechanism. Continue bundling a coherent GTK/libadwaita runtime for
the classic Snap; loading arbitrary host GTK is not a supported replacement.

The
[mesa-2604 provider](https://github.com/canonical/mesa-2604/blob/main/snap/snapcraft.yaml)
advertises Mesa hardware support and distribution-installed NVIDIA integration
through snapd. Its
[wrapper](https://github.com/canonical/mesa-2604/blob/main/scripts/bin/gpu-2604-provider-wrapper.in)
handles Mesa paths, host-driver exports under `/var/lib/snapd/lib`, external
platform configuration, and component integration. This is the appropriate
ownership boundary for a supported GPU content consumer. It is not a promise
that arbitrary classic consumers, driver installation methods or PRIME layouts
are supported.

Strict confinement cannot preserve normal terminal semantics here. Access to
arbitrary host executables, arbitrary project/mount paths, developer toolchains,
agents and sockets is fundamental. `home`, `removable-media`, and lists of
`system-files`/`personal-files` paths cannot express this workflow. Devmode is
not a production substitute: it retains the confined filesystem/runtime model.

## Architecture comparison

The following entries are assessments, not results of hardware testing.

| Option | Maintenance, portability and compatibility | NVIDIA / AMD / Intel | X11 / Wayland / PRIME | Classic, upgrades and local code |
| --- | --- | --- | --- | --- |
| A: Current runtime + NVIDIA symlink mirror | Immediate workaround with one-machine evidence; requires cache invalidation, architecture handling, dependency closure and distro-specific discovery. Retains a coherent GTK base but can still encounter driver ABI mismatch. | Report demonstrates NVIDIA 560 on Ubuntu 22.04; AMD/Intel retain bundled Mesa support and its update limitations. | X11 demonstrated in report. Wayland needs external platform libraries/JSONs too. PRIME needs multiple devices/vendor libraries and preserved selectors. | Works within classic host access. High shell maintenance; each driver/base upgrade can expand discovery logic. Reject as long-term design. |
| B: core24 + gpu-2404/mesa-2404 | Canonical owns Mesa, dispatch, wrappers and graphics cleanup; avoids local driver packaging for a supported consumer. Mature base minimizes GTK change. | Provider integration aims to cover Mesa AMD/Intel and supported NVIDIA host drivers; not every installation method. | GPU runtime supports both display protocols; PRIME remains an end-to-end test obligation. | Official extension excludes classic; content/layout model is incompatible. Excellent for strict applications, infeasible as proposed for Ghostty. Future upgrades change provider generation with the base. Low local GPU code if usable. |
| C: core26 + gpu-2604/mesa-2604 | Same ownership advantage, newer Mesa/base; adds GTK and dependency migration risk. | Newer Mesa benefits newer AMD/Intel hardware; NVIDIA support remains bounded by snapd/provider integration and driver ABI. | Both protocols and modern platform integration are covered by provider infrastructure; offload requires verification. | Also excludes classic. No core26 GNOME extension in 9.1.3. Cannot ship this architecture by changing two YAML fields. Low local GPU code only after an upstream classic mechanism exists. |
| D: Arbitrary host GTK/GL | Very little initial shell code, but dependence on host GTK/libadwaita versions, plugins, libc and distribution packaging defeats consistent runtime support. Older hosts may not satisfy Ghostty's minimum API. | Uses each host's complete driver stack when compatible. Helps one host without establishing portable compatibility. | Works only where the host's desktop stack meets requirements; PRIME behavior follows host setup. Mixing bundled and host libraries remains fragile. | Classic makes paths accessible, not ABI-compatible. Every distro release becomes part of Ghostty's runtime matrix; reject for the general Snap. Native distro packages are a coherent way to use a host stack. |
| E: Classic + coherent bundled GTK + unpatched GLVND boundary (prototype) | Small, generic loader-policy change; avoids driver-name discovery. Keeps existing Mesa maintenance and host-driver ABI risks. Broader driver fallback can expose missing packaged dependencies. | Allows host NVIDIA ICD library lookup without a private mirror; AMD/Intel keep current Mesa. All are unverified locally. | Unpatched EGL/GLX dispatch and host platform JSON directories address both paths. PRIME selectors remain user-controlled and must be tested. | Compatible with classic's filesystem model. Retains current base/RPATH infrastructure. Five dispatch SONAME patterns must be kept consistent across parts. Best immediate experiment, not the fully delegated end-state. |

The preferred long-term ownership model is **classic terminal semantics +
base-matched GTK + Canonical-maintained graphics integration with explicit classic
support**. No currently verified extension meets all three requirements. A
classic-compatible provider contract or a supported selective ELF-patching policy
must precede that migration. This is a capability gap, not a reason to silently
switch confinement. No upstream issue or PR was created during this work.

## Core26 migration findings

The Ubuntu Resolute release package indexes were downloaded from the official
[main archive](https://archive.ubuntu.com/ubuntu/dists/resolute/main/binary-amd64/Packages.xz)
and [universe archive](https://archive.ubuntu.com/ubuntu/dists/resolute/universe/binary-amd64/Packages.xz)
and checked against every explicit build/stage package in this recipe. These are
release-pocket versions; updates/security can provide newer versions. This was
an amd64 metadata check, not an apt dependency-resolution or arm64 build test.

| Item | Verified result / required migration work |
| --- | --- |
| Snapcraft | Use 9.x for supported core26 builds; inspect the exact extension release rather than assuming GNOME support. |
| GTK | Resolute release has `libgtk-4-1`/`libgtk-4-dev` 4.22.2; verify Ghostty's generated bindings, UI and runtime feature gates against these headers. |
| libadwaita | `libadwaita-1-0`/`-dev` 1.9.0. Bundle matching build/runtime versions. |
| GLib | `libglib2.0-0t64` 2.88.0. Check module and schema integration, not just library linkage. |
| libc | GTK depends on libc >= 2.43. Preserve the core26 interpreter/runtime; do not let an older host libc shadow it. |
| GTK media package | `libgtk-4-media-gstreamer` is absent. GTK's GStreamer dependencies are integrated into `libgtk-4-1`; remove the old package entry. |
| LLVM | `libllvm17` is absent. Remove the manual pin and let the staged Mesa packages select their LLVM dependency, rather than hardcoding a replacement. |
| gtk4-layer-shell | `libgtk4-layer-shell-dev` and `libgtk4-layer-shell0` 1.3.0 are available in universe. Add the dev/runtime packages and remove `-fno-sys=gtk4-layer-shell` if adopting the system library. Preserve linkage ordering. |
| Other package names | All other explicit current amd64 build/stage package names resolve in these indexes, including i965, libdrm variants, Mesa Gallium, libunistring5, libelf1t64 and gdk-pixbuf. Their continued necessity needs payload/dependency inspection. |
| Zig | Recipe and repository both use 0.16.0. No base-specific Zig blocker identified; retain the upstream compiler download and validate both architecture builds. A newer base does not justify an unrelated compiler upgrade. |
| RPATH | If retaining the explicit path, change `/snap/core24/current` to core26. Better, verify that Snapcraft's prime-time interpreter/RPATH patching supplies all paths and then remove the redundant Zig option. Do not remove both isolation mechanisms without inspecting final ELFs. |
| GPU and GNOME | GPU extension supports the base but rejects classic; GNOME extension supports neither core26 nor classic. Neither blocker is solved by package-name changes. |

An independent core26 branch is feasible with bundled GTK/Mesa and the above
recipe changes. Recommend validating it after the loader experiment so that
GTK, Mesa and libc changes do not obscure the cause of performance recovery.
Do not promote a core26 GPU-extension migration now under the classic requirement.

## Prototype patch and remaining work

Changed `snap/snapcraft.yaml`:

- Add a `glvnd` part staging only the standard EGL/GL/GLX/GLdispatch/OpenGL
  dispatch libraries with `no-patchelf`.
- Exclude those same libraries from `libs` and `mesa`, so another part cannot
  reapply NODEFLIB. Leave Mesa vendor libraries, DRI and Gallium ownership intact.
- Keep core24, classic, GTK and the executable's isolation/RPATH setup.

Changed `snap/local/launcher`:

- Replace Bash's `source` with POSIX `.` when reading `.last_revision`. This is an
  independent correctness fix; otherwise dash repeatedly regenerates the pixbuf
  cache.
- Include standard host EGL external-platform configuration directories. This
  parallels the existing vendor-JSON fix and does not name a proprietary driver.

Changed `src/renderer/OpenGL.zig`:

- Log GL vendor and renderer after loading GL functions with a current context.
- Warn for `llvmpipe`, `softpipe`, or `Software Rasterizer`; tolerate null strings.
  The warning diagnoses a performance problem, not a fatal condition, and permits
  intentional software rendering.
- Logging happens during context preparation, not in the render loop. It may
  recur when a surface's rendering thread/context is reinitialized.
  EGL vendor/extensions are already logged by `src/renderer/opengl/Device.zig`.

`-Dsnap` remains necessary for host-child environment cleanup:
`src/apprt/gtk/class/surface.zig` filters Snap paths and variables out of shell
environments; `src/os/open.zig` removes bundled `LD_LIBRARY_PATH` for `xdg-open`.
It does not select drivers. Future provider paths must be added to/covered by
that cleanup before adopting any external runtime. Child shells must use host
libraries and retain user PRIME settings.

`-Dpatch-rpath` calls `patchelf --set-rpath` in `src/build/Config.zig`; it does not
set NODEFLIB. Snapcraft separately patches prime-time RPATH/interpreter and can
include dynamically built layer-shell and base dependencies. The explicit Zig
path appears redundant for the final package, but removing it is deferred until
a Linux build proves the resulting ELF paths. The general build option must
remain for Nix users.

The existing local Mesa/DRI/GTK wiring is retained because the compatible shared
runtime is blocked. If a supported classic GPU mechanism becomes available,
remove the Mesa parts, explicit GPU dependencies and GPU environment settings
together, use provider cleanup to remove transitive duplicates, and keep only
application/GTK launch policy. Do not claim this prototype achieves that end-state.

The immediate Linux experiment must establish: no NODEFLIB on GLVND, one coherent
base/application libc, successful host vendor loading, correct desktop/context
selection, and no fallback on the hardware matrix below. It can also reveal
missing base dependencies that were previously masked by ELF patching. If that
occurs, investigate selective RPATH patching with default lookup preserved rather
than adding host library directories globally or making driver-name mirrors.

## Validation performed locally

The work environment is macOS arm64 with no snapd, Snapcraft, Linux GPU, Linux
container runtime or VM runner installed. No Snap build or GPU validation was
possible. A native macOS build would not exercise the modified Linux renderer.

Passed:

- `sh -n snap/local/launcher` and `git diff --check`.
- YAML parse with Ruby Psych; staging ownership checks for all five dispatch
  SONAMEs on both amd64 and arm64 paths.
- Inspection of the five real, original Ubuntu Noble amd64 GLVND shared objects
  downloaded from the official archive: none has `DF_1_NODEFLIB`. This verifies
  the unpatched input policy, not the final Snapcraft prime output or arm64 ELFs.
- Isolated launcher executions for fresh install, same revision, changed
  revision, and missing pixbuf cache on both architecture settings. The cache
  is reused for an unchanged revision; home/inherited paths and host EGL
  directories are preserved. These ran under macOS `/bin/sh`, not Ubuntu dash.
- Zig 0.16.0 `fmt --check` on the modified renderer.
- Compilation/execution of the actual new diagnostic block in a temporary Zig
  harness with mocked GL strings: NVIDIA, AMD and Intel do not warn; the three
  software strings do; null vendor/renderer strings are safe. This does not
  substitute for compiling the full GTK/OpenGL application.

## Linux build and renderer validation

Use a Linux machine with Snapcraft and its normal build provider available.
Build both architectures on suitable builders. The existing CI Snap job only
runs `zig build -Dsnap` in Nix; it does not build/inspect the actual Snap payload.

```sh
snapcraft --version
snapcraft pack --verbose
# Install the exact output filename printed by Snapcraft:
sudo snap install --dangerous --classic ./ghostty_<version>_<arch>.snap
snap list ghostty core24
```

Inspect the installed payload. The dispatch libraries should have no NODEFLIB;
the executable/GTK may still have it. Also inspect the interpreter and RPATH;
absence of NODEFLIB alone is not evidence of acceleration.

```sh
snap run --shell ghostty
case "$SNAP_ARCH" in
  amd64) triplet=x86_64-linux-gnu ;;
  arm64) triplet=aarch64-linux-gnu ;;
  *) exit 1 ;;
esac
readelf -l "$SNAP/bin/ghostty" | grep 'interpreter'
readelf -d "$SNAP/bin/ghostty" | grep -E 'RPATH|RUNPATH|FLAGS_1'
for name in libEGL.so.1 libGL.so.1 libGLX.so.0 libGLdispatch.so.0 libOpenGL.so.0; do
  printf '\n%s\n' "$name"
  readelf -d "$SNAP/usr/lib/$triplet/$name" | grep -E 'RPATH|RUNPATH|FLAGS_1' || true
done
exit
```

Launch from a host terminal with a separate application instance. `GHOSTTY_LOG`
uses Ghostty's logging flags, not a log-level string such as `debug`.

```sh
GHOSTTY_LOG=stderr snap run ghostty --gtk-single-instance=false \
  2>/tmp/ghostty-snap-renderer.log &
ghostty_pid=$!
readlink -f "/proc/$ghostty_pid/exe"
grep -E 'EGL vendor=|GL vendor=|renderer=|software OpenGL' /tmp/ghostty-snap-renderer.log
cat /proc/"$ghostty_pid"/task/*/comm | sort | uniq -c
grep -iE 'nvidia|mesa|egl|glx|gallium|libc\.so|ld-linux' /proc/"$ghostty_pid"/maps
```

Verify that `/proc/<pid>/exe` is the installed Snap Ghostty binary before
interpreting these results. If the launcher hands off to another PID, find it
with `pgrep -af '/snap/ghostty/.*/bin/ghostty'` and assign that single PID to
`ghostty_pid`. Wait until a window renders before examining logs/maps. On
NVIDIA, expect an NVIDIA vendor and hardware renderer plus mapped proprietary
EGL/core libraries. On AMD/Intel, Mesa vendor libraries are normal; check the
renderer for the real device. A mapped Mesa library or an OpenGL version string
alone cannot distinguish hardware rendering from llvmpipe. No llvmpipe threads
is supporting evidence, not a complete proof.

When loading fails, repeat in a fresh independent instance:

```sh
GHOSTTY_LOG=stderr LD_DEBUG=libs,files snap run ghostty \
  --gtk-single-instance=false 2>/tmp/ghostty-snap-loader.log
grep -iE 'libEGL|libGLX|nvidia|not found|version lookup|error' \
  /tmp/ghostty-snap-loader.log
```

Inspect lookup attempts and failures, not just successful `ldconfig -p` output.
Host `glxinfo -B`, `eglinfo` or `nvidia-smi` establish a baseline but do not prove
that the Snap selected the same driver. For a deliberate negative-control test,
start a separate instance with `LIBGL_ALWAYS_SOFTWARE=1` and verify the new warning
when Mesa actually selects a software renderer.

| Hardware/session | Launch from host terminal | Pass criteria |
| --- | --- | --- |
| NVIDIA proprietary + X11 | `GDK_BACKEND=x11 GHOSTTY_LOG=stderr snap run ghostty --gtk-single-instance=false` | NVIDIA GL hardware renderer, matching driver libraries mapped, no llvmpipe/softpipe, responsive selection/scrolling/window creation. |
| NVIDIA proprietary + Wayland | `GDK_BACKEND=wayland GHOSTTY_LOG=stderr snap run ghostty --gtk-single-instance=false` | Same renderer criteria, plus correct EGL Wayland/GBM platform loading, fractional scaling and no context/presentation errors. |
| AMD + Wayland | Same Wayland command | Actual Radeon renderer, working bundled Mesa/DRI, no software warning; check quick terminal on a compositor supporting layer-shell. |
| Intel + Wayland | Same Wayland command | Actual Intel renderer, working Mesa/DRI, no software warning; test both older and newer supported hardware. |
| PRIME/offload + X11 | `__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia GDK_BACKEND=x11 GHOSTTY_LOG=stderr snap run ghostty --gtk-single-instance=false` | Selected NVIDIA renderer versus the default integrated-GPU baseline; validate GTK and Ghostty's own surfaceless EGL context separately. |
| PRIME/offload + Wayland | `__NV_PRIME_RENDER_OFFLOAD=1 GDK_BACKEND=wayland GHOSTTY_LOG=stderr snap run ghostty --gtk-single-instance=false` | Requested hardware renderer if supported by the driver/compositor; selectors alone are not proof, and the surfaceless EGL path may select differently. |

Run the NVIDIA rows on Ubuntu 22.04, 24.04 and 26.04 where drivers are supported;
include distro-packaged drivers and a newer driver requiring newer userspace.
Add amd64/arm64 builds, a non-Ubuntu snapd host, Snap refresh/rollback and driver
upgrade/reboot checks. Compare installed GTK build/runtime versions and mapped
libc paths. Repeat baseline versus prototype on the same host without changing
GTK or driver versions.

In a shell **inside Ghostty**, check ordinary terminal behavior and runtime leaks:

```sh
env | grep -E '^(SNAP|LD_LIBRARY_PATH|GTK_PATH|GIO_MODULE_DIR|GDK_PIXBUF|LIBGL|LIBVA|__EGL|VK_LAYER_PATH|__NV_PRIME|__GLX_VENDOR)'
command -v sh git ssh docker
readlink -f /proc/$$/exe
ls -ld ~/.ssh ~/.config /usr/local/bin
```

Check user-selected project mounts, developer tools, SSH agent and Docker socket
access where configured, `xdg-open`, and launching another Snap from the shell.
User-provided library/offload settings may legitimately remain, but Ghostty's
bundled paths must not force the host child to load the Snap runtime.
