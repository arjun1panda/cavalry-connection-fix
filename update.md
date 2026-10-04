# Cavalry connection fix: update notes (Fedora 44, Cavalry 2.8)

Two findings from getting the patched Wine working on Fedora 44 with Cavalry 2.8. They are independent, so each is written as its own issue.

**Environment (both issues)**
- Fedora 44, KDE Plasma, Wayland session (Wine using X11 via XWayland, started with `env -u WAYLAND_DISPLAY`)
- Hybrid GPU laptop: NVIDIA GTX 1660 Ti + AMD Renoir iGPU
- Cavalry 2.8, Wine 11.13 built with the patch, system Wine 11.0 also installed
- dnf is dnf5 (`dnf5-plugins` was needed for `builddep`)

---

## Issue 1: Fedora build dependencies

**Title:** Fedora 44: `dnf builddep` conflict on ocl-icd-devel, and extra packages needed for a working build

Thanks for the patch! The connection drag now works for me. Getting there on Fedora took a few steps that aren't covered by `install.sh`, so here they are in case they help others.

### Problem 1: `sudo dnf builddep -y wine` aborts

It fails because `ocl-icd-devel` conflicts with the installed `OpenCL-ICD-Loader`:

```
installed package OpenCL-ICD-Loader-3.0.6-7.fc44.x86_64 conflicts with ocl-icd provided by ocl-icd-2.3.4-2.fc44.x86_64
You can try to add to command line: --allowerasing
```

`install.sh` stops there. I didn't want `--allowerasing` removing a system package. `--exclude` and `--skip-broken` aren't accepted by dnf5's `builddep` either ("matches only excluded packages" / "Unknown argument").

### Problem 2: building without builddep silently drops features

I ran `scripts/build-patched-wine.sh` directly, and it built, but `configure` quietly disabled several things. Check `include/config.h`:

```
/* #undef SONAME_LIBXCOMPOSITE */
/* #undef SONAME_LIBXINERAMA */
/* #undef SONAME_LIBXXF86VM */
```

With XComposite missing, the Cavalry canvas, timeline and graph editor were black. The connection drag worked, but GPU-drawn areas didn't. Everything else looked normal.

Also, without OpenCL support, Cavalry exits at start-up:

```
err:module:loader_init "OpenCL.dll" failed to initialize, aborting
```

### What fixed it

```
sudo dnf install dnf5-plugins opencl-headers OpenCL-ICD-Loader-devel \
  libXcomposite-devel libXinerama-devel libXxf86vm-devel libpcap-devel libusb1-devel mingw64-gcc-c++
```

Then re-run `configure`, confirm the three `SONAME_` lines above are `#define`d, and rebuild. After that, canvas, timeline, graph editor, connection drag and rendering all work.

### Suggestions

- In the Fedora branch of `install.sh`, install the packages above explicitly, and don't let a `builddep` conflict stop the whole step (or fall back to them).
- After `configure`, have the build script check for those `SONAME_` lines and print a clear warning if XComposite is missing, since the failure looks like a rendering bug, not a build problem.
- Mention that `Cavalry.desktop` needs `Exec=` and `Path=` to use the real prefix path and the patched wine (and `%u`).

### Data point

My working build has the `dlls/win32u/dce.c` hunk reverted. I did that while debugging the black canvas, and it turned out the canvas problem was the missing libraries. I didn't re-test with the hunk applied, so I can't say if it's needed.

---

## Issue 2: Cavalry 2.8 and Qt 6.9.2

**Title:** Cavalry 2.8 uses Qt 6.9.2, so the Qt663 class-name match no longer applies

With Cavalry 2.8 the bundled Qt is 6.9.2 (`Qt6Core.dll`), and its windows are registered as `Qt692QWindowIcon` (also `Qt692QWindowOwnDCIcon`). The patch matches the class name `Qt663QWindowIcon`, so it never detects the connection window and the drag fails exactly as before.

Changing both occurrences in `dlls/win32u/window.c` fixed it for me:

```
sed -i "s/'Q','t','6','6','3'/'Q','t','6','9','2'/" dlls/win32u/window.c
```

`Qt663QWindowIcon` and `Qt692QWindowIcon` are both 16 characters, so the length check (`len < 16` / the `memcmp` of 16 WCHARs) still works.

Idea: match `Qt6` followed by any three digits plus `QWindowIcon`, instead of one hard-coded version, so a Cavalry update that bumps Qt doesn't silently disable the fix again.

Tested on Fedora 44 + Wine 11.13, Cavalry 2.8: drag works, canvas and graph editor render normally.
