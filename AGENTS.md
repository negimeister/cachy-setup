# AGENTS.md

Ansible playbook provisioning a single CachyOS (Arch) workstation — `ngmstr`'s personal machine. Runs against `localhost` only (`inventory.yml` sets `ansible_connection: local`); tasks freely assume this exact host, user, and pacman environment.

This repo is the **system layer only**: packages, udev rules, docker+containers, tailscale, snapper, distrobox. Dotfiles, the keyboard layout, and secrets are managed by **chezmoi** (separate effort) — do not add dotfile/keyboard deployment here.

## Running

`ansible.cfg` in the repo root sets `inventory = inventory.yml` and `vault_password_file = ~/.secrets/ansible/cachy-setup` — no need to pass `-i` or `--vault-password-file`.

```bash
# Full run
ansible-playbook cachy.yml -e @secrets.enc --ask-become-pass

# Iterate on a subset (don't edit main.yml for this)
ansible-playbook cachy.yml -e @secrets.enc --start-at-task "Deploy udev rules"

# Edit secrets
ansible-vault edit secrets.enc
```

## Architecture

One role, `roles/cachy`. `cachy.yml` sets a UTF-8 locale env and applies it. `roles/cachy/tasks/main.yml` is a list of `import_tasks` lines, **all enabled** — comment a line out to disable a provisioning area. Do not add logic to `main.yml`; wire in task files.

Task files in `roles/cachy/tasks/`, roughly in run order:

- `packages.yml` — pacman system upgrade + canonical package list (includes `yay` from the Cachy repos); imports `cli_apps.yml`, `gui_apps.yml`, `aur_apps.yml`, `gui_aur_apps.yml`. Sets login shell to zsh, adds the user to `uucp`. **Add official-repo packages here or in the cli/gui split.**
- `aur_apps.yml` / `gui_aur_apps.yml` — AUR installs do NOT use a pacman module: they shell out to `yay -S` one group per task with a hand-rolled `changed_when` grepping stdout for `there is nothing to do`. **Copy an existing block and change package names + the `register` var when adding AUR packages.**
- `peripherals.yml` — installs udev rules from `files/*.rules` (Sofle keyboard, hidraw, probe-rs) to `/etc/udev/rules.d/`. The `jpdv` Dvorak keyboard layout and KDE `kxkbrc` are owned by chezmoi, not here.
- `docker.yml` / `containers.yml` — Docker install + Docker Hub login (needs `docker_hub_username`/`docker_hub_password`), LinuxServer containers (Syncthing, Kopia) under `/opt/containers` via `community.docker`.
- `distrobox.yml` — creates the `negibox` container from `negimeister/negibox-desktop:latest`, wired for realtime audio (`--cap-add=sys_nice`, uucp/realtime groups).
- `tailscale.yml` — gated on `tailscale_login_server` + `tailscale_auth_key` being defined.
- `snapper.yml` — Btrfs snapshot configs; iterates `snapper_subvolumes` (empty by default in `defaults/main.yml`).

Variables: `roles/cachy/defaults/main.yml` holds non-secret defaults. `secrets.enc` (ansible-vault) supplies the secret vars tasks reference but never define; several tasks `assert` these and fail early if secrets weren't passed via `-e @secrets.enc`.

## Conventions

- Tasks that shell out (AUR installs, git checkouts) must set a precise `changed_when` — never leave `command`/`shell` tasks reporting perpetual "changed". Match the surrounding grep-stdout style.
- Root operations use `become: yes`; user-scoped operations use `become_user: "{{ ansible_user }}"`.
