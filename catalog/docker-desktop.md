# Docker Desktop

**Component id:** `docker-desktop`

**Channel:** official Docker Desktop `.deb` for Ubuntu amd64

**Upstream:** https://docs.docker.com/desktop/setup/install/linux/ubuntu/

## Guard

```bash
dpkg-query -W -f='${db:Status-Abbrev} ${Version}\n' docker-desktop 2>/dev/null | grep '^ii '
docker --version
```

If both pass, report the installed version and skip installation.

## Requirements

- Ubuntu 24.04 or newer supported LTS, amd64, systemd, and at least 4 GiB RAM.
- KVM device and modules available; QEMU 5.2 or newer.
- GNOME, KDE, or MATE is supported; other desktop environments may work. On Ubuntu, install `gnome-terminal` when using a different desktop environment.
- Docker does not support Docker Desktop for Linux in nested virtualization.
- Docker Desktop runs its own VM and uses the `desktop-linux` Docker context. Its images and containers are separate from a host Docker Engine.

## Install

Run the following on the target Ubuntu machine after confirming its identity:

```bash
set -euo pipefail
source /etc/os-release
test "$ID" = ubuntu
test "$(dpkg --print-architecture)" = amd64
test -c /dev/kvm

apt-get update
DEBIAN_FRONTEND=noninteractive apt-get install -y ca-certificates curl gnome-terminal qemu-system-x86
qemu_version=$(qemu-system-x86_64 --version | sed -n '1s/.*version \([0-9][0-9.]*\).*/\1/p')
test -n "$qemu_version"
install -m 0755 -d /etc/apt/keyrings
curl --fail --location --retry 3 https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
printf 'Types: deb\nURIs: https://download.docker.com/linux/ubuntu\nSuites: %s\nComponents: stable\nArchitectures: amd64\nSigned-By: /etc/apt/keyrings/docker.asc\n' "${UBUNTU_CODENAME:-$VERSION_CODENAME}" > /etc/apt/sources.list.d/docker.sources
apt-get update
curl --fail --location --retry 3 'https://desktop.docker.com/linux/main/amd64/docker-desktop-amd64.deb' -o /tmp/docker-desktop-amd64.deb
DEBIAN_FRONTEND=noninteractive apt-get install -y /tmp/docker-desktop-amd64.deb
```

The official package installs Docker Desktop under `/opt/docker-desktop` and provides its own Docker CLI. Docker Desktop and Docker Engine can coexist, but use separate contexts and storage. Avoid running both engines simultaneously when publishing ports because their port bindings can conflict.

## Verify

```bash
dpkg-query -W -f='${db:Status-Abbrev} ${Version}\n' docker-desktop | grep '^ii '
docker --version
docker compose version
test -x /opt/docker-desktop/bin/docker-desktop
```

Starting Docker Desktop and accepting its subscription agreement are interactive first-run steps in the target user's desktop session. Verify runtime separately after acceptance with `systemctl --user start docker-desktop`, `docker context show`, and `docker info`.

## Known Pitfalls

- Docker does not support Linux Docker Desktop running inside nested virtualization. A guest can expose `/dev/kvm` and still encounter unsupported or failing VM startup.
- On the root Xfce session used by this machine, the Electron UI exits with `Running as root without --no-sandbox is not supported`; adding `--no-sandbox` to the desktop entry did not make the UI work. Use a non-root desktop login for the UI.
- Docker Desktop will not run until its subscription agreement is accepted in the GUI.
- The Linux Docker Desktop `desktop-linux` context uses a separate VM and storage from Docker Engine's `default` context.
- When not using GNOME, the Ubuntu package instructions require `gnome-terminal` for the terminal action in Docker Desktop.
- Docker Desktop Linux uses a per-user socket; SDKs that connect directly to its daemon may need `DOCKER_HOST` configured as documented upstream.
