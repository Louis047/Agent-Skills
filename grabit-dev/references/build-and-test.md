# build & test (Arch)

CI itself builds/lints inside a Debian `trixie-slim` container (see
`.github/workflows/`), not Arch. The packages below give local dev parity —
same libraries, same `make` targets — they aren't what CI literally runs.

## install deps

```sh
# required to build at all (mirrors Makefile's PKGS_CORE + libmagic + wayland-scanner)
sudo pacman -S --needed base-devel pkgconf wayland cairo libxkbcommon file curl json-c dbus

# optional, feature-gated at build time via `pkg-config --exists` (see Makefile)
sudo pacman -S --needed pipewire       # HAVE_PIPEWIRE: enables --record's portal-less pipewire path
sudo pacman -S --needed libjpeg-turbo  # HAVE_JPEG: jpeg output
sudo pacman -S --needed libwebp        # HAVE_WEBP: webp output

# runtime-only, not linked, just exec'd
sudo pacman -S --needed ffmpeg tesseract   # --record encoding / --tesseract OCR

# dev tooling (matches ci.yml / fmt.yml)
sudo pacman -S --needed clang bear     # clang ships both clang-format and clangd; bear generates compile_commands.json
```

Feature flags are compile-time, not runtime — `./build/grabit --version`
prints what actually got compiled in ("capture backends: ...", "recording
sources: ..."), which is the source of truth if you're unsure whether a lib
was picked up.

## make targets

```sh
make               # release build → build/grabit
make sanitize      # ASan+UBSan build → build-san/grabit (no -D_FORTIFY_SOURCE)
make install       # installs to $(DESTDIR)$(PREFIX)/bin — destructive, don't run casually in a dev loop
make clean         # removes build/ and build-san/

make test          # SPDX-header lint over src/ (excludes vendor/); tools/check_headers.c
make apply-headers # auto-adds the missing SPDX header to files `make test` flagged

make fmt           # clang-format -i over all tracked .c/.h (excludes vendor/)
make fmt-check     # dry-run, non-zero exit on diff — what fmt.yml runs in CI

make clangd-check              # parses every source with `clangd --check`, fails on errors
make clangd-check FILE=path/to/file.c   # scope to one file
```

`clangd-check` needs `compile_commands.json`; it generates one with `bear` if
absent, but **not** if it's merely stale. Regenerate manually after
adding/removing source files:

```sh
bear -- make all
```

## pre-change checklist

Run in this order before considering a change finished:

1. `make fmt` — auto-format (tabs, LLVM base style, see conventions.md)
2. `make test` — SPDX headers on any new `.c`/`.h`; `make apply-headers` fixes it
3. `make all` — must build clean; the `Makefile`'s `WARN` flags
   (`-Wall -Wextra -Wpedantic -Wshadow -Wstrict-prototypes
   -Wmissing-prototypes -Wvla -Wformat=2 -Wnull-dereference
   -Wpointer-arith`) are broad — a new warning is a bug, even though the
   build isn't `-Werror`
4. `bear -- make all && make clangd-check` — if you touched headers,
   declarations, or added/removed files
5. `make sanitize` — if you touched memory/buffer handling, Wayland listener
   callbacks, subprocess/pipe handling, or anything in `record/`, `region/`,
   `capture/`, or `plugin/`. Needs a live Wayland session to actually run
   (`./build-san/grabit ...`), not just to build.

## gotchas

- `build/` and `build-san/` are gitignored and fully regenerable; never
  hand-edit `build/protocols/*-protocol.c` (wayland-scanner output, deleted
  right after linking).
- The version string embeds `git describe` output at build time
  (`build/version.h`) — a shallow clone without tags shows a generic/dirty
  version.
- `src/vendor/` (tomlc99, sha256) is excluded from `make test`'s SPDX check
  and built with relaxed warnings — leave its formatting alone.
- Everything here is Wayland-only (wlroots-family + KDE/GNOME via their own
  protocols) — X11 isn't supported, and exercising capture/region/pin/tray/
  record end-to-end needs a real Wayland session, not just a clean compile.
