# ismet/aur

Personal AUR mirror. One subdirectory per AUR package, plus GitHub Actions that keep
each package in sync with its upstream.

## Packages

| Package        | Subdirectory      | Upstream                          | Auto-bump |
|----------------|-------------------|-----------------------------------|-----------|
| `command-code` | `command-code/`   | <https://commandcode.ai> (npm)    | daily     |

## Layout

```
.
├── .github/workflows/   # CI: check-upstream, pr-verify, publish (one set per pkg)
├── command-code/        # the AUR package
│   ├── PKGBUILD
│   ├── .SRCINFO
│   └── LICENSE          # upstream ToS, installed to /usr/share/licenses/
├── AGENTS.md            # maintainer notes for AI agents
└── README.md            # you are here
```

The AUR remote is a strict allow-list. Each package lives in its own subdirectory so
the `publish` workflow can stage only that subdirectory onto AUR. Pushing `.github/`,
`README.md`, or `AGENTS.md` to AUR would be rejected — the deploy action handles
this by staging only `pkgbuild` + `assets`.

## How the automation works

For each AUR package, three workflows run in sequence:

1. **`check-upstream.yml`** — runs daily. Compares the `pkgver` in `PKGBUILD` to the
   latest upstream version. If different, rewrites `pkgver`/`pkgrel`/`sha256sums`
   and opens a PR.
2. **`pr-verify.yml`** — runs on every PR that touches the package. Runs `makepkg`
   in an Arch container; on success, enables auto-merge on the PR.
3. **`publish.yml`** — runs on push to `main` for the package. Uses
   `KSXGitHub/github-actions-deploy-aur` to push the package files onto AUR over SSH.

To add a new package: copy the three workflow files into a new subdirectory under
`.github/workflows/`, prefix each filename with the package name, point them at the
new subdirectory, and add matching `secrets.*` for the new AUR maintainer.

## Required GitHub secrets

| Secret                  | Purpose                                         |
|-------------------------|-------------------------------------------------|
| `AUR_USERNAME`          | AUR account username that owns the packages     |
| `AUR_EMAIL`             | Email used in AUR commit author field           |
| `AUR_SSH_PRIVATE_KEY`   | Private half of the SSH keypair whose **public** half is registered on the AUR account |

The public key MUST be added to the AUR account at
<https://aur.archlinux.org> → My Account → SSH Public Keys. Generate a dedicated
keypair — do not reuse your personal one.

## Local development

```bash
# The AUR remote stays on this repo for now but is no longer pushed to manually.
# Edits flow: local → github (this mirror) → AUR (via publish workflow).

cd command-code
makepkg --printsrcinfo > .SRCINFO    # always after editing PKGBUILD
git add command-code/
git commit -m "command-code: ..."
git push github main
```

## See also

- `AGENTS.md` — non-obvious build details (`--allow-scripts`, `_where` cleanup, etc.)
