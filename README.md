# tcpdump

Container images with [tcpdump](https://www.tcpdump.org/), the command-line packet analyzer, compiled from the signed release tarball against the libpcap of Ubuntu or Alpine. The images are rebuilt when the Tcpdump Group publishes a release and when the base image changes, for `linux/amd64` and `linux/arm64`.

This is an unofficial build, not affiliated with or endorsed by the Tcpdump Group. Report problems with the image in this repository and problems with tcpdump itself [upstream](https://github.com/the-tcpdump-group/tcpdump/issues).

## Quick start

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/tcpdump -n -r capture.pcap
```

Copy the DNS packets of a capture file to a new file:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/tcpdump -r capture.pcap -w dns.pcap 'udp port 53'
```

Watch the traffic of a running container named `web`:

```sh
docker run --rm --user 0 --cap-add NET_RAW --network container:web \
  ghcr.io/randomcontainers/tcpdump -i eth0 -n -l
```

The entrypoint runs `tcpdump` under `tini` in `/work`, so file names are relative to the directory you mount. Stop a live capture with Ctrl-C, or give a packet count with `-c`. Pass `-l` to see packets as they arrive when the output is not a terminal. The [tcpdump manual](https://www.tcpdump.org/manpages/tcpdump.1.html) covers the options, and [pcap-filter](https://www.tcpdump.org/manpages/pcap-filter.7.html) the filter expressions.

## Capturing live traffic

Reading and writing capture files works as any user. Opening a network interface needs the `NET_RAW` capability, which a container process only has when it runs as root, so live captures need `--user 0`. Docker gives root in a container `NET_RAW` by default and Podman does not; `--cap-add NET_RAW` works with both. The binary has no file capabilities: with `cap_net_raw+ep` set it would not start at all under `--cap-drop ALL`, and `no-new-privileges` ignores file capabilities.

`--network container:<name>` captures inside another container's network namespace, and `--network host` captures on the host's interfaces. A capture file written to a mounted directory as root is owned by root on the host, so write it to standard output instead and let your shell create the file:

```sh
docker run --rm --user 0 --cap-add NET_RAW --network host \
  ghcr.io/randomcontainers/tcpdump -i eth0 -c 1000 -w - > capture.pcap
```

## What is in the image

- `tcpdump` in `/usr/local/bin`, linked against the distro's libpcap.
- OpenSSL's libcrypto, which tcpdump uses to decrypt IPsec ESP packets (`-E`) and to check TCP MD5 signatures (`-M`).

Not included: libsmi (SNMP MIB names), libcap-ng, the SMB printer that upstream leaves out by default, and the man page. `tcpdump --version` prints the libpcap and OpenSSL versions, and the configure flags are in `/usr/local/share/randomcontainers/tcpdump/buildinfo`.

## Default or slim

tcpdump's default image adds no other tools, so `latest` and `slim` are the same image: `tcpdump` and the libraries it links against. Use `latest` to run it and the `slim` tags as a base for your own image. The default image of [TShark](https://github.com/randomcontainers/tshark) includes this build of tcpdump.

## Tags

`<version>` is a tcpdump release such as `4.99.7`. `<minor>` and `<major>` are its shorter forms, `4.99` and `4`, and follow the newest release in that series. Each row lists the default tag and its `slim` twin, which point to the same image.

| Tags | Base |
|---|---|
| `latest`, `slim` | Ubuntu |
| `<version>`, `<version>-slim` | Ubuntu |
| `<minor>`, `<minor>-slim`, `<major>`, `<major>-slim` | Ubuntu |
| `ubuntu`, `slim-ubuntu` | Ubuntu |
| `<version>-ubuntu`, `<version>-slim-ubuntu` | Ubuntu |
| `<minor>-ubuntu`, `<minor>-slim-ubuntu`, `<major>-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04`, `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `slim-alpine` | Alpine |
| `<version>-alpine`, `<version>-slim-alpine` | Alpine |
| `<minor>-alpine`, `<minor>-slim-alpine`, `<major>-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24`, `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current tcpdump version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

Live captures run as root instead, see [Capturing live traffic](#capturing-live-traffic).

## Untrusted capture files

tcpdump has its own parsers for more than a hundred protocols, and they have had memory-safety bugs. Reading a capture file needs no network and no capabilities. For files from unknown sources, drop both and mount the directory read-only:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work:ro" \
  --network none --cap-drop ALL --security-opt no-new-privileges \
  ghcr.io/randomcontainers/tcpdump -n -r untrusted.pcap
```

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new tcpdump release and are rebuilt when the base image changes. The packages tcpdump needs are listed in `/usr/local/share/randomcontainers/tcpdump/runtime-deps`. Switch to root to install more, then back:

```dockerfile
FROM ghcr.io/randomcontainers/tcpdump:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends iproute2 \
 && rm -rf /var/lib/apt/lists/*
USER 1000:1000
```

On Alpine, start from `slim-alpine` and use `apk add --no-cache iproute2`. The entrypoint is `["tini", "--", "tcpdump"]`; set your own `ENTRYPOINT` if your image runs something else. To pick up new tcpdump releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/tcpdump:latest \
  --repo randomcontainers/tcpdump --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/tcpdump:latest --format '{{ json .SBOM }}'
```

Before compiling, the build checks the tarball against the SHA-256 recorded in `package.yml` and its signature against the Tcpdump Group's package signing key in `keys/tcpdump-release.gpg` (fingerprint `1F16 6A57 42AB B9E0 249A 8D30 E089 DEF1 D9C1 5D0D`).

## Updates

The project checks the `tcpdump-<version>` tags of [the-tcpdump-group/tcpdump](https://github.com/the-tcpdump-group/tcpdump) every 15 minutes. A release is picked up once it is 24 hours old and its tarball and signature are on tcpdump.org. The new version and the tarball's SHA-256 are then committed to `package.yml` and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes and at least every 7 days, so distro security fixes, including those for libpcap and OpenSSL, reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_SHA256=<sha256 from package.yml> \
  -t tcpdump:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs.

## Licenses

tcpdump's `LICENSE` file is a three-clause BSD license. Most source files carry their own copyright notice under a BSD-style license, some of them with an advertising clause, and a few are under ISC or NTP-style terms, so the image's license label is `BSD-2-Clause AND BSD-3-Clause AND BSD-4-Clause AND BSD-4-Clause-UC AND ISC AND NTP`. A few files, such as the BEEP and DCCP printers, may also be used under the GPL; the image uses them under their BSD terms. `/usr/local/share/randomcontainers/tcpdump/licenses/` has `LICENSE` and `NOTICES`, which collects the copyright notice of each source file that has one. libpcap, OpenSSL and the other distro packages keep their own licenses.

`/usr/local/share/randomcontainers/tcpdump/source` lists the tcpdump.org URLs of the tarball and signature each image was built from. The build applies no patches.

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
