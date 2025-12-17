# Ansible Role: AmneziaWG

Deploy AmneziaWG mesh VPN networks with DPI obfuscation support.

[AmneziaWG](https://github.com/amnezia-vpn/amneziawg-go) is a fork of WireGuard with censorship-resistance features that help disguise VPN traffic from deep packet inspection (DPI).

## Features

- **Mesh networking**: Automatically configures full-mesh or hub-spoke topologies
- **Key management**: Auto-generates keys or uses provided ones; shares public keys via `hostvars`
- **DPI obfuscation**: Full support for AmneziaWG parameters (Jc, Jmin, Jmax, S1, S2, H1-H4)
- **Package installation**: Uses official PPA (Ubuntu/Debian) or COPR (RHEL/Fedora)
- **Kernel module**: Installs native kernel module via DKMS for best performance
- **Live reload**: Uses `awg syncconf` for zero-downtime config updates
- **Multi-OS**: Ubuntu, Debian, RHEL, Rocky, Alma Linux, Fedora

## Requirements

- Ansible 2.14+
- `community.general` collection (for `modprobe` and `copr` modules)
- Target: Ubuntu 22.04+, Debian 11+, or RHEL/Rocky/Alma 8+, Fedora 38+

## Role Variables

### Installation

```yaml
amneziawg_state: 'present'
amneziawg_update_cache: true
amneziawg_auto_reboot: false  # Set to true to allow automatic reboot when kernel upgrade is needed
```

> **Note:** On systems with outdated kernels where headers are unavailable, the role installs a newer kernel. By default (`amneziawg_auto_reboot: false`), the role will fail with instructions to reboot manually. Set `amneziawg_auto_reboot: true` to allow automatic reboot.

### Interface Settings

```yaml
amneziawg_interface: 'awg0'
amneziawg_port: '51820'
amneziawg_remote_directory: '/etc/amneziawg'
```

### Per-Host Variables (host_vars)

```yaml
# Required
amneziawg_addresses:
  - '10.10.0.1/24'

# Optional
amneziawg_endpoint: 'vpn1.example.com'  # Public endpoint (empty = spoke/client)
amneziawg_private_key: ''                # Auto-generated if empty
amneziawg_persistent_keepalive: '25'
amneziawg_dns: '1.1.1.1'
amneziawg_mtu: ''
```

### AmneziaWG Obfuscation (group_vars for mesh-wide consistency)

```yaml
# Junk packets (sent before handshake)
amneziawg_jc: 4       # Count of junk packets
amneziawg_jmin: 40    # Minimum size
amneziawg_jmax: 70    # Maximum size

# Message padding
amneziawg_s1: 0       # Handshake init padding
amneziawg_s2: 0       # Handshake response padding

# Header obfuscation
amneziawg_h1: 0       # Handshake init header
amneziawg_h2: 0       # Handshake response header
amneziawg_h3: 0       # Cookie header
amneziawg_h4: 0       # Transport header
```

### Topology

```yaml
amneziawg_as_spoke: false  # When true, only connects to hubs (hosts with endpoints)
```

## Example: 3-Node Mesh Network

### Inventory (YAML format)

```yaml
---
all:
  hosts:
    node1:
      ansible_host: 192.168.1.10
      ansible_user: root
      amneziawg_addresses:
        - '10.10.0.1/24'
      amneziawg_endpoint: '192.168.1.10'
    node2:
      ansible_host: 192.168.1.11
      ansible_user: root
      amneziawg_addresses:
        - '10.10.0.2/24'
      amneziawg_endpoint: '192.168.1.11'
    node3:
      ansible_host: 192.168.1.12
      ansible_user: root
      amneziawg_addresses:
        - '10.10.0.3/24'
      amneziawg_endpoint: '192.168.1.12'
  vars:
    amneziawg_interface: 'awg0'
    amneziawg_port: 51820

    # Obfuscation parameters (must match on all peers)
    amneziawg_jc: 4
    amneziawg_jmin: 40
    amneziawg_jmax: 70
    amneziawg_s1: 0
    amneziawg_s2: 0

    # Header obfuscation (optional, use large random integers)
    amneziawg_h1: 1234567890
    amneziawg_h2: 987654321
    amneziawg_h3: 1111111111
    amneziawg_h4: 2222222222
```

### Playbook

```yaml
---
- name: Deploy AmneziaWG mesh
  hosts: amneziawg
  roles:
    - role: amneziawg
```

## Example: Hub-Spoke Topology

For clients behind NAT without public endpoints:

### host_vars/hub.yml

```yaml
amneziawg_addresses:
  - '10.10.0.1/24'
amneziawg_endpoint: 'hub.example.com'
```

### host_vars/spoke1.yml

```yaml
amneziawg_addresses:
  - '10.10.0.10/24'
amneziawg_as_spoke: true
# No endpoint = client-only, will connect to hub
```

## Unmanaged Peers

Add peers not in the Ansible inventory:

```yaml
amneziawg_unmanaged_peers:
  mobile-client:
    public_key: 'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
    allowed_ips: '10.10.0.100/32'
    persistent_keepalive: '25'
```

## Generating Client Configurations

Use the included playbook to generate configuration files for desktop/mobile clients and automatically register them as peers on your servers:

```bash
ansible-playbook -i inventory/your-inventory.yml playbooks/generate-client-config.yml \
  -e client_name=my-laptop \
  -e client_address=10.10.0.100/24
```

### Required Variables

| Variable | Description |
|----------|-------------|
| `client_name` | Name for the client (used in filename and peer comments) |
| `client_address` | VPN address for the client (e.g., `10.10.0.100/24`) |

### Optional Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `client_dns` | - | DNS server for the client |
| `output_dir` | `./clients` | Local directory to save the config file |

The playbook will:

1. Generate a new key pair for the client
2. Create a `.conf` file with the client configuration (saved locally)
3. Add the client as a peer to all servers in the inventory
4. Apply the configuration using `awg syncconf` (zero-downtime)

The generated config file includes all AmneziaWG obfuscation parameters from your group_vars and can be imported directly into the [AmneziaWG client](https://github.com/amnezia-vpn/amneziawg-tools) or used with `awg-quick up`.

## Tags

- `amneziawg` - All tasks
- `amneziawg-install` - Installation only
- `amneziawg-keys` - Key generation
- `amneziawg-config` - Configuration only

## License

MIT

## Author

akme
