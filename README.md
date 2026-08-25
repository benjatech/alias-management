# am — Alias Manager

[![CI](https://github.com/mauro-baptista/alias-management/actions/workflows/ci.yml/badge.svg)](https://github.com/mauro-baptista/alias-management/actions/workflows/ci.yml)

`am` is a small command-line tool (a single Rust binary) that manages your
personal bash aliases in one dedicated file, `~/.alias-management`. It works
on **Linux and macOS** with bash. It never touches aliases defined anywhere
else — the only other file it writes is `~/.bash_profile`, once, during
initial setup.

```
$ am list
┌─────────┬───────┬──────────────────────┐
│ type    ┆ alias ┆ action               │
╞═════════╪═══════╪══════════════════════╡
│ command ┆ gp    ┆ git pull             │
│ folder  ┆ p     ┆ cd ~/folder/personal │
│ ssh     ┆ srv   ┆ ssh forge@127.0.0.1  │
└─────────┴───────┴──────────────────────┘
```

It manages three kinds of aliases:

| type | what it does | example |
|---|---|---|
| `command` | run any shell command | `gp` → `git pull` |
| `folder` | jump to a directory | `p` → `cd ~/folder/personal` |
| `ssh` | connect to a server | `srv` → `ssh forge@127.0.0.1` |

## Install

### Option 1 — Homebrew (macOS and Linux)

```bash
brew install benjatech/alias-management/am
```

That taps and installs in one step. The two-step form works too:

```bash
brew tap benjatech/alias-management
brew install am
```

`brew upgrade am` later picks up new releases. The formula installs the
prebuilt binary for your platform, so there is nothing to compile. Homebrew
downloads with curl, which never sets the macOS quarantine flag, so
Gatekeeper stays out of the way.

### Option 2 — download the `am` file and put it in your bin folder

`am` is a single self-contained executable — no runtime, no libraries, no
config. Grab the file for your platform from the
[Releases page](https://github.com/mauro-baptista/alias-management/releases)
— `am-linux-x86_64` (fully static), `am-macos-arm64` (Apple Silicon) or
`am-macos-x86_64` (Intel), with a `SHA256SUMS` file to verify — rename it
to `am`, and put it in a folder on your `PATH`. The easiest way is to let
it install itself:

```bash
chmod +x am        # make the downloaded file executable
./am install       # copies it to /usr/local/bin (or ~/.local/bin without root)
```

`sudo ./am install` forces the system-wide folder, and `./am install ~/bin`
installs into a folder of your choice — am tells you if that folder is not
on your PATH and exactly which line to add. Doing it by hand works just as
well:

```bash
# system-wide (needs sudo)
sudo cp am /usr/local/bin/ && sudo chmod +x /usr/local/bin/am

# or user-only (make sure the folder is on your PATH)
mkdir -p ~/.local/bin && cp am ~/.local/bin/ && chmod +x ~/.local/bin/am
```

On macOS the release also carries a signed, notarized `.pkg` installer
(`am-macos-arm64.pkg`) that puts `am` in `/usr/local/bin` with no Gatekeeper
friction at all — double-click it, or `sudo installer -pkg am-macos-arm64.pkg
-target /`.

The binary must match your OS and CPU: use a Linux build on Linux and a
macOS build on a Mac (and mind x86_64 vs arm64). On macOS, if Gatekeeper
blocks a downloaded binary, clear the quarantine flag with
`xattr -d com.apple.quarantine /usr/local/bin/am`.

### Option 3 — build from source

With Rust 1.85+ installed:

```bash
cargo build --release                    # produces target/release/am
./target/release/am install              # copies it onto your PATH
# or, if ~/.cargo/bin is on your PATH:
cargo install --path .
```

### First run

`am install` only places the binary. Run `am` once afterwards — that first
run sets everything up:

1. Creates `~/.alias-management` if it does not exist.
2. Adds the shell-integration block (see below) to `~/.bash_profile`,
   creating the file if needed — exactly once, never duplicated.

Then restart your terminal or run `source ~/.bash_profile`.

## How to use

### See your aliases

```bash
am list                # everything
am list -c             # only commands (-f folders, -s ssh; combine freely)
am list --search git   # aliases with 'git' in the name or in the action
```

Prints the table shown above: `type | alias | action`, grouped by type
(commands, then folders, then ssh), alphabetical within each group.
`--search` is case-insensitive and looks in the alias name *and* the
action, so it matches folder paths, command lines and ssh targets alike.
Running bare `am` shows the help.

### Create an alias

```bash
am new
```

Everything is asked interactively:

1. **Type** — pick `command`, `folder` or `ssh`.
2. **Alias name** — refused if the name is already taken, either by am or
   by anything else on your system (binaries, builtins, functions, other
   aliases), so an existing command is never shadowed.
3. **The details**, depending on the type:
   - *command*: the command to run, e.g. `git pull`.
   - *folder*: the folder path — defaults to the directory you are
     standing in, so `cd` into the folder first and just press Enter.
   - *ssh*: the user and the host. User `forge` and host `127.0.0.1`
     become the command `ssh forge@127.0.0.1`.

The alias is appended to `~/.alias-management`, and `am` reminds you to run
`source ~/.alias-management` to use it in the shell you are standing in. New
shells pick it up on their own.

**In a hurry?** Give any part up front with `-f` (folder), `-c` (command)
or `-s` (ssh) — am only asks for what is missing:

```bash
am new -f                        # folder alias for the current directory, asks the name
am new -f here                   # folder alias 'here' for the current directory
am new -c                        # command alias, asks name and command
am new -c gp                     # command alias 'gp', asks the command
am new -c gp 'git pull'          # command alias, no prompts
am new -s                        # ssh alias, asks name, user and host
am new -s srv                    # ssh alias 'srv', asks user and host
am new -s srv forge              # ssh alias 'srv', user 'forge', asks the host
am new -s srv forge 127.0.0.1    # ssh alias, no prompts
am new -s srv forge@127.0.0.1    # same thing, user@host form
```

Forms with nothing left to ask (like `am new -c gp 'git pull'`) also work
without a terminal, so they are script-friendly.

### Delete an alias

```bash
am delete          # pick from a list
am delete gp       # delete by name
```

Either way, `am` asks first — **No is the default**, so a stray Enter never
deletes anything:

```
? Do you want to delete command 'gp'? (y/n) › no
```

For scripts and other non-interactive use, skip the prompt with
`am delete gp --yes` (or `-y`). Deleting removes only that alias's line
from `~/.alias-management`; everything else in the file stays untouched.

### Install or update the binary

```bash
am install         # copy this binary to /usr/local/bin (or ~/.local/bin)
sudo am install    # force the system-wide folder
am install ~/bin   # or pick any folder
```

Handy after building from source or downloading a newer version: the
running `am` copies itself into place atomically. `am install` never
touches your dotfiles (safe under sudo) — run any other am command once to
set up the shell integration.

### Help

```bash
am --help          # bare `am` shows the same help
am list --help
am new --help
am delete --help
am install --help
```

## How the shell integration works

Setup installs this block into `~/.bash_profile`:

```bash
# >>> alias-management (am) >>>
# Added by `am` (alias-management). Do not edit this block by hand.
# Loads the managed aliases when the shell starts.
[ -f "$HOME/.alias-management" ] && source "$HOME/.alias-management"
# <<< alias-management (am) <<<
```

That is the whole integration: it sources your aliases when the shell starts,
so every new shell has them.

A child process cannot change the shell that launched it, so the `am` binary
cannot make a brand-new alias appear in the terminal you are already in — only
that shell can load it. After `am new`, run:

```bash
source ~/.alias-management
```

`am new` prints this reminder itself. `am` defines no shell function, so
`which am` reports the binary on your `PATH` and nothing shadows it.

If `am` has to create `~/.bash_profile` from scratch, it also adds a line
sourcing `~/.profile` first: bash reads only the first of `~/.bash_profile`
/ `~/.bash_login` / `~/.profile` for login shells, and without that line a
brand-new `~/.bash_profile` would silently disable an existing `~/.profile`
(and any PATH setup living there).

## Storage format

Plain bash, one alias per line, with a required trailing comment marking
the type:

```bash
alias gp="git pull" #command
alias p="cd ~/folder/personal" #folder
alias srv="ssh forge@127.0.0.1" #ssh
```

Anything else in the file — comments, blank lines, hand-written bash,
alias lines without a type marker — is ignored and preserved verbatim.
Because it is just bash, you can edit the file by hand too; `am` picks the
changes up on the next run.

## Linux and macOS notes

- **macOS**: Terminal.app and iTerm2 start *login* shells, so
  `~/.bash_profile` (and with it your aliases) loads automatically. The
  default shell on modern macOS is zsh; `am` targets bash — switch with
  `chsh -s /bin/bash`, or start `bash` inside your session.
- **Linux**: many desktop terminals start *non-login* shells, which skip
  `~/.bash_profile`. If your aliases only show up after `bash -l`, add
  `source ~/.bash_profile` to your `~/.bashrc` (am deliberately never
  edits `~/.bashrc` itself).

## Development, CI and releases

Two GitHub Actions workflows live in `.github/workflows/`:

- **CI** (`ci.yml`) runs on every pull request and on pushes to `main`:
  `cargo fmt --check`, `cargo clippy -D warnings`, `cargo test` and a
  release build, on both Ubuntu and macOS.
- **Release** (`release.yml`) builds the binaries for all three platforms
  (the Linux one statically against musl, so it runs on any x86_64 distro).
  It also runs on every pull request, so a change that breaks a release
  build is caught in review — those runs stop after the build and attach
  the binaries to the run as downloadable artifacts. Pushing a `v*` tag
  additionally generates `SHA256SUMS` and publishes everything as a GitHub
  release:

  ```bash
  git tag v0.0.1
  git push origin v0.0.1   # pushing the tag is what starts the release
  ```

  The tag must match the `version` in `Cargo.toml` — the workflow checks
  this first and stops with a clear message otherwise, so a release can
  never ship binaries whose `am --version` disagrees with the release
  name. To release a new version: bump `Cargo.toml`, commit, then tag with
  the matching `v` prefix. Tags with a suffix (`v0.1.0-rc.1`) are
  published as prereleases, and re-pushing an existing tag refreshes that
  release's files instead of failing.

Each platform ships twice: the bare binary (`am-macos-arm64`) for a direct
download, and a `.tar.gz` of the same file for Homebrew. `SHA256SUMS` covers
both.

### Signing the macOS builds

macOS binaries are signed with a Developer ID certificate and notarized when
these repository secrets exist. Without them the workflow still succeeds and
publishes unsigned binaries, so forks and pull requests are unaffected.

| Secret | What it is |
|---|---|
| `MACOS_CERT_P12` | Base64 of your *Developer ID Application* certificate and private key, exported from Keychain Access as `.p12` |
| `MACOS_CERT_PASSWORD` | The password you set on that `.p12` |
| `AC_API_KEY_P8` | Base64 of an App Store Connect API key (`AuthKey_XXX.p8`) |
| `AC_API_KEY_ID` | That key's ID |
| `AC_API_ISSUER_ID` | The issuer ID from App Store Connect |
| `MACOS_INSTALLER_CERT_P12` | Base64 of your *Developer ID Installer* certificate, if it is not already inside `MACOS_CERT_P12` |
| `MACOS_INSTALLER_CERT_PASSWORD` | The password on that `.p12` |

Base64-encode the two files with `base64 -i cert.p12 | pbcopy`. The signing
identity is looked up in the keychain automatically; set the optional
`MACOS_SIGNING_IDENTITY` secret to pin a specific one.

This matters for people who download from the Releases page: a browser tags
the file with `com.apple.quarantine`, and Gatekeeper refuses to run an
unsigned quarantined binary. It does **not** matter for Homebrew, which
downloads with curl and never sets that flag.

Signing runs on every build, so a bad certificate shows up in review.
Notarization calls out to Apple and takes minutes, so it runs on tags only.

Two things get notarized, because they are distributed separately:

- **The binary itself**, submitted as a zip. A lone executable has nowhere
  to keep a notarization ticket, so it cannot be stapled — Gatekeeper
  checks with Apple online the first time it runs, which needs a network
  connection.
- **A `.pkg` installer** (`am-macos-arm64.pkg`), built with `pkgbuild` and
  signed with a *Developer ID Installer* certificate, which is a different
  certificate from the one that signs the binary. A `.pkg` can carry its
  ticket, so this one is stapled and verifies with no network at all. It
  installs `am` into `/usr/local/bin`, and files placed by an installer are
  never quarantined.

The `.pkg` is built whenever a Developer ID Installer certificate is in the
keychain and skipped with a notice when it is not, so the rest of the
release is unaffected if you only have the Application certificate.

### The Homebrew tap

Tagging also regenerates the formula in the tap repository, pointing it at
the new tarballs and their checksums. It needs one more secret:

| Secret | What it is |
|---|---|
| `HOMEBREW_TAP_TOKEN` | A token with `contents: write` on the tap repository |

The tap defaults to the `benjatech/homebrew-alias-management` repository.
Homebrew requires the `homebrew-` prefix and drops it in the tap name, which
is why that repository is tapped as `benjatech/alias-management`. It has to be
a separate repository from this one; set the `HOMEBREW_TAP_REPO` repository
*variable* to point somewhere else, for example a shared `homebrew-tap` if you
later publish more than one tool.

Create it with a `Formula/` directory before the first tagged release — the
job is skipped entirely while `HOMEBREW_TAP_TOKEN` is unset.

## Behavior notes and limitations

- Deleting an alias cannot un-define it in shells where it is already
  loaded — sourcing only adds or overwrites definitions. `am delete`
  prints a reminder; run `unalias NAME` in open shells or start a new one.
- Writes are atomic (temp file + rename), so a crash can never truncate
  `~/.alias-management` or `~/.bash_profile`.
- Actions containing **both** single and double quotes cannot be stored as
  a plain bash alias line and are rejected with an explanation. Folder
  paths with spaces are stored quoted; paths containing quotes, `$`,
  backticks, or backslashes are rejected. SSH users and hosts accept the
  usual safe characters (letters, digits, `.`, `_`, `-`, plus `:` for
  IPv6 hosts).
