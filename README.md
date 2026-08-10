[![CI](https://github.com/guidugli/ansible-role-firewall/actions/workflows/CI.yml/badge.svg)](https://github.com/guidugli/ansible-role-firewall/actions/workflows/CI.yml)
[![Release](https://img.shields.io/github/v/tag/guidugli/ansible-role-firewall?label=release)](https://github.com/guidugli/ansible-role-firewall/tags)
[![Galaxy](https://img.shields.io/badge/galaxy-guidugli.firewall-blue)](https://galaxy.ansible.com/ui/standalone/roles/guidugli/firewall/)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)

## Ansible Role: firewall

Install and configure the selected Linux host firewall backend using either firewalld or UFW. The role keeps privilege escalation external to the caller, validates public inputs through argument specs and semantic assertions, and is structured for template-aligned CI, linting, and Molecule coverage.

### Requirements

- Ansible Core compatible with the role metadata.
- Linux targets using a supported package manager and service manager for the selected backend.
- Root-level permissions are required on real hosts for package installation, service management, `/etc/default/ufw`, firewalld runtime/permanent configuration, and firewall rule changes. Provide privilege externally with `become: true` in the play or automation platform.
- Collections from `requirements.yml`: `containers.podman`, `ansible.posix`, and `community.general`, each with an explicit minimum version.

### Features

- Selects a default backend from the target distribution while allowing `firewall_selected` override.
- Installs the required backend packages for firewalld or UFW.
- Stops known conflicting firewall services when service facts show they are present.
- Manages named services, numeric ports, optional firewalld zones, and custom service mappings.
- Supports optional outbound default-deny posture with explicit allowed egress ports.
- Detects IPv6 availability and avoids enabling unsupported UFW IPv6 behavior.
- Keeps SSH connectivity safer by preserving the active SSH port at runtime for SSH-based connections.
- Includes shared Molecule converge and verify playbooks for default and systemd scenarios.

### Supported platforms

Role metadata declares support for Fedora, Ubuntu, and Debian. The repository Molecule shared matrix covers Ubuntu 26.04 and 24.04, Debian 13 and 12, and Fedora 44 and 43 in Podman-backed scenarios.

### Variables

All public inputs are defined in `defaults/main.yml` and mirrored in `meta/argument_specs.yml`.

| Variable | Type | Default | Description |
| --- | --- | --- | --- |
| `firewall_selected` | string | `{{ _suggested_os_firewall }}` | Firewall backend to configure. Valid values are `firewalld` and `ufw`. The default is derived from the distribution in `vars/main.yml`. |
| `firewalld_default_zone` | string | `public` | Default firewalld zone used when a rule does not set `zone`. Applies only when `firewall_selected` is `firewalld`. |
| `firewall_default_protocol` | string | `tcp` | Protocol used for numeric port rules when an item in `firewall_services` does not define `protocol`. Valid values are `tcp` and `udp`. |
| `firewall_default_action` | string | `allow` | Action used when a `firewall_services` item does not define `action`. Valid values are `allow` and `deny`. |
| `firewall_interface_zone` | list(dict) | `[]` | Optional firewalld interface-to-zone mappings. Each item requires `interface` and `zone`. This is valid only with firewalld. |
| `firewall_service_mapping` | list(dict) | `[]` | Maps service names to one or more `port/protocol` strings. Used for custom firewalld services and for expanding named services into UFW port rules. |
| `firewall_output_default_action` | string | `allow` | Default outgoing policy. Use `deny` to add only the ports listed in `firewall_output_allow_ports`. |
| `firewall_output_allow_ports` | list(dict) | HTTP, HTTPS, DNS, and DHCP ports | Outgoing ports allowed when `firewall_output_default_action` is `deny`. Each item requires integer `port` and `protocol`. |
| `firewall_services` | list(dict) | `[]` | Inbound services or ports to manage. Each item requires `name`; optional keys are `action`, `protocol`, and firewalld-only `zone`. |

#### `firewall_services` examples

```yaml
firewall_services:
  - name: ssh
  - name: cockpit
    action: deny
  - name: 8443
    protocol: tcp
    action: allow
```

#### `firewall_service_mapping` examples

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

#### Minimal example

```yaml
---
- name: Configure host firewall
  hosts: all
  become: true
  roles:
    - role: guidugli.firewall
```

#### Explicit UFW backend with outbound default deny

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

#### Explicit firewalld backend with custom service mapping

```yaml
---
- name: Configure firewalld custom service
  hosts: app
  become: true
  vars:
    firewall_selected: firewalld
    firewall_service_mapping:
      - name: api
        short: API service
        description: Application API listener
        ports:
          - 8443/tcp
    firewall_services:
      - name: api
        zone: public
  roles:
    - role: guidugli.firewall
```

### Molecule Testing Instructions

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements-dev.txt
ansible-galaxy collection install -r requirements.yml
python3 scripts/update_matrix.py
python3 scripts/render_inventory.py
yamllint .
ansible-lint .
molecule test -s default
molecule test -s systemd
```

The shared scenario content is under `molecule/shared/`. Scenario inventories are generator-managed and should be refreshed from `molecule/shared/vars.yml` instead of edited directly.

### Execution Notes

- **Privilege model:** the role never declares `become`, `become_user`, or `become_method` in role tasks, handlers, or shared Molecule logic. Use `become: true` externally for real hosts when package, service, or firewall changes require elevated privileges.
- **Container behavior:** Molecule containers generally run as root and scenario converge uses external execution context. UFW runtime rule management and non-systemd container service management are skipped because container runtimes commonly restrict kernel firewall capabilities and init/service state is not stable in ordinary containers.
- **APT cache behavior:** the default Molecule prepare step removes package lists to reduce image size, so the role refreshes apt metadata as a separate non-changing task before package installation.
- **Molecule default scenario:** non-systemd containers do not exercise firewalld on Fedora; those hosts are ended early in `molecule/shared/converge.yml` and `molecule/shared/verify.yml`. The systemd scenario is the coverage path for firewalld.
- **Systemd behavior:** firewalld requires a systemd-managed target. Systemd-backed container coverage is provided by the systemd Molecule scenario, while role logic guards firewalld use on non-systemd hosts.
- **Idempotency:** commands that inspect state set `changed_when: false`; commands that modify firewall state are guarded by existing state checks, module idempotency, accepted return codes, or explicit change handling.

### Release workflow

Generated metadata and inventories are refreshed through repository generator scripts before a release:

```bash
./scripts/update_release_metadata.sh
./scripts/release.sh --version v1.2.0 --message "Release v1.2.0"
```

The release workflow imports tagged releases into Ansible Galaxy. Keep generated files, scenario inventories, `.github/`, and `scripts/` under generator control.

### License

MIT

### Author

Carlos Guidugli
