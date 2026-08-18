# conventions

## style

- C17, tabs (`.clang-format`: `BasedOnStyle: LLVM`, `UseTab: Always`,
  `TabWidth`/`IndentWidth: 4`, `ColumnLimit: 0` — no forced line wrap),
  `PointerAlignment: Right` (`char *p`, not `char* p`), braces attached
  (`if (x) {` not `if (x)\n{`). Always run `make fmt` rather than
  hand-formatting — don't fight the formatter's opinions on individual
  files.
- Every `.c`/`.h` (except `src/vendor/`) needs this exact two-line header
  before anything else, enforced by `make test`:
  ```c
  // SPDX-License-Identifier: AGPL-3.0-or-later
  // Copyright (C) 2026 creations
  ```
  `make apply-headers` inserts it for you on new files.
- No VLAs (`-Wvla`), no implicit declarations, no missing prototypes —
  declare every non-`static` function in a header. Always spell out
  `(void)` for zero-arg functions (`-Wstrict-prototypes`).

## naming prefixes

- `grabit_*` — cross-cutting utilities usable from anywhere (`util/`, `wl/`,
  `wm/`), e.g. `grabit_xasprintf`, `grabit_wl_init`.
- `gapp_*` — the app-orchestration layer in `src/app/`, e.g.
  `gapp_run_upload`, `gapp_acquire_source`.
- domain-prefixed — most subsystems prefix their public functions with the
  domain noun: `capture_*`, `annotation_*`/`ganno_*`, `config_*`,
  `upload_*`/`gup_*`, `plugin_*`, `record_*`, `tray_*`, `pin_*`, `ocr_*`,
  `clipboard_*`.
- `cmd_*` — top-level subcommand entry points called from `main.c`
  (`cmd_set`, `cmd_get`, `cmd_unset`, `cmd_sxcu`, `cmd_plugin`).

## file layout within a subsystem

Public API goes in `<dir>/<dir>.h` (or a specifically-named header).
Structs and helpers private to the subsystem go in `<dir>/*_internal.h`,
included only by that subsystem's own `.c` files — e.g.
`region/annotate_internal.h`, `region/input_internal.h`, `tray/internal.h`,
`upload/internal.h`, `clipboard/clipboard_internal.h`. When adding a new
file to an existing subsystem, follow this split rather than putting
everything in one header.

## error handling

- Functions return `int` (`0` success, nonzero/`-1` failure) or a pointer
  (`NULL` on failure) — no exceptions, no `setjmp`/`longjmp`.
- `log_error`/`log_warn`/`log_info`/`log_debug` (`src/log.h`) for
  user-facing messages. `die()` is reserved for truly unrecoverable startup
  failures — everywhere else, propagate the failure up the call stack
  instead of calling `exit()` deep in a subsystem.
- Memory is manual `malloc`/`free`, no smart pointers. Growable buffers use
  `struct grabit_buf` (`grabit_buf_grow`/`putn`/`puts`/`putc`/`free` in
  `util/buf.c`). Formatted heap strings use `grabit_xasprintf`. Ownership
  ("caller frees") is documented in the producing function's header comment
  — check it before assuming who's responsible for freeing a return value.

## vendored code

`src/vendor/tomlc99` and `src/vendor/sha256` are third-party, exempt from
the SPDX check and built with a relaxed warning set (see the `Makefile`'s
`filter-out` rules for `vendor/%.o`). Treat as a black box; only patch to
fix an actual bug, and keep the diff minimal — don't run `make fmt` over it
or restructure it to match house style.

## common extension recipes

- **new upload service**: add a `{name, base_url_or_NULL, auth_header,
  response_json_path, bool}` entry to `SERVICES[]` in
  `src/upload/services.c`; document the key in `OPTIONS.md`.
- **new config key**: add a validation entry (`VALS_*` array or equivalent)
  in `src/config/schema.c` — `grabit set <key>` becomes self-documenting
  from that table.
- **new CLI flag**: `src/args.c` (parsing) + `src/args.h` (`struct args`
  field) + `src/app/help.c` (usage text).
- **new annotation tool**: extend `enum tool_kind` in `src/region/region.h`,
  then touch `region/annotate_paint.c` (rendering), `region/tool_picker.c` +
  `toolbar_icons.c` (UI), `region/keybinds*.c` (shortcut), and
  `region/edit_persist.c` (persistence). This touches the most files of any
  recipe here — grep an existing tool (e.g. `TOOL_SPOTLIGHT`) across
  `src/region/` to find every call site.
- **new capture backend**: implement against `src/capture/backend.h`'s
  contract; register availability probing alongside `wlr_screencopy.c` /
  `ext_image_copy.c` / `kwin_screenshot.c` in `capture.c`'s capability
  checks.
- **new compositor WM integration**: add `src/wm/<compositor>.c` shaped
  like `hyprland.c`/`niri.c`, wire detection into `src/wm/wm.c`
  (`grabit_wm_current_name` + dispatch).
