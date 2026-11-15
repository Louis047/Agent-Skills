# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Contributor fork (origin `Louis047/windhawk-mods-fork`) of the Windhawk mod marketplace `ramensoftware/windhawk-mods`. PRs go to upstream `main`, GitHub user `Louis047`. System is Windows (PowerShell). `git`, `gh`, `python` are installed. Windhawk 2.0.0-alpha.8 is at `C:\Program Files\Windhawk` (`windhawk-cli.exe`).

## Strict rules
- No em-dashes anywhere (code, comments, README blocks, commits, PR text, replies).
- `CLAUDE.md`, `AGENTS.md`, `.claude/`, `mod-snapshots/`, `mod-logs/`, `mod-tests/` are local only. They are listed in `.git/info/exclude`. Never stage or commit them. Never edit the tracked `.gitignore` (PRs must touch exactly one file).
- Keep the session interactive when a decision or missing requirement could derail the work. Ask rather than guess.

## Branch and PR workflow
- New mod, or update to a mod the author (user) owns: start from a new branch with a relevant name and keep working on it until the user confirms the PR into `main` (or other target) is merged. Then delete the branch (also if the user asks to delete it up front).
- If the branch already exists, switch to it and continue. Ask the user for the PR link to validate.
- Commits: single-line, very short, describing the changes done. On a fresh branch cover the whole change. With existing local/PR commits, cover only what changed since the last commit.
- After any commit or push, check the CI checks (`gh pr checks`). If any fail, read the warnings/errors, fix, re-verify, repeat until all pass.
- Validate any researched fix before applying: confirm it actually resolves the bug, is the most feasible option, and causes no regressions.
- Stick to the PR template and fill in the required areas correctly. Don't use any sort of claude promotions. Even in the commits. It should be under the author/user, not claude or any other models, to keep things clean.

## Versioning
- New mod: ask the user for the starting version and bump method.
- Existing mod with no PR yet: ask the user for the bump.
- Existing mod PR already in progress: follow the mod's own version convention. Do not bump again if the PR already bumped it. Bump only when the user asks.
- `@version` must be new vs the published `versions.json`. Follow the mod's existing scheme (e.g. `1.6`, `1.31`).

## Local folders
- `mod-snapshots/`: snapshot each iteration of the mod being worked on (e.g. `<mod-id>/<version-or-step>.wh.cpp`). Diff against it to catch regressions before finishing. Create the folder if missing.
- `mod-logs/`: manual and automated test logs per mod under a relevant name. Use for cross-referencing. Create if missing.
- `mod-tests/`: side-contributions to a mod where the user is not the main author (e.g. `<mod-id>/<version-or-step>.wh.cpp`). The mod's original author GitHub id is required, so the author's other mods can be looked up and verified easily. Local only and on the no-commit list. Create if missing.
- "CHANGELOGS" from the user: reply with heading `## Changelogs` and very short, single-sentence bullet points for the changes since the last local/PR commit.

## Repo layout and PR constraints
- One mod = `mods/<mod-id>.wh.cpp`. `@id` must equal the file name (`0-9 a-z -` only). A PR adds or modifies exactly one file.
- File structure: `// ==WindhawkMod==` metadata, `// ==WindhawkModReadme==` (`/* markdown */`), `// ==WindhawkModSettings==` (`/* YAML */`), then code.
- `@github` must be `https://github.com/Louis047` (no trailing slash or path). Existing mods cannot change `@id`, `@github`, `@author`, `@twitter`. `@twitter` needs manual verification.
- Update PRs need a changelog between `<!-- changelog:start -->` and `<!-- changelog:end -->` in the PR body. New-mod PRs keep the `## Mod authorship` section. Use `.github/pull_request_template.md`.
- Squash-merge commit messages become the changelog. `changelog_override/<id>/<version>.md` patches bad ones.
- README images only from `https://i.imgur.com` or `https://raw.githubusercontent.com`.
- Validator warnings to avoid: `InternalWh`, `WH_EDITING`, `WH_MOD`, `GWL_WNDPROC`/`GWLP_WNDPROC`, `Wh_FindFirstSymbol`/`Wh_FindNextSymbol`/`Wh_FindCloseSymbol` (use `WindhawkUtils::HookSymbols`), `noUndecoratedSymbols`, `Wh_SetFunctionHookT`, `__ptr64`, `__try`, redefining `Wh_*`/`WH_*`, control characters, lines over 4096 chars, 3+ statements on a 150+ char line. Callback signatures must match exactly (`BOOL Wh_ModInit()`, `void Wh_ModUninit()`, ...).

## CI checks (all warnings fail the job)
- `pr_validation.yml`: run locally with `pip install pyyaml` once, then `python .github/pr_validation.py mods/<id>.wh.cpp Louis047`. Needs network.
- `mod_compatibility_check.yml`: compiles the mod on Windhawk 1.6.1, 1.7.3 and 2.0.0-alpha.8 and fails on compile warnings too. Locally: `& "C:\Program Files\Windhawk\windhawk-cli.exe" --json mod install --disabled --arch all --strict --file mods\<id>.wh.cpp`.
- Keep mods compatible with Windhawk 1.7.3 as well as 2.x. 2.0-only settings annotations (`$format`, `$float`, `$dynamicSelect`, `$min`, `$max`, `$showIf`, `$hideIf`) must be prefixed with `#! ` and, once one is marked in a block, all listed annotations in that block must be. Never mark `$name`/`$description`. `windhawk_tool_mod.h` and `Wh_*DynamicSelectOption` do not exist on 1.7.3.
- `deploy.yml`, `pr_labels.yml`, `pr_commands.yml` and issue workflows are upstream-only. Do not run `deploy.ts` locally.

## Automatic validation (run after every change to a mod file)
Run this loop after each edit session on `mods/<id>.wh.cpp` and before any commit or push. Do not wait to be asked.
1. `pip install pyyaml` (once), then `python .github/pr_validation.py mods/<id>.wh.cpp Louis047`. Any `::warning` line is a CI failure.
2. `& "C:\Program Files\Windhawk\windhawk-cli.exe" --json mod install --disabled --arch all --strict --file mods\<id>.wh.cpp`. Warnings fail too.
3. Check 1.7.3 compatibility by hand (see the `#!` rule above).
4. Snapshot the file to `mod-snapshots/<id>/` (name by version or step) and diff against the previous snapshot to catch regressions.
5. Save validation output to `mod-logs/<id>-validation.txt`.
6. Fix verified problems and repeat until clean. Report any remaining warning and why.

## Upstream PR review process
Source: https://github.com/ramensoftware/windhawk/wiki/Mod-submission-review-process
- Each PR carries one flow label: `waiting-for-author` (set automatically on new PRs and whenever a human requests changes), `waiting-for-ai-review`, `waiting-for-reviewer`. Read the label and the latest comments first to learn a PR's state before acting on it.
- Flow: author comments `/ai-review` (takes 20-30+ minutes), fixes findings and re-runs `/ai-review` as needed, then comments `/ready-for-reviewer`. If a human requests changes the PR returns to `waiting-for-author`. Skipping the AI review sends the PR back.
- Using `/ready-for-reviewer` with unaddressed AI findings requires a one-sentence reason each: the AI is wrong, out of scope or not wanted, or planned for a future version.
- The AI reviewer can be wrong. Verify each finding against the code and Windhawk behavior. Do not change working code just to satisfy it. The author owns the submitted code.
- Only the user posts `/ai-review` and `/ready-for-reviewer` comments. When resuming an existing PR, use `gh pr view <link> --comments` and `gh pr checks <link>` to summarize labels, AI findings, human requests and CI state, then propose the next step.

## Windhawk mod gotchas
Wiki: https://github.com/ramensoftware/windhawk/wiki (Creating-a-new-mod, Development-tips, Global-objects-and-process-shutdown, Mod-lifetime, Injection-targets-and-critical-system-processes, Mods-as-tools:-Running-mods-in-a-dedicated-process, Mod-submission-review-process, Debugging-the-mods, Troubleshooting). Fetch the relevant page before non-trivial work.
- Compiler is Clang 20 mingw-w64, C++23. Extra libs via `@compilerOptions` (e.g. `-lcomctl32`). API headers: `.vscode/windhawk_headers_1.7.3/`.
- Set hooks in `Wh_ModInit` (`Wh_ApplyHookOperations` is slow). Hook functions need the right calling convention (`WINAPI` for WinAPI). Prefer `WindhawkUtils::SetFunctionHook` and `HookSymbols` from `<windhawk_utils.h>`.
- Lifetime: `Wh_ModBeforeUninit` runs with hooks still active, `Wh_ModUninit` after hooks are removed. `Wh_ModUninit` does NOT run on host process exit, but global and function-static destructors do (under loader lock, other threads dead).
- Globals holding `std::thread`, strong WinRT/COM refs, XAML objects, or RAII wrappers around `CoUninitialize`, `GdiplusShutdown`, `UnregisterClass`, `DestroyWindow`, `UnhookWindowsHookEx`, `RemoveWindowSubclass` need `[[clang::no_destroy]]` plus full release in `Wh_ModUninit` (and `Wh_ModSettingsChanged` if it recreates state). Do not add it to safe globals. Join threads before `Wh_ModUninit` returns. Always `UnregisterClass` there.
- `Wh_ModSettingsChanged` can return `FALSE` to unload or set `*bReload = TRUE` to reload. Free string settings (`Wh_FreeStringSetting` or `WindhawkUtils::StringSetting`).
- Critical processes (`csrss`, `dwm`, `lsass`, `winlogon`, `svchost`, `werfault` etc.) ignore pattern `@include`. A mod loads there only if it names the process explicitly. Avoid targeting them.
- Tool mods (run in a dedicated process): `WhTool_ModInit/SettingsChanged/Uninit`, `@include` exactly one allowed host set (`windhawk.exe`, `windhawk-mod.exe`, `windhawk-mod-uiaccess.exe`, `windhawk-mod-elevated.exe`).
- Localization: `@name:fr-FR` style metadata and `$name:fr-FR` in settings. `@architecture` defaults to x86 and x86-64.
- Logging: `Wh_Log(...)`. Collect logs via mod Advanced tab, Debug logging "Detailed debug logs". Empty log means injection failed, `HTTP_STATUS_NOT_FOUND` means symbols are unavailable.
- Debug builds: add `--optimize=0 --debug` to `@compilerOptions` (remove before committing).
- Testing shutdown paths: toggle the mod repeatedly (handle and thread counts must return to baseline) and test graceful host exit (e.g. Exit Explorer).

## Local environment gotchas
- `pip install pyyaml` fails to build on Python 3.15. Use the pure-python `lib/yaml` from the PyYAML sdist on `PYTHONPATH`, with `PYTHONUTF8=1`.
- `windhawk-cli ... mod install` needs write access to `C:\ProgramData\Windhawk`. Run it against an isolated copy via `--app-root` (copy of the install, `windhawk.ini` pointing `AppDataPath` to a user folder and `RegistryKey` to an `HKCU` key), so the real Windhawk is untouched.
- Some mods have mixed CRLF/LF line endings. Edit tools rewrite whole files to LF and produce a full-file diff. After editing, rebuild the file on top of the committed bytes so only changed lines differ, check `git diff --stat` is minimal, and use `git -c core.autocrlf=false` for add, diff and commit.
- Commit identity is not configured. Use one-off `git -c user.name=... -c user.email=...` with the author of the existing branch commits, and do not change git config.
