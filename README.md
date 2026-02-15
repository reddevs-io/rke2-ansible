# RKE2 Kubernetes Cluster Upgrade Playbook

An Ansible playbook for upgrading RKE2 Kubernetes clusters with sequential node upgrades, proper health checks, and error handling.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Inventory Structure](#inventory-structure)
- [Usage](#usage)
- [Configuration Variables](#configuration-variables)
- [Upgrade Process](#upgrade-process)
- [Troubleshooting](#troubleshooting)

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

## Usage

### Basic Upgrade (Latest Version)

```bash
ansible-playbook -i inventory/hosts.yml upgrade_rke2.yml
```

### Upgrade to Specific Version

```bash
ansible-playbook -i inventory/hosts.yml upgrade_rke2.yml -e "rke2_version=v1.28.4+rke2r1"
```

### Dry Run (Check Mode)

```bash
ansible-playbook -i inventory/hosts.yml upgrade_rke2.yml --check
```

### Upgrade Only Servers

```bash
ansible-playbook -i inventory/hosts.yml upgrade_rke2.yml --tags servers
```

### Upgrade Only Agents

```bash
ansible-playbook -i inventory/hosts.yml upgrade_rke2.yml --tags agents
```

### Verbose Output

```bash
ansible-playbook -i inventory/hosts.yml upgrade_rke2.yml -v
```

## Configuration Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `drain_timeout` | `300` | Timeout in seconds for draining nodes |
| `drain_grace_period` | `30` | Grace period in seconds for pod termination |
| `health_check_retries` | `30` | Number of retries for health checks |
| `health_check_interval` | `10` | Interval in seconds between health check retries |
| `service_start_timeout` | `300` | Timeout in seconds for service startup |
| `rke2_version` | `""` | Target RKE2 version (empty for latest) |

### Overriding Variables

**Via command line:**
```bash
ansible-playbook -i inventory/hosts.yml upgrade_rke2.yml -e "drain_timeout=600 health_check_retries=60"
```

**Via inventory file:**
```yaml
[rke2_cluster:vars]
drain_timeout=600
health_check_retries=60
```

**Via vars file:**
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
1. Cordon the node (mark as unschedulable) - runs from localhost
2. Drain the node (evict pods) - runs from localhost
3. Download and install RKE2 update
4. Restart rke2-server service
5. Wait for node to be Ready - runs from localhost
6. Wait for pods to be running - runs from localhost
7. Uncordon the node (mark as schedulable) - runs from localhost

For each agent node:
1. Cordon the node - runs from localhost
2. Drain the node - runs from localhost
3. Download and install RKE2 update
4. Restart rke2-agent service
5. Wait for node to be Ready - runs from localhost
6. Wait for pods to be running - runs from localhost
7. Uncordon the node - runs from localhost

### Safety Features

- **Serial execution**: Nodes are upgraded one at a time to maintain cluster availability
- **Health checks**: Verifies node readiness before proceeding
- **Pod verification**: Ensures pods are running after upgrade
- **Error handling**: Fails fast on critical errors
- **Graceful drain**: Uses appropriate timeouts and grace periods

## Troubleshooting

### Common Issues

#### Drain Timeout

If draining takes too long, increase the timeout:
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

If an upgrade fails, you can rollback to a previous version:
```bash
ansible-playbook -i inventory/hosts.yml upgrade_rke2.yml -e "rke2_version=v1.27.0+rke2r1"
```

### Logs and Debugging

Enable verbose output for detailed logging:
```bash
ansible-playbook -i inventory/hosts.yml upgrade_rke2.yml -vvv
```

## Files

```
.
├── README.md                    # This documentation
├── upgrade_rke2.yml             # Main playbook
└── inventory/
    ├── example_hosts.ini        # Example INI inventory
    └── example_hosts.yml        # Example YAML inventory
```

## License

MIT License

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request
