[![CI](https://github.com/guidugli/ansible-role-firewall/actions/workflows/CI.yml/badge.svg)](https://github.com/guidugli/ansible-role-firewall/actions/workflows/CI.yml)
[![Release](https://img.shields.io/github/v/tag/guidugli/ansible-role-firewall?label=release)](https://github.com/guidugli/ansible-role-firewall/tags)
[![Galaxy](https://img.shields.io/badge/galaxy-guidugli.firewall-blue)](https://galaxy.ansible.com/ui/standalone/roles/guidugli/firewall/)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)

## Ansible Role: firewall

Install and configure a Linux host firewall using either firewalld or UFW. The role keeps privilege escalation external to the caller, validates public inputs through argument specs and semantic checks, and is structured for template-aligned CI, linting, and Molecule coverage.

### Requirements

- Ansible Core compatible with the role metadata.
- Linux targets using a supported package manager and service manager for the selected backend.
- Root-level permissions are required on real hosts for package installation, service management, `/etc/default/ufw`, firewalld, UFW, and firewall rule changes. Provide privilege externally with `become: true`.
- Collections from `requirements.yml`: `containers.podman`, `ansible.posix`, and `community.general`.

### Features

- Selects a default backend from the target distribution while allowing `firewall_selected` override.
- Installs required backend packages for firewalld or UFW.
- Stops known conflicting firewall services when supported by the execution context.
- Manages named services, numeric ports, optional firewalld zones, and custom service mappings.
- Supports optional default-deny outbound policy with explicit allowed egress ports.
- Detects IPv6 availability and avoids enabling unsupported UFW IPv6 behavior.
- Keeps SSH connectivity safer by preserving the active SSH port at runtime for SSH connections.
- Provides default and systemd Molecule scenario support through shared converge and verify logic.

### Supported platforms

Role metadata declares Fedora, Ubuntu, and Debian support. The shared Molecule matrix covers Ubuntu 26.04 and 24.04, Debian 13 and 12, and Fedora 44 and 43 in Podman-backed scenarios.

### Variables

All public inputs are defined in `defaults/main.yml` and mirrored in `meta/argument_specs.yml`.

| Variable | Type | Default | Description |
| --- | --- | --- | --- |
| `firewall_selected` | string | `{{ _suggested_os_firewall }}` | Firewall backend. Valid values are `firewalld` and `ufw`. |
| `firewalld_default_zone` | string | `public` | Default firewalld zone when a rule does not define `zone`. |
| `firewall_default_protocol` | string | `tcp` | Default protocol for numeric port rules. Valid values are `tcp` and `udp`. |
| `firewall_default_action` | string | `allow` | Default action for `firewall_services`. Valid values are `allow` and `deny`. |
| `firewall_interface_zone` | list(dict) | `[]` | Firewalld interface-to-zone mappings. Each item requires `interface` and `zone`. |
| `firewall_service_mapping` | list(dict) | `[]` | Service-name to `port/protocol` mappings for custom services and UFW expansion. |
| `firewall_output_default_action` | string | `allow` | Outgoing policy. Use `deny` with `firewall_output_allow_ports` for allowed egress. |
| `firewall_output_allow_ports` | list(dict) | HTTP, HTTPS, DNS, DHCP | Outgoing ports allowed when the output default is `deny`. |
| `firewall_services` | list(dict) | `[]` | Inbound services or ports to manage. Each item requires `name`. |

#### `firewall_services` example

```yaml
firewall_services:
  - name: ssh
  - name: cockpit
    action: deny
  - name: 8443
    protocol: tcp
```

#### `firewall_service_mapping` example

```yaml
firewall_service_mapping:
  - name: api
    ports:
      - 8443/tcp
  - name: kodi
    short: Allows kodi remotes
    description: Allows kodi remote controls to connect
    ports:
      - 18998/tcp
```

### Example Playbook

```yaml
---
- name: Configure host firewall
  hosts: all
  become: true
  roles:
    - role: guidugli.firewall
```

```yaml
---
- name: Configure application firewall
  hosts: app
  become: true
  vars:
    firewall_selected: ufw
    firewall_output_default_action: deny
    firewall_services:
      - name: ssh
      - name: 8443
        protocol: tcp
  roles:
    - role: guidugli.firewall
```

### Molecule Testing Instructions

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements-dev.txt
ansible-galaxy collection install -r requirements.yml
yamllint .
ansible-lint .
molecule test -s default
molecule test -s systemd
```

### Execution Notes

- **Privilege model:** the role never declares `become`, `become_user`, or `become_method`. Use `become: true` externally for real hosts where package, service, or firewall changes require elevated privileges.
- **Container behavior:** UFW runtime rule management and non-systemd container service management are skipped because container runtimes commonly restrict kernel firewall capabilities and init/service state is not stable in ordinary containers.
- **Molecule default scenario:** non-systemd containers do not exercise firewalld on Fedora. Those hosts are ended early in shared Molecule logic. The systemd scenario is the coverage path for firewalld.
- **APT cache behavior:** the default Molecule prepare step removes package lists to reduce image size, so the role refreshes apt metadata as a separate non-changing task before package installation.
- **Systemd behavior:** firewalld requires a systemd-managed target. The role keeps the real-host safety failure and uses the systemd Molecule scenario for coverage.

### Release workflow

Generated metadata and inventories are refreshed through repository generator scripts before a release:

```bash
./scripts/update_release_metadata.sh
./scripts/release.sh --version v1.2.0 --message "Release v1.2.0"
```

### License

MIT

### Author

Carlos Guidugli
