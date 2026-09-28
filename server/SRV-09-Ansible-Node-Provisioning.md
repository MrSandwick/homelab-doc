---
tags: [homelab, project, ansible, provisioning, kubernetes]
---

# Ansible Node Provisioning

> Status: 🟢 **Working.** Ansible runs from the control-plane node and provisions both cluster nodes with a trimmed `site.yml`; dry run and real run both `failed=0`, `unreachable=0`. DNS and containerd-config management intentionally excluded — see [Playbook scope](#playbook-scope-trimmed-before-running-on-live-nodes).

Picks up after [Helm, Observability, and Ingress](./SRV-08-Helm-Observability-Ingress.md). Commands run on the control-plane node (`<HOSTNAME>`) unless noted.

## Contents

1. [Components](#components)
2. [Installation](#installation)
3. [Test phase on a non-cluster node](#test-phase-on-a-non-cluster-node)
4. [Real troubleshooting #1: SSH host-key and authentication failures](#real-troubleshooting-1-ssh-host-key-and-authentication-failures)
5. [Real troubleshooting #2: `sudo-rs` breaks privilege escalation](#real-troubleshooting-2-sudo-rs-breaks-privilege-escalation)
6. [Inventory and variables](#inventory-and-variables)
7. [Playbook scope: trimmed before running on live nodes](#playbook-scope-trimmed-before-running-on-live-nodes)
8. [Final verified run](#final-verified-run)
9. [Related](#related)

## Components

| Component | Where | Role |
|---|---|---|
| Ansible | Control node only (Ubuntu `ansible` package) | Configuration management over SSH |
| `inventory.ini` | Control node | Groups `k8s_control_plane`, `k8s_workers`, `k8s_cluster` |
| `group_vars/all.yml`, `host_vars/<hostname>.yml` | Control node | Shared and per-node variables |
| `site.yml` | Control node | Node provisioning playbook |
| `learn-playbook.yml` | Control node | Test-phase playbook |
| Classic `sudo` (via `update-alternatives`) | All managed nodes | Replaces `sudo-rs` as active `sudo` |

Overview of each component: [Guide: The Stack](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Stack.md).

## Installation

Introduced to codify the manual provisioning steps from [Kubernetes Installation](./SRV-05-Kubernetes-Installation.md) and [Second Node Setup](./SRV-06-Second-Node-Setup.md). Installed on the control node only; managed nodes require only SSH and Python. See [Guide: Ansible Basics](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Ansible-Basics.md).

```
sudo apt update && sudo apt install ansible -y
```

## Test phase on a non-cluster node

Initial runs targeted a personal laptop (`<TEST_NODE_HOSTNAME>`, `<TEST_NODE_IP>`, different VLAN — see [VLAN Design and Switch Configuration](../network/NET-03-VLAN-Design-and-Switch-Configuration.md)) before the cluster nodes.

SSH key generated on the control node and copied to the test machine; added to `inventory.ini` under a temporary group:

```
ssh-keygen -t ed25519
ssh-copy-id <USERNAME>@<TEST_NODE_IP>
```

```ini
[laptop_test]
<TEST_NODE_HOSTNAME> ansible_host=<TEST_NODE_IP> ansible_user=<USERNAME>
```

Connectivity (Ansible `ping` module):

```
ansible -i inventory.ini laptop_test -m ping
```

Returned `"ping": "pong"`.

### Idempotency check

Same ad-hoc command run twice:

```
ansible -i inventory.ini laptop_test -m apt -a "name=htop state=present" --become --ask-become-pass
```

- First run: `"changed": true`
- Second run: `"changed": false`

Reproduced as a playbook, `learn-playbook.yml`:

```yaml
---
- name: Learning playbook — install htop
  hosts: laptop_test
  become: true
  tasks:
    - name: Ensure htop is installed
      ansible.builtin.apt:
        name: htop
        state: present
```

```
ansible-playbook -i inventory.ini learn-playbook.yml --ask-become-pass
```

Same result on two consecutive runs.

## Real troubleshooting #1: SSH host-key and authentication failures

First playbook run against the cluster nodes:

```
Host key verification failed
```

**Root cause:** `~/.ssh/known_hosts` for this user on the control node was empty — no recorded fingerprint for the worker or for the control node's own address (it manages itself over SSH). **Fix:** one manual SSH to each, accepting the fingerprint:

```
ssh <USERNAME>@<SERVER_IP>     # control node (self)
ssh <USERNAME>@<WORKER_IP>     # worker
```

Next run:

```
Permission denied (publickey,password)
```

**Root cause:** the key had not been copied to either cluster node. **Fix:**

```
ssh-copy-id <USERNAME>@<SERVER_IP>     # control node (self)
ssh-copy-id <USERNAME>@<WORKER_IP>     # worker
```

## Real troubleshooting #2: `sudo-rs` breaks privilege escalation

Every `become` task then failed, with a correct password:

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

**Fix (per node):**

```
update-alternatives --list sudo
# /usr/lib/cargo/bin/sudo   (sudo-rs)
# /usr/bin/sudo.ws          (classic sudo)

sudo update-alternatives --config sudo
# select the classic /usr/bin/sudo.ws entry

sudo --version    # confirms classic sudo
```

Required on all three machines (test laptop and both cluster nodes) — an Ubuntu 26.04 default, not a per-host misconfiguration.

## Inventory and variables

```ini
# inventory.ini
[k8s_control_plane]
<HOSTNAME> ansible_host=<SERVER_IP> ansible_user=<USERNAME>

[k8s_workers]
<WORKER_HOSTNAME> ansible_host=<WORKER_IP> ansible_user=<USERNAME>

[k8s_cluster:children]
k8s_control_plane
k8s_workers
```

- **`group_vars/all.yml`** — shared: `dns_servers` (`<GATEWAY_IP>`, per [Network Configuration](./SRV-03-Network-Configuration.md)), `pod_network_cidr` (`10.244.0.0/16`), `kubernetes_apt_version`.
- **`host_vars/<hostname>.yml`** — per-node interface names (`eno1` on the GMKtec, `enp0s31f6` on the OptiPlex).

## Playbook scope: trimmed before running on live nodes

The original `site.yml` was written for fresh nodes; both targets were already configured by hand and serving cluster traffic. Dry run against them:

```
ansible-playbook -i inventory.ini site.yml --check --diff --ask-become-pass
```

The diff flagged two tasks as unsafe on live nodes:

### 1. Netplan DNS task

Would have written a second netplan file (`99-ansible-dns.yaml`) for an interface already configured in the existing installer-written file — two netplan sources for one interface on a live node.

### 2. containerd config regeneration task

`containerd config default` piped over `/etc/containerd/config.toml`. The live config uses a newer schema (`version = 3`, restructured plugin paths) than the installed binary generates by default; applying it would have replaced a working runtime config with a structurally different one.

### Resulting scope

Both tasks were removed before any real (non-`--check`) run. Remaining tasks:

- base package installation
- kernel module registration
- sysctl settings
- starting and enabling the containerd service (config untouched)
- Kubernetes apt repo and package installation, with version holds

## Final verified run

```
ansible-playbook -i inventory.ini site.yml --check --diff --ask-become-pass   # dry run first
ansible-playbook -i inventory.ini site.yml --diff --ask-become-pass           # then applied for real
```

Both nodes: `failed=0`, `unreachable=0`. `changed` limited to two pending items — `socat` missing on the control-plane node, and an `overlay` line missing from the existing kernel-modules file. All other tasks `ok`.

Re-run: idempotent except the Kubernetes apt-key task (`ansible.builtin.get_url`), which reports `changed` on every run because the download is not checksum-pinned. Functionally harmless; open for a later fix.

## Related

- [Helm, Observability, and Ingress](./SRV-08-Helm-Observability-Ingress.md) — the platform layer this provisioning sits under
- [Kubernetes Installation](./SRV-05-Kubernetes-Installation.md) — the manual steps this playbook now codifies
- [Second Node Setup](./SRV-06-Second-Node-Setup.md) — the worker's manual provisioning
- [Network Configuration](./SRV-03-Network-Configuration.md) — the hand-written netplan and DNS config the playbook deliberately does not manage
- [Guide: Ansible Basics](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Ansible-Basics.md) — companion Guides repository
- [Guide: sudo-rs and Privilege Escalation](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Sudo-rs-and-Privilege-Escalation.md) — companion Guides repository
