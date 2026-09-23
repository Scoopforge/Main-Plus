# Scoop manifest field reference (this repo's conventions)

This document describes the fields the Main-Plus repo (`bucket/*.json`)
actually uses. It is derived from statistics over the 39 manifests here, the
1653-manifest survey of the upstream `ScoopInstaller/Main` bucket recorded in
`references/coverage.md`, and the existing CI config; nothing is invented about
Scoop internals. Treat the
[Scoop Wiki · App Manifests](https://github.com/ScoopInstaller/Scoop/wiki/App-Manifests)
as authoritative.

## 1. File-level conventions

| Item          | Convention                                              | Basis                                                                    |
| :------------ | :------------------------------------------------------ | :----------------------------------------------------------------------- |
| File name     | `<app>.json` where `app` matches `^[a-z0-9][a-z0-9-]*$` | 39/39 comply                                                             |
| Encoding      | UTF-8 without BOM                                       | `.editorconfig`'s `charset = utf-8`                                      |
| Indent        | 4 spaces                                                | `.editorconfig`                                                          |
| Line endings  | CRLF                                                    | `.editorconfig`'s `end_of_line = crlf` plus `.gitattributes`' `eol=crlf` |
| Trailing byte | exactly one newline required                            | `.editorconfig`'s `insert_final_newline = true`                          |
| Non-ASCII     | written literally, never \uXXXX-escaped                 | all 39 manifests are plain ASCII today                                   |

Status: of the 39 files only `typst-ts.json` uses LF endings, the single
formatting deviation (`lint` reports W109; `lint --fix-format` repairs it).

## 2. Top-level fields

### 2.1 Required fields (CI fails when missing)

| Field         | Type             | Notes                                                                                                 | Sample in this repo                                       |
| :------------ | :--------------- | :---------------------------------------------------------------------------------------------------- | :-------------------------------------------------------- |
| `version`     | string           | Upstream version, **without the leading `v`**. A github `checkver` strips the tag's `v` automatically | all 39                                                    |
| `description` | string           | One-line English description. Capitalized, **no trailing period**, length <= 120                      | 39/39 comply                                              |
| `homepage`    | string           | Upstream homepage or repository URL                                                                   | all 39                                                    |
| `license`     | string or object | Prefer an SPDX identifier; use `{"identifier": ..., "url": ...}` when unsure                           | `choose` uses `GPL-3.0-only`; `android-cli` the object form |
| `checkver`    | string or object | How the version is detected, see section 3                                                            | all 39                                                    |
| `autoupdate`  | object           | How URLs change on a version bump, see section 4                                                      | all 39                                                    |

### 2.2 Download and install fields

| Field                              | Type               | Notes                                                                                                                            | Sample in this repo                                  |
| :--------------------------------- | :----------------- | :------------------------------------------------------------------------------------------------------------------------------- | :--------------------------------------------------- |
| `url`                              | string or string[] | Single-architecture direct download URL. A top-level `url` together with `architecture` is redundant (W108)                       | 6 here, e.g. `micromamba`, `cxx2flow`                |
| `hash`                             | string or string[] | sha256 of the download. **Must be 64 lowercase hex chars**; when `url` is an array, `hash` must be an array of the same length    | `micromamba`; the array form is upstream only        |
| `architecture`                     | object             | `{"64bit": {...}, "arm64": {...}}`; `64bit` is mandatory                                                                          | 33/39 use it; `choose` is 64bit-only, `typst-ts` adds arm64 |
| `extract_dir`                      | string             | Inner directory name the archive is extracted into                                                                               | `feynman`, `ltex-ls-plus`, `cargo-update`            |
| `extract_to`                       | string             | Subdirectory under `$dir` to extract into, usually the same value as `extract_dir`                                                | none here; 7 upstream                                |
| `innosetup`                        | bool               | Declares an InnoSetup payload so Scoop unpacks it natively, **instead of** a hand-written `installer.script`                      | none here; 22 upstream (`dvc`, `espanso`)            |
| `installer`                        | object             | `{"script": ...}` for a custom install script (string or string array), or `{"file": ..., "args": [...]}` to run a bundled setup exe | none here; 46 upstream (`bun`, `go`)                 |
| `uninstaller`                      | object             | `{"script": ...}`, a custom uninstall script                                                                                      | none here; 34 upstream (`ant`, `busybox`)            |
| `pre_install` / `post_install`     | string or string[] | Hooks before and after install                                                                                                   | `docker-completion`; upstream `ant`, `busybox`       |
| `pre_uninstall` / `post_uninstall` | string or string[] | Hooks before and after uninstall                                                                                                 | none here; 9 and 7 upstream                          |

### 2.3 Integration fields

| Field          | Type                       | Notes                                                                                                          | Sample in this repo                                  |
| :------------- | :------------------------- | :------------------------------------------------------------------------------------------------------------- | :--------------------------------------------------- |
| `bin`          | string or `[exe, alias][]` | Executables to put on PATH. **This is how this bucket installs almost everything: 38 of 39**                    | `choose` uses a string; `open-code-review` uses a pair |
| `shortcuts`    | `[exe, name][]`            | Start-menu shortcuts. A third and fourth item (arguments, icon) are allowed; **the first two must be strings** | none here; 53 upstream                               |
| `persist`      | string or string[]         | Directories / files kept across versions                                                                        | `micromamba`, `open-code-review`                     |
| `env_set`      | object                     | Environment variables written on install                                                                        | none here; upstream `ant` (`ANT_HOME`)               |
| `env_add_path` | string or string[]         | Directories appended to PATH                                                                                    | `micromamba` (`envs\base`); upstream `ant` (`bin`)   |
| `psmodule`     | object                     | `{"name": ..., "path": ...}` for a PowerShell module package instead of `bin` / `shortcuts`                     | `docker-completion`                                  |
| `suggest`      | object or string           | Packages suggested alongside                                                                                    | `commix`, `n-m3u8dl-re`                              |
| `depends`      | string or string[]         | Hard dependency                                                                                                 | none here; 56 upstream                               |
| `notes`        | string or string[]         | Message printed after install                                                                                   | `commix`, `micromamba` (the array form)              |
| `##`           | string                     | **The documented way to leave a comment inside a manifest.** Scoop ignores it; use it instead of `_comment`     | none here; 14 upstream                               |

### 2.4 The architecture block

This bucket writes an `architecture` block even when there is only one
architecture: 19 of the 39 manifests carry a one-key `architecture.64bit`, and
only 6 use a top-level `url` / `hash`. Upstream agrees (603 against 288), and so
does Extras in the opposite direction only because it is a desktop bucket.

`gen` therefore sets `arch_block` on the recipes that model this bucket's
mainstream shapes -- `github-cli-archive`, `toolchain-env` and
`github-single-exe` -- so a new manifest looks like its neighbours. Pass
`--flat-url` to collapse a single architecture to the top level instead. The
other recipes keep the collapsed form they have always had.

### 2.5 The `#/` fragment in URLs

The trailing `#/name` in a URL decides the file name on disk and
**therefore which way Scoop processes the download**:

| Form                  | Effect                                                                                                              | Sample in this repo               |
| :-------------------- | :------------------------------------------------------------------------------------------------------------------ | :-------------------------------- |
| `...exe#/dl.7z`       | unpack the exe as a 7z archive                                                                                      | none here; 149 upstream URLs      |
| `...exe#/dl.zip`      | unpack as a zip                                                                                                     | none here; 10 upstream URLs       |
| `...exe#/choose.exe`  | pin the saved name and keep the file as an exe                                                                      | `choose`, `cxx2flow`, `json-tui`  |
| `...msi#/setup.msi_`  | keep the MSI as a file; **the trailing `_` hides the extension, so Scoop does not pick `Expand-MsiArchive` for it** | none here; 4 upstream URLs        |

`lib/decompress.ps1` chooses the extraction function from the **on-disk file
name**, matching `\.zip$`, `\.msi$` and (`innosetup` only) `\.exe$`, then falling
back to "is this 7z-readable". A name ending in `_` matches none of those, which
is the whole point of the convention: without the underscore an `.msi` download
is silently unpacked with `Expand-MsiArchive` instead of reaching your
`installer.script`.

The `#/<tool>.exe` form is the one this bucket leans on hardest, because a CLI
release is usually called something like `mytool-x86_64-pc-windows-msvc.exe`
while the shim in `bin` has to be `mytool.exe`. No recipe renames for you: put
the fragment in the URL.

## 3. checkver forms

| Form          | Structure                                            | Use when                                                      | Sample in this repo                                |
| :------------ | :--------------------------------------------------- | :------------------------------------------------------------ | :------------------------------------------------- |
| string        | `"checkver": "github"`                               | the GitHub repo can be derived from `url` / `homepage`        | 35 of 39, e.g. `choose`, `sttr`                    |
| github object | `{"github": "https://github.com/o/r"}`               | the homepage is not GitHub but releases are                   | `calepin`, `typst-ts`                              |
| bare regex    | `"checkver": "Version ([\\d.]+)"`                    | the homepage itself lists the version and a regex can read it | none here; 47 upstream (`cacert`)                  |
| url + regex   | `{"url": ..., "regex": ...}`, optionally `+ replace` | upstream is a website / own CDN / vendor endpoint             | `android-cli`, `docker-completion`, `micromamba`, `n-m3u8dl-re` |
| jsonpath      | `{"url": ..., "jsonpath": ..., "regex": ...}`        | only an API or rolling builds are offered                     | none here; upstream `chromedriver`, `dart`         |
| xpath         | `{"url": ..., "xpath": ..., "regex": ...}`           | the version lives in an XML / RSS document                    | none here; 4 upstream                              |
| sourceforge   | `{"sourceforge": "project/path", "regex": ...}`      | upstream is a SourceForge project                             | none here; upstream `boost`, `gdisk`               |
| script        | `{"script": [...], "regex": ...}`                    | a PowerShell request is needed to get the value               | none here; upstream `cangjie`, `pnpm`              |

Key points:

- The **`github` and `sourceforge` forms need no `regex`**; `url`, `script`,
  `xpath` and `jsonpath` must have one.
- **`checkver` with no `url` scrapes `homepage`** -- `bin/checkver.ps1` sets
  `$url = $json.homepage` under its "Not Specified" branch. A regex on its own
  is the shorthand for exactly that.
- `reverse`, `replace` and `useragent` need the **object** form; a bare string
  cannot carry them.
- A `checkver.github` value must be a repository URL, **never an
  `api.github.com` one**: Scoop appends `/releases/latest` unconditionally, so
  the API path turns into a 404 (W111). Put the API endpoint in `checkver.url`.
  83 upstream manifests get this wrong; this repo currently has none.
- Prefer a `(?<version>...)` named group; without one Scoop takes the first
  group. `n-m3u8dl-re` carries a second one, `(?<date>...)`, into `autoupdate`.
- `jsonpath` may also be spelled `jp`; both are read, and one upstream manifest
  uses `re` as an alias for `regex`.
- A leading `v` in the upstream tag needs no handling; Scoop strips it.
- The `script` form needs a Scoop environment, so this skill's `update --checkver`
  cannot probe it offline and says so explicitly.

## 4. Writing autoupdate

`autoupdate` describes what the URL looks like once the version is `$version`.

| Case                      | Form                                                                            | Sample in this repo             |
| :------------------------ | :------------------------------------------------------------------------------ | :------------------------------ |
| top-level `url`           | `{"url": ".../v$version/app-$version.zip"}`                                     | `micromamba`, `cxx2flow`        |
| `architecture`            | `{"architecture": {"64bit": {"url": ...}, "arm64": {"url": ...}}}`              | `calepin`, `typst-ts`           |
| hash from a checksum file | `{"url": ..., "hash": {"url": "$url.sha256", "regex": "$sha256\\s+$basename"}}` | upstream `cacert`, `ant`        |
| hash from a web page      | `{"url": ..., "hash": {"url": ..., "regex": ...}}`                              | upstream `busybox`              |
| `extract_dir` re-templated | `{"extract_dir": "...-$version", ...}`                                          | `feynman`, `ltex-ls-plus`       |

**Two hard constraints (`lint` checks both)**:

1. If a manifest uses `architecture`, `autoupdate` must also supply
   per-architecture URLs (W103); otherwise Excavator will not update them on a
   bump. This repo has no such finding today; upstream has 29.
2. If the current download URL carries a version, the `autoupdate` URL must
   carry `$version` (W110); otherwise the version rises while the URL stays put
   and the package goes stale forever. This repo has no such finding today.

One thing no rule catches, worth fixing by hand: `typst-ts` installs from
`...-x86_64-pc-windows-msvc.zip` but its `autoupdate` points at the same asset
with a `.tar.gz` suffix. Scoop copes -- both unpack -- but the manifest
mislabels what it will fetch after the next Excavator run.

## 5. Canonical key order

Field order produced by `gen` (`CANONICAL_ORDER` in `sm_lib.py`):

```text
## → version → description → homepage → license → notes → architecture → url
→ hash → pre_install → installer → innosetup → extract_dir → extract_to
→ post_install → psmodule → bin → shortcuts → persist → env_set → env_add_path
→ suggest → depends → uninstaller → pre_uninstall → post_uninstall
→ checkver → autoupdate
```

Nested levels have their own order:

| Parent                      | Order                                                                                                                                          |
| :-------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------- |
| `architecture`              | `64bit` → `arm64`                                                                                                                              |
| `architecture.<arch>`       | `url`, `hash`, `pre_install`, `installer`, `innosetup`, `extract_dir`, `extract_to`, `post_install`, `psmodule`, `bin`, `shortcuts`, `persist` |
| `checkver`                  | `github`, `url`, `sourceforge`, `script`, `jsonpath`, `xpath`, `regex`, `replace`, `reverse`, `useragent`                                      |
| `autoupdate`                | `architecture`, `url`, `hash`, `extract_dir`, `bin`, `shortcuts`                                                                               |
| `installer` / `uninstaller` | `script`, `args`                                                                                                                               |
| `hash`                      | `url`, `regex`, `jsonpath`                                                                                                                     |

`update` **does not rewrite the whole file** (avoiding huge diffs): existing
fields keep their position and only new fields are inserted in the order
above. Pass `--reorder` to rewrite everything.

## 6. The README summary table

The README carries one table of every app, under `## ⭐️ Summary`, with **three
columns**:

| Column          | Filled from                                        |
| :-------------- | :------------------------------------------------- |
| `App`           | `[<name>](<homepage>)`, inserted alphabetically     |
| `Language`      | `--language` (Rust / Go / Python / C++ / ...)       |
| `Auto-Update ?` | `✓`, taken from `readme.auto_mark` in `recipes.jsonc` |

The section is detected, not hard-coded: with a single table in the file
`--section` can be omitted, and the column meaning is read from the header, so a
bucket that labels the third column `Note` instead (Extras-Plus) keeps working.
A column the skill does not recognise is left exactly as it was, which is what
makes re-syncing an existing row a no-op.

Six manifests are currently missing from the table (W105): `android-cli`,
`calepin`, `docker-completion`, `muscle`, `seqkit`, `vsearch`.

## 7. When not to use this skill

- PowerShell build outputs, MSI customisation, or private unpacking logic
  beyond `$PLUGINSDIR` -- writing the manifest by hand is easier.
- Upstream ships an installer that needs interaction and cannot run silently.
- Archives over 2GB (Scoop's `aria2` and hash verification degrade).

## 8. Related files

- Which recipe applies, and what it emits: `references/recipes.md`
- Where the recipes came from, and what is not covered: `references/coverage.md`
- Lint rules: `references/lint-rules.md`
- Recipe data (single source of truth): `assets/recipes.jsonc`
