# Docker Engine + Compose plugin

**Component id:** `docker-engine`

**Channel:** Docker's official signed Ubuntu apt repository

**Upstream:** https://docs.docker.com/engine/install/ubuntu/

This installs the regular system-wide Docker Engine and the Docker Compose V2 plugin. Use `docker compose`; the legacy standalone `docker-compose` command is not installed.

## Guard

```bash
dpkg-query -W -f='${db:Status-Abbrev} ${Version}\n' docker-ce 2>/dev/null | grep '^ii '
systemctl is-active --quiet docker
docker compose version
```

If all commands pass, report the installed Engine and Compose versions and skip installation.

## Install

Run on the confirmed Ubuntu amd64 target:

```bash
set -euo pipefail
source /etc/os-release
test "$ID" = ubuntu
test "$(dpkg --print-architecture)" = amd64
apt-get update
DEBIAN_FRONTEND=noninteractive apt-get install -y ca-certificates curl
install -m 0755 -d /etc/apt/keyrings
curl --fail --location --retry 3 https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
printf 'Types: deb\nURIs: https://download.docker.com/linux/ubuntu\nSuites: %s\nComponents: stable\nArchitectures: amd64\nSigned-By: /etc/apt/keyrings/docker.asc\n' "${UBUNTU_CODENAME:-$VERSION_CODENAME}" > /etc/apt/sources.list.d/docker.sources
apt-get update
DEBIAN_FRONTEND=noninteractive apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
systemctl enable --now docker
```

Do not install the obsolete distro package `docker.io` or legacy Compose V1.

## Verify

```bash
systemctl is-active --quiet docker
docker --version
docker compose version
docker run --rm hello-world
```

Record the Docker Engine server version and Compose version only after all verification commands pass.

## Known Pitfalls

- Docker-published container ports can bypass UFW/firewalld rules. Review firewall policy before publishing ports.
- Docker Desktop's `desktop-linux` context and system Engine's `default` context are separate. Remove Desktop's stale context selection when migrating.
- The Compose V2 plugin is invoked as `docker compose`; scripts that require the legacy `docker-compose` binary need updating.
- The `docker` group grants root-level privileges. This machine uses root for administration, so no non-root group access is configured.
