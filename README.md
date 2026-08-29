# osx-env-sync

Synchronize macOS environment variables for command line and GUI applications from a single source.

## Introduction

On macOS, command line applications and GUI applications are treated differently. It's straightforward to feed command line applications environment variables from your shell profile, but GUI applications are launched by the Dock/Finder and don't read shell profiles. **osx-env-sync** captures the environment of a login shell and publishes it to the user session with `launchctl setenv`, so GUI apps launched afterwards can see it.

## Requirements

- macOS ships everything needed: `/bin/zsh` and `/usr/bin/ruby`. No dependencies.
- Your environment must live in your shell's startup files (e.g. `~/.zshrc` or `~/.zprofile`). The default donor is `/bin/zsh -i`, so both are sourced. Drop `-i` from `DONOR_SHELL` for a strict `.zprofile`-only capture.

## Quick Start

Run it straight from the repo (or any directory):

```sh
git clone <this-repo-url> && cd osx-env-sync
chmod +x osx-env-sync
./osx-env-sync install   # one-time: install the launch agent
./osx-env-sync sync      # re-publish env now, e.g. after editing ~/.zprofile
```

To run it without the `./` prefix from anywhere, symlink or copy it to a directory on your `$PATH`:

```sh
mkdir -p ~/bin
ln -sf "$PWD/osx-env-sync" ~/bin/osx-env-sync   # ~/bin must be on your $PATH
osx-env-sync sync
```

## Usage

Put `osx-env-sync` somewhere on your `$PATH` (e.g. `~/bin`) and make it executable, then:

```sh
osx-env-sync install     # install the launch agent (runs at login)
osx-env-sync sync        # re-publish the environment now (e.g. after editing ~/.zprofile)
osx-env-sync reinstall   # reinstall after changing the script itself
osx-env-sync uninstall   # remove the launch agent
```

Installation is persistent: the plist is written to `~/Library/LaunchAgents` and loaded with `launchctl bootstrap`, so it runs at every login.

## Configuration

`DONOR_SHELL` selects the shell whose environment is captured (default `/bin/zsh -i`). It may carry flags, e.g. drop `-i` to skip `~/.zshrc` for zsh. Set it inline:

```sh
DONOR_SHELL=/bin/zsh osx-env-sync sync
```

Note that `install` writes the plist using the script's default; customize `DONOR_SHELL` inside the script before installing if you need a non-default donor.

## How it works

1. On login, launchd runs the agent (installed via `launchctl bootstrap`).
2. The agent executes `env -i <DONOR_SHELL> --login -c env`, capturing the login shell's environment with variables fully expanded.
3. Each `NAME=value` line is fed to `launchctl setenv NAME value` via ruby, which handles quoting safely.
4. Apps launched by the Dock/Finder afterwards inherit the published variables.

## Known limitations

- **Mid-session refresh and GUI apps:** on modern macOS, GUI apps launched by the Dock/Finder may not pick up `setenv` changes made mid-session. Running `sync` refreshes the session for launchd-spawned processes; already-running apps and some Dock-launched apps may need relaunch or a re-login.
- **Things that mutate env in your shell:** whatever your startup files do also happens in the capture. E.g. `jenv init` unsets `JAVA_HOME` and lets its export hook re-set it — register your JDK with jenv if you rely on it.
- Values containing unusual characters (multiline, etc.) may not survive the `env`/ruby round-trip cleanly.

## Sourcing order

A zsh login shell sources, in order: `/etc/zprofile`, `~/.zprofile`, then `~/.zlogin`. The `env` capture reflects whatever those files (plus `/etc/paths` via `path_helper`) produce. System-wide additions belong in `/etc/paths.d` or `/etc/manpaths.d`.
