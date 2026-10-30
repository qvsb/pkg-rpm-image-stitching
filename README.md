<!--
Copyright (c) Qualcomm Technologies, Inc. and/or its subsidiaries.
SPDX-License-Identifier: BSD-3-Clause
-->
# pkg-rpm-qcom-image-stitching

RPM packaging for the Qualcomm dynamic image-stitching library and its public
development headers.

This repository packages the `qcom-image-stitching` RPM from the
`image-stitching` prebuilt binary release. The
runtime package contains the versioned `libimage_stitching.so.1` shared
library. The `qcom-image-stitching-devel` package contains the public
`image_stitching.h` and `mesh_fusion.h` headers and the `libimage_stitching.so`
linker symlink.

The prebuilt archive is fetched during the RPM build from Qualcomm Software
Center. This repository contains no compiled libraries, binaries, or source
archives. The `iot-core-algs.lnx.0.0` component release dated `261029` is
referenced by `Source0` in the spec, and the checksum in `sources` is verified
before the RPM is built.

## Repository Layout

| File | Purpose |
|---|---|
| `qcom-image-stitching.spec` | Builds the runtime and development RPM packages. |
| `sources` | SHA-512 checksum for the prebuilt image-stitching archive. |
| `.github/workflows/build-on-pr.yml` | Builds the RPM packages for pull requests. |
| `.github/workflows/pkg-release.yml` | Builds and publishes release RPMs. |
| `README.md` | Package and repository documentation. |
| `LICENSE.txt` | License for the RPM packaging repository. |

## Packages

### `qcom-image-stitching`

Runtime package containing the prebuilt image-stitching shared library
`/usr/lib64/libimage_stitching.so.1`. The shared library uses
OpenCV and OpenCL runtime libraries supplied by the target Qualcomm Linux
image; its ELF dependencies are recorded automatically by RPM.

### `qcom-image-stitching-devel`

Development package providing the public headers in `/usr/include` and the
`/usr/lib64/libimage_stitching.so` linker symlink.

The packages are built for `aarch64`; the vendor archive is not compiled or
stripped during the RPM build.

## Installation

Install the runtime and development packages from the configured CentOS Stream
10 repository:

```bash
sudo dnf install -y qcom-image-stitching qcom-image-stitching-devel
```

## Local usage

The `build-on-pr.yml` build can be reproduced on an **aarch64** host with
`docker`, `git`, `curl`, and `bash`. The steps below mirror the
`pkg-build-reusable-workflow.yml` job in
[`qcom-rpm-utils`](https://github.com/qualcomm-linux/qcom-rpm-utils). Step 1
checks out the `c10s` branch; run the later steps from the **repository root**.

1. Clone this repository, check out the `c10s` packaging branch, then fetch the
   build tooling and build the builder image. CI pulls the same image from GHCR;
   building it locally avoids the GHCR login:

   ```sh
   git clone https://github.com/qualcomm-linux/pkg-rpm-image-stitching.git
   cd pkg-rpm-image-stitching
   git checkout c10s
   git clone https://github.com/qualcomm-linux/qcom-rpm-utils.git tool
   docker build -f tool/docker/Dockerfile.rpm-builder -t rpm-builder:centos10 tool/docker/
   ```

2. Resolve the source archive against `sources`. `resolve-sources.sh` appends
   to `$GITHUB_OUTPUT`, so start from an empty file:

   ```sh
   export GITHUB_OUTPUT="$PWD/resolve_sources_output.env"
   : > "$GITHUB_OUTPUT"
   tool/scripts/resolve-sources.sh \
     --sources sources \
     --spec qcom-image-stitching.spec \
     --cache-base-url https://dummy_url \
     --dest "$PWD/sources-cache"
   # tarballs= is an unquoted, space-separated list; sourcing the file would
   # run every path after the first as a command, so read the values instead.
   tarball="$(sed -n 's/^tarball=//p' "$GITHUB_OUTPUT")"
   tarballs="$(sed -n 's/^tarballs=//p' "$GITHUB_OUTPUT")"
   ```

   The lookaside cache is tried first. On a miss, the archive is downloaded from
   the spec's `Source0` (Qualcomm Software Center), and in both cases it is verified against
   the checksum in `sources`.

3. Build:

   ```sh
   tool/scripts/build-rpm.sh \
     --tarball "$tarballs" \
     --spec qcom-image-stitching.spec \
     --output output \
     --builder-image rpm-builder:centos10
   ```

   The package has no cross-package `BuildRequires`; the OpenCV and OpenCL
   libraries it links against are only needed at runtime.

   `build-rpm.sh` copies every file beside the spec into the build workspace,
   so the spec can be built straight from the repository root.

Binary and source RPMs are written to `output/`. Nothing is uploaded; publishing
happens only through `pkg-release.yml`. The build leaves `tool/`,
`sources-cache/`, `output/`, and `resolve_sources_output.env` in the repository
root; they are not part of the package and can be deleted.

## Updating the Package Version

1. Update `Version:` and `vendor_build_date` in `qcom-image-stitching.spec`, and
   update `Source0` if the Qualcomm Software Center artifact location changes.
   The archive filename must remain
   `image-stitching_<version>_armv8a.tar.gz`.
2. Confirm that the staged archive still contains the files validated by the
   `%prep` checks in the spec.
3. Regenerate the source checksum:

   ```bash
   sha512sum --tag image-stitching_<version>_armv8a.tar.gz > sources
   ```

4. Commit the spec and `sources`, then open a pull request against `c10s`.
5. After the pull request is merged, run `pkg-release.yml` to publish the
   RPMs.

## License

This packaging repository is licensed under the BSD 3-Clause License. See
`LICENSE.txt` for the complete license text.

The vendor binary payload is distributed under Qualcomm's binary license. The
`LICENSE.qcom-2` file from the vendor archive is included in the runtime RPM.
