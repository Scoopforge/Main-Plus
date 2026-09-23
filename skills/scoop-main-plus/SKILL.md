---
name: scoop-main-plus
version: 1.5.0
description: >
  Generate, update and lint Main-Plus Scoop bucket manifests (bucket/*.json).
  Three trigger commands: generate builds a skeleton from one of 18 built-in
  recipes and fills in version, URL, hash, checkver, autoupdate, bin and
  shortcuts, optionally syncing the README summary row; update edits fields by
  dotted path, bumps the version while rewriting hard-coded URLs, recomputes
  hashes and probes upstream for the latest release (batch sweep supported);
  lint runs 22 rules against this repo's CI and .editorconfig conventions and
  repairs formatting with --fix-format. The target bucket is resolved at run
  time from $Scoop, so an installed copy writes new manifests into
  $Scoop/buckets/main-plus from any working directory.
  Triggers: generate manifest, new manifest, update manifest, lint manifest,
  scoop-main-plus, main-plus, scoop manifest, bucket manifest, checkver,
  autoupdate, hash verification, version bump, Excavator, Scoop bucket
  maintenance, lint bucket.
display_name: "Scoop Main-Plus Manifest Forge"
visibility: "public"
agent_created: true
---

# Scoop Main-Plus Manifest Forge

Turn "upstream shipped something new" or "upstream shipped a new version" into a
single command. All three trigger commands -- **generate / update / lint** --
share the recipe catalog and the rule engine. Python standard library only, and
everything runs offline except `--checkver`, `--fetch-hash` and `--rehash`.

Package layout:

- `scripts/sm_lib.py` shared layer: paths, serialization, the 18 builders,
  checkver, rule engine, README table sync
- `scripts/scoop_manifest.py` the three-command CLI
- `scripts/sm_selftest.py` self-check: recipes <-> builders, docs <-> code,
  repo round-trip, lint baseline
- `references/manifest-fields.md` manifest field reference (this repo's rules)
- `references/recipes.md` when each of the 18 recipes applies, and what it emits
- `references/lint-rules.md` the 22 rules and how to fix each one
- `references/coverage.md` the upstream survey behind the catalog, and the gaps
- `assets/recipes.jsonc` the single source of truth for recipes. The `.jsonc`
  suffix is deliberate -- see the hard constraints below

Scripts derive the package root themselves, so **they run from any cwd**. The
target bucket is resolved on every run, in this order:

1. `--repo <path>`, when given;
2. the cwd, walked upwards -- running inside any bucket edits that bucket;
3. **`$Scoop/buckets/main-plus`** -- the global fallback, which is what lets an
   installed copy write into this bucket from anywhere else.

`$Scoop` is read from the environment (`SCOOP`, then `Scoop`). The expanded path
is **never** stored in the skill -- the same package has to work on any machine.

```bash
python scripts/scoop_manifest.py <command> [options]
python scripts/sm_selftest.py
```

Pure standard library, so any Python 3.11+ works. On this machine the gates run
on the managed interpreter at
`~/.workbuddy/binaries/python/versions/3.13.12/python.exe`.

## 1. Hard constraints

- **Output**: `<repo>/bucket/<app>.json`, optionally plus a README summary row.
  Never write to `bin/`, `scripts/` or `.github/` -- those belong to Scoop's
  official scripts and to this repo's CI.
- **Never bake `$Scoop` in.** No file may hold the resolved path
  (`<drive>:\...\buckets\...`); the environment is read on every run, so one
  package works on every machine. The self-check fails if a literal reappears.
- **Never add a `.json` data file inside this skill.** The bucket's CI runs
  `Import-Bucket-Tests.ps1`, which validates every *changed* `.json` in the
  repository against scoop's manifest schema -- repo-wide, because
  `Get-GitChangedFile -Include '*.json'` ignores its `-Path` when listing files.
  A non-manifest `.json` here turns CI red. That is why the catalog is
  `assets/recipes.jsonc`; use `.jsonc` (or a `.py` module) for any further data.
- **Preserve existing order**: `update` only slots **new** fields into their
  canonical position; existing fields keep their place. A full reorder needs an
  explicit `--reorder`.
- **Self-check before writing**: the result goes through the rule engine first,
  and error-level findings block the write (`--force` overrides).
- **This bucket is bin-first**: it installs 39 of its 40 packages through `bin`
  and declares no `shortcuts` at all. Reach for a shortcut only when the package
  really is a desktop app, and expect `github-cli-archive` to refuse one.
- **README is controlled**: the table lives under `## ⭐️ Summary` with the three
  columns `App / Language / Auto-Update ?`. A missing section skips the sync with
  an explanation, and a column the skill does not recognise is never touched.

## 2. The three trigger commands

| Command | Alias | Job | Main options |
| :--- | :--- | :--- | :--- |
| **generate** | `gen` | Build a manifest from a recipe and fill it in, optionally sync README | `--list-recipes`, `--from`, `--recipe`, `--fetch-hash`, `--hash-from-file`, `--language`, `--flat-url`, `--dry-run` |
| **update** | `upd` | Edit fields / bump version + rewrite URLs / recompute hashes / probe upstream | `--name`, `--all`, `--set`, `--unset`, `--version`, `--rehash`, `--readme`, `--checkver [--apply]` |
| **lint** | `check` | Run the 22 rules, repair formatting | `--name`, `--json`, `--strict`, `--fix-format`, `--rules` |

Shared option `--repo <bucket repo root>`. Without it the script walks up from
the cwd looking for a directory holding both `bucket/` and `README.md`, and
falls back to `$Scoop/buckets/main-plus` when there is none -- so a copy that is
installed elsewhere still writes into this bucket.

## 3. generate

**Settle six things first** and ask the user for anything missing; do not guess:

1. Who is upstream: a GitHub repo, or a website / own CDN?
2. What ships: portable archive / NSIS installer / InnoSetup / bare exe / a
   vendor toolchain that is never shimmed?
3. Version number (without the leading `v`)
4. What goes on PATH: the exe `bin` should point at (relative to `$dir`), and any
   command-line alias. Most packages in this bucket need nothing else.
5. Whether a Start-menu shortcut is genuinely wanted. If it is, the package is
   not a CLI tool and `github-cli-archive` is the wrong recipe.
6. README: the implementation language (Rust / Go / Python / C++ / ...)

Unsure about the recipe? Run `--list-recipes` first; it prints when each recipe
applies, the required and optional parameters, and same-kind samples (from this
repo where a manifest of that shape exists, from the upstream main bucket
otherwise). Then compare against `references/recipes.md`.

```bash
python scripts/scoop_manifest.py gen --name mytool --recipe github-cli-archive \
  --version 3.4.5 --desc "Super fast text transformer" \
  --homepage https://github.com/o/r --license MIT \
  --url64 "https://github.com/o/r/releases/download/v3.4.5/tool-x86_64-pc-windows-msvc.zip" \
  --repo-url https://github.com/o/r \
  --bin-exe tool.exe --language Rust --dry-run

python scripts/scoop_manifest.py gen --from specs.json --language Go
```

`--from` reads a spec file, which suits batches: an object or an array of
objects whose keys are the `param_docs` names from `recipes.jsonc`, plus `name`,
`recipe` and `language`. Command-line options win over the file.

**Architecture**: `--arch 64bit+arm64` emits an `architecture` block with
`url64` / `url_arm64`. A single architecture still gets a block for the recipes
that set `arch_block` (`github-cli-archive`, `toolchain-env`,
`github-single-exe`), because that is what 20 of the 40 manifests here do; pass
`--flat-url` to collapse it to a top-level `url` / `hash` instead.

**32bit is not supported.** This bucket ships 64bit and arm64 only, so `arch`
accepts just those two values and there is no `--url32` / `--hash32`. Passing
`32bit` fails with a deliberate message rather than a generic typo complaint.
Upstream still carries it on 528 files, which is why `references/coverage.md`
keeps it in the distribution table -- that is survey data, not a supported
option.

**Pick one of three ways to obtain the hash, never invent it**: `--fetch-hash`
streams the download and computes it; `--hash-from-file <path>` uses a package
already on disk; if neither is given, run `bin/checkhashes.ps1` afterwards (the
command prints that hint).

**Rhythm**: `--dry-run` to preview, then drop it to write and sync the README,
then `lint --name <app>` to confirm.

## 4. update

`--set` takes a dotted path and parses the value as JSON, falling back to a
string. New fields land in their canonical key position (`persist` goes between
`extract_dir` and `env_set`, not at the end of the file); `--unset` deletes.

```bash
python scripts/scoop_manifest.py upd --name mytool \
  --set 'description=Fast text transformer' --set 'persist=data' \
  --set 'bin.0.1=tt'

python scripts/scoop_manifest.py upd --name mytool --checkver   # report
python scripts/scoop_manifest.py upd --name mytool --checkver --apply --rehash
python scripts/scoop_manifest.py upd --name mytool --version 3.5.0  # manual
python scripts/scoop_manifest.py upd --all --checkver --apply --rehash  # sweep
```

`--version` rewrites the old version hard-coded in every download URL.

`--checkver` understands the `github` string, `{"github": ...}`, bare-string
regex (scraped from `homepage`), `{"url", "regex"}`, `{"url", "jsonpath",
"regex", "replace"}`, `{"url", "xpath", ...}` and `{"sourceforge": ...}`.
**The `{"script": ...}` form needs a Scoop environment and explicitly reports that
it cannot probe offline**; use `bin/checkver.ps1` instead.

Safety net: the rule engine runs after every change and error-level findings
**block the write** (`--force` overrides); `--dry-run` previews and
`--print-json` dumps the result. `upd` leaves the README alone unless `--readme`
is passed, which syncs the row and keeps every cell it does not own.

## 5. lint

```bash
python scripts/scoop_manifest.py lint                  # full run, about a second
python scripts/scoop_manifest.py lint --name choose    # a single app
python scripts/scoop_manifest.py lint --json           # machine-readable report
python scripts/scoop_manifest.py lint --strict         # warnings fail too
python scripts/scoop_manifest.py lint --fix-format     # formatting only
python scripts/scoop_manifest.py lint --rules          # print the rule catalog
```

`--fix-format` touches formatting only (indent / CRLF / trailing newline) and
never JSON semantics.

Exit code: error-level findings give 1; warnings alone give 0, or 1 with
`--strict`. Rules and their fixes live in `references/lint-rules.md`.

**Baseline (40 manifests)**: 0 errors, 13 warnings, 28 fully clean. Real issues
found so far:

| manifest | Issue | Rule |
| :--- | :--- | :--- |
| `calepin`, `docker-completion`, `muscle`, `seqkit`, `vsearch` | missing from the README summary table | W105 |
| `commix` | `license` is a URL, not an SPDX identifier | W106 |
| `qlty`, `rheo`, `shiroa` | `license` is prose (`Business Source License 1.1`, `Apache-2.0 license`) | W106 |
| `micromamba`, `n-m3u8dl-re`, `typst-ts` | `version` carries non-numeric parts (`2.9.0-0`, `0.6.0-beta`, `0.8.0-rc3`), which autoupdate can mishandle | W107 |
| `typst-ts` | the only file in the repo using LF endings | W109 |

One issue no rule catches, worth fixing by hand: `typst-ts` installs from a
`.zip` while its `autoupdate` points at a `.tar.gz`.

## 6. Boundaries

Not for: installers that need interaction, MSI customisation, or packages with
private unpacking logic beyond `$PLUGINSDIR` (hand-writing is easier); archives
over 2GB (aria2 and hash verification degrade); `.jar` launchers, which need a
hand-written `.cmd` shim; and any change under `bin/`, `scripts/` or `.github/`.

## 7. Maintenance

```bash
python scripts/sm_selftest.py            # full self-check (offline)
python scripts/sm_selftest.py --verbose  # print every detail
```

The self-check has 7 groups, numbered in run order: recipe catalog shape and
architecture policy -> virtual rendering of all 18 recipes from their own
declared parameters -> skill package consistency (`lint-rules.md` matches
`RULES` word for word, `recipes.md` maps one-to-one onto `recipes.jsonc`, and
`SKILL.md`'s `name` equals the directory name) -> path resolution policy (no
expanded `$Scoop` baked in, and `$Scoop/buckets/main-plus` really is the
fallback) -> repo serialization round-trip -> README table round-trip,
row-insert idempotence and the no-op re-sync -> the lint baseline over the real
bucket.

**Adding a recipe** (4 steps, and the self-check catches omissions): add an entry
to the `recipes` array in `recipes.jsonc` (`id` / `label` / `when` / `builder` /
`required` / `optional` / `refs`) and document any new parameters in
`param_docs` -> register a builder of the same name in `BUILDERS` in
`sm_lib.py` -> add a `## <recipe id>` section to `recipes.md` -> run
`sm_selftest.py`. Base a new recipe on a population recorded in
`references/coverage.md`, not on a single manifest, and update section 9 there
in the same pass.

**Adding a rule**: change `RULES` in `sm_lib.py` and the table in
`lint-rules.md` together, keeping the wording identical.

**Editing docs**: all four markdown files pass `rumdl check` at its default
(width <= 80 columns); finish with `rumdl fmt`.
