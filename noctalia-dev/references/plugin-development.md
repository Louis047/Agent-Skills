# Plugin (Luau) development

User-installed extensions (extra bar widgets, launcher providers, desktop widgets, panels, background services,
compositor-specific extras, third-party integrations) run as **Luau** scripts, not C++. The runtime lives under
`src/scripting/`.

## Anatomy of a plugin

Every plugin is a directory with a `plugin.toml` manifest plus one or more Luau script files. Minimal example
(from `tests/config_validate/plugin-source/wallhaven/plugin.toml`):

```toml
id = "noctalia/wallhaven"
name = "Wallhaven"
version = "1.0.0"
plugin_api = 3

[[panel]]
id = "browser"
entry = "panel.luau"
width = 920
height = 780
placement = "attached"
position = "auto"
```

- `id` - reverse-namespace-style unique identifier (`author/plugin-name`).
- `plugin_api` - the API version this plugin was written against. Noctalia enforces a supported range; check
  `src/scripting/plugin_api.h` for `kOldestSupportedPluginApiVersion` and `kCurrentPluginApiVersion` before
  writing a plugin or bumping the version a plugin targets. Each capability the plugin API gained is tagged
  with the version it landed in (e.g. persistent panels, system stats, sound playback, module `require`) - grep
  that file for the constant matching a feature you want to use to know the minimum `plugin_api` value needed.
- `[[panel]]` blocks declare a UI surface backed by a Luau entry script (`entry`), with layout/placement config.

## Settings fields

Plugin-configurable settings are declared in the manifest and typed via `ManifestFieldType`
(`src/scripting/plugin_manifest.h`): `Bool, Int, Double, String, StringList, StringMap, File, Folder, Glyph,
Select, Color`. Each field carries a mandatory `labelKey` and optional `descriptionKey` - these resolve against
the *plugin's own* `translations/<lang>.json`, not the main Noctalia catalog. A typed default lives directly on
the field struct (`boolDefault`, `numberDefault`, `stringDefault`, ...) matching its declared `type`.

## Scripting API surface

- `src/scripting/plugin_api.h` - versioned capability constants (source of truth for "can I use feature X").
- `src/scripting/ui_prelude.h` / `ui_handler_table.h` - the Luau-facing UI API surface plugins call into.
- `src/scripting/plugin_bindings.cpp` - the actual C++ bindings exposed to Luau scripts.
- `src/scripting/script_api_context.h` - the execution context passed into running scripts.

Read the relevant one directly rather than guessing signatures - this surface changes over time (that's exactly
what the versioned constants in `plugin_api.h` track).

## Where plugins live at runtime

Local plugin sources resolve under the user's XDG data directory, `<data-dir>/plugins/` (see
`src/scripting/plugin_source_paths.h` for the exact resolution logic - it also supports git-sourced and
registry-sourced plugins, with materialized roots kept separate from the local source root). A `catalog.toml`
can index multiple plugins from one source (`src/scripting/plugin_catalog.cpp`).

## CLI and linting

Noctalia ships a `plugins` subcommand (`noctalia plugins ...`, dispatched from `src/main.cpp` into
`scripting::plugins::runCli` / `plugin_lint.cpp`), including a `lint` command that validates a plugin directory
against manifest and script rules (`lintPluginDir` in `plugin_lint.h`). Run this against a plugin you're editing
before assuming it will load correctly:

```sh
just build
./build-debug/noctalia plugins lint /path/to/plugin-dir
```

## Testing plugin changes in this repo

If you're changing the **plugin runtime itself** (not authoring a third-party plugin), the relevant coverage is:

- `tests/plugin_manifest_test.cpp` - manifest parsing/validation
- `tests/plugin_lifecycle_test.cpp` - load/enable/disable/reload behavior
- `tests/plugin_catalog_test.cpp` - multi-plugin catalog resolution
- `tests/plugin_bindings_test.cpp` - Luau-facing API bindings
- `tests/plugin_i18n_test.cpp` - plugin translation catalog handling
- `tests/plugin_git_export_test.cpp`, `plugin_source_locks_test.cpp`, `plugin_source_paths_test.cpp`,
  `plugin_process_test.cpp` - source management, locking, sandboxing

Run the targeted subset while iterating (`just test debug --print-errorlogs plugin_manifest_test` etc. -
`meson test` accepts a test name filter), then the full `just test debug` before finishing.

## Checklist for plugin runtime changes

- [ ] Bump `kCurrentPluginApiVersion` (and add a new named constant) in `plugin_api.h` if you're adding a new
      capability plugins can opt into - don't silently change behavior under an existing version number
- [ ] Update or add coverage in the relevant `tests/plugin_*_test.cpp`
- [ ] Run `just build && just test debug --print-errorlogs && just lint && just format`
- [ ] If the manifest schema changed, update the wallhaven fixture or add a new fixture under
      `tests/config_validate/plugin-source/`
