# Coverage: where the recipes come from

`assets/recipes.jsonc` was not drawn from a blank page, and it is not meant to be
exhaustive. This document records the survey that sized the catalog, maps every
recipe onto the population it covers, and states what was deliberately left out.
Read it when a manifest in front of you does not obviously match a recipe: the
two questions are "which upstream shape is this?" and "is it one of the known
gaps?".

Sections 1-2 are the method and the two corpora, 3-8 the survey, 9 the mapping,
10 the gaps, 11 the re-run recipe.

## 1. Method

Two corpora, both read offline:

- `C:\Scoop\buckets\main\bucket` -- the upstream `ScoopInstaller/Main` bucket,
  **1653** `*.json` manifests and **6340** URLs. This is the corpus the catalog
  is sized against now, because Main-Plus is an enhancement of that bucket and
  inherits its shape.
- `C:\Scoop\buckets\main-plus\bucket` -- this repo, **39** manifests. Quoted
  separately wherever it disagrees with upstream, which it often does.
- The earlier survey of `C:\Scoop\buckets\extras\bucket` (upstream
  `ScoopInstaller/Extras`, **2389** manifests, **7853** URLs) is still quoted in
  section 2, because it is what the original recipes were shaped by.

Every file is `json.loads`-ed and tallied; nothing is downloaded. "manifests" or
"files" below means *how many files* show a trait, and one file counts at most
once per trait. Rows that count URLs instead say so.

Measured 2026-09-20. Upstream moves, so the absolute numbers are a snapshot and
the proportions are the durable part. Section 11 has the re-run recipe.

## 2. Two buckets, two shapes

The two upstream buckets are not variations on a theme; they are opposites in
the one place that matters most, namely how a package reaches the user.

| Trait                    |      Extras (2389) | Main (1653) | This repo (39) |
| :----------------------- | -----------------: | ----------: | -------------: |
| `bin`                    |        1428 (60%) |  1517 (92%) |       38 (97%) |
| `shortcuts`              |        1843 (77%) |     53 (3%) |              0 |
| `architecture`           |               1688 |        1365 |             33 |
| `checkver`               |               2199 |        1567 |             39 |
| `autoupdate`             |               2210 |        1575 |             39 |
| top-level `url` + `hash` |                831 |         322 |              6 |
| `persist`                |                704 |         152 |              3 |
| `env_set`                |                 44 |         106 |              1 |
| `env_add_path`           |                 28 |          88 |              1 |
| `installer`              |                154 |          46 |              0 |
| `innosetup`              |                124 |          22 |              0 |
| `psmodule`               |                 18 |           9 |              1 |
| `##`                     |                 54 |          14 |              0 |

Extras is a **desktop** bucket: a Start-menu shortcut is the normal way to hand
a package over, and `bin` is the exception. Main is a **CLI** bucket: `bin` is
near-universal and `shortcuts` is rounding error. That is why the catalog leads
with `github-cli-archive`, and why `shortcut_exe` stopped being a required
parameter of `github-nsis-7z`, `github-innosetup` and `github-single-exe` -- in
this repo it would have been wrong more often than right.

## 3. Corpus shape

The four required fields are on all 1653 files by definition. Everything else:

| Field                      | Files | Field            | Files |
| :------------------------- | ----: | :--------------- | ----: |
| `version` / `description`  |  1653 | `post_install`   |    74 |
| `homepage` / `license`     |  1653 | `depends`        |    56 |
| `autoupdate`               |  1575 | `shortcuts`      |    53 |
| `checkver`                 |  1567 | `installer`      |    46 |
| `bin`                      |  1517 | `uninstaller`    |    34 |
| `architecture`             |  1365 | `innosetup`      |    22 |
| `url` + `hash` (top level) |   322 | `##`             |    14 |
| `suggest`                  |   245 | `psmodule`       |     9 |
| `extract_dir`              |   241 | `pre_uninstall`  |     9 |
| `notes`                    |   171 | `extract_to`     |     7 |
| `pre_install`              |   161 | `post_uninstall` |     7 |
| `persist`                  |   152 |                  |       |
| `env_set`                  |   106 |                  |       |
| `env_add_path`             |    88 |                  |       |

Four figures matter for maintenance:

- **74 manifests have neither `checkver` nor `autoupdate`** (86 lack checkver
  alone, 78 lack autoupdate alone). Excavator can never refresh those. `lint`
  reports it as a warning, not an error, because it is a legitimate choice for
  abandoned software. Extras had 175 of 2389, so the habit is more common there.
- **83 manifests put an `api.github.com` URL in `checkver.github`** (5.0%, against
  5.9% in Extras). Scoop appends `/releases/latest` unconditionally, so those
  values resolve to `.../releases/latest/releases/latest` and 404. Rule `W111`;
  the fix is to move the endpoint into `checkver.url`.
- **29 manifests carry `architecture` but give `autoupdate` only a flat `url`**,
  so Excavator refreshes one URL and silently leaves the per-architecture ones
  pinned. Rule `W103`; Extras had 148.
- **`license` is an object on 127 files** (`{"identifier": ..., "url": ...}`) and
  a plain string on 1526. Both forms are first class in `param_docs`.

## 4. checkver shapes

The shape is `json`-normalised: `str:` means the value is a bare string,
`{a,b}` means an object with exactly those keys.

| Shape                             | Files | Covered by              |
| :-------------------------------- | ----: | :---------------------- |
| `{github}`                        |   527 | every `github-*` recipe |
| `str:github`                      |   506 | every `github-*` recipe |
| `{regex,url}`                     |   228 | `webpage-regex`         |
| *(no checkver)*                   |    86 | --                      |
| `{github,jsonpath,regex}`         |    82 | `github-asset-jsonpath` |
| `{github,regex}`                  |    51 | `github-asset-jsonpath` |
| `str:<regex>`                     |    47 | `webpage-regex`         |
| `{jsonpath,url}`                  |    30 | `api-jsonpath`          |
| `{regex,replace,url}`             |    21 | `webpage-regex`         |
| `{jsonpath,regex,url}`            |    16 | `api-jsonpath`          |
| `{regex,reverse,url}`             |    11 | `webpage-regex`         |
| `{regex,script}`                  |    10 | `checkver-script`       |
| `{github,jsonpath,regex,replace}` |     6 | `github-asset-jsonpath` |
| `{github,jsonpath}`               |     5 | `github-asset-jsonpath` |
| `{regex,url,useragent}`           |     5 | `webpage-regex`         |
| `{regex}`                         |     4 | `checkver-script`       |
| `{jsonpath,regex,reverse,url}`    |     4 | `api-jsonpath`          |
| `{regex,sourceforge}`             |     3 | `sourceforge`           |
| `{url,xpath}` / `{regex,url,xpath}` | 2 / 2 | `webpage-regex`       |
| `{regex,replace}`                 |     2 | hand-written            |
| `{github,re}`                     |     1 | `github-asset-jsonpath` (`re` aliases `regex`) |
| `{sourceforge}`                   |     1 | `sourceforge`           |
| `{regex,replace,reverse,url}` / `{regex,script,url}` / `{github,jsonpath,regex,script}` | 1 each | hand-written |

Per-modifier totals, which the recipes accept as optional parameters:

| Modifier                     | Files | Recipe parameter               |
| :--------------------------- | ----: | :----------------------------- |
| GitHub checkver (both forms) |  1033 | `repo_url` / `checkver_github` |
| `checkver.regex`             |   448 | `checkver_regex`               |
| `checkver.url`               |   321 | `checkver_url`                 |
| `checkver.jsonpath`          |   144 | `checkver_jsonpath`            |
| `checkver.replace`           |    30 | `checkver_replace`             |
| `checkver.reverse`           |    16 | `checkver_reverse`             |
| `checkver.script`            |    12 | `checkver_script`              |
| `checkver.useragent`         |     5 | `checkver_useragent`           |
| `checkver.xpath`             |     4 | `checkver_xpath`               |
| `checkver.sourceforge`       |     4 | `checkver_sourceforge`         |

GitHub is 66% of the corpus, denser than Extras (57%), so the six GitHub recipes
carry most of this bucket. The bare-regex shorthand is still used 47 times: Scoop
reads a bare string as a regex run against `$json.homepage`, which is why
`webpage-regex` works without `checkver_url`.

## 5. autoupdate shapes

| Shape                            | Files | Shape                             | Files |
| :------------------------------- | ----: | :-------------------------------- | ----: |
| `{architecture}`                 |   749 | `{extract_dir,hash,url}`          |    18 |
| `{architecture,hash}`            |   510 | `{architecture,extract_dir,hash}` |    12 |
| `{url}`                          |   145 | `{architecture,bin}`              |     3 |
| *(no autoupdate)*                |    78 | `{architecture,hash,url}`         |     3 |
| `{hash,url}`                     |    63 | `{architecture,url}`              |     3 |
| `{extract_dir,url}`              |    37 | `{bin,url}`                       |     1 |
| `{architecture,extract_dir}`     |    31 |                                   |       |

Inside `autoupdate.architecture.<arch>` the member shape is almost always
minimal: `{url}` on 1823 branches, `{extract_dir,url}` on 286, `{hash,url}` on
77, `{extract_dir}` on 13, `{extract_dir,hash,url}` on 9.

`autoupdate` is architecture-first (1259 of 1575 files) for the same reason
`checkver` is GitHub-first: the two halves are built from the same release asset
list.

## 6. Architecture combinations

| Combination                 | Main | Extras |
| :-------------------------- | ---: | -----: |
| `64bit` only                |  603 |    860 |
| `32bit` + `64bit`           |  348 |    468 |
| no `architecture` block     |  288 |    701 |
| `64bit` + `arm64`           |  233 |    230 |
| `32bit` + `64bit` + `arm64` |  180 |    128 |
| `arm64` only                |    1 |      2 |

Two consequences for the builders:

- **One architecture in a block beats no block at all** in this bucket: 603
  against 288 upstream, and 19 against 6 in this repo. A recipe that always
  collapses a single architecture into a top-level `url` would fight the local
  convention, which is why `github-cli-archive`, `toolchain-env` and
  `github-single-exe` set `arch_block` by default and `--flat-url` exists to
  override it. The other recipes keep the historical collapsed form.
- **32bit is surveyed but not supported.** It still appears on 528 upstream
  files, so the distribution above has to account for it -- but this skill
  refuses the value: `arch_list` rejects `32bit` outright and there is no
  `--url32` / `--hash32` to feed it. arm64 is the only second architecture
  worth modelling, at 414 files (360 in Extras), and it is never the only
  architecture in practice.

## 7. Download shapes

Counted **per URL**, not per file, across every `url` / `url64` / `url32` /
`url_arm64` at any nesting depth -- **6340** URLs in total:

| Extension | URLs |     | Extension   | URLs |
| :-------- | ---: | :-- | :---------- | ---: |
| `.zip`    | 2902 |     | `.tar.lzma` |   31 |
| `.exe`    |  945 |     | `.ps1`      |   30 |
| `.tar.gz` |  422 |     | `.tar.zst`  |   24 |
| `.txt`    |  316 |     | `.jar`      |   23 |
| `.sha256` |  171 |     | `.tar.xz`   |   22 |
| `.msi`    |  127 |     | `.nupkg`    |   17 |
| `.7z`     |  110 |     | `.sha512`   |   17 |
| `.html`   |   75 |     | `.tgz`      |   17 |
| `.json`   |   51 |     | `.gz`       |   12 |

**516 URLs carry no extension at all**, and most of those are not packages:
`.txt`, `.sha256` and `.sha512` together account for 504 URLs, which is the
sidecar-checksum population behind `autoupdate.hash` and the `au_hash_url`
parameter. `.html`, `.json` and `.atom` point at checkver targets, not downloads.

`#` fragments, again per URL:

| Fragment       |         URLs | Meaning                                        |
| :------------- | -----------: | :--------------------------------------------- |
| `#/dl.7z`      |          149 | rename an NSIS shell so Scoop unpacks it as 7z |
| `#/dl.zip`     |           10 | the same rename trick, for a zip payload       |
| `#/<tool>.exe` | the long tail | pin the saved name and keep it unpacked       |
| `#/dl.msi`     |            4 | hand an MSI to Scoop untouched                 |

The named `#/<tool>.exe` fragment is far more common here than in Extras -- it is
what a CLI release needs when the asset is called
`mytool-x86_64-pc-windows-msvc.exe` and the shim has to be `mytool.exe`. Pass it
in the URL; no recipe does the renaming for you.

Fifteen manifests download a `.tar.gz` but install from a `.zip` (or the other
way round) because their `autoupdate` template disagrees with the checked-in URL.
`typst-ts` in this repo is one of them: `url` ends in `.zip`, `autoupdate.url` in
`.tar.gz`. Scoop copes, but the mismatch is a latent surprise.

## 8. Install mechanics

This is the section the catalog actually turns on.

| Trait                                           | Files |
| :---------------------------------------------- | ----: |
| `bin`, no `shortcuts`                           |  1469 |
| `bin`, no `shortcuts`, no `extract_dir`, no env |  1222 |
| ... of those, with an `architecture` block      |  1066 |
| ... of those, flat `url`/`hash`                 |   156 |
| ... of those, a `.tar.*` payload                |   109 |
| no `bin`, no `shortcuts`, no `psmodule`         |   123 |
| ... of those, `env_add_path`                    |    67 |
| ... of those, `extract_dir`                     |    34 |
| ... of those, `env_set`                         |    33 |
| ... of those, `persist`                         |    28 |
| `installer.keep`                                |     2 |

Two recipes were added on the strength of this table:

- **`github-cli-archive`** for the 1469 top rows: an archive or bare exe whose
  entire install is a PATH entry. It also swallows the `.tar.*` releases (142
  files; 114 `.tar.gz`, 9 `.tgz`, 7 `.tar.xz`, 7 `.tar.lzma`, several files
  carrying more than one), of which only 11 have a version-stamped `extract_dir`
  -- the tarball usually expands in place, so no `extract_dir` is needed.
- **`toolchain-env`** for the 123 bottom rows: a compiler / SDK / runtime that is
  never shimmed and is wired up through `env_add_path` and `env_set` instead.
  `ant` is the cleanest example -- no `bin`, `env_add_path: bin`,
  `env_set: {ANT_HOME: $dir}`, a version-stamped `extract_dir`, and an
  `uninstaller.script` that copies user libraries back to `persist_dir`.

`github-cli-archive` and `github-portable-zip` can produce the same file. The
overlap is deliberate: `github-cli-archive` is the stricter default for this
repo -- it demands `bin_exe` (so you have to say what goes on PATH) and rejects
`shortcut_exe` (a Start-menu entry means the package is not a CLI tool). Reach
for `github-portable-zip` when the shortcut, the `extract_dir` or the
environment variables are the point.

## 9. Recipe to upstream pattern

"Population" is the number of upstream files whose shape that recipe is built
for; a file is quoted under the shape it fits best, so the column is not a
partition.

| Recipe                  | Upstream shape                                             |                Population |
| :---------------------- | :--------------------------------------------------------- | ------------------------: |
| `github-cli-archive`    | `bin` with no `shortcuts`                                  |                      1469 |
| `github-portable-zip`   | archive plus a shortcut / `extract_dir` / env              |                       254 |
| `webpage-regex`         | non-GitHub `url` + `regex`, plus the bare-string shorthand |                  228 + 47 |
| `toolchain-env`         | no shim at all; env-driven                                 |                       123 |
| `github-asset-jsonpath` | GitHub checkver carrying `jsonpath`                        | 82 + 51 + 6 + 5 + 1 = 145 |
| `api-jsonpath`          | non-GitHub `jsonpath` checkver                             |                  30 + 16 + 4 = 50 |
| `github-msi`            | `.msi` among the download URLs                             |                        50 |
| `github-exe-installer`  | an `installer` block the release actually runs              |                        46 |
| `github-nsis-7z`        | `#/dl.7z` on the URL                                       |                        40 |
| `github-innosetup`      | `innosetup: true`                                          |                        22 |
| `github-single-exe`     | the download is itself the executable                      |                        19 |
| `redirect-arch`         | version-less permanent link (`/latest/`)                   |                        18 |
| `checkver-script`       | `checkver.script`                                          |                        12 |
| `powershell-gallery`    | a `psmodule` block and a `.nupkg` download                  |                         9 |
| `github-source-archive` | `archive/refs/tags`, or a version-stamped `extract_dir`    |                         6 |
| `portable-multifile`    | `url` and `hash` as arrays                                 |                         6 |
| `sourceforge`           | `checkver.sourceforge`                                     |                         4 |
| `github-git-clone`      | plugin cloned into a host app                              |           0 (Extras-Plus) |

Three of those deserve a caveat rather than a number:

- `sourceforge`: **23** files download from a SourceForge mirror, but only 4 use
  the dedicated `checkver.sourceforge` key. The rest scrape the project page and
  land in `webpage-regex`. Prefer the dedicated key -- it does not break when the
  page is redesigned.
- `github-git-clone`: zero manifests here and upstream contain `git clone`. It is
  an Extras-Plus pattern (`comfyui-manager`) and stays in the catalog for that
  reason alone.
- `github-single-exe`: in a bin-only bucket it lands on the same JSON as
  `github-cli-archive`. Its reason to exist is the recipe's own text -- "the
  download is the executable" -- which is the only case where a `#/<name>.exe`
  fragment is the whole install.

Every **Samples** list in `recipes.md` cites files that exist in this corpus:
this repo where a manifest of that shape exists there (11 of 18 recipes), and
`ScoopInstaller/Main` otherwise. The samples replaced the
`ScoopInstaller/Extras` ones the previous edition of the catalog carried, so a
sample can now be opened next to the recipe and read.

## 10. Not covered

Known gaps, in rough order of how likely they are to bite:

- **`checkver.script` cannot be probed offline.** The 12 files that use it, and
  `checkver-script`, need a live Scoop environment, so `update --checkver`
  reports that it cannot run rather than guessing. Use `bin/checkver.ps1`.
- **`autoupdate.bin` and `autoupdate.shortcuts`.** Four manifests
  (`avr-gcc`, `capnp`, `influxdb`, `lua`) rewrite `bin` on every update. No
  parameter emits that block; hand-edit those four.
- **`installer.keep`.** `groovyserv` and `vcpkg` keep the installer file in
  `$dir` after running it. Two files did not justify a parameter.
- **`.jar` launchers.** 23 URLs are `.jar`, and they need a generated `.cmd`
  shim in `installer.script`, which no recipe writes. Hand-write it.
- **MSI customisation.** `github-msi` covers the two mechanical modes (unpack
  with `Expand-MsiArchive`, or hand to `msiexec`). Per-feature install, MST
  transforms and advertised shortcuts are not modelled.
- **Installer interaction.** Anything that needs a click, a licence dialog or a
  driver prompt is out of scope; `github-exe-installer` scaffolds the script, but
  the body has to be written by hand.
- **Archives over 2 GB.** aria2 and hash verification degrade, so no recipe is
  tuned for them.
- **`.msix`.** Zero in this corpus (12 in Extras). `github-msi` is the nearest
  recipe; expect to hand-edit `pre_install`.
- **The one-off shapes.** `{regex,replace}`, `{regex,replace,reverse,url}`,
  `{regex,script,url}` and `{github,jsonpath,regex,script}` appear once each.
  Left to hand-writing by design.

## 11. Refreshing the survey

The survey script is deliberately **not** shipped: it is a one-off analysis, and
keeping it in the package would make it look like a supported tool. To redo it,
point a read-only script at the bucket to measure and tally the same things --
top-level keys, the four shape families, architecture combinations, URL
extensions, `#` fragments, and the install-mechanics table in section 8. If the
proportions move far enough to invalidate a recipe (say `shortcuts` climbs back
above 10% and the bin-first default stops being obvious), update the recipe and
the corresponding row here together, the way `sm_selftest.py` requires for
`recipes.md`.
