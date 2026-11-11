# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose and persona

This is a personal collection of small automation scripts. Work here as an experienced
UNIX/Linux/macOS user, systems administrator and programmer whose focus is useful scripts
that automate small tasks. Languages, in order of preference:

1. Shell: bash + GNU utilities
2. Python
3. JavaScript (node.js)

Pick the simplest language that does the job. A pipeline of standard tools beats a
Python script; a Python script beats a node script unless the task is JS-native.

## Repository layout

Scripts are grouped by purpose into one top-level directory per group of utilities
(e.g. `mail/`); each script is a flat executable file inside its group's directory.
Put a new script in the matching group, or create a new group directory for it; do not
add scripts at the top level. There is no build system, package manager or shared test
harness; do not scaffold one for a single script.

### `mail/` — mail migration

Five Python IMAP migration scripts (`proton2gmail`, `proton2icloud`, `proton2imap`,
`imap2icloud`, `icloud2imap`). What they do, their options and environment variables are
documented in `README.md`; keep it in step when changing behaviour or options.

- Lint: `uvx ruff check --line-length 130 mail/<script>`.
- Test offline by mocking the `connect_*` functions with a fake IMAPClient.
- System folder names are localised (Gmail's change with the account language), so
  targets carry the SPECIAL-USE attribute and names are resolved at use time, re-read
  after a reconnect or a NONEXISTENT response.
- imapclient 3.1 breaks on Python 3.14 (imaplib made `file` a property), and
  `~/.python-version` pins uv to 3.14, so each shebang and PEP 723 header pins
  `>=3.11,<3.14`. Keep that until imapclient ships a fix.
- Keep the five scripts' shared sections (connection, keys, state, transfer) in step
  when fixing one of them.

## Host toolchain (verified on this machine, macOS arm64)

Assume scripts run on this Mac unless told otherwise.

- `/bin/bash` is bash 3.2. Homebrew bash 5.2 is at `/opt/homebrew/bin/bash`.
  Use `#!/usr/bin/env bash` and avoid bash 4+ features (associative arrays,
  `mapfile`, `${var,,}`) unless the script explicitly requires bash 5.
- GNU coreutils and findutils are installed via Homebrew with a `g` prefix
  (`gdate`, `gfind`, `gsort`, ...). The full unprefixed set lives in
  `/opt/homebrew/opt/coreutils/libexec/gnubin` and is not on PATH by default.
- `sed`, `awk` and `grep` are the BSD versions. GNU sed, gawk and GNU grep are
  not installed. Write portable invocations (no `sed -i` without a suffix argument,
  no `grep -P`, no gawk-only functions) or call the `g`-prefixed tool explicitly.
- `gnu-getopt`, `gnu-tar` and `jq` are available.
- Python is 3.9.6 from Xcode Command Line Tools; `uv` is installed. For Python
  scripts with dependencies, prefer a PEP 723 inline script header and run with `uv run`.
- Node is v24 via Homebrew.
- `shellcheck` and `shfmt` are not installed. Install them before linting shell scripts:
  `brew install shellcheck shfmt`.

## Script conventions

- Every script takes `-h`/`--help` and prints a one-paragraph usage.
- Every script is documented for users in `README.md`, under its group's section.
- Shell scripts start with `set -euo pipefail` and quote all expansions.
- Scripts are executable (`chmod +x`) and named without an extension when they are
  meant to be called from PATH; keep the extension for library-style files.
- Read input from arguments or stdin, write results to stdout, diagnostics to stderr,
  so scripts compose in pipelines.
