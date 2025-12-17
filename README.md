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

### Inventory

```ini
[amneziawg]
node1 ansible_host=192.168.1.10
node2 ansible_host=192.168.1.11
node3 ansible_host=192.168.1.12
```

### group_vars/amneziawg.yml

```yaml
# Shared obfuscation settings (must match on all peers)
amneziawg_jc: 4
amneziawg_jmin: 40
amneziawg_jmax: 70
# H1-H4 should be left at 0 (see Known Limitations below)
```

### host_vars/node1.yml

```yaml
amneziawg_addresses:
  - '10.10.0.1/24'
amneziawg_endpoint: 'node1.example.com'
```

### host_vars/node2.yml

```yaml
amneziawg_addresses:
  - '10.10.0.2/24'
amneziawg_endpoint: 'node2.example.com'
```

### host_vars/node3.yml

```yaml
amneziawg_addresses:
  - '10.10.0.3/24'
amneziawg_endpoint: 'node3.example.com'
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

## Known Limitations

### Custom H1-H4 Values

The H1-H4 parameters (header obfuscation) should be left at their default values (0). Setting custom values causes `awg setconf` to fail with "Invalid argument" due to a bug in the current amneziawg-tools package.

When H1-H4 are set to 0, the kernel module uses its default values (h1=1, h2=2, h3=3, h4=4).

The Jc/Jmin/Jmax (junk packet) and S1/S2 (padding) parameters work correctly and provide effective DPI bypass functionality.

## License

MIT

## Author

akme
