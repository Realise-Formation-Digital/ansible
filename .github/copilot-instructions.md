<!-- Copilot instructions for agents working on this Ansible repo -->
# Repo overview

This repository is a small set of Ansible playbooks for provisioning Debian/Ubuntu hosts with Docker, system packages, and basic user/SSH hardening. Key entry points:

- Main top-level playbook: [site.yml](site.yml)
- First-installation orchestration: [first-installation/main.yml](first-installation/main.yml)
- Installation subtasks: [first-installation/packages/docker.yml](first-installation/packages/docker.yml), [first-installation/packages/packages.yml](first-installation/packages/packages.yml)
- User/SSHD adjustments: [first-installation/users/users.yml](first-installation/users/users.yml)
- Update workflow: [updates/main.yml](updates/main.yml) and [updates/updates.yml](updates/updates.yml)

# Big picture (what to know first)

- This project uses simple playbooks (no roles). `include_tasks` stitches smaller yml files together (see `first-installation/main.yml`).
- The repo configures apt sources, installs Docker & related packages, creates a `docker` group and user membership, pulls a default image and creates multiple containers using `community.docker` modules.
- Updates are handled separately under `updates/` and include an automated reboot step that uses a hard-coded `test_command` ping target (adjust before running in new environments).

# Project-specific patterns and conventions

- No Ansible roles: follow the file-level task includes pattern (use `include_tasks` rather than adding roles).
- Variables are declared inline in `first-installation/main.yml` and `updates/main.yml` (for example `container_count`, `default_container_image`, etc.). Override these with `--extra-vars` when running playbooks.
- Uses `with_sequence` to create multiple Docker containers: see `first-installation/packages/docker.yml` (container names: `{{ default_container_name }}{{ item }}`).
- Docker operations use `community.docker.*` modules (ensure the collection is installed on the control machine).
- Tasks commonly use `ansible.builtin.*` modules (APT, service, stat, reboot). Look for `when:` guards paired with `register` + `stat`.

# Integration points & external dependencies

- External apt repositories and GPG keys are fetched directly (Docker GPG and apt repos). Review these network calls before running offline.
- The Docker tasks expect the target to be Debian/Ubuntu-like (apt-based) and to be reachable with `become: yes` privileges.
- The reboot task in `updates/updates.yml` runs `ansible.builtin.reboot` and tests with `ping -c 4 192.168.100.3` — replace that IP for your environment.
- Ensure `community.docker` collection is available: `ansible-galaxy collection install community.docker` (recommended).

# Common developer workflows (commands & examples)

- Run entire site (requires an inventory):

  ansible-playbook -i <inventory> site.yml -K

- Run the first-installation sequence only:

  ansible-playbook -i <inventory> first-installation/main.yml -K

- Run updates only:

  ansible-playbook -i <inventory> updates/main.yml -K

- Run only Docker-related tasks using tags (see `site.yml`):

  ansible-playbook -i <inventory> site.yml --tags install_docker -K

- Override variables at runtime (example):

  ansible-playbook -i <inventory> first-installation/main.yml --extra-vars "container_count=2 default_container_image=alpine" -K

- Debugging: add `-vvv` to increase verbosity and inspect registered vars (e.g., `deb_architecture` in `site.yml`).

# Safety notes & gotchas discovered in code

- There is no inventory in the repository — you must supply one (`-i`).
- The reboot `test_command` contains a static IP (192.168.100.3). Edit `updates/updates.yml` before use in different networks.
- The repo assumes a target user `realise` exists when adding them to the `docker` group in `first-installation/packages/docker.yml`.

# Quick references (useful lines/examples)

- `first-installation/main.yml`: uses `include_tasks` to run `updates.yml` and packages (`packages/docker.yml`, `packages/packages.yml`).
- `packages/docker.yml`: adds apt keys/repos, installs docker packages, creates group `docker`, adds user `realise`, pulls images and creates containers via `community.docker`.
- `updates/updates.yml`: performs `apt` upgrade, checks `/var/run/reboot-required` and reboots if present.

# If something is unclear

- Tell me which command or flow you want to run (example: "run on a VM" or "test container creation"), and I will update these instructions with exact command arguments and any inventory/variables you'll need.
