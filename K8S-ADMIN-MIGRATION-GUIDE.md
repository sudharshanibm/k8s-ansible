# k8s-admin User Migration Guide for IBM Z (s390x) Systems

## Overview

Due to IBM Cloud's security policy change (effective February 23, 2026), root SSH access is disabled by default on new VPC-VSI instances. This guide explains how to migrate from root to k8s-admin user for Kubernetes cluster management.

## Background

- **Affected Systems**: IBM Z (s390x) architecture only
- **Policy Change**: Root SSH access disabled on new IBM Cloud VPC-VSI instances
- **Solution**: Use k8s-admin user with sudo privileges for all operations
- **Power Systems (ppc64le)**: Not affected, continues to use root user

## Prerequisites

1. Existing cluster with root SSH access OR new cluster with k8s-admin user created via Terraform
2. SSH private key for authentication (`/root/.ssh/k8s-test-env/id_rsa`)
3. Ansible installed on control node
4. Network connectivity to bastion and cluster nodes

## Migration Process

### Step 1: Verify Current Access

First, verify you can connect to the bastion as root:

```bash
ssh -i /root/.ssh/k8s-test-env/id_rsa root@<BASTION_IP>
```

### Step 2: Run Migration Playbook

The migration playbook creates the k8s-admin user and copies SSH keys:

```bash
cd /path/to/k8s-ansible

ansible-playbook -v -i examples/k8s-build-cluster/hosts.yml migrate-to-k8s-admin.yaml \
  -e ansible_user=root \
  -e ansible_ssh_private_key_file=/root/.ssh/k8s-test-env/id_rsa
```

**What this does:**
- Phase 1: Creates k8s-admin user on all nodes
- Phase 2: Validates SSH and sudo access
- Phase 3: (Optional) Disables root SSH login

### Step 3: Verify Migration

After successful migration, test k8s-admin access:

```bash
# Test bastion access
ssh -i /root/.ssh/k8s-test-env/id_rsa k8s-admin@<BASTION_IP>

# Test sudo access
ssh -i /root/.ssh/k8s-test-env/id_rsa k8s-admin@<BASTION_IP> "sudo whoami"
# Should output: root
```

### Step 4: Install Kubernetes

Now you can proceed with Kubernetes installation using k8s-admin:

```bash
ansible-playbook -v -i examples/k8s-build-cluster/hosts.yml install-k8s-ha.yaml \
  -e ansible_user=k8s-admin \
  -e ansible_ssh_private_key_file=/root/.ssh/k8s-test-env/id_rsa \
  -e @group_vars/bastion_configuration
```

## Troubleshooting

### Issue 1: "Connection closed by remote host" Error

**Symptom:**
```
kex_exchange_identification: Connection closed by remote host
Connection closed by UNKNOWN port 65535
```

**Cause:** SSH keys not properly copied to k8s-admin user

**Solution:** Run the emergency fix playbook:

```bash
ansible-playbook -v -i examples/k8s-build-cluster/hosts.yml fix-k8s-admin-ssh.yml
```

This will:
1. Connect as root
2. Verify root's authorized_keys exists
3. Force copy keys to k8s-admin
4. Verify the fix worked

### Issue 2: Empty authorized_keys File

**Symptom:** Migration succeeds but subsequent connections fail

**Diagnosis:**
```bash
ssh root@<BASTION_IP>
ls -la /home/k8s-admin/.ssh/authorized_keys
# Shows 0 bytes
```

**Solution:**
```bash
# On the bastion/node
sudo cp /root/.ssh/authorized_keys /home/k8s-admin/.ssh/authorized_keys
sudo chown k8s-admin:k8s-admin /home/k8s-admin/.ssh/authorized_keys
sudo chmod 600 /home/k8s-admin/.ssh/authorized_keys
```

### Issue 3: Sudo Password Required

**Symptom:** Ansible asks for sudo password

**Solution:** Verify passwordless sudo is configured:

```bash
ssh root@<BASTION_IP>
sudo cat /etc/sudoers.d/k8s-admin
# Should contain: k8s-admin ALL=(ALL) NOPASSWD: ALL
```

If missing, run:
```bash
echo "k8s-admin ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/k8s-admin
sudo chmod 0440 /etc/sudoers.d/k8s-admin
```

### Issue 4: ProxyCommand Failures

**Symptom:** Cannot reach worker/master nodes through bastion

**Diagnosis:** Check hosts.yml configuration:

```yaml
[masters:vars]
ansible_ssh_common_args='-o ProxyCommand="ssh -W %h:%p -i /root/.ssh/k8s-test-env/id_rsa -q k8s-admin@<BASTION_IP>"'

[workers:vars]
ansible_ssh_common_args='-o ProxyCommand="ssh -W %h:%p -i /root/.ssh/k8s-test-env/id_rsa -q k8s-admin@<BASTION_IP>"'
```

**Solution:** Ensure:
1. Bastion IP is correct
2. SSH key path is correct
3. User is k8s-admin (not root)

## Configuration Files

### hosts.yml Example

```yaml
[bastion]
149.81.33.136

[masters]
10.243.0.32
10.243.0.35
10.243.0.30

[workers]
10.243.0.24
10.243.0.33
10.243.0.26

[all:vars]
ansible_user=k8s-admin
ansible_ssh_private_key_file=/root/.ssh/k8s-test-env/id_rsa
ansible_become=true
ansible_become_method=sudo
ansible_become_user=root

[masters:vars]
ansible_ssh_common_args='-o ProxyCommand="ssh -W %h:%p -i /root/.ssh/k8s-test-env/id_rsa -q k8s-admin@149.81.33.136"'

[workers:vars]
ansible_ssh_common_args='-o ProxyCommand="ssh -W %h:%p -i /root/.ssh/k8s-test-env/id_rsa -q k8s-admin@149.81.33.136"'
```

### group_vars/all Changes

The `group_vars/all` file automatically detects s390x architecture:

```yaml
ansible_user: "{{ 'k8s-admin' if ansible_architecture == 's390x' else 'root' }}"
```

This means:
- s390x systems: Use k8s-admin
- ppc64le systems: Use root (no change)

## Best Practices

1. **Always test migration on a single node first** before running on entire cluster
2. **Keep root access available** until k8s-admin is fully verified
3. **Backup SSH keys** before making changes
4. **Document your bastion IP and key paths** for team members
5. **Use the fix-k8s-admin-ssh.yml playbook** if you encounter issues

## Emergency Recovery

If you lose k8s-admin access and still have root access:

```bash
# Connect as root
ssh -i /root/.ssh/k8s-test-env/id_rsa root@<BASTION_IP>

# Manually fix k8s-admin
sudo cp /root/.ssh/authorized_keys /home/k8s-admin/.ssh/authorized_keys
sudo chown k8s-admin:k8s-admin /home/k8s-admin/.ssh/authorized_keys
sudo chmod 600 /home/k8s-admin/.ssh/authorized_keys

# Verify
sudo -u k8s-admin ssh-keygen -l -f /home/k8s-admin/.ssh/authorized_keys
```

## Disabling Root SSH (Optional)

After confirming k8s-admin works, you can disable root SSH:

```bash
ansible-playbook -v -i examples/k8s-build-cluster/hosts.yml migrate-to-k8s-admin.yaml \
  -e ansible_user=k8s-admin \
  -e ansible_ssh_private_key_file=/root/.ssh/k8s-test-env/id_rsa \
  -e disable_root_ssh=true
```

**WARNING:** Only do this after thoroughly testing k8s-admin access!

## Support

For issues or questions:
1. Check this guide's troubleshooting section
2. Review Ansible output for specific error messages
3. Use `fix-k8s-admin-ssh.yml` for common SSH issues
4. Verify network connectivity and firewall rules

## Related Files

- `migrate-to-k8s-admin.yaml` - Main migration playbook
- `fix-k8s-admin-ssh.yml` - Emergency recovery playbook
- `roles/setup-k8s-admin-user/` - User creation role
- `examples/k8s-build-cluster/hosts.yml` - Inventory template
- `group_vars/all` - Global Ansible variables