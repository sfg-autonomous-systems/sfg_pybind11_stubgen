# SFG `pybind11-stubgen`

[![Version][version-badge]][workflow] [![Ubuntu][ubuntu-badge]][apt-repo] [![Build][build-badge]][build] 

[version-badge]: https://img.shields.io/badge/dynamic/yaml?url=https%3A%2F%2Fraw.githubusercontent.com%2Fsfg-autonomous-systems%2Fsfg_pybind11_stubgen%2Fmain%2F.github%2Fworkflows%2Fbuild_deb.yaml&query=%24.env.UPSTREAM_VERSION&label=pybind11-stubgen&color=A81D33&logo=debian&logoColor=white
[ubuntu-badge]: https://img.shields.io/badge/dynamic/yaml?url=https%3A%2F%2Fraw.githubusercontent.com%2Fsfg-autonomous-systems%2Fsfg_pybind11_stubgen%2Fmain%2F.github%2Fworkflows%2Fbuild_deb.yaml&query=%24.jobs.build.strategy.matrix.release%5B*%5D&label=Ubuntu&color=E95420&logo=ubuntu&logoColor=white
[build-badge]: https://img.shields.io/github/actions/workflow/status/sfg-autonomous-systems/sfg_pybind11_stubgen/build_deb.yaml?branch=main&label=build&logo=githubactions&logoColor=white

[workflow]: .github/workflows/build_deb.yaml
[build]: https://github.com/sfg-autonomous-systems/sfg_pybind11_stubgen/actions/workflows/build_deb.yaml?query=branch%3Amain
[apt-repo]: https://github.com/sfg-autonomous-systems/sfg_apt_repo

Backport builds of [`pybind11-stubgen`](https://github.com/pybind/pybind11-stubgen) for Ubuntu releases that do not ship it, rebuilt from Debian's source package with a backport version suffix (e.g. `3.0.0-1~jammy1`, `3.0.0-1~noble1`).

## Debian Packaging

- [Source Package Page](https://packages.debian.org/source/forky/pybind11-stubgen)
- [Packaging Repository](https://salsa.debian.org/science-team/pybind11-stubgen)

### Updating to a Newer Upstream Version

1. Find the new version on the [source package page](https://packages.debian.org/source/forky/pybind11-stubgen).
2. Update `UPSTREAM_VERSION` (and `DEBIAN_REVISION`, if Debian bumped it) in the workflow `env:` block.
3. Update `DSC_SHA256`. Only the `.dsc` needs pinning, it carries the checksums of both tarballs, and `dpkg-source -x` verifies them while unpacking.
4. Push and let CI rebuild.

**A Note on Sid/Testing (Forky) Packages:**
Debian only keeps the single newest package revision in the active pool for non-stable releases. Once a new revision (e.g., `-2`) is accepted, the older `.dsc` (e.g., `-1`) is instantly purged from the main mirrors.

### Why Two Packages?

That split is Debian's, not ours: `debian/control` declares two binary entries and two `.install` files route the files.

| Package                    | Contents                                          | Depends                                   |
| -------------------------- | ------------------------------------------------- | ----------------------------------------- |
| `python3-pybind11-stubgen` | `/usr/lib/python3/dist-packages/pybind11_stubgen` | `python3:any`                             |
| `pybind11-stubgen`         | `/usr/bin/pybind11-stubgen`                       | `python3-pybind11-stubgen`, `python3:any` |

Per Debian Python policy, `python3-pybind11-stubgen` is the importable library and `pybind11-stubgen` is CLI tool. Installing the CLI pulls in the library, so installing `pybind11-stubgen` alone is enough.

## Workflow Jobs

- **`build`:** Rebuilds the pinned Debian source package in a `debian:trixie` container. This works because it is a pure Python package with no additional runtime dependencies.
- **`smoke-test`:** Installs and checks the packages in each target Ubuntu release.
- **`submit`:** Publishes each release to the SFG APT repository on pushes to `main`. The repository accepts one distribution per run, so there is one job per release.

## Adding an Ubuntu Release

Add its codename to `matrix.release` in `.github/workflows/build_deb.yaml`.
