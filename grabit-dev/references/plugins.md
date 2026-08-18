# plugin development

`PLUGINS.md` in the repo root is the authoritative spec (manifest schema,
install/update semantics, env vars, dispatch precedence) — read it for the
full picture. This file only covers the local-dev-loop details it doesn't
spell out.

## what a plugin is

A plugin is a **separate binary/repo**, not part of grabit's own build.
grabit dispatches to it by exec — the plugin can be any language. A
single-file C helper, `include/grabit-plugin.h`, is available to vendor into
a plugin repo if the plugin is C and wants to call back into grabit
(`grabit_plugin_capture()`, `grabit_plugin_pin()`, etc.) — see PLUGINS.md's
function table for exact contracts and who frees what.

## manifest.toml (minimum)

```toml
name = "myplugin"
description = "what it does"

[build]
cmd = "make"                 # run via /bin/sh -c from the plugin dir
binary = "build/myplugin"    # path relative to the plugin dir
```

or, for a prebuilt binary (`sha256` mandatory, https-only):

```toml
name = "myplugin"

[prebuilt]
url = "https://example.com/myplugin-x86_64"
sha256 = "<hex>"
```

## local dev loop

```sh
grabit plugin install /path/to/plugin-repo   # accepts local paths, not just URLs
grabit myplugin [args]                        # exec it
grabit plugin show myplugin                   # inspect the parsed manifest
grabit plugin update myplugin                  # re-pull + rebuild after changes
grabit plugin remove myplugin                  # uninstall
```

**Caveat**: `install`/`update` do `git clone --depth 1` / `git fetch` under
the hood, even for a local path — **uncommitted changes in the plugin repo
won't be picked up**. Commit before `install`/`update` when iterating
locally, or symlink `~/.config/grabit/plugins/myplugin/` at your working
tree if you want live edits (bypasses the manifest-driven install flow
entirely, which is fine for iteration but skip that for anything you're
about to test end-to-end).

## dispatch caveats worth knowing when testing

- `set`, `get`, `unset`, `sxcu`, `plugin`, and `help` are matched before
  plugin dispatch — a plugin named any of those is unreachable.
- `grabit --silent myplugin` never dispatches (dispatch only looks at
  `argv[1]`, and only when it doesn't start with `-`).
- `[capture] auto = true` in the manifest auto-captures a screenshot
  in-process and prepends its path as the plugin's first positional arg,
  *unless* the caller already passed one. Per-call overrides:
  `--capture` / `--no-capture` (stripped before forwarding to the plugin).
  Useful for a plugin that always wants "the current screenshot" without
  the caller having to invoke `grabit -o` first.
- `grabit -p myplugin [args]` runs the plugin, takes the last line of its
  stdout as a file path, and execs `grabit --pin -f <path>` — handy for
  testing a plugin that produces an image/file and should land pinned on
  the desktop immediately.
