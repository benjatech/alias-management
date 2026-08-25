# am — Alias Manager

[![CI](https://github.com/benjatech/alias-management/actions/workflows/ci.yml/badge.svg)](https://github.com/benjatech/alias-management/actions/workflows/ci.yml)

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
| `function` | a command taking arguments | `gc "fix"` → `git commit -m "fix"` |
| `folder` | jump to a directory | `p` → `cd ~/folder/personal` |
| `ssh` | connect to a server | `srv` → `ssh forge@127.0.0.1` |

## Install

Pick one route below, then do the [one-time setup](#after-installing-do-this-once)
— it is required whichever route you take. `am` is a single self-contained
executable: no runtime, no libraries, nothing to configure.

### macOS

**Homebrew** — the least work, and it handles upgrades:

```bash
brew install benjatech/alias-management/am
```

Three segments: owner, tap, formula. `brew install benjatech/alias-management`
fails, because that is the tap name with no formula on the end. The two-step
form does the same thing:

```bash
brew tap benjatech/alias-management
brew install am
```

`brew upgrade am` picks up later releases.

**Installer package** — download `am-macos-arm64.pkg` (Apple Silicon) or
`am-macos-x86_64.pkg` (Intel) from the
[latest release](https://github.com/benjatech/alias-management/releases/latest)
and double-click it, or:

```bash
sudo installer -pkg am-macos-arm64.pkg -target /
```

It installs `am` into `/usr/local/bin`. The package is signed and notarized,
so Gatekeeper lets it through without complaint.

**Plain binary**:

```bash
curl -fsSL -o am https://github.com/benjatech/alias-management/releases/latest/download/am-macos-arm64
chmod +x am
./am install        # copies it to /usr/local/bin, or ~/.local/bin without root
```

Not sure which file? `uname -m` prints `arm64` on Apple Silicon and `x86_64`
on Intel.

Downloading with `curl` also sidesteps macOS quarantine entirely. A browser
tags downloads with `com.apple.quarantine`; the binaries are notarized, so
Gatekeeper should clear them anyway, but if one is ever refused,
`xattr -c am` removes the tag.

### Linux

**Homebrew**, if you use it:

```bash
brew install benjatech/alias-management/am
```

**Plain binary** — statically linked against musl, so it runs on any x86_64
distribution regardless of glibc version:

```bash
curl -fsSL -o am https://github.com/benjatech/alias-management/releases/latest/download/am-linux-x86_64
chmod +x am
./am install
```

### Both platforms

`sudo ./am install` forces the system-wide folder, and `./am install ~/bin`
installs into a folder of your choice — `am` tells you if that folder is not
on your `PATH`, and exactly which line to add. By hand works just as well:

```bash
sudo cp am /usr/local/bin/ && sudo chmod +x /usr/local/bin/am    # system-wide
mkdir -p ~/.local/bin && cp am ~/.local/bin/                     # user-only
```

Every release ships a `SHA256SUMS` file covering every asset. To check one
download, pick its line out and pipe that in — the file must still have the
name it was released under:

```bash
base=https://github.com/benjatech/alias-management/releases/latest/download
curl -fsSLO "$base/am-macos-arm64"      # the asset, under its own name
curl -fsSLO "$base/SHA256SUMS"

grep " am-macos-arm64$" SHA256SUMS | shasum -a 256 -c -   # macOS
grep " am-linux-x86_64$" SHA256SUMS | sha256sum -c -      # Linux
```

Expect `am-macos-arm64: OK`. Rename it to `am` afterwards.

### Build from source

With Rust 1.85+ installed:

```bash
cargo build --release                    # produces target/release/am
./target/release/am install              # copies it onto your PATH
# or, if ~/.cargo/bin is on your PATH:
cargo install --path .
```

### After installing, do this once

Installing only places the binary. Run any `am` command once to set up the
rest:

```bash
am list                  # any command will do
source ~/.bash_profile   # or just open a new terminal
```

That first run:

1. Creates `~/.alias-management` if it does not exist.
2. Adds the shell-integration block (described below) to `~/.bash_profile`,
   creating that file if needed — exactly once, never duplicated.

Without it `am` still runs, but your aliases are never loaded into a shell.

### When *not* to run `am install`

`am install` copies the running binary onto your `PATH`. That is the right
move for a downloaded binary or one you just built, and the wrong move when
something else already manages the file:

| Installed with | Run `am install`? |
|---|---|
| Homebrew | **No** — brew already put it on your `PATH` |
| `.pkg` installer | **No** — it installed to `/usr/local/bin` |
| Downloaded binary | Yes |
| `cargo build` | Yes |
| `cargo install --path .` | No — cargo puts it in `~/.cargo/bin` |

Running it anyway under Homebrew on Apple Silicon leaves a second copy in
`/usr/local/bin` that brew does not manage and `brew upgrade` will not
update. If you have already done that, `rm /usr/local/bin/am` removes the
stray copy; the Homebrew one is untouched.

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

### Commands that take arguments

An alias cannot see the arguments it is called with — bash substitutes the
text and appends whatever you typed, so `$1` inside one is never your
argument. Write `$1`, `$2`, ... anyway and `am` notices, storing a shell
function instead:

```bash
$ am new -c gc 'git commit -a -m "$1"'
Saved: gc() { [ $# -ge 1 ] || { echo "am: gc needs 1 argument" >&2; return 2; }; git commit -a -m "$1"; } #function
Saved as a shell function, because it takes 1 argument — an alias cannot.
```

```bash
$ gc "a real message"      # [master 47d70ef] a real message
$ gc                       # am: gc needs 1 argument
```

The check in front of the body is why the arguments have to be numbered.
`$*` and `$@` swallow whatever they are handed, so a forgotten argument would
quietly become an empty string; numbered parameters can be counted, and `am`
refuses `$*` and `$@` for that reason. They must also start at `$1` and leave
no gaps — `$2` with no `$1` is only ever a mistake:

```bash
$ am new -c bad 'echo "$2"'
error: '$2' is used but '$1' is not: numbered arguments must start at $1 with no gaps
```

`am list` shows these as type `function` and prints the command you typed
rather than the generated check. `--function` filters to just them. Deleting
one reminds you to `unset -f` rather than `unalias`, since that is what
removes a function from an open shell.

Two smaller differences from an alias: the body may contain both `'` and `"`,
because nothing wraps it, and it may not contain `#`, which would comment out
the rest of the line.

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
gc() { [ $# -ge 1 ] || { echo "am: gc needs 1 argument" >&2; return 2; }; git commit -a -m "$1"; } #function
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

[docs/releasing.md](docs/releasing.md) walks through obtaining each one. The
signing identity is looked up in the keychain automatically; set the optional
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
