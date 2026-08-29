# osx-env-sync

Synchronize macOS environment variables for login shells and GUI apps from a login shell's environment (default donor: `/bin/zsh` login shell → `~/.zprofile`).

This repo is a single self-contained zsh script. Keep it that small. Do not introduce a package manager, Python/Node layout, or extra abstraction unless the user asks.

## Repository map

| Path | Role |
|------|------|
| `osx-env-sync` | The whole tool: `install` / `uninstall` / `reinstall` / `sync` subcommands. Installed to a directory on `$PATH` (README uses `~/bin`). |
| `README.md` | Install, reload, and login-shell sourcing order. |
| `LICENSE` | Apache 2.0. |

Verified absences: no tests, no CI, no `.gitignore`, no package manifest, no Python/Node layout, no SQL, no secrets/config files, no orchestration beyond this one LaunchAgent.

One installed path is not a repo path: `install` writes `~/Library/LaunchAgents/osx-env-sync.plist`, which hard-codes the script's absolute path (captured at install time via `${0:A}`).

## Core architecture and control flow

1. `osx-env-sync install` writes the plist to `~/Library/LaunchAgents/osx-env-sync.plist` and loads it with `launchctl bootstrap gui/$(id -u)`.
2. At login, launchd runs the agent (`RunAtLoad`, `LimitLoadToSessionType: Aqua`).
3. The agent runs `env -i $DONOR_SHELL --login -c env` (default `DONOR_SHELL=/bin/zsh`). `--login` makes it a login shell so `/etc/zprofile` → `~/.zprofile` → `~/.zlogin` are sourced; `env -i` starts from a clean environment so captured values come only from those files.
4. Each `NAME=value` line is piped into ruby, which calls `launchctl setenv NAME value` per line (ruby handles quoting/escaping).
5. The result is published to the user session. GUI apps launched by launchd/Dock afterwards inherit it. Already-running apps do not.
6. Mid-session refresh: `osx-env-sync sync` runs the capture directly, no launchd round-trip.
7. `uninstall` removes the agent with `launchctl bootout gui/$(id -u)/osx-env-sync` and deletes the plist. `reinstall` chains uninstall + install.

```
~/.zprofile  --(login zsh)-->  env capture  --(ruby)-->  launchctl setenv  -->  user session / GUI apps
                 ^
                 |
    osx-env-sync.plist (launchd, RunAtLoad, written by `install`)
                 ^
                 |
        osx-env-sync install / sync / reinstall / uninstall
```

## Script contracts

### `osx-env-sync`

- Shebang: `#!/usr/bin/env zsh`. Depends only on stock macOS `/bin/zsh` and `/usr/bin/ruby`.
- `DONOR_SHELL` env var selects the capture shell, default `/bin/zsh`. Zsh exports living in `~/.zshrc` are not seen by `--login` — users must use `~/.zprofile` or a donor invocation with `-i` (documented gotcha).
- `SCRIPT=${0:A}` resolves the script's absolute path at runtime; the generated plist points launchd at it.
- `write_plist` writes the plist via heredoc. `install`/`uninstall` use modern `launchctl bootstrap`/`bootout` verbs, not deprecated `load`/`unload`.
- `sync` is the capture-and-publish core: `env -i $DONOR_SHELL --login -c env | ruby -ne '...system "launchctl", "setenv", $1, $2'`.
- No output on success (except launchd status in `launchctl list`).

## Configuration and environments

- Source of truth is the user's login-shell profile (`~/.zprofile` for stock zsh), not anything in this repo.
- `DONOR_SHELL` is the only knob; it can be set inline per invocation or edited in the script before `install`.
- README documents zsh login-shell order and `path_helper` (`/etc/paths`, `/etc/paths.d`, `/etc/manpaths.d`).
- No project-local env files, secrets, or identity/access config.

Verified on macOS 26 (Tahoe): `sync` publishes vars from `~/.zprofile` to the session; `install`/`uninstall` round-trip cleanly. Known limitation: mid-session `setenv` does **not** reach GUI apps launched via LaunchServices (Dock/Finder) on modern macOS — those inherit env captured at login. The login-time `RunAtLoad` path is the working delivery route; mid-session `sync` mainly helps launchd-spawned processes.

## Testing

Not present. There is no harness. If adding one, the in-repo behavior worth asserting: the ruby one-liner calls `launchctl setenv` once per `NAME=value` line, and `install`/`uninstall` idempotently manage the plist + launchd registration.

## Architectural constraints

- Stay a single zsh script. Do not grow a CLI framework, installer, or second source-of-truth format.
- Preserve the login-shell trick (`--login -c env`) so values resolve after the user's login profile is sourced.
- Preserve the `env -i` clean-environment capture — it is what guarantees captured values come from the profile, not the launching context.
- Preserve the bootstrap/bootout verbs; do not reintroduce deprecated `load`/`unload`.
- `DONOR_SHELL` is an override, not a config file. Keep configuration env-var-shaped.

## How to extend safely

| Change | Where | Existing example |
|--------|--------|------------------|
| New env var | User's login profile as an `export NAME=…` line, then `sync` | README |
| Reload without logout | `osx-env-sync sync` | the `sync` subcommand |
| Install / persist at login | `osx-env-sync install` (writes plist + `launchctl bootstrap`) | that subcommand |
| Use a different shell / profile set | set `DONOR_SHELL` (e.g. `/bin/bash`, or zsh with `-i` for `.zshrc` users) | — |
| Tests / CI | not present | — |

Lazier default when asked to "improve" this: edit the one script and the README. Do not add a package, installer, or second source-of-truth format.
