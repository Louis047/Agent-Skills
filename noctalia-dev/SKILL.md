---
name: noctalia-dev
description: Local development brain for the Noctalia repo (github.com/noctalia-dev/noctalia), a native C++23 Wayland desktop shell built with Meson (no Qt/GTK). Use this skill for ANY work inside a noctalia checkout, even if the user doesn't say "noctalia" explicitly - adding or editing bar/dock/launcher/control-center/notification widgets, writing or debugging Luau plugins, changing the TOML config schema, fixing bugs, touching D-Bus services (network, bluetooth, mpris, tray, upower, polkit...), the GLES render/scene-graph layer, Wayland protocol handling, i18n strings, or running the build/test/lint/format loop. Also use it when the user mentions files under src/, tests/, assets/, meson.build, justfile, or CONTRIBUTING.md/BUILDING.md that match this project's layout. Treat this as the persistent project memory for agentic tools (Claude Code, Antigravity, Opencode, etc.) working on this codebase locally.
---

# Noctalia Development

Noctalia is a native Wayland desktop shell (bars, dock, launcher, notifications, control center, lockscreen,
theming, wallpaper, OSDs, a Luau plugin system) written directly against Wayland + OpenGL ES. No Qt, no GTK.
Build system is Meson, wrapped by a `justfile`. Sources are C++23 (GCC 13+ / Clang 16+).

**Confirm you're in the right repo** before applying anything below: repo root has `justfile`, `meson.build`,
`CONTRIBUTING.md`, and a `src/` tree matching the map in this file. If any of that is missing, this skill doesn't apply.

## The dev loop - run this before calling anything done

This mirrors CI exactly (`.github/workflows/ci.yml` runs format-check, build, test, lint as hard gates). Don't
consider a task finished until it passes locally.

```sh
just configure          # first time only, or after changing build options - creates build-debug/
just build               # compiles the noctalia binary
just test debug --print-errorlogs   # builds + runs the unit test suite (tests/ are NOT part of `just build`)
just lint                 # clang-tidy, warnings-as-errors
just format               # clang-format; also auto-runs via lefthook pre-commit if installed
```

Use `just configure release` / `just build release` / `just test release` for the optimized build. Debug is the
normal loop for local iteration; `just run` builds and launches your local debug binary.

If you only touched non-C++ files (assets, translations, docs), `just build`/`just test` still matter whenever
they read those assets at runtime or are covered by a fixture test - check before skipping.

## Project map

```text
src/
  main.cpp          Entry point
  app/               Application bootstrap, main loop, poll sources (start here to trace startup/shutdown)
  auth/              PAM and fingerprint authentication
  calendar/          CalDAV, Google Calendar, iCalendar parsing, polling
  capture/           Screenshots, screencopy capture, region overlay
  cli/               CLI parsing, help, schema output
  compositors/       Compositor detection, runtime adapters, workspace/output/keyboard backends
  config/            Config schema, validation, hot reload, state store, overrides
    schema/          Typed config schema engine (see references/config-schema.md)
  core/              Logging, timers, process helpers, resource paths, shared utilities
  dbus/              Session/system bus wrappers and service integrations
    accounts/ bluetooth/ idle/ logind/ mpris/ network/ notification/ polkit/ power/ tray/ upower/
  debug/             Debug D-Bus service (dev.noctalia.Debug - see Debugging below)
  hooks/             User hook state and hook manager
  i18n/              Translation catalog and language tag handling
  idle/              Idle manager, inhibitor, grace overlay
  ipc/               IPC client/service and CLI command parsing
  launcher/          Launcher providers (apps, emoji, calculator, sessions, windows, plugins)
  net/               HTTP client, URI parsing, URL opening
  notification/      Notification model, manager, filtering, history
  pipewire/          PipeWire audio service, sound playback, spectrum analyzer
  render/
    animation/       Animation manager, easing, motion settings
    backend/         GLES render backend, framebuffers, texture manager
    core/            Render data types, image decoders/encoders, texture caches
    programs/        GLES shader programs
    scene/           Scene graph nodes and pointer input dispatch
    text/            Cairo/Pango text and glyph rendering
  scripting/         Luau plugin runtime, manifests, registry, source mgmt, bindings (see references/plugin-development.md)
  shell/
    bar/             Bar surface, instance, widget base/factory + widgets/ (see references/adding-a-widget.md)
    clipboard/ control_center/ desktop/ dock/ greeter/ launcher/ lockscreen/ notification/ osd/
    panel/ polkit/ screen_corners/ session/ settings/ setup_wizard/ surface/ switcher/ tooltip/
    tray/ wallpaper/ widgets_editor/
  system/            Desktop entries, brightness, weather, location, system monitor, hardware services
  theme/             Palette generation, templates, theme service, template application
  time/              Time service and polling
  ui/
    controls/        Reusable controls (Button, Input, Label, Select, Slider, Box, ...)
    dialogs/         File, color, glyph picker dialogs
    visuals/         Shared visualizer controls
  util/              Generic helpers
  wayland/           Wayland connection, seats, surfaces, clipboard, toplevels, text input
    hyprland/        Hyprland-specific protocol helpers
assets/              fonts/ sounds/ templates/ translations/  (shipped runtime data - see Runtime assets below)
protocols/           Vendored Wayland protocol XML files
tests/               Unit tests, one file per unit, plus config_validate/ fixtures
tools/               Developer/translation helper scripts (i18n-check.py, palette-generator-analysis.py, ...)
nix/                 Nix package, module, dev shell definitions
third_party/         Vendored: Wuffs (images), Luau (plugin runtime), fzy (fuzzy match), material_color_utilities
```

When a task doesn't obviously map to one of these, grep first rather than guessing - the layout is consistent
enough that `grep -r` for a config key, D-Bus interface name, or widget type string usually lands you in the
right file fast.

## Naming conventions (enforced by clang-format/clang-tidy in CI)

| | Convention | Example |
|---|---|---|
| Files | snake_case | `widget_factory.cpp` |
| Directories | snake_case | `shell/bar/widgets/` |
| Types / Classes | PascalCase | `WidgetFactory` |
| Functions / Methods | camelCase | `createWidget()` |
| Variables / Parameters | camelCase | `busName` |
| Private members | m_camelCase | `m_changeCallback` |
| Macros / Enum values | SCREAMING_SNAKE_CASE | `MAX_SIZE` |

Exception: D-Bus wire-protocol string literals (e.g. `player["bus_name"]`) stay snake_case - they're wire names,
not C++ identifiers. Don't "fix" these to camelCase.

## Critical gotcha: meson.build lists every source file explicitly

There's no globbing. Adding a new `.cpp`/`.h` under `src/` does nothing until you also add the `.cpp` path to the
`_noctalia_sources` list in `meson.build` - it will silently fail to compile in (no error, the file is just
never built). Same applies to any new file you split off from an existing one.

Adding a new unit test requires **two** edits, not one:
1. Create `tests/<name>_test.cpp` using `TEST_CHECK(...)` macros from `tests/test_check.h`.
2. Add `'<name>'` to the `_cpp_test_names` list in `meson.build` (search for `_cpp_test_names`). The test target
   is auto-generated from that name plus the matching file - both must exist and match exactly.

## i18n rules

User-facing strings go in `assets/translations/en.json` only. Never hand-edit or machine-translate other locale
files unless the task is explicitly about translation tooling/import-export sync, or a maintainer asked for that
specific locale. Non-English translations come from the separate Noctalia Translate workflow.

After adding or renaming translation keys, run:

```sh
python3 tools/i18n-check.py            # fast: missing-key check only
python3 tools/i18n-check.py --all      # + unused keys + raw (un-wrapped) UI string literals
```

Wrap new user-visible strings in `i18n::tr` (or the `labelKey`/`descriptionKey`/`titleKey` pattern used by
widget and plugin field schemas) rather than hardcoding literals in `src/ui/` or `src/shell/`.

## Config schema changes

Schema lives under `src/config/schema/`. See `references/config-schema.md` before adding, renaming, or
retyping a config key - most schema changes need a migration entry (`src/config/config_migrations.h`) and are
covered by `tests/config_schema_roundtrip_test.cpp`, so skipping either is a common source of CI failures.

## Adding or editing a bar/dock/desktop widget

Widgets follow a declarative three-piece pattern (implementation, field definition, factory registration). See
`references/adding-a-widget.md` for the full walkthrough with a real example (`battery_widget*`).

## Plugin (Luau) development

User-installed plugins are Luau scripts described by a `plugin.toml` manifest, loaded through
`src/scripting/`. See `references/plugin-development.md`.

## Runtime assets

Noctalia needs the `assets/` tree at runtime, not just the binary - lookup order is `NOCTALIA_ASSETS_DIR` env
var, then relative to the executable, then the Meson install path, then the source-tree `assets/` as a dev
fallback. If you add a new asset (font, sound, template), make sure it's actually reachable via one of these
paths when testing with `just run` from a source checkout.

## Debugging a running instance

All debug commands go through the `dev.noctalia.Debug` D-Bus service:

```sh
gdbus call --session --dest dev.noctalia.Debug --object-path /dev/noctalia/Debug \
  --method dev.noctalia.Debug.SetVerboseLogs true

gdbus call --session --dest dev.noctalia.Debug --object-path /dev/noctalia/Debug \
  --method dev.noctalia.Debug.EmitInternalNotification "Noctalia" "Test" "Hello from debug" 5000 1
```

## Before wrapping up any change - PR checklist

This mirrors `.github/PULL_REQUEST_TEMPLATE.md`. Walk through it explicitly before telling the user something
is done:

- [ ] Ran `just format` (clang-format v22+)
- [ ] Ran the relevant build/test commands (`just build`, `just test debug --print-errorlogs`, `just lint`), or can explain why one was skipped
- [ ] Self-reviewed the diff for new compiler warnings or clang-tidy findings
- [ ] Added/updated `assets/translations/en.json` for any new user-facing string, and touched no other locale file
- [ ] Used existing canonical names for config keys, IPC names, paths, and identifiers rather than inventing new ones
- [ ] Flagged if end-user documentation (docs.noctalia.dev) needs a follow-up update for user-facing config/behavior changes
- [ ] For UI/visual/animation/layout changes: noted what compositor(s) this was tested on (Niri, Hyprland, Sway, ...) or that it wasn't tested live

## Quick lookup

| Working on... | Look under |
|---|---|
| Bar/dock/launcher/control-center widgets | `src/shell/<area>/widgets/`, pattern in `references/adding-a-widget.md` |
| A specific D-Bus integration (bluetooth, network, mpris, tray, upower, polkit, power) | `src/dbus/<name>/` |
| Rendering, animation, shaders, texture handling | `src/render/{animation,backend,core,programs,scene,text}/` |
| Wayland protocol / compositor-specific behavior | `src/wayland/`, `src/compositors/`, `protocols/*.xml` |
| Config keys, validation, migrations, hot reload | `src/config/`, `src/config/schema/`, `references/config-schema.md` |
| Plugins / Luau scripting API | `src/scripting/`, `references/plugin-development.md` |
| Notifications, OSDs, lockscreen, session actions | `src/shell/{notification,osd,lockscreen,session}/` |
| Theming, palette generation, templates | `src/theme/`, `assets/templates/` |
| CLI / IPC commands | `src/cli/`, `src/ipc/` |
| Translations | `assets/translations/en.json`, `tools/i18n-check.py` |
