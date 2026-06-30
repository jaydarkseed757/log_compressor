# compress_logs.sh

Compress uncompressed log files while keeping each log family consistent with the compression format it already uses. A `gzip` family stays `gzip`, a `bzip2` family stays `bzip2`, a `7z` family stays `7z`, and brand-new families fall back to a configurable default. Parallel compressors (`pigz`, `lbzip2`, threaded `xz`) are used automatically when installed, and every run ends with a detailed report.

## Why

Tools like `logrotate` leave a mix of compressed and uncompressed rotated logs:

```
messages          # active log, still being written
messages.1        # rotated, not yet compressed
messages.2        # rotated, not yet compressed
messages.3.gz     # rotated, already gzip
```

Blindly running `bzip2` over everything would produce a `messages.1.bz2` sitting next to `messages.3.gz` — a split-format family that is awkward to manage and search. This script instead reads the convention from the family's existing compressed members and matches it, so `messages.1` and `messages.2` become `messages.1.gz` and `messages.2.gz`.

## Requirements

- `bash` 4.2 or newer (RHEL 7/8/9 all qualify)
- `find` and GNU coreutils (`stat`, `sort`, `rm`, `date`, `chmod`, `chown`, `touch`)
- At least one compressor for the formats you use: `gzip`/`pigz`, `bzip2`/`lbzip2`, `xz`, and/or `7z`/`7za`/`7zr`
- `lsof` is optional; if present, open files are skipped, otherwise that check is skipped with a warning

On RHEL the parallel and 7z tools come from EPEL:

```
sudo dnf install pigz lbzip2 p7zip    # xz threading is built in (xz >= 5.2)
```

## Installation

```
cp compress_logs.sh /usr/local/bin/
chmod +x /usr/local/bin/compress_logs.sh
```

## Quick start

```
# Preview what would happen to /var/log (no changes made)
compress_logs.sh -n

# Compress /var/log, top level only
compress_logs.sh

# Recurse into subdirectories
compress_logs.sh -r

# Preview recursively with full per-file detail
compress_logs.sh -r -n -v
```

## How it works

The script runs in two passes over the target directory.

**Pass 1 — learn each family.** Every filename is split into a family key by stripping the rotation number and any compression extension. `messages`, `messages.1`, `messages.2`, and `messages.3.gz` all reduce to the family `messages`. The family key is qualified by directory, so `messages` under `/var/log` and `messages` under `/var/log/app/` are treated as separate families. For each family, the already-compressed members determine the convention (the format of `messages.3.gz` makes `messages` a gzip family).

**Pass 2 — compress.** Each eligible uncompressed file is compressed with its family's compressor — using the parallel variant if available — preserving the original file's mode, owner, and modification time. (For 7z families, see [7z support](#7z-support) below, since 7z is an archiver rather than an in-place stream compressor.)

A file is **skipped** when it is:

- already compressed (`.gz .bz2 .xz .7z .zst .lz4 .lzo .Z .zip .zz`)
- empty
- currently open by another process (detected with a single batched `lsof` call)
- the active, rotation-less member of a family that has other members (i.e. the live log) — override with `--include-active`
- part of a family whose format this host cannot produce (e.g. a `.zst` family with no `zstd` installed), in which case it is left alone rather than mixed
- already accompanied by an existing target file (no clobbering)

Families with no compressed members yet use the default compressor (`-c`, default `bzip2`).

## Compressor selection

| Format | Preferred (parallel) | Fallback |
|--------|----------------------|----------|
| gzip   | `pigz`               | `gzip`   |
| bzip2  | `lbzip2`             | `bzip2`  |
| xz     | `xz -T` (threaded)   | —        |
| 7z     | `7z` / `7za` / `7zr` (multithreaded by default) | — |

The startup banner reports which tools were actually selected, for example:

```
[INFO]  Compressors   : gz=pigz  bz2=lbzip2  xz=xz  7z=7z
```

A format whose tool is not installed shows as `none`; files in such a family are skipped rather than compressed into a different format.

Thread count is controlled with `-T/--threads`; `0` (the default) lets each parallel tool use all available cores (for 7z this maps to `-mmt=on`).

## 7z support

7z is supported as both a recognized family convention and a `-c` default, but it works differently from the stream compressors and the script accounts for that:

- **It is an archiver, not an in-place stream compressor.** `gzip`, `bzip2`, and `xz` each rewrite a single file in place. 7z instead creates a separate `.7z` container. The script runs 7z from inside the file's directory using the basename, so `messages.1` becomes an archive `messages.1.7z` whose single stored entry is `messages.1` — it extracts back to a plain file, not a nested directory tree.
- **Metadata is preserved explicitly.** A 7z archive is created with default permissions, so the script copies the original file's mode, owner, and modification time onto the `.7z` (via `chmod`/`chown`/`touch --reference`) before removing the original. The stream compressors get this for free; 7z needs the extra step.
- **The original is removed only after a verified-good archive.** The script does not use `7z -sdel`; it archives, confirms success, copies metadata, then deletes the source. If 7z fails, any partial archive is removed and the original is left untouched. 7z's non-fatal warning exit status is treated conservatively as a failure (roll back) rather than trusting a questionable archive.
- The detected binary is whichever of `7z`, `7zz`, `7za`, or `7zr` is found first.

Because 7z performs more work per file (extra metadata calls), the fast in-place path is still used for `gzip`/`bzip2`/`xz`; only 7z families incur the extra steps.

## Options

| Option | Description |
|--------|-------------|
| `-d DIR` | Directory to process (default: `/var/log`) |
| `-c COMPRESSOR` | Default compressor for families with no compressed members: `bzip2`, `gzip`, `xz`, or `7z` (default: `bzip2`) |
| `-T, --threads N` | Threads for parallel compressors; `0` = auto/all cores (default: `0`) |
| `-r` | Recurse into subdirectories |
| `--include-active` | Also compress active (rotation-less) logs that have siblings |
| `-n, --dry-run` | Show what would happen without changing anything |
| `-v` | Verbose output and per-file detail in the report |
| `-h, --help` | Show help and exit |

## The end-of-run report

Every run prints a structured report. Sections appear only when they have content.

- **Header** — host, directory, recurse mode, default compressor, thread setting, elapsed time. Dry runs are clearly labeled.
- **Families** — each family with its resolved convention and tool, member count, and how many were compressed this run. Mixed-format families show `mixed→<default>`.
- **Compressed** (or **Would compress** in a dry run) — per-file table with the tool used, before/after sizes, and percent saved. Dry runs show original sizes only.
- **Skipped (by reason)** — a count per skip reason; with `-v`, a per-file detail listing follows.
- **Failed** — any files whose compression errored (shown only if non-empty).
- **Totals** — files compressed/skipped/failed, original vs compressed size, total space reclaimed, and elapsed time.

Example (trimmed):

```
════════════════════════════════════════════════════════════════════════
  COMPRESSION REPORT
════════════════════════════════════════════════════════════════════════
  Host          prod-web-01
  Directory     /var/log
  Recurse       yes
  Default       bzip2 (.bz2)
  Threads       auto (all cores)
  Elapsed       3s

────────────────────────────────────────────────────────────────────────
  FAMILIES
────────────────────────────────────────────────────────────────────────
  Family                             Convention       Members       Done
  audit                              xz (xz)                2          1
  messages                           gzip (pigz)            4          2
  secure                             bzip2 (lbzip2)         2          1

────────────────────────────────────────────────────────────────────────
  TOTALS
────────────────────────────────────────────────────────────────────────
  Files compressed       4
  Files skipped          6
  Files failed           0
  Original size          1.2 MiB
  Compressed size        3.4 KiB
  Space reclaimed        1.2 MiB (99%)
  Elapsed                3s
════════════════════════════════════════════════════════════════════════
```

## Exit codes

| Code | Meaning |
|------|---------|
| `0` | All eligible files compressed successfully (or nothing to do) |
| `1` | One or more files failed to compress |
| `3` | Script, dependency, or argument error |

These are suitable for monitoring integrations (Nagios, Icinga, Zabbix, etc.) that interpret exit codes.

## Cron usage

```
# Compress /var/log daily at 3am, recursively
0 3 * * * /usr/local/bin/compress_logs.sh -r

# Default brand-new families to gzip, 4 threads
0 3 * * * /usr/local/bin/compress_logs.sh -r -c gzip -T 4
```

The script hardens `PATH` internally, so it behaves consistently under cron. The active live log of each rotated family is skipped automatically, and open files are skipped when `lsof` is available, so it is safe to run against a live `/var/log`.

## Notes and safety

- Stream formats (`gzip`/`bzip2`/`xz`) compress **in place**, so the original file's permissions, owner, and timestamp are preserved automatically — important for sensitive logs such as `secure` (often `0600 root`). This matches how `gzip` and `logrotate` behave. The `7z` archiver preserves the same metadata, copied onto the archive explicitly (see [7z support](#7z-support)).
- An existing target (e.g. `messages.1.gz` already present) causes the source to be skipped rather than overwritten.
- A family with two different existing compression formats is reported as mixed and compressed with the default, with a one-time warning.
- Run as `root` (or a user with read/write access to the log directory) since `/var/log` files are typically root-owned. Owner preservation for 7z archives requires the privilege to `chown`; without it, the archive keeps the running user's ownership and a non-fatal best-effort attempt is made.

## Examples

```
# Investigate before acting, recursively, with detail
compress_logs.sh -d /var/log -r -n -v

# Compress an application's log directory, gzip by default for new families
compress_logs.sh -d /opt/myapp/logs -r -c gzip

# Single-threaded run (e.g. to limit load on a busy box)
compress_logs.sh -r -T 1

# Default brand-new families to 7z archives
compress_logs.sh -d /opt/myapp/logs -r -c 7z
```
