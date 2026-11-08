# Baseline Ansible Role

The **`baseline` role** provides baseline Linux configuration and identity management for Debian and Ubuntu systems.

User definitions are merged deterministically from inventory, applied to the operating system, and persisted for consumption by downstream roles. This allows a single source of truth for Linux users, access management, and platform-specific integrations.

In addition to identity management, the role can install baseline packages, configure timezone settings, and manage optional NFS mounts.

---

## Features

### User Management

- Create Linux users
- Create matching primary groups
- Manage supplemental Linux groups
- Optional UID and GID assignment
- Configure SSH authorized keys
- Configure passwordless sudo access
- Support system accounts
- Merge user definitions across inventory scopes
- Persist merged user metadata for downstream consumers

### Timezone Management

- Configure the system timezone
- Enabled by default
- Can be disabled per inventory scope

### Package Management

- Install baseline packages
- Skip installation when the package list is empty
- Enabled by default
- Can be disabled per inventory scope

### NFS Mounts

- Install NFS client packages
- Create mount points
- Configure persistent mounts
- Support custom mount options

### Metadata Persistence

- Persist merged user definitions
- Provide a consistent data contract for downstream roles
- Enable platform-wide identity reuse

---

## User Definition Model

This role intentionally does **not** allow direct use of:

```yaml
baseline_users:
```

Inventory should instead define users through explicit scopes:

| Scope | Variable |
|---------|---------|
| Global | `baseline_users_global` |
| Group | `baseline_users_group` |
| Host | `baseline_users_host` |

These scopes are processed in order:

```text
baseline_users_global
        ↓
baseline_users_group
        ↓
baseline_users_host
```

Definitions are merged by username.

When duplicate users exist:

- Scalar values are overridden by later scopes.
- Lists are merged using `append_rp`.
- Host values have the highest precedence.

Users can define `authorized_keys` to manage SSH public keys. The `authorized_keys_exclusive` variable defaults to `true` and controls whether unmanaged keys are removed.

Users can optionally define `groups` to manage supplemental Linux group membership. Referenced groups are created automatically and assigned to the user.

---

## Platform Metadata

User definitions may contain additional fields consumed by downstream roles.

### Example

```yaml
baseline_users_global:
  - name: admin
    groups:
      - docker
    department: infrastructure
    external_roles:
      - administrator
      - operator
```

The baseline role does not interpret these fields.

They are persisted to `/etc/ansible/baseline_users.yml` and may be consumed by downstream roles, allowing a single user definition to drive multiple platform integrations.

---

## Inventory Examples

### Global Users

```yaml
baseline_users_global:
  - name: admin
    uid: 1000
    sudo: true
    groups:
      - docker
    authorized_keys:
      - ssh-ed25519 AAAA...
```

### Group-Specific Users

```yaml
baseline_users_group:
  - name: postgres
    uid: 101
    system: true
```

### Host-Specific Overrides

```yaml
baseline_users_host:
  - name: postgres
    uid: 999
```

### Supplemental Groups

```yaml
baseline_users_global:
  - name: konkele

    groups:
      - docker
      - libvirt
      - wireshark
```

---

## Persisted User Metadata

After execution, the role writes the merged user model to:

```text
/etc/ansible/baseline_users.yml
```

This file acts as the canonical identity contract for downstream roles.

### Example

```yaml
baseline_users:
  - name: admin
    uid: 1000
    gid: 1000
    home: "/home/admin"
    shell: "/bin/bash"
    sudo: true
    system: false
    authorized_keys_exclusive: true

    groups:
      - docker

    authorized_keys:
      - ssh-ed25519 AAAA...

    tags:
      - platform-admin

  - name: postgres
    uid: 999
    gid: 999
    home: "/var/lib/postgresql"
    shell: "/usr/sbin/nologin"
    sudo: false
    system: true
    authorized_keys_exclusive: true
```

---

## Downstream Role Integration

Roles can load the generated user model and extend identities into external systems or platform-specific services.

```yaml
- name: Verify baseline user metadata exists
  ansible.builtin.stat:
    path: /etc/ansible/baseline_users.yml
  register: baseline_users_file

- name: Load baseline user metadata
  ansible.builtin.include_vars:
    file: /etc/ansible/baseline_users.yml
  when: baseline_users_file.stat.exists

- name: Abort if metadata is missing
  ansible.builtin.fail:
    msg: >-
      The baseline role must be executed before this role.
      Missing /etc/ansible/baseline_users.yml.
  when: not baseline_users_file.stat.exists
```

---

## Timezone Configuration

### Configure the System Timezone

```yaml
baseline_timezone: America/Chicago
```

### Disable Timezone Management

```yaml
baseline_timezone_manage: false
```

---

## Package Management

Define packages to install using `baseline_packages`:

```yaml
baseline_packages:
  - curl
  - htop
  - jq
  - unzip
```

Packages are installed using APT with `state: present`.

An empty list skips package installation.

### Disable Package Management

```yaml
baseline_packages_manage: false
```

### Run Only Package Tasks

```bash
ansible-playbook site.yml --tags packages
```

---

## NFS Mounts

### Single Mount

```yaml
baseline_nfs_mounts:
  - src: nas01:/exports/media
    dest: /mnt/media
    opts: rw,sync,hard,intr
    state: mounted
```

### Multiple Mounts

```yaml
baseline_nfs_mounts:
  - src: nas01:/exports/media
    dest: /mnt/media

  - src: nas01:/exports/backups
    dest: /mnt/backups
```

### Custom Mount Options

```yaml
baseline_nfs_mounts:
  - src: nas01:/exports/archive
    dest: /mnt/archive
    opts: rw,sync,hard,intr,noatime
```

---

## Example Playbook

```yaml
- name: Baseline Configuration
  hosts: all
  become: true

  roles:
    - role: baseline
```

---

## Tags

| Tag | Description |
|------|-------------|
| `users` | User and SSH configuration |
| `timezone` | Timezone configuration |
| `nfs` | NFS client configuration and mounts |
| `packages` | Package installation |