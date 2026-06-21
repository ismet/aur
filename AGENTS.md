# AGENTS.md — ismet/aur (AUR mirror)

Personal AUR mirror. Each subdirectory is an AUR package. Only `command-code/` currently.

## Layout

Only `command-code/{PKGBUILD,.SRCINFO,LICENSE}` pushed to AUR. Workflows, README, AGENTS.md stay on GitHub only — AUR rejects unexpected files.

`command-code/LICENSE` is the upstream Terms of Service (Langbase, Inc. d/b/a Command Code), **not** an open-source license. The AUR `license=('LicenseRef-command-code')` is correct; do not change it to `0BSD` or similar.

## CI workflow chain

- **check-upstream.yml** — daily cron 04:00 UTC: compare pkgver vs npm registry, rewrite PKGBUILD with `updpkgsums`, regenerate `.SRCINFO`, open PR.
- **pr-verify.yml** — PRs touching `command-code/`: `makepkg` in `archlinux:base-devel` container, auto-merge on success.
- **publish.yml** — push to main: deploy to AUR via SSH (requires secrets: `AUR_USERNAME`, `AUR_EMAIL`, `AUR_SSH_PRIVATE_KEY`).

## Build & verify

Local build needs `base-devel`, `nodejs`, `npm`, `jq` (the first two from the repo, the latter two are makedepends):

```bash
sudo pacman -S --needed base-devel nodejs npm jq
cd command-code
rm -rf src pkg
makepkg -f
sudo pacman -U command-code-*-x86_64.pkg.tar.zst
cmd --version
ls /usr/share/licenses/command-code/LICENSE
```

Regenerate `.SRCINFO` after editing PKGBUILD:
```bash
cd command-code && makepkg --printsrcinfo > .SRCINFO
```

## Manual AUR push (fallback)

The CI chain (`check-upstream.yml` → `pr-verify.yml` → `publish.yml`) is the primary path. Use this only if that chain is broken and you need to push `PKGBUILD` / `.SRCINFO` / `LICENSE` directly to AUR.

```bash
git clone ssh://aur@aur.archlinux.org/command-code.git /tmp/cc
cp command-code/{PKGBUILD,.SRCINFO,LICENSE} /tmp/cc/
cd /tmp/cc && makepkg --printsrcinfo > .SRCINFO
git add . && git commit -m "command-code: update to X.Y.Z" && git push
```

## PKGBUILD gotchas

Non-obvious. Don't remove when refactoring.

- **Wrapper `update` intercept** — `package()` deletes the four npm-installed `usr/bin/*` symlinks (`cmd`, `cmdc`, `command-code`, `commandcode`) and reinstalls them as bash wrappers. The wrapper exits 0 on `update` and otherwise `exec`s the real binary with `COMMANDCODE_SKIP_UPDATES=1`. Prevents the package from self-upgrading out from under pacman. The env var alone is not sufficient — upstream also gates updates on `isLocalDevelopmentBuild()`.
- **`noextract=("${pkgname}-${pkgver}.tgz")`** — npm handles extraction via `--prefix`; without this makepkg extracts and npm re-extracts.
- **`options=('!strip')`** — prevents stripping ELF from `sharp` native modules (pointless for JS).
- **`--allow-scripts sharp --allow-scripts protobufjs`** — both need install scripts (sharp downloads binaries, protobufjs generates code).
- **`_where` cleanup** — `find "$pkgdir" -name package.json -print0 | xargs -r -0 sed -i '/\_where/d'` removes npm-embedded build paths.
- **jq underscore regex** — use `test("^_")` not `\_.+` (Arch wiki pattern fails on current jq).
- **`LICENSE` in `source=`** — needed so it reaches `$srcdir` for `install -Dm644`. Use `SKIP` checksum if fetched from URL.

## Build artifacts (do not commit)

`.gitignore` covers `*.tgz` and `*.pkg.tar.zst`. Also avoid committing `command-code/src/` and `command-code/pkg/` (makepkg working dirs). The `.commandcode/` dir is runtime state produced by the `command-code` tool itself when run in this repo — ignored by the user-level `.*` rule, not by this repo.
