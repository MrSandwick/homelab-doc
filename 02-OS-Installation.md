---
tags: [homelab, project, os, ubuntu]
---

# OS Installation — Ubuntu Server 26.04 LTS

Installed on: **GMKtec M8** (primary node)

## Why Ubuntu Server (no GUI)

- Long-term support release, stable and widely documented
- No desktop environment overhead — every resource goes to services
- Standard choice for homelab and production Linux servers alike

## Installation media

- Tool: **Rufus** (Windows)
- Image: `ubuntu-26.04-live-server-amd64.iso`
- Settings used:
  - Partition scheme: **GPT** (required for UEFI boot on modern hardware)
  - Target system: UEFI
  - Write mode: **ISO Image mode** (Rufus default recommendation for ISOHybrid images)

> Rufus does not permanently consume the USB drive — it can be reformatted to plain FAT32/exFAT afterward, or overwritten directly with a new ISO for the next install.

## Installer walkthrough (key choices made)

| Step | Choice |
|---|---|
| Base install | **Ubuntu Server** (not "minimized") |
| Third-party drivers | Skipped |
| Network configuration | Skipped at install time ("Continue without network") — configured later, see [Network Configuration](./03-Network-Configuration.md) |
| Storage layout | Guided, entire disk, **LVM enabled**, **no LUKS encryption** |
| Root partition | 100GB allocated out of ~473GB available in the LVM volume group (remaining space left free, can be extended later with `lvextend`) |
| Ubuntu Pro | Skipped |
| SSH | **OpenSSH server installed**, password authentication over SSH allowed |
| Featured Server Snaps | None selected — all services installed manually afterward for full control |

## Why LVM without LUKS

- LVM (Logical Volume Manager) makes it easy to resize partitions later as storage needs grow (e.g. Docker/Kubernetes volumes).
- LUKS (disk encryption) was intentionally skipped — it would require entering a passphrase on every physical boot, which conflicts with the goal of a headless server managed entirely over SSH.

## Post-install

First login was performed with a physical monitor + keyboard. All subsequent administration is done remotely — see [SSH Remote Access](https://github.com/MrSandwick/homelab-guides/blob/main/Guide-SSH-Remote-Access.md).

To shut the machine down safely at any point:
```
sudo poweroff
```

## Related

- [Network Configuration](./03-Network-Configuration.md) — static IP setup, done after this install
- [SSH Remote Access](https://github.com/MrSandwick/homelab-guides/blob/main/Guide-SSH-Remote-Access.md) — companion Guides repository — how remote access was established
