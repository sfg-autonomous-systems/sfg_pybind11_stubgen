# SFG `pybind11-stubgen`

![Ubuntu](https://img.shields.io/badge/Ubuntu-jammy%20%7C%20noble-E95420?logo=ubuntu&logoColor=white)

Backport builds of [`pybind11-stubgen`](https://github.com/pybind/pybind11-stubgen) for Ubuntu releases that do not ship it, rebuilt from Debian's **3.0.0-1** source package with a backport version suffix (`3.0.0-1~jammy1`, `3.0.0-1~noble1`).

## Debian Packaging

- [Source Package Page](https://packages.debian.org/source/forky/pybind11-stubgen)
- [Packaging Repository](https://salsa.debian.org/science-team/pybind11-stubgen)

### Updating to a Newer Upstream Version

1. Find the new version on the [source package page](https://packages.debian.org/source/forky/pybind11-stubgen).
2. Update `UPSTREAM_VERSION` (and `DEBIAN_REVISION`, if Debian bumped it) in the workflow `env:` block.
3. Update `DSC_SHA256`. Only the `.dsc` needs pinning — it carries the checksums of both tarballs, and `dpkg-source -x` verifies them while unpacking.
4. Push and let CI rebuild.

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

