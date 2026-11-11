# utils

A personal collection of small scripts that automate everyday tasks. Scripts are grouped
by purpose, one directory per group. Every script prints its usage with `-h`/`--help`.

## `mail/`: mail migration

Five Python scripts that copy a mailbox from one provider to another over IMAP without
creating duplicates. They need [uv](https://docs.astral.sh/uv/), which fetches Python
(3.11 to 3.13) and the `imapclient` dependency on first run:

```sh
mail/proton2gmail --help
```

| Script          | From                       | To                |
|-----------------|-----------------------------|-----------------|
| `proton2gmail`  | Proton Mail (Proton Bridge) | Gmail           |
| `proton2icloud` | Proton Mail (Proton Bridge) | iCloud Mail     |
| `proton2imap`   | Proton Mail (Proton Bridge) | any IMAP server |
| `imap2icloud`   | any IMAP server             | iCloud Mail     |
| `icloud2imap`   | iCloud Mail                 | any IMAP server |

### How they work

- Every source folder is copied to a folder (or Gmail label) of the same nested name,
  reused when it exists and created otherwise. System folders (Inbox, Sent, Drafts,
  Archive, Junk, Trash) map to the target's own, whatever language the account uses.
- Messages are matched on Message-ID, so a message already on the target is never copied
  again. Gmail adds the missing label instead; targets without labels (iCloud, plain
  IMAP) get each message exactly once, in the first folder it is found in; there, Proton
  labels are processed after Proton folders, so they only receive mail found nowhere else.
- The source is only read, never changed.
- Progress is saved in a state file under `~/.cache/<script>/`. When a run is
  interrupted or throttled, simply run it again and it resumes where it stopped.

### Common options

| Option            | Effect                                                    |
|-------------------|-----------------------------------------------------------|
| `--dry-run`       | show the folder mapping and counts, change nothing        |
| `--folder NAME`   | only migrate this source folder (repeatable)              |
| `--include-trash` | also migrate Trash                                        |
| `--include-spam`  | also migrate Spam/Junk                                    |
| `--state FILE`    | use a different state file                                |
| `-v`, `--verbose` | log every message                                         |

Exit status: 0 ok, 1 usage or configuration error, 2 connection failure, 3 throttled by
the target (re-run later).

### Credentials

Credentials and connection details are read from environment variables.

**Proton Mail** (source for the `proton2*` scripts) is reached through the local
[Proton Bridge](https://proton.me/mail/bridge):

| Variable               | Meaning                                  | Default     |
|------------------------|------------------------------------------|-------------|
| `PROTON_IMAP_USER`     | Proton address                           | required    |
| `PROTON_IMAP_PASS`     | Bridge password (not your Proton password) | required  |
| `PROTON_IMAP_HOST`     | Bridge host                              | `127.0.0.1` |
| `PROTON_IMAP_PORT`     | Bridge IMAP port                         | `1143`      |
| `PROTON_IMAP_SECURITY` | `auto`, `ssl`, `starttls` or `none`      | `auto`      |
| `PROTON_IMAP_NOVERIFY` | `1` accepts Bridge's self-signed certificate | `1`     |

**Gmail** (`proton2gmail`): `GMAIL_IMAP_USER` and `GMAIL_IMAP_PASS`, an
[app password](https://myaccount.google.com/apppasswords).

**iCloud Mail** (`proton2icloud`, `imap2icloud`, `icloud2imap`): `ICLOUD_IMAP_USER`, your
Apple ID address, and `ICLOUD_IMAP_PASS`, an app-specific password created at
[appleid.apple.com](https://appleid.apple.com).

**Any IMAP server** as target (`proton2imap`, `icloud2imap`) uses the `IMAP_*` variables;
as source (`imap2icloud`) the same set prefixed with `SOURCE_`:

| Variable        | Meaning                                    | Default                      |
|-----------------|--------------------------------------------|------------------------------|
| `IMAP_HOST`     | server                                     | required                     |
| `IMAP_USER`     | login                                      | required                     |
| `IMAP_PASS`     | password                                   | required                     |
| `IMAP_PORT`     | port                                       | `993` for ssl, `143` otherwise |
| `IMAP_SECURITY` | `ssl`, `starttls` or `none`                | `ssl`                        |
| `IMAP_NOVERIFY` | `1` skips certificate verification         | `0`                          |

### Script notes

- **`proton2imap`, `icloud2imap`**: the server's hierarchy delimiter and namespace prefix
  (such as `INBOX.`) are honoured, and a missing system folder is created. Many servers
  cannot store a `.` in a folder name, so dots are rewritten: dropped before a space or at
  the end (`G. Blake` becomes `G Blake`), otherwise replaced by `_` (`One.com` becomes
  `One_com`). Pass `--keep-dots` for a server that accepts them.
- **`imap2icloud`**: Gmail's Starred and Important folders are skipped. All Mail is
  processed last into iCloud Archive, so it only adds mail found in no other folder, such
  as archived Gmail mail; `--skip-all-mail` leaves it out. State is kept per source
  account in `~/.cache/imap2icloud/USER@HOST.json`.
- **`icloud2imap`**: state is kept per account pair in
  `~/.cache/icloud2imap/ICLOUD_USER--USER@HOST.json`.

### Example

```sh
export ICLOUD_IMAP_USER=me@icloud.com ICLOUD_IMAP_PASS=abcd-efgh-ijkl-mnop
export IMAP_HOST=imap.example.com IMAP_USER=me@example.com IMAP_PASS=secret
mail/icloud2imap --dry-run   # check the folder mapping first
mail/icloud2imap
```

## License

See [LICENSE](LICENSE).
