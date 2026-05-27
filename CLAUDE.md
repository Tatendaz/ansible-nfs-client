# CLAUDE.md

Guidance for AI assistants working in this repository.

## What this is

An Ansible **role** (not a full playbook project) that installs and configures
an NFS **client** on a host. It mounts a remote NFS share using systemd
`.mount` and `.automount` units rather than `/etc/fstab`. Target platform is
Ubuntu 22.

## Layout

Standard `ansible-galaxy init` role structure:

- `defaults/main.yml` — user-overridable variables (the role's public API)
- `vars/main.yml` — internal vars (currently empty)
- `tasks/` — task files, orchestrated by `tasks/main.yml`
- `handlers/main.yml` — handlers triggered via `notify`
- `templates/` — Jinja2 systemd unit templates (`.j2`)
- `meta/main.yml` — Galaxy metadata (still placeholder values)
- `tests/` — `inventory` + `test.yml` playbook for manual runs
- `README.md`, `LICENSE`

## Execution flow

`tasks/main.yml` imports task files in order:

1. `install.yml` — `apt` install of `nfs-common`
2. `create.yml` — create the mount-point directory (`client_share_folder`)
3. `configure_mount.yml` — render `nfsshare.mount.j2` to
   `/etc/systemd/system/{{ client_share_folder_systemd }}.mount`
4. `configure_automount.yml` — render `nfsshare.automount.j2` to the matching
   `.automount` unit, then `notify: "enable automount"`

Handlers in `handlers/main.yml`:

- `restart mount` — `systemd_service` restart + `daemon_reload` of the mount unit
- `enable automount` — `systemctl enable` the automount unit

`tasks/reload.yml` exists but is **not** wired into `main.yml` (the import is
commented out).

## Key variables (`defaults/main.yml`)

- `client_share_folder` — local mount point and remote export path (e.g. `/nfs/example`)
- `nfs_server` — NFS server IP/host
- `TimeoutIdleSec` — automount idle timeout
- `client_share_folder_systemd` — systemd unit base name; **must** equal
  `client_share_folder` with path separators replaced by `-` (except the
  leading slash). E.g. `/nfs/example` → `nfs-example`. systemd requires this
  encoding; a mismatch produces broken/duplicate units.

## Known gotchas

- **Handlers hardcode `nfs-example.mount` / `nfs-example.automount`** instead of
  using `client_share_folder_systemd`. If a user changes the share name, the
  restart/enable handlers will act on the wrong unit. Prefer wiring these to the
  variable when touching handlers.
- `client_share_folder_systemd` is a manual, error-prone duplicate of
  `client_share_folder`. Be careful keeping them consistent.
- `meta/main.yml` author/license/description are placeholders.

## Conventions

- All YAML files start with `---` and a `# ... file for ansible-nfs-client` comment.
- Prefer fully-qualified module names (`ansible.builtin.*`); some legacy tasks
  use short names (`apt`, `template`) — match the surrounding file when editing.
- Unit files are written `root:root` mode `u=rwx,g=,o=`.
- Use `notify` handlers for service restarts rather than restarting inline.

## Testing / running

No CI or lint config is present. To exercise the role manually, edit
`tests/inventory` with a reachable Ubuntu 22 host and run:

```
ansible-playbook -i tests/inventory tests/test.yml
```

Recommended local checks before committing:

```
ansible-lint
yamllint .
ansible-playbook --syntax-check -i tests/inventory tests/test.yml
```

(These tools are not vendored; install them if not present.)
