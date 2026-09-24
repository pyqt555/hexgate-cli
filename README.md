# hexgate

`hexgate` is a command-line tool for installing, updating, verifying and
selectively extracting Riot game content (League of Legends, Valorant, and other
Riot products). It resolves official release manifests from public catalogs,
downloads only the chunks it needs from the Riot CDN, and maintains the game's
`Game.db` in Riot's own format. It is also mostly an Ai created wrapper around my hexgate lib, i didnt test it to throurougly, so treat with the appropriate caution

All work happens locally: `hexgate` talks only to the public Riot catalog,
`api.hexgate.lol` and CDN endpoints, and keeps its state in the platform
config/cache directories described below.

- Browse every published manifest from the RiotArchiveProject catalog or the
  low-latency `api.hexgate.lol` index.
- Pin an install to a specific build, then install or update it in place.
- Verify and repair an existing installation, or rebuild its `Game.db`.
- Extract selected game files into a normal folder without touching an install.
- Simulate a migration between two builds before committing to it.

This guide ships with the release bundle (`hexgate-<version>-<target>.zip` /
`.tar.gz` archives plus `SHA256SUMS`).

## Install

| Platform | Archive | Binary |
| --- | --- | --- |
| Windows x64 | `hexgate-<version>-x86_64-pc-windows-msvc.zip` | `hexgate.exe` |
| Linux x64 (static) | `hexgate-<version>-x86_64-unknown-linux-musl.tar.gz` | `hexgate` |
| Linux arm64 (static) | `hexgate-<version>-aarch64-unknown-linux-musl.tar.gz` | `hexgate` |
| macOS arm64 | `hexgate-<version>-aarch64-apple-darwin.tar.gz` | `hexgate` |
| macOS x64 | `hexgate-<version>-x86_64-apple-darwin.tar.gz` | `hexgate` |

**Windows**

```
Expand-Archive .\hexgate-<version>-x86_64-pc-windows-msvc.zip -DestinationPath "$env:LOCALAPPDATA\Programs\hexgate"
$env:Path = "$env:LOCALAPPDATA\Programs\hexgate;$env:Path"   # add permanently via System settings
hexgate --version
```

Windows may show a SmartScreen prompt because the binary is not code-signed;
choose "More info" and "Run anyway". The build needs the Microsoft Visual C++
2015-2022 Redistributable (`VCRUNTIME140.dll`); most systems already have it,
otherwise install it from Microsoft.

**Linux**

```sh
tar -xzf hexgate-<version>-x86_64-unknown-linux-musl.tar.gz
install -m 755 hexgate ~/.local/bin/hexgate     # or /usr/local/bin with sudo
hexgate --version
```

The Linux builds are static musl binaries: no glibc or other runtime
dependencies.

**macOS** (13 or newer)

```sh
tar -xzf hexgate-<version>-aarch64-apple-darwin.tar.gz
sudo install -m 755 hexgate /usr/local/bin/hexgate
hexgate --version
```

The macOS builds are unsigned, so Gatekeeper may block the first run. Clear the
quarantine flag once (`xattr -d com.apple.quarantine /usr/local/bin/hexgate`)
or allow it under System Settings -> Privacy & Security.

**Verify the download** before installing (run from the folder with the
archives):

```sh
sha256sum -c SHA256SUMS                 # Linux
shasum -a 256 -c SHA256SUMS             # macOS
(Get-FileHash .\hexgate-<version>-x86_64-pc-windows-msvc.zip -Algorithm SHA256).Hash
# compare with the matching line in SHA256SUMS
```

`hexgate --help` lists the command groups; `hexgate <group> --help` and
`hexgate <group> <command> --help` show every flag.

**Uninstall:** delete the binary. State lives in the config and cache paths
listed under [Configuration and state](#configuration-and-state);
`hexgate config path` and `hexgate manifest cache path` print them.

## Quick start

```
# 1. create the global config (cache directory, default product/source)
hexgate config init

# 2. fetch the Riot release catalog (about 6 MB, full replace)
hexgate manifest sync

# 3. see what is available
hexgate manifest list --product lol --limit 10

# 4. pin an install named "live" to the newest LoL build
#    (paths default to ./live and ./live.db; add --root/--db to point elsewhere)
hexgate manifest resolve --latest --source catalog --append --as live

# 5. download and install it - a full game download of tens of GiB
#    (prompts before starting; add --yes for non-interactive use)
hexgate install latest live
```

Later, keep it current:

```
hexgate install update live
```

Already have the game on disk? Jump to `install verify` / `install repair` /
`install db` in the command reference, or see *Register an install for a manual
path* under Common tasks.

## How it works

Two manifest **sources** are queried one at a time:

| Source | What it is | Coverage |
| --- | --- | --- |
| `catalog` (default) | RiotArchiveProject `catalog.json` | all Riot products and realms (including PBE) plus historical builds |
| `api` | `api.hexgate.lol` incremental index | League of Legends only; lower latency for very recent builds |

Sources return manifest **IDs**, not bytes. Any command that needs actual
manifest data fetches the RMAN blob from the product CDN
(`https://{product}.secure.dyn.riotcdn.net/...`) into the cache on first use.

The same build is often published as several **artifact types** (for example
`lol-game-client`, `lol-standalone-client-content` or `league-client`).
`manifest list` and `resolve` show the type and `--artifact` filters it
(substring, case-insensitive); `resolve` warns when the picked version exists
as other artifacts.

An **install** is a named entry in your configuration pointing at a root
directory, a `Game.db` and a pinned manifest ID. Multiple installs are normal:
every *other* install with a pinned manifest is offered as a local chunk source
while patching, so shared data is copied instead of downloaded. Patching
operations take an exclusive lock, so two `hexgate` processes cannot operate on
the same install at once.

Configuration is three layers, highest priority first:

1. command-line arguments,
2. `./hexgate.toml` in the working directory (or `--work-dir`),
3. the platform-global config file.

`--scope none` changes exist only for the current invocation and are labelled
`ephemeral` in output and `config list --all`.

## Global options

These apply to every command and may appear before or after the command name.

| Option | Meaning |
| --- | --- |
| `--work-dir <DIR>` | Directory used for the local config layer and relative default paths (default: current directory) |
| `--no-global` | Ignore the global config layer for reads |
| `--no-local` | Ignore the local `./hexgate.toml` layer for reads |
| `--isolated` | Ignore both config layers for reads |
| `--scope <SCOPE>` | Where mutations are written: `global`, `local` (default), `both` or `none` |
| `--cache-dir <DIR>` | Override the effective cache directory |
| `--offline` | Forbid all network access; use only cached manifests and source data |
| `--json` | Machine-readable output (see [Scripting](#scripting)) |
| `--no-color` | Disable ANSI colors |
| `-y, --yes` | Answer yes to confirmations; required on a non-interactive session for destructive commands |
| `-v, --verbose` | Add library logs (`info`/`debug`); `-vv` adds `trace` |
| `-q, --quiet` | Suppress progress and info output; warnings and errors are still shown |
| `-h, --help` / `-V, --version` | Standard help and version |

## Configuration and state

`config` manages the config files and install entries.

```
hexgate config init [--force]            # write defaults (global unless --scope)
hexgate config path [--all]              # print config file path(s)
hexgate config list [--all]              # effective install entries (--all shows layer)
hexgate config show <name>               # one entry
hexgate config set <name> <key> <value>  # create/update an entry key
hexgate config remove <name> [--yes]     # delete an entry
```

`config set` keys: `product`, `root-dir`, `db-path`, `languages`, `manifest`,
`version`, and the global `source` and `timestamp` settings. `manifest`
accepts a selector (literal ID, version prefix, `latest`, `@install`) and, when
the cached source knows it, stores the matching version too - so a hand-pinned
entry is not mistaken for a fresh install. An ID the source cache does not know
is refused unless you pass `--version SS.PP.MINOR` explicitly.

Typical layout:

| Platform | Global config | Default cache |
| --- | --- | --- |
| Windows | `%APPDATA%\hexgate\hexgate.toml` | `%LOCALAPPDATA%\hexgate\cache` |
| Linux | `~/.config/hexgate/hexgate.toml` | `~/.cache/hexgate` |
| macOS | `~/Library/Application Support/hexgate/hexgate.toml` | `~/Library/Caches/hexgate` |

A config file looks like this:

```toml
[settings]
cache_dir = "C:/hexgate/cache"
source = "catalog"                 # catalog | api
product = "lol"
languages = ["windows", "en_US"]
timestamp = "datetime"             # datetime | raw | relative

[[installs]]
name = "live"
product = "lol"
root_dir = "C:/Riot Games/League of Legends/Game"
db_path = "C:/Riot Games/League of Legends/Game.db"
manifest = "1D0BDEC9762D3CC3"      # 16-char hex manifest ID; all-zero = unpinned
languages = ["windows", "en_US"]

[installs.version]
season = 16
patch = 9
minor = 7721032

[[installs]]
name = "pbe"
product = "lol"
root_dir = "C:/Riot Games/League of Legends PBE/Game"
db_path = "C:/Riot Games/League of Legends PBE/Game.db"
manifest = "0B7343FB7841F598"
languages = ["windows", "en_US"]

[installs.version]
season = 16
patch = 9
minor = 7750779
```

`version` is a table and must stay last within each `[[installs]]` block. An
optional `version_text` string holds an opaque source version (for example
Teamfight Tactics `rls-18.4.0...`); when present it is displayed instead of the
numeric version, and the numeric `version` is a best-effort form for the
library - good enough for install/update bookkeeping:

```toml
[[installs]]
name = "tft"
product = "teamfighttactics"
root_dir = "C:/Riot Games/TFT/Game"
db_path = "C:/Riot Games/TFT/Game.db"
manifest = "054222AFD3A58261"
version_text = "rls-18.4.0.5537496.Set18FullPC.Shipping.live"

[installs.version]
season = 18
patch = 4
minor = 0
```

`timestamp` controls how human-readable timestamps are rendered: `datetime`
(default, `YYYY-MM-DD HH:MM:SS UTC`), `raw` (the source's original string when
present) or `relative` (`6 hours ago`). It applies to every human timestamp
(list, resolve, ...); `--json` output always carries the raw values.

The cache directory holds:

```
catalog.json                  raw RiotArchiveProject catalog
api-versions.db               api.hexgate.lol index (SQLite)
manifests/{ID}.manifest       raw RMAN blobs fetched from the CDN
```

Manage it with `hexgate manifest cache path | info | clear` (`clear` supports
`--manifests` and `--index`).

Because install entries merge by name from the bottom layer up, the same install
name can shadow a global entry in a project directory - handy for testing
against a second game copy.

## Manifest selectors

Commands that take `<SEL>` accept any of these:

| Selector | Meaning |
| --- | --- |
| `1D0BDEC9762D3CC3` | literal manifest ID, case-insensitive and zero-padding optional |
| `16.9.7721032`, `16.9`, `16` | version, exact or dotted prefix; the newest match wins |
| `latest` | newest match from the selected source and filters |
| `@live` | the current/pinned manifest of install `live` |

Version prefixes and `latest` need a source (the config default, or `--source`).
A literal ID passes through without touching the source.

## Commands

### `config`

See [Configuration and state](#configuration-and-state).

### `manifest`

```
hexgate manifest sync   [--source S] [--force]
hexgate manifest list   [--source S] [--product P] [filters] [--limit N] [--sort version|timestamp]
hexgate manifest resolve [--source S] [--product P] [filters] [--id HEX | --latest]
                        [--append [--as NAME]] [--root DIR] [--db FILE] [--languages LIST]
hexgate manifest fetch  <ID> [--product P] [--force]
hexgate manifest inspect <ID|path> [--product P]
hexgate manifest tree   <ID|path> [--product P] [filters] [--depth N] [--flat]
hexgate manifest cache  path | info | clear [--manifests] [--index]
```

- `sync` refreshes the selected source's cache. `catalog` always re-downloads
  the full file; `api` is incremental and `--force` resets its cursor.
- `list` browses without selecting. On `catalog`, omitting `--product` lists
  every product. The table shows id, product, version, artifact type,
  realms/servers, platform, date and size.
- `resolve` picks exactly one manifest. Without `--id`/`--latest` it opens an
  interactive picker (a TTY is required). When the same product and version
  exists as multiple artifact types, the chosen one is shown and a warning
  names the others (`--artifact` chooses explicitly). `--append` pins the
  result onto an install entry: `--as NAME` names it, or without `--as` your
  single existing install is updated. `--root`, `--db` and `--languages` are
  stored on the entry; paths default to `<work-dir>/<name>` and
  `<work-dir>/<name>.db`.
- `fetch` downloads the RMAN blob and prints its path.
- `inspect` reports sizes, tags, bundles, chunking parameters and file totals.
  The target may also be a local `.manifest` file (parsed as untrusted input;
  only inspect files you trust).
- `tree` renders the file tree, or a flat list with `--flat`; `--depth N` keeps
  paths of at most `N` segments (`--depth 1` is root files only, `DATA/x` is
  depth 2).
- A literal ID is looked up in the source cache so the correct product CDN is
  used (a Valorant ID fetches from the Valorant CDN); `--product` overrides
  when the ID is not in the cache.

Source filters (shared by `list`/`resolve`):

| Filter | Applies to |
| --- | --- |
| `--version <VERSION>` | exact or dotted prefix; numeric segments ignore leading zeros (`9.8` matches `09.08.00.x`) |
| `--realm <REALM>` / `--realms` | catalog |
| `--platform <PLATFORM>` / `--platforms` | both |
| `--artifact <TYPE>` / `--artifacts` | catalog |
| `--server <SERVER>` / `--servers` | api |
| `--since <DATE>` / `--until <DATE>` | both (`YYYY-MM-DD` or RFC3339) |
| `--min-size` / `--max-size` | both (suffixes: `KiB`, `MiB`, `GiB`) |
| `--limit <N>` / `--sort version\|timestamp` | both |

Realm, platform, artifact and server filters are case-insensitive **substring**
matches (`--realm tmnt` matches `LOLTMNT99`, `--platform win` matches
`Windows`); comma-separated values are ORed.

File filters (`--path-regex`, `--name-regex`, `--glob`, `--tags`,
`--untagged`, `--min-size`, `--max-size`) are shared by `manifest tree` and
`extract`. The glob matches the manifest-relative path, and `*` also matches
`/` (`DATA/*` covers every depth).

### `install`

```
hexgate install new    <name> --product <P> [--root DIR] [--db FILE] [--languages LIST]
hexgate install latest <name> [--product P] [--source S] [--resolve]
                             [--root DIR] [--db FILE] [--languages LIST]
                             [--dry-run] [--skip-verify] [--yes]
hexgate install update <name> [--to SEL] [--dry-run] [--skip-verify] [--yes]
hexgate install verify <name> [--no-repair]
hexgate install repair <name>
hexgate install db     <name>
```

- `new` only registers an entry: no network, nothing is downloaded. The entry
  starts unpinned at version `0.0.0`.
- `latest` creates the entry if needed (upsert) and then runs the install/update
  flow. If the entry is pinned, that pin is the target; `--resolve` ignores the
  pin, resolves `latest` again from the source and re-pins. `--product` defaults
  to the entry's product or the configured default. A pinned entry whose version
  is still `0.0.0` (usually hand-pinned) warns that it will run a fresh install.
- `update` requires an existing entry and targets its pin, or `--to <SEL>`.
  A target that is not newer than the current version still proceeds (the
  library does not enforce version order) but prints a warning.
- `db` deletes and rewrites `Game.db` from the current manifest and on-disk
  files. Use it after a lost DB or manual file edits; it is not verification.
- `verify` checks file metadata and chunk hashes, repairs through the product
  CDN unless `--no-repair`, and exits `3` when damage remains.
- `repair` re-hashes every chunk and repairs the failures explicitly.

Before patching, every other managed install with a pinned manifest is offered
as a local chunk source, so shared data is copied locally instead of downloaded.
`--dry-run` prints the plan (files, download and disk bytes) without changing
anything.

### `extract`

```
hexgate extract (--manifest SEL | --from-install NAME) --dest DIR
                [--path-regex R] [--name-regex R] [--glob G]
                [--tags T ...] [--untagged] [--min-size B] [--max-size B]
                [--files ID ...] [--files-from FILE] [--all]
                [--list] [--yes] [--no-interactive]
```

- No install is created or modified. Files are written to `--dest`.
- `--from-install` uses that install's manifest, languages and - when present -
  its `Game.db` as a local chunk source for delta extraction.
- With no explicit `--files`/`--files-from`/`--all`, a TTY opens the interactive
  picker; otherwise every file matching the filters is selected.
- `--list` prints the exact planned files (path, size, source: `local` or
  `network`) without downloading; `--list --json` (and the execution envelope)
  include the same entries in a `files` array. File IDs for `--files` come from
  `manifest tree --json` and are `0x`-prefixed hex.
- Extraction cannot be verified - there is no install database - so the summary
  says so.
- If requested tags match nothing in the manifest the command fails instead of
  silently extracting only untagged files.

### `diff`

```
hexgate diff <FROM> <TO> [--product P] [--source S] [--tags T ...] [--from-install NAME]
```

Read-only migration simulation between two selectors: target file counts,
updated/new and unchanged files, unique chunks, local reuse, network and disk
bytes, and the paths that would be removed. Nothing is downloaded or written.
Add `--from-install` to use an install's DB for the local-reuse numbers (the
only case where a DB is opened).

## Common tasks

**First-time setup with a cache on another drive**

```
hexgate config init
# cache_dir is materialized by `config init`; edit it in hexgate.toml, e.g.
#   cache_dir = "D:/hexgate/cache"
# or override per invocation:
hexgate --cache-dir D:/hexgate/cache manifest sync
# prefer the low-latency LoL index as the default source
hexgate config set settings source api
```

(`config set` ignores the entry name for the `source` setting.)

**Find the newest LoL build and pin it**

```
hexgate manifest list --product lol --limit 5
hexgate manifest resolve --latest --source catalog --append --as live
```

**Pin a specific build or a PBE build**

```
hexgate manifest resolve --version 16.9 --append --as live
hexgate manifest resolve --latest --realm PBE1 --append --as pbe
```

**Register an install for a manual path, then install it**

```
hexgate install new live --product lol --root "C:/Riot Games\League of Legends\Game"
hexgate config set live manifest 1D0BDEC9762D3CC3   # also sets the version when the source cache knows the ID
hexgate install latest live
```

**Preview an update, then apply it**

```
hexgate install update live --dry-run
hexgate install update live --yes
```

**Verify and repair**

```
hexgate install verify live         # repairs what it finds, exit 3 if damage remains
hexgate install verify live --no-repair
hexgate install repair live
hexgate install db live             # rebuild Game.db from the files on disk
```

**Browse what is inside a build**

```
hexgate manifest inspect 1D0BDEC9762D3CC3
hexgate manifest tree 1D0BDEC9762D3CC3 --depth 2
hexgate manifest tree 1D0BDEC9762D3CC3 --glob "DATA/FINAL/Champions/*" --flat
hexgate manifest tree 1D0BDEC9762D3CC3 --tags windows,en_US --min-size 100MiB
hexgate manifest fetch 1D0BDEC9762D3CC3             # download the blob, print its path
```

**Extract selected files**

```
# list what would be extracted first
hexgate extract --manifest @live --dest ./dump --name-regex "\.wad\.client$" --tags windows,en_US --list

# flags-only extraction (non-interactive)
hexgate extract --manifest @live --dest ./dump --glob "DATA/FINAL/Champions/Ahri.*" --all --yes

# exact file IDs from `manifest tree --json` (0x-prefixed hex)
hexgate extract --manifest @live --dest ./dump --files 0xD4076C6EC6892967 --yes

# or just open the picker
hexgate extract --manifest @live --dest ./dump
```

**Delta-extract using an existing install as the local source**

```
hexgate extract --from-install live --dest ./dump --tags en_US --all --yes
```

**Compare two builds before updating**

```
hexgate diff @live latest --tags windows,en_US
hexgate diff 12ED9F21C0DA9C90 1D0BDEC9762D3CC3 --from-install live
```

**Work fully offline**

```
hexgate --offline manifest list
hexgate --offline manifest inspect 1D0BDEC9762D3CC3
hexgate --offline extract --manifest @live --dest ./dump --all --yes
```

**Use a project-local install that shadows the global one**

```
cd ~/test-build
hexgate config set live root-dir .\Game --scope local
hexgate config set live db-path .\Game.db --scope local
hexgate config list --all
```

**Script it with JSON**

```
hexgate --json manifest list --product lol --limit 10
hexgate --json install update live --dry-run
hexgate --json extract --manifest @live --dest ./dump --all --list
```

## Interactive pickers

When stdout is a terminal and `--json`/`--yes`/`--no-interactive` are absent,
`manifest resolve` (without `--id`/`--latest`) and `extract` (without an
explicit selection) open a full-screen picker.

| Key | Action |
| --- | --- |
| `up` / `down`, `page-up` / `page-down`, `home` / `end` | Move the cursor |
| `space` | Toggle the file/folder, or the highlighted tag in the filter bar |
| `right` / `enter` | Expand a folder |
| `left` | Collapse a folder, or jump to its parent |
| `left` / `right` (tags row) | Move between `untagged` (first) and the tags |
| `a` / `n` / `x` | Select all visible / clear visible / clear all |
| `/` | Focus the filter bar |
| `tab` / `shift-tab` | Cycle filter fields |
| `ctrl-enter` or `d` | Confirm (extract), `enter` selects (resolve) |
| `q` / `esc` | Cancel without touching disk |

The filter bar supports path/name regexes, a glob, size bounds and tag toggles;
it applies live. Cancelling exits `0` and writes nothing.

## Scripting

With `--json`, results are one envelope per command on stdout:

```json
{"command":"install.update","status":"ok","data":{"...":"..."}}
{"command":"install.update","status":"error","error":{"kind":"network","message":"..."}}
```

During patching, progress is emitted as NDJSON events on stdout and logs stay on
stderr:

```json
{"event":"progress","percent":12.3,"chunks":{"done":10,"total":100},"speed_mbps":4.2,"eta":"02:10","downloaded_mib":12.3,"download_total_mib":100.0,"written_mib":4.0,"write_total_mib":240.0}
```

JSON schemas are stable within a major version; new fields may be added. Exit
codes still apply.

## Exit codes

| Code | Meaning |
| --- | --- |
| `0` | success (also a cancelled picker) |
| `1` | runtime failure (network, I/O, config, library error) |
| `2` | usage error (bad arguments, or a destructive command on a non-interactive session without `--yes`) |
| `3` | verification found unresolved damage |

## Notes and limitations

- **Verification repairs.** The library's chunk verification always repairs
  through the product CDN when it finds damage. `install verify --no-repair`
  performs a single check pass and reports detected damage (exit `3`) without a
  follow-up confirmation pass. With `--offline`, verification checks file
  metadata only; `install repair` requires network access.
- **`--skip-verify`** skips the CLI's explicit verification after
  `install latest`/`install update`. The library still runs its own internal
  chunk verification during an update, so this only avoids the extra pass.
- **Extraction is not verifiable** - no install database is involved.
- **Tag filtering** uses tag names (`windows`, `en_US`, ...) mapped to the
  manifest's tag bitmask. A tag filter that matches nothing is an error for
  `extract` rather than silently degrading to untagged files.
- **One process per install.** Patching commands hold an advisory lock; a second
  `hexgate` process targeting the same install fails fast.
- **Progress may stop below 100%.** The reporter stops with its workers; every
  command prints a final summary you can rely on.
- **Live `Game.db` files are WAL.** If the game is running, close it (or copy
  `Game.db`, `Game.db-wal` and `Game.db-shm`) before using an install as a local
  source for `--from-install` or as a diff source.
- **`--latest` is realm-agnostic.** It picks the newest timestamp in the
  selected source, which is frequently a PBE build. Filter with `--realm`
  (e.g. `--realm pbe`, `--realm na1`) to target a specific realm.
- **`api` source is LoL-only.** For other products or historical builds use
  `catalog`.
- **Teamfight Tactics** manifests can be browsed, fetched, inspected and
  extracted (`--product teamfighttactics`, or `tft` where accepted), and
  installs work through the opaque `version_text` field. Version-prefix
  filtering cannot match `rls-...` strings; use `--realm`, `--artifact` or date
  filters instead.
- **Local manifest files** passed to `manifest inspect`/`tree` are parsed with
  an unchecked parser; only inspect files you trust.
