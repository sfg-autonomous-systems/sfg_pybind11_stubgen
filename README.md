# SFG pybind11-stubgen

Backport builds of [`pybind11-stubgen`](https://github.com/pybind/pybind11-stubgen) for Ubuntu releases that do not ship it.

Debian already packages `pybind11-stubgen` **3.0.0-1**. Instead of writing packaging from scratch, this repository rebuilds that exact Debian source package with a backport version suffix (`3.0.0-1~jammy1`, `3.0.0-1~noble1`).

Debian Packaging Resources:

- [Source package page](https://packages.debian.org/source/forky/pybind11-stubgen)
- [Packaging repository](https://salsa.debian.org/science-team/pybind11-stubgen)
- [Source files (pool)](https://deb.debian.org/debian/pool/main/p/pybind11-stubgen/)

## Supported Releases

Upstream requires Python >= 3.10, so jammy (Ubuntu 22.04) is the oldest supported version. Currently packages are built for jammy and noble (Ubuntu 24.04).

## Getting the Packages

Run the **Build Debian Packages** workflow, or push to `main`. Each matrix entry produces an artifact `pybind11-stubgen-<release>` containing two `.deb`s:

```bash
sudo apt install ./python3-pybind11-stubgen_3.0.0-1~jammy1_all.deb \
                 ./pybind11-stubgen_3.0.0-1~jammy1_all.deb
```

Or install the packages from [`sfg_apt_repo`](https://github.com/sfg-autonomous-systems/sfg_apt_repo#sfg-apt-repo).

### Why Two Packages?

That split is Debian's, not ours: `debian/control` declares two binary stanzas and two `.install` files route the files.

| Package                    | Contents                                          | Depends                                   |
| -------------------------- | ------------------------------------------------- | ----------------------------------------- |
| `python3-pybind11-stubgen` | `/usr/lib/python3/dist-packages/pybind11_stubgen` | `python3:any`                             |
| `pybind11-stubgen`         | `/usr/bin/pybind11-stubgen`                       | `python3-pybind11-stubgen`, `python3:any` |

Per Debian Python policy, `python3-*` is the importable library and the unprefixed name is the user-facing tool. Installing the CLI pulls in the library, so installing `pybind11-stubgen` alone is enough.

## Workflow

- **Build**: Rebuilds the pinned Debian source package in the newer `debian:trixie` container; this works because it is a pure Python package with no additional runtime dependencies.
- **Smoke Test**: Installs and checks the packages in each target Ubuntu release.
- **Submit**: Publishes each release to the SFG APT repository on pushes to `main`; the repository accepts one distribution per run, so there is one job per release.

## Design Notes

- **Version suffix.** `3.0.0-1~jammy1` sorts *below* the real `3.0.0-1`, so if a future Ubuntu release ships the package, the distro version wins and apt upgrades to it.
- **`X-Python3-Version: >= 3.10`.** `dh_python3` can emit a constraint such as `python3:any (>= 3.13)` when built on a newer Debian, which would make the package impossible to install on jammy and noble. The field caps it, and is a no-op when no constraint would have been added.
- **Tests are disabled.** The Debian packaging sets `PYBUILD_DISABLE := test`; the upstream tests are integration tests needing compiled pybind11 modules. The smoke test only checks install, import, and CLI startup.

## Adding an Ubuntu Release

1. Confirm the release ships Python >= 3.10.
2. Add its codename to `matrix.release` in `.github/workflows/build_deb.yml`.
3. Add the release and its Ubuntu version to **Supported Releases** above.
4. Push the change and check that build, smoke-test, and submit pass for the new matrix entry.

The workflow derives the package suffix from the codename. No other workflow changes are needed.

## Updating to a Newer Upstream Version

1. Find the new version on the [source package page][src] or in the [pool][pool].
2. Update `UPSTREAM_VERSION` (and `DEBIAN_REVISION`, if Debian bumped it) in the workflow `env:` block.
3. Update `DSC_SHA256`. Only the `.dsc` needs pinning — it carries the checksums of both tarballs, and `dpkg-source -x` verifies them while unpacking.
4. Push and let CI rebuild.

[src]: https://packages.debian.org/source/forky/pybind11-stubgen
[pool]: https://deb.debian.org/debian/pool/main/p/pybind11-stubgen/
