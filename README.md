# RKE2 Kubernetes Cluster Upgrade Playbook

An Ansible playbook for upgrading RKE2 Kubernetes clusters with sequential node upgrades, preflight validation, proper health checks, downgrade protection, and error handling that never leaves a node silently cordoned.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Inventory Structure](#inventory-structure)
- [Architecture](#architecture)
- [Usage](#usage)
- [Configuration Variables](#configuration-variables)
- [Upgrade Process](#upgrade-process)
- [Adding Worker Nodes](#adding-worker-nodes)
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

This playbook uses a role-based modular design with 6 specialized roles. A preflight play runs first on the control node, then each node is upgraded sequentially through 5 roles (version check now runs before system preparation so no-op upgrades skip the OS patching and reboot):

### Role Overview

| Role | Purpose |
|------|---------|
| [`rke2_preflight`](roles/rke2_preflight/) | Validates kubectl reachability, inventory groups, server readiness, and inventory/hostname match before any host is touched |
| [`rke2_version_check`](roles/rke2_version_check/) | Version detection and comparison (installed vs. target); sets the `skip_upgrade` fact and enforces the downgrade guard |
| [`rke2_prepare`](roles/rke2_prepare/) | System preparation (apt update, dist-upgrade, conditional reboot) — skipped on no-op runs unless `prepare_always_runs: true` |
| [`rke2_node_drain`](roles/rke2_node_drain/) | Cordoning and draining nodes before upgrade |
| [`rke2_install`](roles/rke2_install/) | Installing/upgrading RKE2 binaries |
| [`rke2_node_restore`](roles/rke2_node_restore/) | Health checks and uncordoning nodes after upgrade, with a `rescue` block that leaves the node cordoned on failure |

### Skip Logic

The playbook includes intelligent skip logic to avoid unnecessary work:

- The `rke2_version_check` role compares the currently installed RKE2 version against the target version
- If the installed version matches the target version, the `skip_upgrade` fact is set to `true` for that node
- `rke2_prepare` is then skipped (unless `prepare_always_runs: true`), avoiding an unnecessary `dist-upgrade` and reboot
- Subsequent roles (`rke2_node_drain`, `rke2_install`, `rke2_node_restore`) also check this fact and skip their tasks accordingly

### Downgrade Protection

By default the playbook refuses to downgrade RKE2. If the installed version is newer than the target version, the `rke2_version_check` role fails with an actionable message. To allow a downgrade (e.g. for rollback), pass:

```bash
-e "allow_downgrade=true"
```

### Execution Flow

1. **Preflight** (control node, once) — kubectl reachability, non-empty `rke2_servers` group, at least one server `Ready`, and inventory hostname matches a Kubernetes node name
2. **Version Check** — determine installed and target versions; set `skip_upgrade`; enforce downgrade guard
3. **System Preparation** (skipped if version matches, unless `prepare_always_runs: true`) — apt update, dist-upgrade, reboot only when `/var/run/reboot-required` exists
4. **Node Drain** (skipped if version matches) — cordon and drain from localhost
5. **RKE2 Installation** (skipped if version matches) — download and install new version
6. **Node Restore** (skipped if version matches) — wait for `Ready`, wait for pods `Running`/`Succeeded`, uncordon; on failure the node stays cordoned and the rescue block prints remediation guidance
7. **Post-Upgrade Verification** (control node, once) — all nodes `Ready` and all `kubeletVersion` values match the target version prefix

Servers are upgraded first (one at a time, `any_errors_fatal: true`), then agents (one at a time).

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

Shared variables for all hosts in the `rke2_cluster` group live in [`group_vars/rke2_cluster.yml`](group_vars/rke2_cluster.yml). Override them via `-e`, inventory vars, or a vars file (see [Overriding Variables](#overriding-variables) below).

### Preflight Role (`rke2_preflight`)

| Variable | Default | Description |
|----------|---------|-------------|
| `preflight_strict_hostname_check` | `true` | When `true`, a mismatch between `inventory_hostname` and the Kubernetes node name is a hard failure. Set to `false` to downgrade to a warning |

### System Preparation Role (`rke2_prepare`)

| Variable | Default | Description |
|----------|---------|-------------|
| `prepare_always_runs` | `false` | When `false`, the prepare role (apt dist-upgrade + reboot) is skipped on no-op upgrades. Set to `true` to keep OS patching on every run regardless of the RKE2 skip decision |
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
| `github_token` | `""` | Optional GitHub token to raise API rate limit from 60 to 5000 req/hr |
| `allow_downgrade` | `false` | When `false`, the role refuses to downgrade RKE2. Set to `true` to allow rollback scenarios |

### Node Drain Role (`rke2_node_drain`)

| Variable | Default | Description |
|----------|---------|-------------|
| `drain_timeout` | `300` | Timeout in seconds for draining nodes |
| `drain_grace_period` | `30` | Grace period in seconds for pod termination |
| `drain_ignore_daemonsets` | `true` | Whether to ignore daemonsets during drain |
| `drain_force` | `true` | Whether to force drain |
| `drain_delete_emptydir_data` | `true` | Whether to delete pods using an emptyDir volume during drain |

### Install Role (`rke2_install`)

| Variable | Default | Description |
|----------|---------|-------------|
| `rke2_channel` | `latest` | RKE2 installation channel |
| `rke2_install_url` | `https://get.rke2.io` | RKE2 installation script URL |
| `rke2_version` | `""` | Target RKE2 version (empty for latest) |
| `service_check_retries` | `20` | Number of retries for service status checks |
| `service_check_delay` | `15` | Delay in seconds between service status checks |

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

1. **Preflight Checks** (control node) — Validate kubectl, inventory groups, server readiness, and hostname match
2. **Server Nodes** - Upgraded sequentially (one at a time, `any_errors_fatal: true`)
3. **Agent Nodes** - Upgraded sequentially (one at a time)
4. **Post-Upgrade Verification** - Final cluster health and version check

### Per-Node Upgrade Steps

For each server node:

1. **Version Check**
   - Get currently installed RKE2 version
   - Fetch target version (latest from GitHub API or specified version)
   - Compare versions and set `skip_upgrade` fact
   - Refuse downgrade unless `allow_downgrade: true`

2. **System Preparation** (skipped if version matches, unless `prepare_always_runs: true`)
   - Update apt cache (respects `apt_cache_valid_time`)
   - Perform dist-upgrade
   - Reboot only when `/var/run/reboot-required` exists, then wait for connection

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
   - Wait for all pods on the node to be `Running` or `Succeeded` - runs from localhost
   - Uncordon the node (mark as schedulable) - runs from localhost
   - On failure, the node stays cordoned and the rescue block prints remediation guidance

For each agent node, the same steps apply, but:
- Uses `rke2-agent` service instead of `rke2-server`
- Agents are upgraded after all servers are complete

### Post-Upgrade Verification

After all nodes are processed, the control node verifies:
- All nodes report `Ready` status
- All nodes report a `kubeletVersion` matching the target version prefix (the RKE2 build suffix `+rke2r1` is not part of the kubelet version string)

### Safety Features

- **Preflight validation**: Fails fast on missing kubectl, empty inventory groups, no Ready server, or inventory/hostname mismatch before touching any host
- **Serial execution**: Nodes are upgraded one at a time to maintain cluster availability
- **`any_errors_fatal` on servers**: A failed server halts the entire run to protect etcd quorum
- **Downgrade guard**: Refuses to downgrade RKE2 unless `allow_downgrade: true`
- **Version skip logic**: Avoids unnecessary upgrades (and OS patching) when version already matches
- **Conditional reboot**: Reboots only when `/var/run/reboot-required` exists, not on every run
- **Health checks**: Verifies node readiness before proceeding
- **Pod verification**: Requires every pod on the node to be `Running` or `Succeeded` before uncordoning
- **Failure-aware restore**: On failure, the node stays cordoned (safer than auto-uncordoning) with an explicit remediation message
- **Post-upgrade version assertion**: Confirms nodes actually report the target version, not just `Ready`
- **Graceful drain**: Uses configurable timeouts, grace periods, and daemonset/emptyDir flags
- **System preparation**: Ensures OS is up-to-date before RKE2 upgrade (gated on the skip flag)

## Adding Worker Nodes

The `add_workers_rke2.yml` playbook joins fresh nodes to an existing RKE2 cluster as agent (worker) nodes. It does not modify any existing server or agent nodes.

### Prerequisites

- A healthy RKE2 cluster with at least one reachable `rke2_servers` host
- The join token from any server node:
  ```bash
  sudo cat /var/lib/rancher/rke2/server/node-token
  ```
- The server URL (typically `https://<server-ip-or-vip>:9345`)
- New worker nodes listed under the `rke2_new_agents` inventory group
- kubectl installed on the control node (optional; post-join verification is skipped gracefully if unavailable)

### Required Extra-Vars

| Variable | Description |
|----------|-------------|
| `rke2_token` | Join token obtained from an existing server node |
| `rke2_server_url` | URL of the RKE2 server, e.g. `https://192.168.1.10:9345` |

### Inventory

Add new worker nodes to the `rke2_new_agents` group. After a successful join, move them into `rke2_agents` so they are included in future upgrade runs.

**INI format:**
```ini
[rke2_new_agents]
rke2-new-worker-01 ansible_host=192.168.1.30
```

**YAML format:**
```yaml
rke2_new_agents:
  hosts:
    rke2-new-worker-01:
      ansible_host: 192.168.1.30
```

### Usage

**Using INI inventory:**
```bash
ansible-playbook -i inventory/hosts.ini add_workers_rke2.yml \
  -e "rke2_server_url=https://192.168.1.10:9345" \
  -e "rke2_token=K10abc...::server:xyz..."
```

**Using YAML inventory:**
```bash
ansible-playbook -i inventory/hosts.yml add_workers_rke2.yml \
  -e "rke2_server_url=https://192.168.1.10:9345" \
  -e "rke2_token=K10abc...::server:xyz..."
```

### Execution Flow

1. **Preflight Checks** — Validate required extra-vars and inventory groups before touching any remote host
2. **Version Detection** — Auto-detect the RKE2 version from the first `rke2_servers` host; new workers will install the same version to avoid version skew
3. **Join Workers** — For each host in `rke2_new_agents` (one at a time):
   - System preparation (apt update, dist-upgrade, conditional reboot)
   - Write agent configuration (`/etc/rancher/rke2/config.yaml`) with server URL and token
   - Install RKE2 agent binary
   - Enable `rke2-agent` service at boot
   - Post-join verification: wait for node to be `Ready` (skipped if kubectl is unavailable)

### Configuration Variables

#### Agent Join Role (`rke2_agent_join`)

| Variable | Default | Description |
|----------|---------|-------------|
| `rke2_token` | `""` | Join token (required, pass via `-e`) |
| `rke2_server_url` | `""` | Server URL (required, pass via `-e`) |
| `rke2_config_dir` | `/etc/rancher/rke2` | Directory for RKE2 configuration |
| `rke2_config_file` | `/etc/rancher/rke2/config.yaml` | Path to the agent config file |

All variables from the `rke2_prepare` and `rke2_install` roles also apply (see [Configuration Variables](#configuration-variables) above).

### Post-Join Step

After a successful join, move the host from `rke2_new_agents` to `rke2_agents` in your inventory so it is included in future upgrade runs.

## Troubleshooting

### Common Issues

#### Preflight — Inventory Hostname Mismatch

The preflight play fails with:

```
The following inventory hostnames are not Kubernetes node names: [...]
```

This means an entry in `rke2_servers` or `rke2_agents` does not match a node name known to Kubernetes. The playbook uses `inventory_hostname` directly in every `kubectl cordon/drain/uncordon` call, so a mismatch would silently target the wrong node. Fix the inventory so each host's `inventory_hostname` equals the OS hostname (which RKE2 registers as the node name). To downgrade to a warning instead of a hard failure:

```bash
ansible-playbook -i inventory/hosts.ini upgrade_rke2.yml -e "preflight_strict_hostname_check=false"
```

#### Preflight — Downgrade Refused

The version check fails with:

```
Installed version vX.Y.Z is newer than target vA.B.C. Downgrade is dangerous and refused by default.
```

This is the downgrade guard. If you intentionally want to roll back, pass `allow_downgrade=true` (see [Rollback](#rollback)).

#### Node Left Cordoned After Upgrade Failure

If `rke2_node_restore` cannot confirm the node is `Ready` and its pods are `Running`/`Succeeded`, the rescue block leaves the node **cordoned** and prints a remediation message. This is deliberate — do not uncordon a node whose health is unverified. Investigate first:

```bash
journalctl -u rke2-server -f   # or rke2-agent on worker nodes
kubectl describe node <node-name>
```

Once you have confirmed the service is healthy, manually uncordon:

```bash
kubectl uncordon <node-name>
```

#### Adding Worker Nodes — Token Mismatch

If you see `failed to validate token` in `journalctl -u rke2-agent`, verify the token matches the value on a server:
```bash
sudo cat /var/lib/rancher/rke2/server/node-token
```

#### Adding Worker Nodes — Firewall

Ensure TCP ports 9345 (supervisor) and 6443 (kube-api) are reachable from the new worker to the servers.

#### Adding Worker Nodes — Node Stuck NotReady

Check the agent logs:
```bash
journalctl -u rke2-agent -f
```

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

### Rollback

If an upgrade fails, you can rollback to a previous version. Because the playbook refuses downgrades by default, you must explicitly opt in with `allow_downgrade=true`:

**Using INI inventory:**
```bash
ansible-playbook -i inventory/hosts.ini upgrade_rke2.yml \
  -e "rke2_version=v1.27.0+rke2r1" \
  -e "allow_downgrade=true"
```

**Using YAML inventory:**
```bash
ansible-playbook -i inventory/hosts.yml upgrade_rke2.yml \
  -e "rke2_version=v1.27.0+rke2r1" \
  -e "allow_downgrade=true"
```

> **Warning:** Downgrading RKE2 is not supported by upstream and may leave the cluster in an inconsistent state. Prefer rolling forward to the next patch release when possible.

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
├── add_workers_rke2.yml              # Playbook to add worker nodes to an existing cluster
├── upgrade_rke2.yml                  # Main upgrade playbook
├── ansible.cfg                       # Ansible configuration
├── group_vars/
│   └── rke2_cluster.yml              # Shared variables for all rke2_cluster hosts
├── inventory/
│   ├── example_hosts.ini             # Example INI inventory
│   └── example_hosts.yml             # Example YAML inventory
└── roles/
    ├── rke2_preflight/
    │   ├── defaults/
    │   │   └── main.yml              # Default variables for preflight checks
    │   └── tasks/
    │       └── main.yml              # kubectl reachability, group, readiness, hostname checks
    ├── rke2_agent_join/
    │   ├── defaults/
    │   │   └── main.yml              # Default variables for agent join
    │   └── tasks/
    │       └── main.yml              # Agent configuration and join tasks
    ├── rke2_prepare/
    │   ├── defaults/
    │   │   └── main.yml              # Default variables for system preparation
    │   └── tasks/
    │       └── main.yml              # apt update, dist-upgrade, conditional reboot
    ├── rke2_version_check/
    │   ├── defaults/
    │   │   └── main.yml              # Default variables for version checking
    │   └── tasks/
    │       └── main.yml              # Version detection, comparison, skip flag, downgrade guard
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
            └── main.yml              # Health checks, uncordon, and rescue block
```

## License

MIT License

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request
