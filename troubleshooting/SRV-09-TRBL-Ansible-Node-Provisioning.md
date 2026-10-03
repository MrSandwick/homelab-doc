---
tags: [homelab, project, ansible, troubleshooting]
---

# Troubleshooting — Ansible Node Provisioning

> Companion to [Ansible Node Provisioning](../server/SRV-09-Ansible-Node-Provisioning.md), which records only the working procedure. This file records the problems hit during that work, in the order they occurred.

## Contents

| Incident | Relates to |
|---|---|
| [SSH host-key and authentication failures](#ssh-host-key-and-authentication-failures) | [SSH access to the cluster nodes](../server/SRV-09-Ansible-Node-Provisioning.md#ssh-access-to-the-cluster-nodes) |
| [`sudo-rs` breaks privilege escalation](#sudo-rs-breaks-privilege-escalation) | [Privilege escalation: classic `sudo`](../server/SRV-09-Ansible-Node-Provisioning.md#privilege-escalation-classic-sudo) |

## SSH host-key and authentication failures

**Symptom 1:** first playbook run against the cluster nodes:

```
Host key verification failed
```

**Root cause:** `~/.ssh/known_hosts` for this user on the control node was empty — no recorded fingerprint for the worker or for the control node's own address (it manages itself over SSH).

**Fix:** one manual SSH to each, accepting the fingerprint:

```
ssh <USERNAME>@<SERVER_IP>     # control node (self)
ssh <USERNAME>@<WORKER_IP>     # worker
```

**Symptom 2:** next run:

```
Permission denied (publickey,password)
```

**Root cause:** the key had been copied only to the test machine, not to either cluster node.

**Fix:**

```
ssh-copy-id <USERNAME>@<SERVER_IP>     # control node (self)
ssh-copy-id <USERNAME>@<WORKER_IP>     # worker
```

## `sudo-rs` breaks privilege escalation

**Symptom:** every `become` task failed, with a correct password:

```
Timeout (12s) waiting for privilege escalation prompt:
```

**Diagnosis:** `-vvv` showed Ansible issuing `sudo -S -p "[sudo via ansible, key=...] password:"` and never receiving that prompt back.

```
ansible-playbook -i inventory.ini site.yml -vvv --ask-become-pass
sudo --version
dpkg -l | grep sudo
```

**Root cause:** Ubuntu 26.04 ships `sudo-rs` alongside classic `sudo`, with `sudo-rs` as the active `update-alternatives` choice. Its prompt handling is incompatible with Ansible's `become`. See [Guide: sudo-rs and Privilege Escalation](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Sudo-rs-and-Privilege-Escalation.md).

**Fix (per node):** switch the active `sudo` to the classic implementation — commands in [Privilege escalation: classic `sudo`](../server/SRV-09-Ansible-Node-Provisioning.md#privilege-escalation-classic-sudo). Required on all three machines (test laptop and both cluster nodes) — an Ubuntu 26.04 default, not a per-host misconfiguration.

## Related

- [Ansible Node Provisioning](../server/SRV-09-Ansible-Node-Provisioning.md) — the working procedure
- [Guide: sudo-rs and Privilege Escalation](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Sudo-rs-and-Privilege-Escalation.md) — companion Guides repository
