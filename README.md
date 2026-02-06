# k8s-ansible

Ansible utility to deploy and manage alpha/release version Kubernetes clusters on multi-architecture infrastructure (ppc64le, s390x).

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Deploying a Kubernetes Cluster](#deploying-a-kubernetes-cluster)
- [Migrate to k8s-admin User](#migrate-to-k8s-admin-user)
- [Upgrade Containerd](#upgrade-containerd)

---

## Prerequisites

Add node IP + hostname entries under `/etc/hosts` on the deployer machine:

```
[root@kubetest2-tf1 hack]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
1.2.3.4 <Nodename 1>
1.2.3.5 <Nodename 2>
```

**Inventory file** (`hosts.yml`) must define the following groups:

```yaml
[bastion]
<bastion-ip>

[masters]
<control-plane-ip-1>
<control-plane-ip-2>

[workers]
<worker-ip-1>
<worker-ip-2>

[all:vars]
ansible_user=k8s-admin
ansible_ssh_private_key_file=/path/to/private/key
ansible_become=true
ansible_become_method=sudo
ansible_become_user=root

[masters:vars]
ansible_ssh_common_args='-o ProxyCommand="ssh -W %h:%p -i /path/to/key -q k8s-admin@<bastion-ip>"'

[workers:vars]
ansible_ssh_common_args='-o ProxyCommand="ssh -W %h:%p -i /path/to/key -q k8s-admin@<bastion-ip>"'
```

---

## Deploying a Kubernetes Cluster

### Method 1: Using hosts.yml and extra-vars

Modify `examples/containerd-cluster/hosts.yml` and `examples/containerd-cluster/extra-vars-k8s.json`, then run:

```bash
ansible-playbook -i examples/containerd-cluster/hosts.yml install-k8s.yml \
  --extra-vars "@examples/containerd-cluster/extra-vars-k8s.json"
```

For HA (multi control-plane) clusters:

```bash
ansible-playbook -i examples/containerd-cluster/hosts.yml install-k8s-ha.yaml \
  --extra-vars "@examples/containerd-cluster/extra-vars-k8s.json"
```

### Method 2: Using hack/k8s-installer.sh

The `k8s-installer.sh` utility provides options for alpha or release Kubernetes installations:

```bash
# Deploy latest alpha release
./k8s-installer.sh -w X.X.X.X -c Y.Y.Y.Y -a -y

# Deploy latest stable release
./k8s-installer.sh -w X.X.X.X -c Y.Y.Y.Y -r -y

# Deploy stable release for perf-tests
./k8s-installer.sh -p install-k8s-perf.yml -w X.X.X.X -c Y.Y.Y.Y -r -y
```

---

## Migrate to k8s-admin User

**Playbook:** `migrate-to-k8s-admin.yaml`

Migrates existing nodes from using `root` SSH access to a dedicated `k8s-admin` user with passwordless sudo. This is the recommended security setup for s390x (IBM Z) clusters.

### What it does

| Phase | Description |
|-------|-------------|
| **Phase 1** | Creates `k8s-admin` user on all nodes (connects as `root` since user doesn't exist yet) |
| **Phase 2** | Validates SSH connectivity and sudo access as `k8s-admin`, verifies kubectl on masters |
| **Phase 3** | Optionally disables root SSH login (disabled by default for safety) |

### Usage

**Step 1 — Run the migration (creates user + validates)**

```bash
ansible-playbook -i <inventory> migrate-to-k8s-admin.yaml
```

> **Note:** Phase 1 overrides `ansible_user` to `root` internally, so your inventory can already have `ansible_user=k8s-admin` set — the playbook handles this.

**Step 2 — Verify the output**

Phase 2 prints validation results for each node:

```
=== Migration Validation Results ===
SSH user (should be k8s-admin): k8s-admin
Sudo user (should be root):    root
kubectl output:                <node list> (masters only)
====================================
```

If either check fails, the playbook will abort with a clear error message.

**Step 3 — Disable root SSH (optional, when ready)**

```bash
ansible-playbook -i <inventory> migrate-to-k8s-admin.yaml \
  -e "disable_root_ssh=true"
```

> **Warning:** Only run this after confirming `k8s-admin` works on all nodes. This modifies `/etc/ssh/sshd_config` to set `PermitRootLogin no` and restarts sshd.

### Example: Full migration on production

```bash
# Dry-run on a single node first
ansible-playbook -i examples/k8s-build-cluster/hosts.yml migrate-to-k8s-admin.yaml \
  --limit 10.243.0.213

# Run on all nodes
ansible-playbook -i examples/k8s-build-cluster/hosts.yml migrate-to-k8s-admin.yaml

# After verifying, disable root SSH
ansible-playbook -i examples/k8s-build-cluster/hosts.yml migrate-to-k8s-admin.yaml \
  -e "disable_root_ssh=true"
```

---

## Upgrade Containerd

**Playbook:** `upgrade-containerd.yaml`

Performs a rolling containerd upgrade across the cluster — one node at a time, workers first, then control-plane nodes — without any cluster downtime.

### What it does

| Phase | Hosts | Description |
|-------|-------|-------------|
| **Phase 1** | Workers (serial: 1) | Drain → stop containerd → backup config → download new binary → regenerate config → restart containerd + kubelet → uncordon → verify |
| **Phase 2** | Masters (serial: 1) | Same as Phase 1, but for control-plane nodes |
| **Phase 3** | First master | Runs `kubectl get nodes -o wide` and checks kube-system pod health |

### Key features

- **Rolling upgrade** — processes one node at a time, cluster stays healthy throughout
- **Idempotent** — nodes already at the target version are automatically skipped
- **Config preserved** — backs up existing `config.toml` and regenerates from the Jinja2 template
- **PDB bypass** — uses `--disable-eviction` to handle PodDisruptionBudgets that block eviction
- **Version check** — compares current vs target version and skips if already upgraded

### Configuration

The target version is controlled by `containerd_version` in `group_vars/all`:

```yaml
containerd_version: 2.1.6
```

### Usage

**Step 1 — Update the target version in `group_vars/all`**

```yaml
# Before
containerd_version: 1.7.13

# After
containerd_version: 2.1.6
```

**Step 2 — Test on a single worker node first**

```bash
ansible-playbook -i <inventory> upgrade-containerd.yaml --limit <worker-ip>
```

Verify the node:

```bash
kubectl get node <worker-hostname> -o wide
# Should show containerd://2.1.6
```

**Step 3 — Roll out to all nodes**

```bash
ansible-playbook -i <inventory> upgrade-containerd.yaml
```

Already-upgraded nodes are skipped automatically.

**Step 4 — Verify the cluster**

```bash
kubectl get nodes -o wide
kubectl get pods -A | grep -v Running | grep -v Completed
```

### Override version without editing group_vars

```bash
ansible-playbook -i <inventory> upgrade-containerd.yaml \
  -e "containerd_target_version=2.1.6"
```

### Handling PDB-protected pods

If the drain step gets stuck on pods with strict PodDisruptionBudgets (e.g., `maxUnavailable: 0`), the playbook already uses `--disable-eviction` to bypass PDBs. If pods still time out due to long `terminationGracePeriodSeconds`, force-delete them before running:

```bash
# Find and delete the blocking pod
kubectl get pods -n <namespace> -o wide | grep <node-name>
kubectl delete pod <pod-name> -n <namespace> --grace-period=0 --force

# Uncordon the node if it was left cordoned
kubectl uncordon <node-name>

# Re-run the playbook for that node
ansible-playbook -i <inventory> upgrade-containerd.yaml --limit <node-ip>
```

### Rollback

Each node keeps a timestamped backup at `/etc/containerd/config.toml.bak.<timestamp>`. To rollback:

```bash
ansible-playbook -i <inventory> upgrade-containerd.yaml \
  -e "containerd_target_version=1.7.13" --limit <node-ip>
```

### Example: Full production upgrade

```bash
# 1. Update group_vars/all with target version
sed -i 's/containerd_version: .*/containerd_version: 2.1.6/' group_vars/all

# 2. Test on a single worker
ansible-playbook -i examples/k8s-build-cluster/hosts.yml upgrade-containerd.yaml \
  --limit 10.243.0.213

# 3. Verify the test node
kubectl get node worker-s390x-1 -o wide

# 4. Roll out to all remaining nodes
ansible-playbook -i examples/k8s-build-cluster/hosts.yml upgrade-containerd.yaml

# 5. Verify the entire cluster
kubectl get nodes -o wide
kubectl get pods -n kube-system
```

### Version compatibility notes

| Containerd Version | Config v2 Template | K8s 1.33 | Status |
|--------------------|-------------------|----------|--------|
| 1.7.x | Works | Supported (1.7.24+) | EOL Sep 2026 (extended) |
| 2.0.x | Works (auto-migrates) | Supported (2.0.4+) | EOL Nov 2025 |
| **2.1.x** | **Works (auto-migrates)** | **Supported (2.1.0+)** | **Active, EOL May 2026** |
| 2.2.x | Breaks (requires v3 rewrite) | Supported | Active, EOL ~Nov 2026 |

> **Recommendation:** Use **containerd 2.1.x** — it is the latest actively maintained version that works with the existing `config.toml.j2` template without any changes. Upgrading to 2.2.x requires rewriting the config template to version 3 format and migrating registry mirrors to `/etc/containerd/certs.d/`.
