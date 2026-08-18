---
name: grabit-dev
description: >-
  Local development brain for the grabit repo — a C17 screenshot/screen-recording/OCR/uploader tool for wlroots-family Wayland compositors (hyprland, sway, niri, river, KDE, GNOME), built with a plain Makefile. Use this whenever working in this repository — building, fixing bugs, adding features, writing or debugging grabit plugins (manifest.toml, the grabit-plugin.h helper header), or navigating its src/ tree. Trigger even without an explicit request whenever the task touches this repo, such as mentions of grabit, files like main.c, capture.c, or region/annotate.c, types like struct grabit_wl_state or struct args, make/make fmt/make test in this context, or the plugin CLI (grabit plugin install/update/show).
---

# grabit

Screenshot, screen-recording, OCR, and uploader tool for wlroots-family
Wayland compositors. C17, AGPL-3.0-or-later. Primary repo:
heliopolis.live/creations/grabit — github.com/Creationsss/grabit is a
mirror.

## orient yourself, in this order

1. `references/architecture.md` — src/ subsystem map + control-flow
   walkthrough. Read this before your first change in an unfamiliar area.
2. `references/build-and-test.md` — Arch build deps, every `make` target,
   the pre-change checklist.
3. `references/conventions.md` — naming/style/error-handling conventions,
   plus recipes for common extension points (new upload service, new config
   key, new CLI flag, new annotation tool, new capture backend, new
   compositor integration).
4. `references/plugins.md` — the local plugin dev loop (installing from a
   local path, iterating, testing dispatch). `PLUGINS.md` in the repo root
   is the full spec this supplements.
5. Root docs, as needed rather than up front: `README.md` (feature/install
   overview), `OPTIONS.md` (full config schema + CLI reference, 600+
   lines — grep it for the key you need rather than reading it whole).

## quick facts

- C17, no C++, plain Makefile (no cmake/meson). ~185 `.c` files under
  `src/`, one directory per subsystem.
- Every `.c`/`.h` needs a 2-line SPDX header — enforced by `make test`,
  auto-fixed by `make apply-headers`. See conventions.md for the exact text.
- Tabs, LLVM-based clang-format, `ColumnLimit: 0` — run `make fmt`, don't
  hand-format.
- Wayland-only. No X11. Exercising most subsystems end-to-end needs a real
  Wayland session (hyprland/sway/niri/river/KDE/GNOME).

## fast build

```sh
sudo pacman -S --needed base-devel pkgconf wayland cairo libxkbcommon file curl json-c dbus
sudo pacman -S --needed pipewire libjpeg-turbo libwebp   # optional: --record, jpeg/webp output
sudo pacman -S --needed clang bear                        # dev tooling: fmt, clangd, compile_commands.json
make && ./build/grabit --version
```

Full target list, dependency rationale, and CI parity notes:
`references/build-and-test.md`.

## before calling a change done

1. `make fmt`
2. `make test` (SPDX headers — `make apply-headers` fixes it)
3. `make all` — must build with no new warnings (the `Makefile`'s `WARN`
   flags are broad even without `-Werror`)
4. `bear -- make all && make clangd-check` — if headers/declarations changed
5. `make sanitize` — if you touched memory/buffers, Wayland listener
   callbacks, subprocess/pipe handling, or `record/`/`region/`/`capture/`/
   `plugin/`

Details and exact commands: `references/build-and-test.md`.

## gotchas

- `build/` and `build-san/` are gitignored/regenerable; never hand-edit
  `build/protocols/*-protocol.c` (wayland-scanner output, deleted after
  linking).
- CI (`.github/workflows/`) builds/lints in a Debian trixie-slim container —
  the pacman packages above are for local dev parity, not literally what CI
  runs.
- `src/vendor/` (tomlc99, sha256) is exempt from the SPDX check and built
  with relaxed warnings — don't reformat or restyle it.
- Feature support (`--record`'s pipewire path, jpeg/webp output) is
  compile-time via `pkg-config --exists`, not runtime — `./build/grabit
  --version` reports what's actually compiled in.
