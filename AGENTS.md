# AGENTS.md — command-code AUR Package

AUR package for [Command Code](https://commandcode.ai), an AI coding agent distributed as a npm tarball. Maintainer: Ismet Togay <ismet.togay@gmail.com>. License: `LicenseRef-command-code` (proprietary).

Upstream version: **0.38.4** (pkgrel 1).

## Layout

The repo is a personal AUR mirror with one package per subdirectory. For `command-code`:

```
.
├── .github/workflows/
│   ├── check-upstream.yml   # cron: poll npm, open PR on new version
│   ├── pr-verify.yml        # PR:  run makepkg to verify, then auto-merge
│   └── publish.yml          # push to main: stage command-code/* and push to AUR
├── command-code/            # the AUR package — must be pushable to AUR as-is
│   ├── PKGBUILD
│   ├── .SRCINFO
│   ├── LICENSE              # 0BSD for the PKGBUILD itself
│   └── command-code.license # upstream ToS, installed at /usr/share/licenses/command-code/LICENSE
├── AGENTS.md                # this file
├── LICENSE                  # n/a — moved into command-code/
└── README.md                # mirror-level docs (not pushed to AUR)
```

**Hard rule:** the deploy action pushes only `command-code/` to AUR. `git push aur master` from this repo is no longer safe — workflow files would be rejected, and the AUR server is strict about file locations. Use the GitHub workflow.

## What the package installs

Four wrapper scripts in `/usr/bin/` (`cmd`, `cmdc`, `command-code`, `commandcode`) that all `exec` `/usr/lib/node_modules/command-code/dist/index.mjs` with `COMMANDCODE_SKIP_UPDATES=1`. The npm tarball ships 4 symlinks pointing at the same entry; we delete them and install wrappers.

## Build & verify

```bash
cd command-code
rm -rf src pkg                  # clean any prior build
makepkg -f                      # build
sudo pacman -U command-code-*-x86_64.pkg.tar.zst
cmd --version                   # any of the 4 bin names works
ls /usr/share/licenses/command-code/LICENSE
```

The `pr-verify.yml` workflow runs the same `makepkg` step on a GitHub Actions runner. If the user wants to verify on a specific arch or with custom flags, edit the workflow.

## Version bump (automated)

Routine version bumps are fully automated:

1. `check-upstream.yml` runs daily at 04:00 UTC (also `workflow_dispatch`-able).
2. It compares the current `pkgver` in `command-code/PKGBUILD` to `https://registry.npmjs.org/command-code/latest`.
3. On a new version, it rewrites `pkgver`/`pkgrel`/`sha256sums` via `updpkgsums`, regenerates `.SRCINFO`, and opens a PR titled `command-code: update to X.Y.Z`.
4. `pr-verify.yml` builds the package. On success, it enables GitHub auto-merge on the PR.
5. The PR merges to `main`. `publish.yml` then stages `command-code/PKGBUILD`, `.SRCINFO`, and the two license files and pushes them to AUR over SSH.

**Escape hatch (CI is broken):** manually edit `command-code/PKGBUILD` (new `pkgver`, `pkgrel=1`, recompute `sha256sums`), regenerate `.SRCINFO`, commit, push to `github main` with `[skip ci]` to bypass the workflows, then push the package files to AUR directly from a separate clone (e.g., `git clone ssh://aur@aur.archlinux.org/command-code.git /tmp/cc && cp command-code/{PKGBUILD,.SRCINFO,command-code.license,LICENSE} /tmp/cc/ && cd /tmp/cc && makepkg --printsrcinfo > .SRCINFO && git add . && git commit -m "..." && git push`).

## Gotchas (PKGBUILD internals)

These are the non-obvious bits. Don't remove them when refactoring the PKGBUILD.

**Why wrapper scripts intercept `update`.** `performAutoUpdate()` in the upstream code only checks `isLocalDevelopmentBuild()` — the `COMMANDCODE_SKIP_UPDATES` env var alone does **not** block it. The wrappers short-circuit at the shell level: any `update` subcommand prints a message and exits 0. Without this, the AUR install would self-upgrade out from under the package manager.

**`_where` cleanup.** npm embeds `$srcdir`/`$pkgdir` paths in the `_where` attribute of every `package.json` under the install tree. Remove them:

```bash
find "$pkgdir" -name package.json -print0 | xargs -r -0 sed -i '/\_where/d'
```

**jq regex for underscore-prefixed keys.** The Arch wiki's `\_.+` fails to compile on current jq. Use `^_`:

```bash
jq '.|=with_entries(select(.key|test("^_")|not))' "$pkgjson"
```

**`options=('!strip')`.** Without it, makepkg strips ELF binaries out of `sharp`'s native modules — slow and pointless for a JavaScript package.

**`noextract=("${pkgname}-${pkgver}.tgz")`.** npm handles extraction via `--prefix`; without `noextract`, makepkg extracts the tarball into `$srcdir` and npm re-extracts it again.

**`--allow-scripts sharp --allow-scripts protobufjs`.** Suppresses `npm warn allow-scripts` for the two packages that need install scripts (`sharp` downloads native binaries, `protobufjs` generates code). Using `--userconfig` is the older approach and was replaced in commit `9a2094a`.

**`command-code.license` must be in `source=`.** Otherwise it won't be copied to `$srcdir` and the `install -Dm644` line in `package()` fails. Use `SKIP` for its checksum.

## Working tree

Untracked files are build/runtime artifacts, **not** source:

- `command-code/command-code-*.tgz` — the downloaded npm tarball (after build)
- `command-code/command-code-*-x86_64.pkg.tar.zst` — built package
- `command-code/src/`, `command-code/pkg/` — makepkg build dirs
- `.commandcode/` — runtime data from using the tool (already ignored via the global `.*` rule)

`.gitignore` covers the first two. Never `git add` them; never commit the tarball alongside `PKGBUILD`.

## References

- [PKGBUILD(5)](https://man.archlinux.org/man/PKGBUILD.5)
- [Node.js package guidelines](https://wiki.archlinux.org/title/Node.js_package_guidelines)
- [Nonfree applications package guidelines](https://wiki.archlinux.org/title/Nonfree_applications_package_guidelines)
- [AUR submission](https://wiki.archlinux.org/title/Arch_User_Repository#Submitting_packages)
