# etcd Docker image (Alpine + Bitnami script layer)

Self-contained etcd image, fully decoupled from Bitnami's online endpoints
(`bitnami/minideb` base image and `downloads.bitnami.com` binary downloads),
while keeping 100% of the Bitnami chart's operational features.

## Layout

| Path | Origin | Purpose |
|---|---|---|
| `Dockerfile.alpine` | this repo | Multi-stage build: compile etcd from this repo's source, package on `alpine:3.21` |
| `prebuildfs/` | [bitnami/containers](https://github.com/bitnami/containers) (Apache-2.0, vendored) | Bitnami helper libraries (`lib*.sh`, `install_packages`). Local change: welcome banner in `libbitnami.sh` points at this fork instead of Bitnami |
| `rootfs/` | bitnami/containers (Apache-2.0, vendored, unchanged) | etcd orchestration scripts: `libetcd.sh` (cluster bootstrap, member add on scale-up, rejoin, disaster recovery, RBAC), `preupgrade.sh` (stale member removal on scale-down), `healthcheck.sh`, snapshotter |

The Bitnami script layer is what powers the `bitnami/etcd` Helm chart's
automation (scale up/down member management, snapshots, RBAC, TLS). Keeping it
intact means the chart works unchanged — only `image.registry/repository/tag`
needs to point at this image.

## Build

The builder stage clones the etcd source (`ETCD_REPO`, CI passes upstream
`etcd-io/etcd`) at `ETCD_REF` (a release tag) and runs
`./scripts/build.sh` (`CGO_ENABLED=0`, static binaries). Source is cloned by
ref — not taken from the build context — because upstream release tags do not
contain this `docker/` directory. The context (repo root on `main`) only
supplies `docker/prebuildfs` + `docker/rootfs`.

```bash
docker buildx build -f docker/Dockerfile.alpine \
  --build-arg ETCD_REF=v3.6.7 \
  --platform linux/amd64,linux/arm64 \
  -t <your-registry>/etcd:<tag> --push .
```

CI (`.github/workflows/build-images.yml`) resolves the highest stable `v3.*`
tag automatically, builds it, and publishes `ghcr.io/<owner>/etcd:{3,X.Y.Z,latest}`.

## Alpine compatibility notes

The Bitnami scripts are written for a Debian/GNU userland. The runtime stage
installs `bash coreutils grep sed gawk findutils` to provide GNU semantics the
scripts rely on (`stat -c`, `comm -23`, `sync -d`, ...). `getent` (used for
headless-service DNS discovery during cluster bootstrap) ships with the Alpine
base image. `yq` comes from the Alpine package (same mikefarah Go project that
Bitnami packages) and is symlinked to `/opt/bitnami/common/bin/yq`.

## Upstream sync

`.github/workflows/sync-upstream.yml` merges `etcd-io/etcd` `main` and release
tags into this fork on a weekly schedule (and on manual dispatch).
