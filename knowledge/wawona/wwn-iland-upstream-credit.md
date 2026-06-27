# wwn-iland lineage and upstream credit

## Upstream inspiration

**[CoreBedtime/iland](https://github.com/CoreBedtime/iland)** pioneered replacing
macOS WindowServer/SkyLight with a custom `IOSurface`/Metal-backed framebuffer
that mimics Linux **DRM/KMS/EGL/GBM** — the "Mode A" in-window present path for
Wayland GL clients on macOS.

All credit for the original macOS-only approach goes to bedtime and the
CoreBedtime/iland project.

## Wawona's fork history

1. Wawona initially forked iland as **`github.com/Wawona/iland`** (a GitHub fork
   of CoreBedtime/iland).
2. That fork was **deleted** after extraction into the dedicated flake repo
   **`github.com/Wawona/wwn-iland`** — the current home for all iland-related
   Nix packaging, shims, and Wawona-specific extensions.

Do not reference `Wawona/iland`; use **`wwn-iland`**.

## What wwn-iland adds beyond upstream macOS iland

Upstream iland targeted **macOS only**. `wwn-iland` implements substantial
Wawona-specific work:

- **KMS/DRM on iOS** and across the Apple platform family (iPadOS, tvOS,
  visionOS) — not just macOS.
- **Android** graphics-compat paths where applicable.
- Nix-wrapped, cross-compiled recipes via **`wwn-toolchain`** (`registryFragment`
  for `iland` and `iland-gl-clients`).
- Vendored upstream sources + extended shim tree under
  `dependencies/libs/iland/upstream/shims/` (DRM, EGL, GBM, udev stubs).
- Consumed by **`wwn-weston`** for the apple-mobile compositor DRM/GL path via
  flake input `wwn-iland` and `extraArgs.ilandSrc = wwn-iland`.

## Indexing note (WWN-MCP)

- **`project=iland`, source `wwn-iland`** — Wawona's packaging + shims (edit here).
- **`project=iland`, source `iland` (CoreBedtime/iland)** — upstream reference
  implementation (read-only context; do not edit for Wawona patches).

When answering "how does iland work in Wawona?", prefer **`wwn-iland`** recipes
and knowledge docs over vanilla CoreBedtime/iland behavior — the mobile/Android
paths exist only in `wwn-iland`.
