# Recipe catalog (18 recipes)

A recipe is a template for one combination of fields, autoupdate shape and
checkver form. The data source is `assets/recipes.jsonc`; this document is the
readable version, kept in sync by `scripts/sm_selftest.py`.

Decision order: **look at what upstream publishes first, then at how to unpack
it.**

```text
GitHub Release
  CLI tool, everything goes on PATH → github-cli-archive
  portable archive (.zip/.7z)      → github-portable-zip
  NSIS .exe with a 7z payload      → github-nsis-7z
  InnoSetup .exe                   → github-innosetup
  .msi installer                   → github-msi
  installer must actually run      → github-exe-installer
  bare .exe (portable build)       → github-single-exe
  source tag archive               → github-source-archive
  asset list carries extra fields  → github-asset-jsonpath
vendor toolchain / SDK / runtime   → toolchain-env
non-GitHub / own CDN
  HTML page / version API          → webpage-regex
  JSON API (rolling build)         → api-jsonpath
  version needs a script           → checkver-script
  SourceForge project              → sourceforge
  PowerShell Gallery module        → powershell-gallery
  fixed redirect URL               → redirect-arch
several downloads in one package   → portable-multifile
plugin (into a host app)           → github-git-clone
```

Where each recipe came from, and what is still uncovered, is in
`references/coverage.md`. The **Samples** line under each recipe cites a
manifest that really has that shape: files from this repo where they exist, and
files from the upstream `ScoopInstaller/Main` bucket otherwise. `comfyui-manager`
is the one sample that lives in the sibling Extras-Plus bucket.

## github-cli-archive

**Use when** the release is a command-line tool: an archive or a bare exe
holding one or more executables, and the manifest's whole job is putting them on
PATH. This is the normal shape here -- 38 of the 39 manifests in this bucket,
and 1469 of the 1653 upstream.

**Required** `version`, `desc`, `homepage`, `license`, `url64`, `bin_exe`

**Optional** `repo_url`, `arch`, `url_arm64`, `hash64`, `hash_arm64`,
`extract_dir`, `bin_alias`, `bin_entries`, `env_set`, `env_add_path`,
`persist`, `suggest`, `notes`, `comment`

**Key point** the recipe **refuses** `shortcut_exe` / `shortcut_entries` instead
of ignoring them: a Start-menu entry is almost always a sign that the package
belongs to `github-portable-zip`. The URL extension does not matter --
`.tar.gz`, `.tar.xz`, `.tar.zst` and `.tar.lzma` releases (142 upstream files,
109 of them this shape) take the same route as a `.zip`.

**Samples** `cargo-dist`, `calepin`, `sttr`, `tex-fmt`, `typdiff`, `sendme`,
`act`, `air`

```powershell
scoop_manifest.py gen --name mytool --recipe github-cli-archive `
  --version 1.4.0 --desc "Fast command line text transformer" `
  --homepage https://github.com/acme/mytool --license MIT `
  --url64 "https://github.com/acme/mytool/releases/download/v1.4.0/mytool-x86_64-pc-windows-msvc.zip" `
  --bin-exe mytool.exe --language Rust --rehash
```

## github-portable-zip

**Use when** the release ships an archive that works as-is, with no installer,
and installing it means more than a PATH entry: a Start-menu shortcut, a
version-stamped `extract_dir`, or environment variables.

**Required** `version`, `desc`, `homepage`, `license`, `url64`

**Optional** `repo_url`, `arch`, `url_arm64`, `extract_dir`,
`shortcut_exe`, `shortcut_name`, `bin_exe`, `bin_alias`, `env_set`,
`env_add_path`, `persist`, `suggest`, `notes`, `comment`

**Output** with a single architecture the `url` / `hash` pair lands at the top
level; listing several in `arch` emits an `architecture` block instead.

**Samples** `cargo-update`, `cryptomator-cli`, `feynman`, `ltex-ls-plus`,
`commix`, `micromamba`, `nebula`, `typst-ts`

```powershell
scoop_manifest.py gen --name myapp --recipe github-portable-zip `
  --version 1.2.3 --desc "Portable note taking app" `
  --homepage https://github.com/acme/myapp --license MIT `
  --url64 "https://github.com/acme/myapp/releases/download/v1.2.3/app.zip" `
  --repo-url https://github.com/acme/myapp --extract-dir myapp-1.2.3 `
  --shortcut-exe myapp.exe --shortcut-name MyApp
```

## github-nsis-7z

**Use when** the release is an NSIS `.exe` whose real payload is a 7z under
`$PLUGINSDIR`. The installer is a self-extracting shell; treating the shell as a
7z is the cleanest route. This covers any such container, not only the
electron-builder ones the shape was first noticed on.

**Required** `version`, `desc`, `homepage`, `license`, `url64`

**Optional** `repo_url`, `arch`, `url_arm64`, `nsis_payload`, `shortcut_exe`,
`shortcut_name`, `bin_exe`, `bin_alias`, `env_set`, `env_add_path`, `persist`

**Key point** the URL must carry `#/dl.7z`. With one architecture the inner
payload defaults to `app-64.7z`; add `url_arm64` and the arm64 branch gets its
own `installer` entry pointing at `app-arm64.7z`, with a matching
per-architecture `autoupdate`. For a CLI payload use `bin_exe` and skip the
shortcut -- it is optional here.

**Samples** `azure-functions-core-tools`, `exercism`, `ghostscript`,
`git-annex`, `gpac`, `graphviz`, `iverilog`, `elm`

Output (single architecture):

```json
{
    "url": "https://github.com/o/r/releases/download/v1.2.3/App-1.2.3.exe#/dl.7z",
    "installer": {
        "script": "7z x $original_dir/`$PLUGINSDIR/app-64.7z -o\"$original_dir\""
    },
    "extract_dir": "$PLUGINSDIR",
    "extract_to": "PLUGINSDIR",
    "post_install": "Remove-Item -RECURSE $original_dir/`$PLUGINSDIR"
}
```

## github-innosetup

**Use when** the release is an InnoSetup `.exe`. Scoop unpacks it natively, so
**just set `"innosetup": true`** and do not write an `installer.script`.

**Required** `version`, `desc`, `homepage`, `license`, `url64`

**Optional** `repo_url`, `arch`, `url_arm64`, `shortcut_exe`,
`shortcut_name`, `bin_exe`, `bin_alias`, `env_set`, `env_add_path`, `persist`

**Samples** `dvc`, `espanso`, `get-iplayer`, `imagemagick`, `lynx`, `mercurial`,
`microsoft-coreutils`, `openssl`

## github-exe-installer

**Use when** the installer genuinely must run -- it writes registry keys,
installs drivers, or files have to move out of a subdirectory into `$dir`.

**Required** `version`, `desc`, `homepage`, `license`, `url64`, plus either
`installer_script` or `installer_file`

**Optional** `repo_url`, `installer_args`, `pre_install`, `uninstaller_script`,
`post_install`, `pre_uninstall`, `post_uninstall`, `shortcut_exe`,
`shortcut_name`, `bin_exe`, `bin_alias`, `persist`

**Note** do not add a `#/dl.*` fragment to the URL, or Scoop unpacks the
installer instead of running it. Use `installer_file` + `installer_args` when
the vendor's own silent flags are enough and no script is needed.

**Samples** `bind`, `bun`, `cygwin`, `erlang`, `gitea`, `go`, `gnupg`,
`git-machete`

## github-single-exe

**Use when** the release is one bare exe, usable as-is. This bucket's repos
publish one exe per architecture, so set `arch` to `64bit+arm64` and give
`url_arm64` as well; the default `arch=64bit` keeps the flat `url`/`hash` pair.

**Required** `version`, `desc`, `homepage`, `license`, `url64`

**Optional** `repo_url`, `arch`, `url_arm64`, `hash_arm64`,
`shortcut_exe`, `shortcut_name`, `bin_exe`, `bin_alias`, `env_set`,
`env_add_path`, `persist`, `suggest`, `notes`

**Samples** `android-cli`, `choose`, `gauth`, `json-tui`, `muscle`, `seqkit`,
`shimmy`, `wthrr`, `yutu`

## toolchain-env

**Use when** a compiler / SDK / runtime is never shimmed: the archive is
unpacked, one of its subdirectories goes on PATH with `env_add_path`, and one or
more `*_HOME` variables point at `$dir` with `env_set`. Upstream, 123 manifests
install nothing through `bin`, `shortcuts` or `psmodule`, and 88 of them reach
for `env_add_path`.

**Required** `version`, `desc`, `homepage`, `license`, `url64`, plus either
`env_add_path` or `env_set` -- `gen` refuses the recipe without one, because
there would be nothing to install.

**Optional** `repo_url`, `arch`, `url_arm64`, `extract_dir`,
`pre_install`, `uninstaller_script`, `bin_exe`, `bin_alias`, `persist`,
`suggest`, `notes`, `comment`

**Key point** when `extract_dir` embeds the version, `gen` also writes
`autoupdate.extract_dir` with `$version`; a vendor tarball is always named after
the release, so Excavator would otherwise bump the URL and keep the old
directory name.

**Samples** `ant`, `glfw`, `gnutls`, `ldc`, `libvips`, `llvm`, `maven`, `mingw`

```powershell
scoop_manifest.py gen --name ant --recipe toolchain-env `
  --version 1.10.18 --desc "Java build tool" `
  --homepage https://ant.apache.org/ --license Apache-2.0 `
  --url64 "https://dlcdn.apache.org/ant/binaries/apache-ant-1.10.18-bin.zip" `
  --extract-dir apache-ant-1.10.18 --env-add-path bin `
  --env-set ANT_HOME='$dir'
```

## webpage-regex

**Use when** upstream is not GitHub; the version comes from a web page or a
version endpoint that a plain regex can read.

**Required** `version`, `desc`, `homepage`, `license`, `url`, `checkver_regex`

**Optional** `checkver_url` (the page to scrape; **omit it to scrape
`homepage`**, which collapses `checkver` to the bare regex string), `url_au` (an
autoupdate template; when omitted the version in `url` is swapped for
`$version`), `au_hash_url` + `au_hash_regex` (hash from a checksum file),
`checkver_reverse` (take the last match), `checkver_xpath` (for XML pages),
`checkver_useragent`, `extract_dir`, `installer_script`, `innosetup`,
`shortcut_exe`, `bin_exe`, `bin_alias`, `persist`, `suggest`, `notes`

**Key point** the bare-string form is the one upstream uses 147 times and it is
what Scoop reads as "run this regex over `homepage`". It is only emitted when
the regex is the sole checkver key, because `replace` / `reverse` / `useragent`
need the object form. `ant` and `cacert` show the same recipe on vendor CDNs
rather than on GitHub releases.

**Samples** `ant`, `autoit`, `busybox`, `cacert`, `cmake`, `julia`, `msys2`,
`netcat`

```powershell
scoop_manifest.py gen --name myapp --recipe webpage-regex `
  --version 2.0.1 --desc "Data recovery utility" `
  --homepage https://example.com --license Shareware `
  --url "https://example.com/dl/MyApp-2.0.1.exe" `
  --checkver-url https://example.com/download.php `
  --checkver-regex "MyApp-([\d.]+)\.exe" `
  --shortcut-exe MyApp.exe --shortcut-name MyApp
```

## api-jsonpath

**Use when** upstream only offers an API, or only rolling builds, so the
version has to be picked out of a JSON document.

**Required** `version`, `desc`, `homepage`, `license`, `url`, `checkver_url`,
`checkver_jsonpath`, `checkver_regex`

**Optional** `checkver_replace` (rearranges named groups into a version),
`checkver_reverse`, `depends`, `installer_script`, `extract_dir`, `url_au`,
`au_hash_url` + `au_hash_regex`, `shortcut_exe`, `bin_exe`, `bin_alias`,
`persist`, `notes`

**Samples** `antigravity-cli`, `audiowaveform`, `bfg`, `chromedriver`, `dart`,
`devtunnel`, `dotnet-sdk`, `edgedb`

A commit-timestamp version, the shape `comfyui-manager` uses:

```json
"checkver": {
    "url": "https://api.github.com/repos/Comfy-Org/ComfyUI-Manager/commits/main",
    "jsonpath": "$.commit.committer.date",
    "regex": "(?<year>\\d{4})-(?<month>\\d{2})-(?<day>\\d{2})T(?<hour>\\d{2}):(?<min>\\d{2}):(?<sec>\\d{2})Z",
    "replace": "${year}.${month}.${day}.${hour}${min}${sec}"
}
```

## github-asset-jsonpath

**Use when** the release filename carries more than the version -- a build
number, a codename -- so swapping `$version` in the download URL is not enough.
Read the asset list from the GitHub API and carry the extra fields through
`$match*` placeholders.

**Required** `version`, `desc`, `homepage`, `license`, `url64`,
`checkver_url`, `checkver_jsonpath`, `checkver_regex`, `url_au`

**Optional** `arch`, `url_arm64`, `extract_dir`, `shortcut_exe`,
`shortcut_name`, `bin_exe`, `bin_alias`, `persist`, `suggest`, `notes`

**Key point** put the API endpoint in `checkver_url`, **never** in
`checkver.github`: Scoop appends `/releases/latest` to the `github` value
whatever it holds, which turns an API URL into a 404 (rule W111). 83 of the 1653
upstream manifests do this today. A named group `(?<build>...)` in
`checkver_regex` becomes `$matchBuild` in `url_au`; `(?<name>...)` becomes
`$matchName`.

**Samples** `aptos-cli`, `arc`, `aria2`, `biome`, `capstone`, `chroma`, `clink`,
`cloak`

```json
{
    "checkver": {
        "url": "https://api.github.com/repos/86Box/86Box/releases/latest",
        "jsonpath": "$.assets[*].browser_download_url",
        "regex": "v(?<version>[\\d.]+)/86Box-Windows-64-b(?<build>\\d+)\\.zip"
    },
    "autoupdate": {
        "url": "https://github.com/86Box/86Box/releases/download/v$matchVersion/86Box-Windows-64-b$matchBuild.zip"
    }
}
```

## github-source-archive

**Use when** the release is a source snapshot (`archive/refs/tags/v$version`)
rather than a built binary, or the archive expands to a version-stamped
directory that autoupdate has to re-template.

**Required** `version`, `desc`, `homepage`, `license`, `url64`, `extract_dir`

**Optional** `repo_url`, `hash64`, `au_hash_url` + `au_hash_regex`,
`pre_install`, `shortcut_exe`, `bin_exe`, `bin_alias`, `persist`, `depends`,
`suggest`, `notes`

**Key point** when `extract_dir` contains the version, `gen` also writes
`autoupdate.extract_dir` with `$version`, otherwise Excavator would bump the URL
and leave the directory name behind.

**Samples** `emscripten`, `memcached`, `offlineinsiderenroll`, `pdf2svg`,
`vcpkg`, `z.lua`

## github-msi

**Use when** the download is an `.msi`.

**Required** `version`, `desc`, `homepage`, `license`, `url64`

**Optional** `repo_url`, `msi_mode`, `msi_args`, `shortcut_exe`,
`shortcut_name`, `bin_exe`, `bin_alias`, `persist`, `suggest`, `notes`

**Key point** name the file with a `#/*.msi_` fragment (`#/dl.msi_`,
`#/setup.msi_`). The trailing `_` tells Scoop not to unpack it. Then pick a
mode:

- `msi_mode=extract` (default) -- `Expand-MsiArchive` into `$dir`, so the
  package stays portable and needs no admin rights
- `msi_mode=install` -- hand it to `msiexec /i`, and `msiexec /x` in
  `pre_uninstall`. Both steps check `is_admin` and re-launch elevated

**Samples** `7zip`, `aws-sam-cli`, `cppcheck`, `espeak-ng`, `fio`, `juliaup`,
`kalker`, `kiro-cli`

## powershell-gallery

**Use when** the package is a PowerShell module published to the PowerShell
Gallery as a `.nupkg`, installed through the `psmodule` block instead of
`bin` / `shortcuts`.

**Required** `version`, `desc`, `homepage`, `license`, `url`, `checkver_url`,
`checkver_regex`, `psmodule_name`

**Optional** `psmodule_path`, `pre_install`, `url_au`

**Key point** the default `pre_install` drops the NuGet packaging leftovers
(`_rels`, `package`, `*Content*.xml`) so Scoop treats the directory as a plain
module folder. `checkver_url` is the gallery page for the module.

**Samples** `acmesharp`, `gsudo`, `importexcel`, `pester`,
`powershell-beautifier`, `powershell-yaml`, `psgithub`, `winget-ps`

## checkver-script

**Use when** the version only becomes visible after a request the manifest
itself has to make: following a redirect, or calling an endpoint whose URL is
computed at run time.

**Required** `version`, `desc`, `homepage`, `license`, `url`,
`checkver_script`, `checkver_regex`

**Optional** `url_au`, `au_hash_url` + `au_hash_regex`, `extract_dir`,
`pre_install`, `shortcut_exe`, `bin_exe`, `bin_alias`, `persist`, `notes`

**Key point** the script's return value is the text `checkver_regex` runs
against. It needs a Scoop environment, so `update --checkver` reports that it
cannot probe it offline and points at `bin/checkver.ps1`.

**Samples** `cangjie`, `castxml`, `edgedriver`, `ijhttp`, `pnpm`, `postgresql`,
`qrencode`, `selenium-manager`

## sourceforge

**Use when** upstream lives on SourceForge. The `sourceforge` checkver key
queries the project RSS on its own, so no page scraping is involved.

**Required** `version`, `desc`, `homepage`, `license`, `url`,
`checkver_sourceforge`, `checkver_regex`

**Optional** `url_au`, `au_hash_url` + `au_hash_regex`, `extract_dir`,
`pre_install`, `shortcut_exe`, `bin_exe`, `bin_alias`, `persist`, `notes`

**Key point** only four upstream manifests use the dedicated key, while 23
download straight from a SourceForge mirror and land in `webpage-regex` or
`github-cli-archive` instead. Prefer the dedicated key: it survives a project
page redesign.

**Samples** `boost`, `gdisk`, `rhash`, `uncrustify`

## redirect-arch

**Use when** upstream serves a permanent `latest/redirect`-style link that
contains **no version number**. The URL never changes, so the version must be
probed elsewhere (e.g. the winget-pkgs commit history).

**Required** `version`, `desc`, `homepage`, `license`, `url64`, `url_arm64`,
`checkver_url`, `checkver_regex`, `shortcut_exe`

**Note** such a manifest **carries no `hash`** -- the content behind the same
URL changes, so a pinned hash would fail as soon as upstream ships. 18 upstream
manifests use a versionless link; check whether the vendor also offers a
versioned one before reaching for this recipe.

**Samples** `android-cli`, `dart`, `ddev`, `istioctl`, `nuclei`,
`quick-lint-js`, `sonar-scanner`, `spotdl`

## portable-multifile

**Use when** one package needs several downloads: the main archive plus a
helper script, an icon, or a hotfix archive. `url` and `hash` become arrays.

**Required** `version`, `desc`, `homepage`, `license`, `url64`, `hash64`,
`extra_urls`, `extra_hashes`

**Optional** `extract_dir`, `pre_install`, `post_install`, `shortcut_exe`,
`shortcut_name`, `bin_entries`, `persist`, `env_add_path`, `env_set`,
`suggest`, `notes`

**Key point** the two arrays must line up one for one; `gen` refuses a mismatch
instead of writing a manifest Scoop cannot verify. `autoupdate` tracks only the
main download, because Scoop cannot re-derive a sidecar URL.

**Samples** `diffutils`, `jsign`, `pdfbox`, `pkg-config`, `say`, `shasum`

## github-git-clone

**Use when** the package is not an executable but a plugin / extension for a
host application and must be `git clone`d into the host directory.

**Required** `version`, `desc`, `homepage`, `license`, `url`, `depends`,
`installer_script`, `checkver_url`, `checkver_jsonpath`, `checkver_regex`

**Optional** `checkver_replace`, `extract_dir`, `uninstaller_script`,
`post_uninstall`, `notes`

**Samples** `comfyui-manager` (in the sibling Extras-Plus bucket; no manifest
here or upstream has this shape)

## Adding a recipe

1. Add an entry to the `recipes` array in `assets/recipes.jsonc` with `id` /
   `label` / `when` / `builder` / `required` / `optional` / `refs`, and add the
   new parameters to `param_docs`.
2. Register a builder of the same name in `BUILDERS` in `scripts/sm_lib.py`.
3. Add a `## <recipe id>` section to this document.
4. Run `python scripts/sm_selftest.py` -- it catches four kinds of drift:
   no builder for a recipe, an unreferenced builder, an undocumented required
   parameter, and a missing doc section.
