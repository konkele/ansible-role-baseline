# Ansible Role: Baseline

The **`baseline` role** provides baseline Linux configuration and identity management for Debian and Ubuntu systems.

User definitions are merged deterministically from inventory, applied to the operating system, and persisted for consumption by downstream roles. This allows a single source of truth for Linux users, access management, and platform-specific integrations.

In addition to identity management, the role can install baseline packages, configure timezone settings, and manage optional NFS mounts.

---

## Requirements

This role performs system-level configuration and executes privileged tasks using Ansible privilege escalation.

The account used to run the role must be permitted to use sudo on the target host.

---

## Features

### User Management

- Create Linux users
- Create matching primary groups
- Manage supplemental Linux groups
- Optional UID and GID assignment
- Configure SSH authorized keys
- Manage optional user passwords
- Explicitly remove user passwords
- Configure optional sudo privileges
- Support passwordless and password-required sudo
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
- Exclude plaintext passwords from persisted metadata

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

### Password Management

Users can optionally define a password or explicitly request password removal.

| Attribute | Default | Description |
|---|---|---|
| `password` | Undefined | Sets or updates the user's password |
| `password_remove` | `false` | Removes the user's existing password |

Passwords are supplied as plaintext variables, preferably protected using Ansible Vault. The role converts them into SHA-512 password hashes before applying them to Linux accounts.

The password hash uses a deterministic salt derived from the inventory hostname and username. This ensures repeated playbook runs produce the same hash rather than continually changing the account password.

Example:

```yaml
baseline_users_host:
  - name: admin
    password: "{{ vault_admin_password }}"

  - name: ansible
    password_remove: true

  - name: developer
```

Password behavior:

- `password` defined: Set or update the account password.
- `password` changed: Update the account password on the next run.
- `password_remove: true`: Remove the existing account password.
- Neither attribute defined: Leave the existing password unchanged.

Do not define `password` and `password_remove` together.

**Important:** Removing a password is not the same as locking an account. An account without a password may permit passwordless authentication through some local authentication configurations. Consider the security implications before removing passwords from normal user accounts.

Password values are excluded from `/etc/ansible/baseline_users.yml`. The role also suppresses task output for password-related operations.

#### Ansible Vault Example

Store the password in an encrypted Vault variable:

```yaml
vault_admin_password: "your-password"
```

Reference it from inventory:

```yaml
baseline_users_host:
  - name: admin
    password: "{{ vault_admin_password }}"
```

Encrypt a variable directly:

```bash
ansible-vault encrypt_string \
  --stdin-name 'vault_admin_password'
```

This keeps plaintext passwords out of ordinary inventory files.

### Sudo Configuration

Users can optionally define two attributes to manage sudo privileges:

| Attribute | Default | Description |
|---|---|---|
| `sudo` | `false` | Grants sudo privileges |
| `sudo_nopass` | `false` | Allows passwordless sudo; requires `sudo: true` |

Passwordless sudo is not enabled by default.

Example:

```yaml
baseline_users_global:
  - name: admin
    sudo: true
    sudo_nopass: true

  - name: ansible
    sudo: true
    sudo_nopass: true

  - name: developer
    sudo: true

  - name: postgres
    system: true
```

The resulting behavior:

- `sudo: false` or omitted: No sudo privileges.
- `sudo: true`: Sudo requires a password.
- `sudo: true` and `sudo_nopass: true`: Passwordless sudo.
- `sudo_nopass: true` without `sudo: true`: Invalid configuration.

Both attributes must be YAML booleans (`true` or `false`).

When sudo is disabled, the role removes its previously managed sudoers configuration for that user.

---

## Platform Metadata

User definitions may contain additional fields consumed by downstream roles.

### Example

```yaml
baseline_users_global:
  - name: admin

    groups:
      - docker

    sudo: true
    sudo_nopass: true

    department: infrastructure
    external_roles:
      - administrator
      - operator
```

The baseline role does not interpret these additional fields.

Any additional user attributes are preserved during the merge process and persisted to the generated metadata file, with the exception of plaintext password values. This allows a single user definition to act as the source of truth for Linux account management and platform-specific integrations.

Downstream roles can consume custom fields without reimplementing inventory merging logic.

---

## Persisted User Metadata

After execution, the role writes the fully merged user model to:

```text
/etc/ansible/baseline_users.yml
```

This file acts as the canonical identity contract for downstream roles.

Plaintext password values are never included in this file.

### Example

```yaml
baseline_users:
  - name: admin
    uid: 1000
    gid: 1000
    home: /home/admin
    shell: /bin/bash
    sudo: true
    sudo_nopass: true
    system: false
    authorized_keys_exclusive: true

    groups:
      - docker

    authorized_keys:
      - ssh-ed25519 AAAA...

    tags:
      - platform-admin

    department: infrastructure

    external_roles:
      - administrator
      - operator

  - name: postgres
    uid: 999
    gid: 999
    home: /var/lib/postgresql
    shell: /usr/sbin/nologin
    sudo: false
    system: true
    authorized_keys_exclusive: true
```

---

## Downstream Role Integration

Roles can consume the merged user model generated by the baseline role to extend identities into external systems, applications, and platform services.

The metadata file is stored on the managed host:

```text
/etc/ansible/baseline_users.yml
```

### Load Baseline User Metadata

```yaml
- name: Verify baseline user metadata exists
  ansible.builtin.stat:
    path: /etc/ansible/baseline_users.yml
  register: baseline_users_file

- name: Abort if metadata is missing
  ansible.builtin.fail:
    msg: >-
      The baseline role must be executed before this role.
      Missing /etc/ansible/baseline_users.yml.
  when: not baseline_users_file.stat.exists

- name: Read baseline user metadata
  ansible.builtin.slurp:
    src: /etc/ansible/baseline_users.yml
  register: baseline_users_raw

- name: Parse baseline user metadata
  ansible.builtin.set_fact:
    baseline_users_metadata: >-
      {{
        baseline_users_raw.content
        | b64decode
        | from_yaml
      }}
```

### Example Usage

Iterate over the merged user definitions:

```yaml
- name: Create application access
  ansible.builtin.debug:
    msg: "Managing external access for {{ item.name }}"
  loop: "{{ baseline_users_metadata.baseline_users }}"
```

Consume custom metadata fields:

```yaml
- name: Configure application roles
  ansible.builtin.debug:
    msg:
      user: "{{ item.name }}"
      department: "{{ item.department | default('') }}"
      roles: "{{ item.external_roles | default([]) }}"
  loop: "{{ baseline_users_metadata.baseline_users }}"
```

### Available Data

Downstream roles receive the fully merged user model produced by the baseline role, including any custom metadata defined in inventory.

Example persisted data:

```yaml
baseline_users:
  - name: admin
    department: infrastructure

    external_roles:
      - administrator
      - operator
```

A downstream role can directly reference:

```yaml
{{ item.name }}
{{ item.department }}
{{ item.external_roles }}
{{ item.groups }}
{{ item.authorized_keys }}
{{ item.sudo }}
{{ item.sudo_nopass }}
```

without duplicating merge logic or maintaining separate identity inventories.

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

  roles:
    - role: baseline
```

---

## Tags

| Tag | Description |
|------|-------------|
| `users` | User, password, sudo, and SSH configuration |
| `timezone` | Timezone configuration |
| `nfs` | NFS client configuration and mounts |
| `packages` | Package installation |