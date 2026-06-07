# Ansible Role: firewall

[![CI](https://github.com/guidugli/ansible-role-firewall/actions/workflows/CI.yml/badge.svg)](https://github.com/guidugli/ansible-role-firewall/actions/workflows/CI.yml)
[![Release](https://github.com/guidugli/ansible-role-firewall/actions/workflows/release.yml/badge.svg)](https://github.com/guidugli/ansible-role-firewall/actions/workflows/release.yml)
[![Galaxy](https://img.shields.io/badge/galaxy-guidugli.firewall-blue.svg)](https://galaxy.ansible.com/ui/standalone/roles/guidugli/firewall/)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

## Overview

This role installs and configures a host firewall using either **firewalld** or **ufw** on the supported platforms declared in `meta/main.yml`.

The role follows a modernized structure with:

- explicit public defaults in `defaults/main.yml`
- automatic input validation through `meta/argument_specs.yml`
- semantic validation in `tasks/assert.yml`
- clean task dispatch in `tasks/main.yml`
- generated metadata with `templates/meta_main.yml.j2` as the source of truth
- shared Molecule verification content in `molecule/shared/`

## Features

- installs the selected firewall backend and starts the related service
- stops conflicting firewall services when they are present
- supports named services and direct port rules
- supports firewalld custom services through `firewall_service_mapping`
- supports optional outgoing-policy tightening through `firewall_output_default_action`
- keeps behavior idempotent and readable

## Supported platforms

Current metadata declares support for:

- Fedora 42 and 43
- Ubuntu 22.04 (jammy) and 24.04 (noble)
- Debian 12 (bookworm) and 13 (trixie)

## Role variables

Defaults are defined in `defaults/main.yml`.

### Public variables

```yaml
---
firewall_selected: "{{ _suggested_os_firewall }}"
firewalld_default_zone: public
firewall_default_protocol: tcp
firewall_default_action: allow
firewall_interface_zone: []
firewall_service_mapping: []
firewall_output_default_action: allow
firewall_output_allow_ports:
  - port: 80
    protocol: tcp
  - port: 443
    protocol: tcp
  - port: 53
    protocol: tcp
  - port: 53
    protocol: udp
  - port: 67
    protocol: tcp
  - port: 67
    protocol: udp
firewall_services: []
```

### `firewall_services`

Each item may define:

- `name`: required; a service name (for example `ssh`) or a numeric port
- `action`: optional; `allow` or `deny`
- `protocol`: optional; `tcp` or `udp`
- `zone`: optional; firewalld only

Example:

```yaml
firewall_services:
  - name: ssh
  - name: cockpit
    action: deny
  - name: 8443
    protocol: tcp
```

### `firewall_service_mapping`

Useful when a service name is not natively known by the target backend.

Example:

```yaml
firewall_service_mapping:
  - name: cockpit
    ports:
      - 9090/tcp
  - name: kodi
    short: Allows kodi remotes
    description: Allows kodi remote controls to connect
    ports:
      - 18998/tcp
```

## How it works

1. Optional SSH-port detection runs for SSH-based connections.
2. Ansible applies `meta/argument_specs.yml` automatically.
3. `tasks/assert.yml` performs semantic checks that do not belong in argument specs.
4. The role installs the selected backend and disables conflicting services if present.
5. Backend-specific tasks apply rules for `firewalld` or `ufw`.

## Usage

### Standard usage

```yaml
---
- name: Configure firewall
  hosts: all
  become: true
  roles:
    - role: guidugli.firewall
```

### Example with explicit backend and custom rules

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
      - name: api
    firewall_service_mapping:
      - name: api
        ports:
          - 8443/tcp
  roles:
    - role: guidugli.firewall
```

## Design notes

- **Privilege escalation**: the role does not force `become` in tasks or handlers. Callers should set `become: true` in the play when required.
- **Validation model**: argument specs validate shape and type; `tasks/assert.yml` validates semantics and role-specific constraints.
- **Metadata source of truth**: `templates/meta_main.yml.j2` should be updated first, then `meta/main.yml` regenerated.
- **Molecule shared structure**: `molecule/shared/verify.yml` is provided so scenario files can reuse shared verification logic once the standard scenario wiring is in place.

## Molecule testing

Recommended commands:

```bash
yamllint .
ansible-lint
molecule test -s default
molecule test -s systemd
```

## Release workflow

- CI runs linting and Molecule tests.
- Release publishing is handled by the repository release workflow.
- Metadata should be regenerated from the template before tagging a release.

## Repository structure

```text
defaults/
handlers/
meta/
molecule/
tasks/
templates/
vars/
```

## License

MIT

## Author

Carlos Guidugli
