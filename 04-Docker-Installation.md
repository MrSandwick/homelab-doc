---
tags: [homelab-project, homelab, note, docker, project]
---

# Docker Installation

## Purpose

Docker is installed for **local image building and testing** (`docker build`, `docker run`, `docker-compose`). It is *not* the runtime used by Kubernetes itself — see [Docker vs Containerd](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/Guide-Docker-vs-Containerd.md) for why these are separate concerns.

## Packages installed

```
sudo apt update && sudo apt upgrade -y
sudo apt install docker.io docker-compose-v2 git curl wget htop fastfetch -y
```

Note: `neofetch` was attempted first and is no longer available in Ubuntu repositories (the upstream project is archived). `fastfetch` is the maintained replacement.

## Running Docker without `sudo`

```
sudo usermod -aG docker $USER
```

This adds the current user to the `docker` group. **Group membership only takes effect on a new login session** — the SSH session must be closed and reopened:

```
exit
```
then reconnect:
```
ssh <USERNAME>@<SERVER_IP>
```

Verify with:
```
groups
```
`docker` should now appear in the list.

### Reverting this change

```
sudo gpasswd -d $USER docker
```

### Security note

Docker group membership is effectively equivalent to root access on the host (a container can mount the host filesystem). This is acceptable for a single-user home server; it would need reconsideration on a shared/production system.

## Verification

```
docker --version
docker run hello-world
```

A successful "Hello from Docker!" message confirms the daemon is working correctly.

## Related

- [Docker vs Containerd](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/Guide-Docker-vs-Containerd.md) — companion Guides repository
- [Kubernetes Installation](./05-Kubernetes-Installation.md) — where a *separate* containerd instance is configured specifically for Kubernetes
