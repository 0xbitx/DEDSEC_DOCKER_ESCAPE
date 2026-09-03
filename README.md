<p align="center">
<img src="https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Foss.bytezonex.com%2Flogos%2Fee95d582e512302d0a8e.png&f=1&nofb=1&ipt=abbcd7804972ac8778f62dfdf7642266752ac8a10559961b81c1e13a515429ed", width="300", height="300">
</p>

<h1 align="center">DOCKER ESCAPE</h1>
<p align="center"><code>Docker Container Escape PoC for Red-Team and Privilege-Escalation Research</code></p>

---

## DESCRIPTION

DOCKER ESCAPE builds a Docker image that breaks out of a container. You give it a
command (whoami, a reverse shell, anything), it wraps that command in an
entrypoint that walks the usual escape tricks one by one until a path opens up,
then it ships the whole thing as a portable `.tar` you can load and run on any
Docker host you have a foothold on. No source needed on the other end, just the
archive and a misconfigured runtime.

A container is only as safe as the flags you start it with. A plain `docker run`
is locked down fine, but the moment someone adds `--privileged`, mounts the
Docker socket, or bind mounts a host path, that safety evaporates. Each of those
choices hands an attacker a different route straight to the host. This tool shows
each mistake on its own so you can see how a single bad flag turns a sandbox into
a shell on the host:

- **Privileged mode** (`--privileged`). Exposes host devices and all
  capabilities. The escape hunts for host block devices, mounts the host root,
  and `chroot`s into it.
- **Docker socket** (`-v /var/run/docker.sock:...`). Gives the container control
  over Docker. The escape spawns its own `--privileged` container with the host
  root bind-mounted and runs the payload from there.
- **cgroup v1 `release_agent`**. With a writable cgroup and `CAP_SYS_ADMIN`, a
  script can be kicked off as root on the host. The escape mounts a writable
  cgroup, sets `release_agent`, and fires it.
- **Host bind mount** (`-v /:/host`). A host filesystem mapped straight in. The
  easiest and most reliable path, needs nothing but the mount.

---

## Threat Model & Purpose

For people who need realistic container-escape samples:

- **Build test samples** for detection engineering and sandbox testing.
- **Check detections** against privileged containers and socket exposure.
- **Study the tricks** (host mounts, socket pivoting, cgroup release_agent,
  chroot).
- **Harden docker hosts** by knowing what to look for.

The payload rides inside the image and only runs on the host after a stage in the
chain succeeds.

---

## Features

### Core Escape Chain

- **Privileged check**: uses `ip link add dummy0` to see if `--privileged` is set.
- **Root device scan**: loops `/dev` for block devices (`sd*`, `vd*`, `hd*`,
  `xvd*`, `nvme*`, `mmcblk*`) and reads filesystem type with `blkid`.
- **Chroot escape**: mounts the host root to `/mnt/host` and `chroot`s into it
  before running the payload.
- **Socket escape**: finds `/var/run/docker.sock` and pivots through it to spawn
  a `--privileged` Alpine/Busybox container with the host bind-mounted.
- **release_agent escape**: resolves the host overlay path from `/etc/mtab` and
  triggers `release_agent` to run the payload on the host.

| Tool Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/171e27f8-e67c-4331-a110-65a0d15fa348) |
| ![screenshot](https://github.com/user-attachments/assets/986ba748-3d39-45c3-b1e2-80f2bab61f8e) |

---

## Installation

```bash
git clone https://github.com/0xbitx/DEDSEC_DOCKER_ESCAPE.git
cd DEDSEC_DOCKER_ESCAPE
sudo apt install docker.io
./dedsec_docker_escape
```

#### Step-by-step flow

1. Input the payload and image name.
2. Run the generator. It builds, exports to `.tar`.
3. Move the `.tar` to the target host.
4. `docker load` it and run with one of the three modes.

---

## Escape Methods

The image tries these in order:

| # | Method | Trigger |
|---|--------|---------|
| 1 | Privileged host mount | `--privileged` |
| 2 | Docker socket pivot | `-v /var/run/docker.sock:/var/run/docker.sock` |
| 3 | cgroup v1 `release_agent` | `CAP_SYS_ADMIN` + writable cgroup |
| 4 | Host bind mount | `-v /:/host` |

---

## Usage

Load and run on the target:

```bash
docker load -i escape_image.tar
```

Pick a mode:

```bash
# 1. Privileged (host device mount)
docker run --privileged -it escape_image

# 2. Docker socket
docker run -v /var/run/docker.sock:/var/run/docker.sock -it escape_image

# 3. Both (best odds)
docker run --privileged -v /var/run/docker.sock:/var/run/docker.sock -it escape_image
```

---

## How It Works

### Host Root Device Mount

Scans `/dev` for block devices and checks filesystem type with `blkid`. The first
device matching `ext4` / `xfs` / `btrfs` etc. gets mounted to `/mnt/host`, then
the payload runs via `chroot /mnt/host sh -c <payload>`.

### Docker Socket Pivot

If `/var/run/docker.sock` is there, it launches a `--privileged` container with
the host root bind-mounted to `/host` and runs the payload through
`chroot /host`.

### cgroup v1 `release_agent`

With `CAP_SYS_ADMIN` and a writable cgroup, it mounts the `rdma` cgroup, reads the
host overlay path from `/etc/mtab`, writes the payload script, and sets
`release_agent` to run it on the host. Patched on modern kernels and cgroup v2.

---

### Run instructions

```text
[1] Privileged container (host device mount escape):
    docker run --privileged -it escape_image

[2] Docker socket escape:
    docker run -v /var/run/docker.sock:/var/run/docker.sock -it escape_image

[3] Combine both (maximum chance of success):
    docker run --privileged -v /var/run/docker.sock:/var/run/docker.sock -it escape_image
```

---

## Requirements

| Dependency | Purpose | Install |
|-----------|---------|---------|
| Python 3.8+ | Runtime | `apt install python3` |
| Docker | Build and load | `apt install docker.io` |

---

## Legal Disclaimer

Educational and security research only. Unauthorized access to computer systems
is illegal. The author is not responsible for any misuse. Only use on systems you
own or have explicit permission to test.
