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

- `proton2gmail` — Python, migrates Proton Mail (via Proton Bridge IMAP) to Gmail
  IMAP, deduplicating on Message-ID and resuming from `~/.cache/proton2gmail/state.json`.
  Run: `mail/proton2gmail --help` (shebang uses `uv run --script`; credentials via env
  vars listed in the help). Lint: `uvx ruff check --line-length 130 mail/proton2gmail`.
  Test offline by mocking `connect_proton`/`connect_gmail` with a fake IMAPClient.
  Gmail system folder names are localised and change with the account language, so
  targets carry the SPECIAL-USE attribute and names are resolved at use time, re-read
  after a reconnect or a NONEXISTENT response.
  imapclient 3.1 breaks on Python 3.14 (imaplib made `file` a property), and
  `~/.python-version` pins uv to 3.14, so the script's shebang and PEP 723 header
  both pin `>=3.11,<3.14`. Keep that until imapclient ships a fix.
- `proton2icloud` — Python, same shape as `proton2gmail` but for iCloud Mail
  (`imap.mail.me.com`, app-specific password via `ICLOUD_IMAP_USER`/`ICLOUD_IMAP_PASS`).
  iCloud has no labels or All Mail, so the dedup index spans every iCloud folder, each
  message is copied once, and Proton labels are processed after folders. System folder
  names are re-resolved by SPECIAL-USE attribute after every reconnect. Same run, lint
  and test approach as `proton2gmail`; state in `~/.cache/proton2icloud/state.json`.
- `proton2imap` — Python, `proton2icloud` generalised to any IMAP server. Target via
  `IMAP_HOST`/`IMAP_PORT`/`IMAP_USER`/`IMAP_PASS`/`IMAP_SECURITY` (ssl, starttls, none)
  and `IMAP_NOVERIFY`. Honours the server's hierarchy delimiter and personal namespace
  prefix (NAMESPACE, or inferred `INBOX.`), replaces the delimiter inside folder names,
  finds system folders by SPECIAL-USE with a table of common names as fallback, and
  creates a missing one under a default name. Dots in folder names are rewritten by
  default because many servers cannot store them: dropped before a space or at the end
  (`G. Blake` to `G Blake`), otherwise replaced by `_` (`One.com` to `One_com`);
  `--keep-dots` opts out except where `.` is the delimiter. State in
  `~/.cache/proton2imap/state.json`.
- `imap2icloud` — Python, `proton2icloud` with any IMAP server as the source. Source via
  `SOURCE_IMAP_HOST`/`SOURCE_IMAP_PORT`/`SOURCE_IMAP_USER`/`SOURCE_IMAP_PASS`/
  `SOURCE_IMAP_SECURITY` (ssl, starttls, none) and `SOURCE_IMAP_NOVERIFY`. Strips the
  source's namespace prefix and translates its delimiter; source system folders found by
  SPECIAL-USE or a common-name table (incl. Gmail `[Gmail]/...`). Starred/Important are
  skipped; All Mail goes to iCloud Archive, processed last so it only gets mail found
  nowhere else (`--skip-all-mail` opts out). State is per source account in
  `~/.cache/imap2icloud/USER@HOST.json`.
  Keep the four scripts' shared sections (connection, keys, state, transfer) in step
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
- Shell scripts start with `set -euo pipefail` and quote all expansions.
- Scripts are executable (`chmod +x`) and named without an extension when they are
  meant to be called from PATH; keep the extension for library-style files.
- Read input from arguments or stdin, write results to stdout, diagnostics to stderr,
  so scripts compose in pipelines.
