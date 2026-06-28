# Ansible NFS Client

> An Ansible role that mounts a remote **NFS share** on Ubuntu using **systemd `.mount` +
> `.automount` units** (not `/etc/fstab`) — install `nfs-common`, create the mount point, and
> let systemd mount the share on demand.

[![Ansible Role](https://img.shields.io/badge/Ansible-role-EE0000.svg?logo=ansible&logoColor=white)](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html)
[![Platform: Ubuntu 22](https://img.shields.io/badge/platform-Ubuntu%2022-E95420.svg?logo=ubuntu&logoColor=white)](https://ubuntu.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

## Overview

This role configures an NFS client on a Debian/Ubuntu host and mounts a remote export using
**systemd units** rather than `/etc/fstab`. With an `.automount` unit, the share is mounted
lazily on first access and unmounted again after an idle timeout. The role:

1. Installs the `nfs-common` package.
2. Creates the local mount-point directory.
3. Renders a systemd `.mount` unit (`Type=nfs4`) describing the remote share.
4. Renders a matching `.automount` unit and enables it (on-demand, idle-timed mounting).

It pairs with the companion [`ansible-nfs-server`](https://github.com/Tatendaz/ansible-nfs-server)
role, which exports the share.

## Features

- Idempotent install + configuration of the NFS client.
- systemd `.mount` / `.automount` units instead of `/etc/fstab` — on-demand mounting with an
  idle timeout.
- Mount point created automatically if it doesn't exist.
- `nfs4` mount with sensible defaults.
- Handlers restart the mount unit (with `daemon-reload`) and enable the automount unit on
  change.

## Requirements

- **Target OS:** Ubuntu 22 (Debian-family; uses the `apt` module, `nfs-common`, and systemd).
- **Ansible:** 2.1+ (`min_ansible_version` in `meta/main.yml`).
- **Privileges:** root (the play writes unit files under `/etc/systemd/system` and manages
  services).
- A reachable NFS server exporting the share (see `ansible-nfs-server`).

## Role variables

Defined in [`defaults/main.yml`](defaults/main.yml) — override them in your inventory, group/host
vars, or playbook:

| Variable | Default | Description |
|---|---|---|
| `client_share_folder` | `/nfs/example` | Local mount point **and** the remote export path to mount. |
| `nfs_server` | `10.128.0.5` | IP/hostname of the NFS server to mount from. |
| `TimeoutIdleSec` | `60` | Automount idle timeout (seconds) before the share is unmounted. |
| `client_share_folder_systemd` | `nfs-example` | Base name for the generated systemd units. **Must** equal `client_share_folder` with path separators replaced by `-` (no leading dash) — e.g. `/nfs/example` → `nfs-example`. systemd requires this encoding. |

> **Important:** `client_share_folder_systemd` is a manual, systemd-encoded mirror of
> `client_share_folder`; keep the two in sync or systemd will create broken/duplicate units.

## Example playbook

```yaml
- hosts: nfs_client
  become: yes
  roles:
    - role: ansible-nfs-client
      vars:
        nfs_server: "10.128.0.5"
        client_share_folder: "/nfs/example"
        client_share_folder_systemd: "nfs-example"
        TimeoutIdleSec: "60"
```

Minimal form (using the defaults):

```yaml
- hosts: all
  remote_user: root
  roles:
    - ansible-nfs-client
```

## How it works

`tasks/main.yml` imports the task files in order:

1. **`install.yml`** — `apt` install of `nfs-common`.
2. **`create.yml`** — create the mount-point directory (`client_share_folder`).
3. **`configure_mount.yml`** — render `nfsshare.mount.j2` →
   `/etc/systemd/system/{{ client_share_folder_systemd }}.mount`.
4. **`configure_automount.yml`** — render `nfsshare.automount.j2` → the matching `.automount`
   unit, then notify the *enable automount* handler.

Handlers (`handlers/main.yml`): **restart mount** runs a `systemd_service` restart with
`daemon_reload`; **enable automount** enables the automount unit.

> **Caveat:** the handlers currently reference the default unit name (`nfs-example.*`) rather
> than `client_share_folder_systemd`. If you change the share name, update the handlers too so
> they act on the right unit. (`tasks/reload.yml` exists but is intentionally not wired into
> the main flow.)

## Project structure

```
.
├── defaults/main.yml             # client_share_folder, nfs_server, TimeoutIdleSec, ... (public API)
├── vars/main.yml                 # internal vars (empty)
├── tasks/
│   ├── main.yml                  # orchestrates install → create → mount → automount
│   ├── install.yml               # apt install nfs-common
│   ├── create.yml                # create the mount-point directory
│   ├── configure_mount.yml       # render the systemd .mount unit
│   ├── configure_automount.yml   # render the systemd .automount unit + enable it
│   └── reload.yml                # standalone daemon-reload (not wired into main.yml)
├── handlers/main.yml             # restart mount + enable automount
├── templates/
│   ├── nfsshare.mount.j2         # systemd .mount unit (Type=nfs4)
│   └── nfsshare.automount.j2     # systemd .automount unit (idle timeout)
├── meta/main.yml                 # Galaxy metadata
└── tests/                        # inventory + test.yml for manual runs
```

## Testing

Point [`tests/inventory`](tests/inventory) at a reachable Ubuntu 22 host, then run the test
playbook:

```sh
ansible-playbook -i tests/inventory tests/test.yml
```

Recommended local checks before committing:

```sh
ansible-lint
yamllint .
ansible-playbook --syntax-check -i tests/inventory tests/test.yml
```

## License

[MIT](LICENSE) © Tatenda Zhou.
