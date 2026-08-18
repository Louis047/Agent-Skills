# architecture

grabit is a single C17 binary (`src/main.c`) that dispatches to one of several
"actions" (upload / copy / save / ocr / record / pin / tray / plugin exec).
There's no library boundary between subsystems — everything links into one
binary — but the `src/` tree is cleanly partitioned by directory, and each
subsystem exposes a small public header plus (usually) a `*_internal.h` for
its own private structs.

## control flow

1. `main()` pre-scans argv for `--silent`/`--debug`, installs signal handlers
   (SIGINT/TERM/HUP → `gapp_unlink_tmpfile`), then branches on `argv[1]`:
   `--version`, `--help`, `help`, `set`/`get`/`unset`, `sxcu`, `plugin`, `-p
   <plugin>`, or a bare word that isn't a flag (→ try plugin dispatch,
   `gapp_try_dispatch_plugin` in `src/app/dispatch.c`). Anything else falls
   through to `args_parse()` → `struct args` (`src/args.c`/`.h`).
2. `run(&a)` in `main.c`: loads config (`config_load_full`), migrates state,
   applies config-derived defaults onto the parsed args (`default_action`,
   `edit.default`, `region.repeat_last`, `capture.delay`), then switches on
   the effective `enum action`:
   - `ACTION_UPLOAD/COPY/OUTPUT/PIN` → `gapp_run_*` in `src/app/actions.c`
   - `ACTION_OCR` → `gapp_run_ocr` in `src/app/ocr.c`
   - `ACTION_RECORD` → `record_toggle` in `src/record/record.c`
   - `ACTION_PIN_GRAB/RELEASE/CLOSE_ALL` → `src/pin/pin_ipc.c`
   - `ACTION_TRAY` → `tray_app_run` in `src/tray/apptray.c`
3. Most `gapp_run_*` handlers call `gapp_acquire_source()`
   (`src/app/source.c`) to get pixels: either a fresh capture (region select,
   `-F`/fullscreen, `-w`/window) via `src/capture` + `src/region`, or an
   existing `--file`. The result is handed to clipboard / upload / save / ocr.
4. Region selection opens its own Wayland layer-shell surface
   (`src/region/wlr_layer.c`) over a frozen screenshot
   (`src/capture/freeze.c`), runs its own input+render loop
   (`region/input_*.c`, `region/render_*.c`) with the annotator, toolbar, and
   color picker, then returns the finalized rect + annotation list back up.

## subsystem map

| dir | purpose | notable files |
|---|---|---|
| `src/app/` | orchestration glue between args/config and every subsystem | `dispatch.c` (plugin exec), `source.c` (acquire/release capture source), `actions.c`, `ocr.c`, `help.c` |
| `src/config/` | TOML config + state files, schema validation, `set`/`get`/`unset` CLI, `grabit help` text | `schema.c` (key validation tables), `cli.c`, `save.c`, `patch.c`, `zipline.c` |
| `src/wl/` | core Wayland client: registry binding, display pump, output/monitor tracking, foreign-toplevel tracking | `wl.c`, `registry.c`, `output.c`, `monitors.c`, `toplevel.c` |
| `src/wm/` | compositor-specific active-window/workspace queries, dispatched by detected compositor | `wm.c` (dispatch), `hyprland.c`, `niri.c`, `ipc.c` (shared IPC helper) |
| `src/capture/` | screenshot backends + encode | `wlr_screencopy.c`, `ext_image_copy.c`, `kwin_screenshot.c`/`kwin_reply.c` (KDE), `freeze.c`, `save.c`/`png.c`/`jpeg.c`/`webp.c` |
| `src/region/` | interactive region-selector/annotator UI (its own layer-shell surface) | `wlr_layer.c`/`wlr_render.c`/`wlr_input*.c`, `annotate*.c` (paint tools), `toolbar*.c`, `color_picker*.c`, `keybinds*.c`, `input_*.c`, `edit_persist.c` |
| `src/record/` | screen recording | `record.c` (orchestrate), `loop.c`/`segments.c`/`ring.c`, `ffmpeg.c` (subprocess pipe), `pw.c` (PipeWire, `HAVE_PIPEWIRE`-gated), `sc_kde.c`/`sc_gnome.c` (screencast D-Bus), `overlay.c`/`controls*.c` (on-screen HUD), `pid.c` (start/stop toggle) |
| `src/tray/` | StatusNotifierItem D-Bus tray icon | `apptray.c`, `sni.c`/`sni_props.c`, `menu.c`/`menu_layout.c` |
| `src/upload/` | uploader | `upload.c` (orchestrate), `services.c` (built-in service registry — see conventions.md recipes), `zipline.c` (chunked uploads), `sxcu_*.c` (ShareX `.sxcu` custom uploaders), `errors.c` |
| `src/plugin/` | plugin manager (see `PLUGINS.md` + `references/plugins.md`) | `manifest.c`, `install.c`, `update.c`, `fetch.c` (prebuilt + sha256), `spawn.c`/`dispatch.c` (exec), `lock.c` |
| `src/pin/` | desktop-pinned screenshot windows | `pin.c`, `pin_render.c`/`pin_input.c`/`pin_cursor.c`/`pin_outputs.c`, `pin_ipc.c` (grab/release/close-all), `text_card.c`, `preview.c` |
| `src/ocr/` | OCR + translation | `tesseract.c` (shells out), `translate.c` (dispatch), `translate_libre.c`, `translate_deepl.c` |
| `src/clipboard/` | clipboard read/write | `clipboard.c`, `wlr_data_control.c` / `ext_data_control.c` (two protocol backends), `send.c` |
| `src/notify/dbus.c` | freedesktop desktop notifications | — |
| `src/sound/sound.c` | shutter/notification sound | — |
| `src/util/` | grab-bag utilities | `util.c` (xasprintf, paths, spawn/capture, edit distance), `buf.c` (`struct grabit_buf`), `shm.c` (POSIX-shm-backed `wl_buffer`), `proc.c`, `json_path.c` (dotted-path JSON extraction) |
| `src/vendor/` | third-party, vendored | `tomlc99` (TOML parser), `sha256` — see conventions.md, do not reformat |
| top-level | entry point + plumbing | `main.c`, `args.c`, `cursor.c`, `log.c`, `mime.c`, `paths.c`, `template.c` (filename templates) |

## protocols

`protocols/*.xml` are Wayland protocol XML specs (upstream + wlroots +
KDE-specific). `wayland-scanner` code-generates headers/sources from them into
`build/protocols/` at build time (see `Makefile`'s `WL_PROTOCOLS` list) — this
generated code is deleted right after the final link. Don't hand-edit
anything under `build/`.

## compositor / DE support matrix

- hyprland, sway, niri, river — screenshot + record via wlr protocols
  (`wlr_screencopy.c`), no portal dialog.
- KDE Plasma — screenshots via kwin's `ScreenShot2` D-Bus interface
  (`kwin_screenshot.c`), recording via kwin's screencast protocol
  (`sc_kde.c`).
- GNOME — recording-only, via mutter's screencast D-Bus API (`sc_gnome.c`).
- X11 is not supported at all.
