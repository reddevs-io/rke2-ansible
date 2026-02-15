# RKE2 Kubernetes Cluster Upgrade Playbook

An Ansible playbook for upgrading RKE2 Kubernetes clusters with sequential node upgrades, proper health checks, and error handling.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Inventory Structure](#inventory-structure)
- [Architecture](#architecture)
- [Usage](#usage)
- [Configuration Variables](#configuration-variables)
- [Upgrade Process](#upgrade-process)
- [Troubleshooting](#troubleshooting)
- [Files](#files)

## Prerequisites

### Control Node Requirements

- Ansible 2.9 or later
- SSH access to all RKE2 nodes
- Python 3.x installed on the control node
- **kubectl installed and configured** with access to the RKE2 cluster
  - The kubeconfig must be properly configured (default: `~/.kube/config` or via `KUBECONFIG` environment variable)
  - Test with: `kubectl get nodes` - should successfully list all cluster nodes

### Target Node Requirements

- RKE2 already installed and running
- SSH access with sudo privileges
- Python 3.x installed on all nodes
- **jq installed** (required for JSON parsing during version checks)
- Internet access to download RKE2 updates (or configured local mirror)

### Cluster Requirements

- At least one RKE2 server node must be operational

## Inventory Structure

The playbook requires two inventory groups:

| Group | Description |
|-------|-------------|
| `rke2_servers` | RKE2 server/control-plane nodes |
| `rke2_agents` | RKE2 agent/worker nodes |

### Example Inventory (INI format)

```ini
[rke2_servers]
rke2-server-01 ansible_host=192.168.1.10
rke2-server-02 ansible_host=192.168.1.11
rke2-server-03 ansible_host=192.168.1.12

[rke2_agents]
rke2-agent-01 ansible_host=192.168.1.20
rke2-agent-02 ansible_host=192.168.1.21
rke2-agent-03 ansible_host=192.168.1.22

[rke2_cluster:children]
rke2_servers
rke2_agents

[rke2_cluster:vars]
ansible_user=rke2admin
ansible_ssh_private_key_file=~/.ssh/id_rsa
ansible_python_interpreter=/usr/bin/python3
```

### Example Inventory (YAML format)

```yaml
---
all:
  children:
    rke2_cluster:
      children:
        rke2_servers:
          hosts:
            rke2-server-01:
              ansible_host: 192.168.1.10
            rke2-server-02:
              ansible_host: 192.168.1.11
            rke2-server-03:
              ansible_host: 192.168.1.12
        rke2_agents:
          hosts:
            rke2-agent-01:
              ansible_host: 192.168.1.20
            rke2-agent-02:
              ansible_host: 192.168.1.21
            rke2-agent-03:
              ansible_host: 192.168.1.22
      vars:
        ansible_user: rke2admin
        ansible_ssh_private_key_file: ~/.ssh/id_rsa
        ansible_python_interpreter: /usr/bin/python3
```

## Architecture

This playbook uses a role-based modular design with 5 specialized roles that execute sequentially for each node:

### Role Overview

| Role | Purpose |
|------|---------|
| [`rke2_prepare`](roles/rke2_prepare/) | System preparation (apt update, dist-upgrade, conditional reboot) |
| [`rke2_version_check`](roles/rke2_version_check/) | Version detection and comparison (installed vs. target) |
| [`rke2_node_drain`](roles/rke2_node_drain/) | Cordoning and draining nodes before upgrade |
| [`rke2_install`](roles/rke2_install/) | Installing/upgrading RKE2 binaries |
| [`rke2_node_restore`](roles/rke2_node_restore/) | Health checks and uncordoning nodes after upgrade |

### Skip Logic

The playbook includes intelligent skip logic to avoid unnecessary upgrades:

- The `rke2_version_check` role compares the currently installed RKE2 version against the target version
- If the installed version matches the target version, the upgrade process is skipped for that node
- This is controlled by the `skip_upgrade` fact set during version comparison
- Subsequent roles (`rke2_node_drain`, `rke2_install`, `rke2_node_restore`) check this fact and skip their tasks accordingly

### Execution Flow

For each node (servers first, then agents, one at a time):

1. **System Preparation** - Update apt cache, perform dist-upgrade, reboot if needed
2. **Version Check** - Determine if upgrade is necessary
3. **Node Drain** - Cordon and drain the node (if upgrade needed)
4. **RKE2 Installation** - Download and install new version (if upgrade needed)
5. **Node Restore** - Verify health and uncordon (if upgrade needed)

## Usage

### Basic Upgrade (Latest Version)

**Using INI inventory:**
```bash
ansible-playbook -i inventory/hosts.ini upgrade_rke2.yml
```

**Using YAML inventory:**
```bash
ansible-playbook -i inventory/hosts.yml upgrade_rke2.yml
```

### Upgrade to Specific Version

**Using INI inventory:**
```bash
ansible-playbook -i inventory/hosts.ini upgrade_rke2.yml -e "rke2_version=v1.28.4+rke2r1"
```

**Using YAML inventory:**
```bash
ansible-playbook -i inventory/hosts.yml upgrade_rke2.yml -e "rke2_version=v1.28.4+rke2r1"
```

### Dry Run (Check Mode)

**Using INI inventory:**
```bash
ansible-playbook -i inventory/hosts.ini upgrade_rke2.yml --check
```

**Using YAML inventory:**
```bash
ansible-playbook -i inventory/hosts.yml upgrade_rke2.yml --check
```

### Verbose Output

**Using INI inventory:**
```bash
ansible-playbook -i inventory/hosts.ini upgrade_rke2.yml -v
```

**Using YAML inventory:**
```bash
ansible-playbook -i inventory/hosts.yml upgrade_rke2.yml -v
```

## Configuration Variables

### System Preparation Role (`rke2_prepare`)

| Variable | Default | Description |
|----------|---------|-------------|
| `apt_cache_valid_time` | `3600` | Apt cache validity time in seconds |
| `reboot_timeout` | `600` | Timeout in seconds for reboot to complete |
| `reboot_connect_timeout` | `5` | Connection timeout during reboot check |
| `reboot_pre_delay` | `0` | Delay before initiating reboot |
| `reboot_post_delay` | `30` | Delay after reboot before checking connection |
| `wait_for_connection_delay` | `10` | Delay before starting connection checks |
| `wait_for_connection_timeout` | `300` | Timeout for connection to be established after reboot |

### Version Check Role (`rke2_version_check`)

| Variable | Default | Description |
|----------|---------|-------------|
| `rke2_version` | `""` | Target RKE2 version (empty for latest) |
| `rke2_github_api_url` | `https://api.github.com/repos/rancher/rke2/releases/latest` | GitHub API URL for fetching latest RKE2 release |

### Node Drain Role (`rke2_node_drain`)

| Variable | Default | Description |
|----------|---------|-------------|
| `drain_timeout` | `300` | Timeout in seconds for draining nodes |
| `drain_grace_period` | `30` | Grace period in seconds for pod termination |
| `drain_ignore_daemonsets` | `true` | Whether to ignore daemonsets during drain |
| `drain_force` | `true` | Whether to force drain |

### Install Role (`rke2_install`)

| Variable | Default | Description |
|----------|---------|-------------|
| `rke2_channel` | `latest` | RKE2 installation channel |
| `rke2_install_url` | `https://get.rke2.io` | RKE2 installation script URL |
| `rke2_version` | `""` | Target RKE2 version (empty for latest) |
| `service_check_retries` | `20` | Number of retries for service status checks |
| `service_check_delay` | `15` | Delay in seconds between service status checks |

> **Note:** The variable `service_start_timeout` is defined in the playbook but not currently used by the install role.

### Node Restore Role (`rke2_node_restore`)

| Variable | Default | Description |
|----------|---------|-------------|
| `health_check_retries` | `30` | Number of retries for health checks |
| `health_check_interval` | `10` | Interval in seconds between health check retries |

### Overriding Variables

**Via command line:**

Using INI inventory:
```bash
ansible-playbook -i inventory/hosts.ini upgrade_rke2.yml -e "drain_timeout=600 health_check_retries=60"
```

Using YAML inventory:
```bash
ansible-playbook -i inventory/hosts.yml upgrade_rke2.yml -e "drain_timeout=600 health_check_retries=60"
```

**Via inventory file (INI format):**
```ini
[rke2_cluster:vars]
drain_timeout=600
health_check_retries=60
```

**Via inventory file (YAML format):**
```yaml
rke2_cluster:
  vars:
    drain_timeout: 600
    health_check_retries: 60
```

**Via vars file:**

Using INI inventory:
```bash
ansible-playbook -i inventory/hosts.ini upgrade_rke2.yml -e "@vars.yml"
```

Using YAML inventory:
```bash
ansible-playbook -i inventory/hosts.yml upgrade_rke2.yml -e "@vars.yml"
```

## Upgrade Process

### Execution Order

1. **Server Nodes** - Upgraded sequentially (one at a time)
2. **Agent Nodes** - Upgraded sequentially (one at a time)
3. **Verification** - Final cluster health check

### Per-Node Upgrade Steps

For each server node:

1. **System Preparation**
   - Update apt cache (respects `apt_cache_valid_time`)
   - Perform dist-upgrade
   - Check if reboot is required
   - Reboot if needed and wait for connection

2. **Version Check**
   - Get currently installed RKE2 version
   - Fetch target version (latest from GitHub or specified version)
   - Compare versions and set `skip_upgrade` fact

3. **Cordon/Drain** (skipped if version matches)
   - Cordon the node (mark as unschedulable) - runs from localhost
   - Drain the node (evict pods) - runs from localhost

4. **RKE2 Installation** (skipped if version matches)
   - Download RKE2 install script
   - Execute install script with target version
   - Restart rke2-server service

5. **Service Health Check** (skipped if version matches)
   - Wait for RKE2 service to be active
   - Verify service is running properly

6. **Node Restore** (skipped if version matches)
   - Wait for node to be Ready - runs from localhost
   - Wait for pods to be running - runs from localhost
   - Uncordon the node (mark as schedulable) - runs from localhost

For each agent node, the same steps apply, but:
- Uses `rke2-agent` service instead of `rke2-server`
- Agents are upgraded after all servers are complete

### Safety Features

- **Serial execution**: Nodes are upgraded one at a time to maintain cluster availability
- **Version skip logic**: Avoids unnecessary upgrades when version already matches
- **Health checks**: Verifies node readiness before proceeding
- **Pod verification**: Ensures pods are running after upgrade
- **Error handling**: Fails fast on critical errors
- **Graceful drain**: Uses appropriate timeouts and grace periods
- **System preparation**: Ensures OS is up-to-date before RKE2 upgrade

## Troubleshooting

### Common Issues

#### Drain Timeout

If draining takes too long, increase the timeout:

**Using INI inventory:**
```bash
ansible-playbook -i inventory/hosts.ini upgrade_rke2.yml -e "drain_timeout=900"
```

**Using YAML inventory:**
```bash
ansible-playbook -i inventory/hosts.yml upgrade_rke2.yml -e "drain_timeout=900"
```

#### PDB Blocking Drain

If a PodDisruptionBudget is blocking the drain, you may need to manually handle it:
```bash
kubectl get pdb -A
```

#### Node Not Ready After Upgrade

Check the RKE2 service logs:
```bash
journalctl -u rke2-server -f  # On server nodes
journalctl -u rke2-agent -f   # On agent nodes
```

#### kubectl Connection Issues

Verify kubectl is properly configured on your local machine:
```bash
kubectl get nodes
```

If using a custom kubeconfig:
```bash
kubectl --kubeconfig /path/to/kubeconfig get nodes
```

Or set the KUBECONFIG environment variable:
```bash
export KUBECONFIG=/path/to/kubeconfig
kubectl get nodes
```

#### jq Not Found

If you get an error about `jq` not being found, install it on the target nodes:
```bash
# Ubuntu/Debian
sudo apt-get install -y jq

# RHEL/CentOS
sudo yum install -y jq
```

### Rollback

If an upgrade fails, you can rollback to a previous version:

**Using INI inventory:**
```bash
ansible-playbook -i inventory/hosts.ini upgrade_rke2.yml -e "rke2_version=v1.27.0+rke2r1"
```

**Using YAML inventory:**
```bash
ansible-playbook -i inventory/hosts.yml upgrade_rke2.yml -e "rke2_version=v1.27.0+rke2r1"
```

### Logs and Debugging

Enable verbose output for detailed logging:

**Using INI inventory:**
```bash
ansible-playbook -i inventory/hosts.ini upgrade_rke2.yml -vvv
```

**Using YAML inventory:**
```bash
ansible-playbook -i inventory/hosts.yml upgrade_rke2.yml -vvv
```

## Files

```
.
├── README.md                         # This documentation
├── upgrade_rke2.yml                  # Main playbook
├── inventory/
│   ├── example_hosts.ini             # Example INI inventory
│   └── example_hosts.yml             # Example YAML inventory
└── roles/
    ├── rke2_prepare/
    │   ├── defaults/
    │   │   └── main.yml              # Default variables for system preparation
    │   └── tasks/
    │       └── main.yml              # System preparation tasks
    ├── rke2_version_check/
    │   ├── defaults/
    │   │   └── main.yml              # Default variables for version checking
    │   └── tasks/
    │       └── main.yml              # Version detection and comparison tasks
    ├── rke2_node_drain/
    │   ├── defaults/
    │   │   └── main.yml              # Default variables for node draining
    │   └── tasks/
    │       └── main.yml              # Cordon and drain tasks
    ├── rke2_install/
    │   ├── defaults/
    │   │   └── main.yml              # Default variables for RKE2 installation
    │   └── tasks/
    │       └── main.yml              # RKE2 installation tasks
    └── rke2_node_restore/
        ├── defaults/
        │   └── main.yml              # Default variables for node restoration
        └── tasks/
            └── main.yml              # Health check and uncordon tasks
```

## License

MIT License

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request
