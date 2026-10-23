# sfg_pybind11_stubgen

Backport builds of [`pybind11-stubgen`](https://github.com/pybind/pybind11-stubgen) for Ubuntu releases that do not ship it.

Debian already packages `pybind11-stubgen` **3.0.0-1**. Instead of writing packaging from scratch, this repository rebuilds that exact Debian source package with a backport version suffix (`3.0.0-1~jammy1`, `3.0.0-1~noble1`).

Upstream packaging lives in Debian:

- Source package page: <https://packages.debian.org/source/forky/pybind11-stubgen>
- Packaging repository: <https://salsa.debian.org/science-team/pybind11-stubgen>
- Source files (pool): <https://deb.debian.org/debian/pool/main/p/pybind11-stubgen/>

Everything lives in one file: `.github/workflows/build_deb.yml`.

## Supported releases

Upstream requires Python >= 3.10, so jammy is the floor.

| Release       | System Python | Supported |
| ------------- | ------------- | --------- |
| jammy (22.04) | 3.10          | yes       |
| noble (24.04) | 3.12          | yes       |
| focal (20.04) | 3.8           | no        |

## Getting the packages

Run the **Build Debian Packages** workflow, or push to `main`. Each matrix entry produces an artifact `pybind11-stubgen-<release>` containing two `.deb`s:

```bash
sudo apt install ./python3-pybind11-stubgen_3.0.0-1~jammy1_all.deb \
                 ./pybind11-stubgen_3.0.0-1~jammy1_all.deb
```

### Why two packages?

That split is Debian's, not ours: `debian/control` declares two binary stanzas and two `.install` files route the files.

| Package                    | Contents                                          | Depends                                   |
| -------------------------- | ------------------------------------------------- | ----------------------------------------- |
| `python3-pybind11-stubgen` | `/usr/lib/python3/dist-packages/pybind11_stubgen` | `python3:any`                             |
| `pybind11-stubgen`         | `/usr/bin/pybind11-stubgen`                       | `python3-pybind11-stubgen`, `python3:any` |

Per Debian Python policy, `python3-*` is the importable library and the unprefixed name is the user-facing tool. Installing the CLI pulls in the library, so installing `pybind11-stubgen` alone is enough.

## Workflow structure

| Job          | Container          | Purpose                                 |
| ------------ | ------------------ | --------------------------------------- |
| `build`      | `debian:trixie`    | Rebuild the Debian source package       |
| `smoke-test` | `ubuntu:<release>` | Install the `.deb` and verify it works  |
| `submit`     | —                  | Publish the `.deb`s to the SFG APT repo |

`build` and `smoke-test` run inside a container image (no `docker run`); `submit` runs on the host because it only talks to the GitHub API, and only on pushes to `main`.

`defaults.run.shell` is `bash --noprofile --norc -euo pipefail {0}`, because container jobs default to `sh`, which lacks arrays and `pipefail`. The built-in `bash` keyword already implies `-eo pipefail`; the template adds `-u` so an unset variable fails loudly.

### `build`

1. **Resolve Versions** — derives `SOURCE_VERSION` from `UPSTREAM_VERSION` and `DEBIAN_REVISION` via `$GITHUB_ENV`.
2. **Install Build Dependencies.**
3. **Fetch and Verify Source** — downloads the `.dsc`, checks it against a pinned SHA-256, then downloads the tarballs it names and extracts with `dpkg-source -x`.
4. **Build Package** — patches `debian/control` and `debian/changelog`, runs `dpkg-buildpackage -us -uc -b`, and copies the `.deb`s to `dist/` (`dpkg-buildpackage` writes to the directory *above* the source tree).

### `smoke-test`

Installs the artifact in `ubuntu:<release>` and checks that the module imports and the CLI starts, against the real target Python.

### `submit`

One job per release, because the APT repository accepts a single distribution per run. Each leg passes `artifact_name: pybind11-stubgen-<release>` to the `submit_packages` action from `sfg_apt_repo`, so the packages are not uploaded twice. The `.deb`s must be at the **root** of the artifact — see *Artifact Layout* in the `sfg_apt_repo` README.

## Design notes

- **Version suffix.** `3.0.0-1~jammy1` sorts *below* the real `3.0.0-1`, so if a future Ubuntu release ships the package, the distro version wins and apt upgrades to it.
- **`X-Python3-Version: >= 3.10`.** `dh_python3` can emit a constraint such as `python3:any (>= 3.13)` when built on a newer Debian, which would make the package uninstallable on jammy and noble. The field caps it, and is a no-op when no constraint would have been added.
- **Tests are disabled.** The Debian packaging sets `PYBUILD_DISABLE := test`; the upstream tests are integration tests needing compiled pybind11 modules. The smoke test only checks install, import, and CLI startup.

## Updating to a newer upstream version

1. Find the new version on the [source package page][src] or in the [pool][pool].
2. Update `UPSTREAM_VERSION` (and `DEBIAN_REVISION`, if Debian bumped it) in the workflow `env:` block.
3. Update `DSC_SHA256`. Only the `.dsc` needs pinning — it carries the checksums of both tarballs, and `dpkg-source -x` verifies them while unpacking.
4. Push and let CI rebuild.

[src]: https://packages.debian.org/source/forky/pybind11-stubgen
[pool]: https://deb.debian.org/debian/pool/main/p/pybind11-stubgen/
