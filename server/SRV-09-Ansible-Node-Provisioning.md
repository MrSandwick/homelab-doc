---
tags: [homelab, project, ansible, provisioning, kubernetes]
---

# Ansible Node Provisioning

> Status: 🟢 **Working.** Ansible runs from the control-plane node and provisions both cluster nodes with a deliberately trimmed, production-safe `site.yml`: a dry run and a real run both finished with `failed=0`, `unreachable=0`. DNS and containerd-config management were **intentionally left out** — see [the decision to trim the playbook](#the-decision-trimming-the-playbook-before-running-it-on-live-nodes).

Picks up after [Helm, Observability, and Ingress](./SRV-08-Helm-Observability-Ingress.md). Commands are run from the control-plane node (`<HOSTNAME>`) unless noted.

## Why Ansible, and why only on the control node

Both nodes were built by hand: the same packages, kernel modules, sysctl settings and Kubernetes repo steps run twice, once per machine ([Kubernetes Installation](./SRV-05-Kubernetes-Installation.md) and [Second Node Setup](./SRV-06-Second-Node-Setup.md)). That is exactly the kind of repeated, order-sensitive work configuration management is for — and it would have caught the `br_netfilter` persistence slip documented in [Kubernetes Installation, Step 8](./SRV-05-Kubernetes-Installation.md#step-8--installing-flannel-cni). See [Guide: Ansible Basics](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Ansible-Basics.md).

Ansible is installed **only on the control node** (`<HOSTNAME>`). It is agentless: managed nodes need nothing beyond SSH and Python, which Ubuntu Server already has.

```
sudo apt update && sudo apt install ansible -y
```

## Learning phase on a throwaway node

Before pointing anything at the real cluster, everything was practised on a disposable target: a personal laptop (`<TEST_NODE_HOSTNAME>`, `<TEST_NODE_IP>`) on a different VLAN (see [VLAN Design and Switch Configuration](../network/NET-03-VLAN-Design-and-Switch-Configuration.md)). If a mistake was going to happen, it would happen there.

An SSH key was generated on the control node and copied to the test machine, which went into `inventory.ini` under a temporary group:

```
ssh-keygen -t ed25519
ssh-copy-id <USERNAME>@<TEST_NODE_IP>
```

```ini
[laptop_test]
<TEST_NODE_HOSTNAME> ansible_host=<TEST_NODE_IP> ansible_user=<USERNAME>
```

Connectivity check with Ansible's `ping` **module** — an SSH-plus-Python round trip, not ICMP:

```
ansible -i inventory.ini laptop_test -m ping
```

A healthy target replies `"ping": "pong"`.

### Idempotency, demonstrated

The same ad-hoc command was run twice:

```
ansible -i inventory.ini laptop_test -m apt -a "name=htop state=present" --become --ask-become-pass
```

- **First run:** `"changed": true` — `htop` was installed.
- **Second, identical run:** `"changed": false` — the package was already there, so nothing was done.

That is the core property of Ansible modules: they describe a **desired state** and only act if reality differs. It comes from using a state-aware module like `apt`; the `command` and `shell` modules have no such awareness and report `changed` every time.

The same thing as a minimal file-based playbook, `learn-playbook.yml`:

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

Run twice, it behaved the same way: `changed` the first time, not the second.

## Real troubleshooting #1: SSH host-key and authentication failures

Pointing the first playbook at the real cluster nodes failed immediately:

```
Host key verification failed
```

**Root cause:** `~/.ssh/known_hosts` for this user on the control node was empty. The control node had never SSH'd into itself, nor explicitly verified the worker under this user account, so it had no recorded fingerprint for either. **Fix:** connect to each once by hand and accept the fingerprint:

```
ssh <USERNAME>@<SERVER_IP>     # the control node itself
ssh <USERNAME>@<WORKER_IP>     # the worker
```

Note that the control node manages *itself* over SSH like any other node, so it needs a `known_hosts` entry for its own address too.

That exposed a second, separate failure:

```
Permission denied (publickey,password)
```

**Root cause:** the SSH key had never actually been copied to either node in this flow. **Fix:** a fresh `ssh-copy-id` for both — including the control node, so it can manage itself:

```
ssh-copy-id <USERNAME>@<SERVER_IP>     # self — needed for the control node to manage itself
ssh-copy-id <USERNAME>@<WORKER_IP>     # worker
```

## Real troubleshooting #2: `sudo-rs` breaks Ansible's privilege escalation

With SSH working, every playbook run against the real nodes then failed the same way:

```
Timeout (12s) waiting for privilege escalation prompt:
```

**Diagnosis.** Re-running with maximum verbosity showed what Ansible does to escalate:

```
ansible-playbook -i inventory.ini site.yml -vvv --ask-become-pass
```

Ansible sends a `sudo -S -p "[sudo via ansible, key=...] password:"` command and then **waits for that exact prompt string** before submitting the password. Whatever `sudo` was answering was not producing it.

**Root cause.** Checking the node showed why:

```
sudo --version
dpkg -l | grep sudo
```

Ubuntu 26.04 (Resolute Raccoon) ships **`sudo-rs`** — a Rust reimplementation of sudo — and `dpkg` showed both `sudo` and `sudo-rs` installed side by side, with `sudo-rs` registered as the active `update-alternatives` choice. Its prompt handling doesn't match what Ansible's `become` mechanism expects from classic GNU sudo, so every privilege-escalated task hangs until it times out, **even with a correct password**. Note the misleading symptom: it looks like an authentication or SSH problem, but is neither. See [Guide: sudo-rs and Privilege Escalation](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Sudo-rs-and-Privilege-Escalation.md).

**Fix — switch the active `sudo` back to the classic implementation, per node:**

```
update-alternatives --list sudo
# /usr/lib/cargo/bin/sudo   (sudo-rs)
# /usr/bin/sudo.ws          (classic sudo)

sudo update-alternatives --config sudo
# select the classic /usr/bin/sudo.ws entry

sudo --version    # confirm it reports the classic GNU sudo version, not sudo-rs
```

This was hit and fixed on **three separate machines** during this work — the test laptop, then both real cluster nodes — which points to an Ubuntu 26.04-wide default rather than a one-off misconfiguration. Worth checking proactively on any freshly-provisioned 26.04 host before assuming Ansible's `become` will just work.

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

- **`group_vars/all.yml`** — settings shared by every node: the DNS server list (the router, `<GATEWAY_IP>`, consistent with the fix in [Network Configuration](./SRV-03-Network-Configuration.md)), `pod_network_cidr` (`10.244.0.0/16`, matching the Flannel setup in [Kubernetes Installation](./SRV-05-Kubernetes-Installation.md)), and `kubernetes_apt_version`.
- **`host_vars/<hostname>.yml`** — per-node network interface names. They legitimately differ between the two machines (`eno1` on the GMKtec, `enp0s31f6` on the OptiPlex), which is precisely what host variables are for.

## The decision: trimming the playbook before running it on live nodes

This is the most consequential part of the work. The originally planned `site.yml` was written as if provisioning a fresh, unconfigured node. Both real nodes were **already configured by hand and serving live cluster traffic**. Before applying anything, a dry run with a diff was made against them:

```
ansible-playbook -i inventory.ini site.yml --check --diff --ask-become-pass
```

`--check` makes no changes and `--diff` shows what each task *would* change. Two tasks that were fine on a fresh node turned out to be unsafe to apply here:

### 1. The netplan DNS task

It would have written a **second, separate netplan file** (`99-ansible-dns.yaml`) describing an interface that the existing installer-written netplan file already configures correctly, by hand. Two netplan sources describing one interface, on a node carrying live cluster traffic, is an invitation for conflicting configuration and a network outage.

### 2. The containerd-config regeneration task

It ran `containerd config default` and wrote the output over `/etc/containerd/config.toml`. The diff showed this would **overwrite the live, working config** with a freshly generated default. The live file uses a newer schema (`version = 3`, with restructured plugin paths) than what the installed `containerd config default` would generate identically, so applying it risked a structurally different config — and a destabilised runtime on a node actively running the cluster's containers.

### The trimmed scope

Both tasks were **removed from `site.yml` before any real (non-`--check`) run.** What remained is only the safe, additive and genuinely idempotent parts:

- base package installation
- kernel module registration
- sysctl settings
- starting and enabling the containerd service (its config untouched)
- the Kubernetes apt repo and package installation, with version holds

The lesson: an infrastructure-as-code task being *correct* is not the same as it being *safe to apply to a system that is already correct in some other way*. A dry run against the live system is what separates the two. See [Guide: Ansible Basics](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Ansible-Basics.md), the section on applying a playbook to already-configured infrastructure.

## Final verified run

```
ansible-playbook -i inventory.ini site.yml --check --diff --ask-become-pass   # dry run first
ansible-playbook -i inventory.ini site.yml --diff --ask-become-pass           # then applied for real
```

Result on both nodes: `failed=0`, `unreachable=0`. `changed` was limited to genuinely pending items — a `socat` package missing on the control-plane node, and an `overlay` line missing from the existing kernel-modules file. Everything else reported `ok`, confirming the trimmed playbook matches live reality.

A subsequent identical re-run was almost fully idempotent. The one exception is the Kubernetes apt-key download task (`ansible.builtin.get_url`), which still reports `changed` every time because it re-downloads the key on each run and isn't checksum-pinned. That's a known, minor imperfection — functionally harmless, and a candidate for a later tidy-up rather than a bug.

## Related

- [Helm, Observability, and Ingress](./SRV-08-Helm-Observability-Ingress.md) — the platform layer this provisioning sits under
- [Kubernetes Installation](./SRV-05-Kubernetes-Installation.md) — the manual steps this playbook now codifies
- [Second Node Setup](./SRV-06-Second-Node-Setup.md) — the worker's manual provisioning
- [Network Configuration](./SRV-03-Network-Configuration.md) — the hand-written netplan and DNS config the playbook deliberately does not manage
- [Guide: Ansible Basics](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Ansible-Basics.md) — companion Guides repository
- [Guide: sudo-rs and Privilege Escalation](https://github.com/MrSandwick/OVault/blob/main/homelab-docs/homelab-guides/server/Guide-Sudo-rs-and-Privilege-Escalation.md) — companion Guides repository
