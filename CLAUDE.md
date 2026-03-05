# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is an Ansible repository for managing Ubuntu-based Linux hosts. It targets servers in a homelab/self-hosted environment (Proxmox) and assumes SSH access with privilege escalation (`become: yes`).

## Setup

Install required Ansible collections before running any playbook:

```bash
ansible-galaxy collection install -r requirements.yml
```

Sensitive variables live in `group_vars/all/vault.yml` (loaded automatically for all hosts). Before first use, fill in the values and encrypt the file:

```bash
# Edit values first, then encrypt
ansible-vault encrypt group_vars/all/vault.yml

# Edit an already-encrypted file
ansible-vault edit group_vars/all/vault.yml
```

Run all playbooks with `--ask-vault-pass` (or `--vault-password-file`) after encrypting.

## Running Playbooks

```bash
# Run the full first-installation playbook against all hosts
ansible-playbook first-installation/main.yml -i <inventory>

# Run only system updates
ansible-playbook updates/main.yml -i <inventory>

# Run with a specific tag
ansible-playbook first-installation/main.yml -i <inventory> --tags install_docker

# Dry run (check mode)
ansible-playbook first-installation/main.yml -i <inventory> --check

# Lint playbooks
ansible-lint
```

## Architecture

Two entry points:

- **`first-installation/main.yml`** — Full setup for new hosts, in order:
  1. `updates/updates.yml` — apt upgrade + conditional reboot
  2. `packages/docker.yml` — Docker CE + Portainer Agent (port 9001)
  3. `packages/packages.yml` — monitoring, diagnostics, and utility tools
  4. `security/security.yml` — ufw firewall, fail2ban, unattended-upgrades
  5. `users/users.yml` — `maintenance` user (Semaphore UI) + SSH hardening

- **`updates/main.yml`** — Runs only `updates/updates.yml` for routine patching, used by Semaphore UI via the `maintenance` user.

### Key notes

- `docker.yml` targets Ubuntu; uses `dpkg --print-architecture` and `ansible_distribution_release` for the repository URL.
- ufw allows only ports 22, 9001 (Portainer Agent), 9100 (Prometheus node exporter); all other incoming traffic is denied.
- `maintenance` user has passwordless sudo and is in the `docker` group — add the Semaphore SSH public key to `/home/maintenance/.ssh/authorized_keys`.
- `updates/updates.yml` reboots if `/var/run/reboot-required` exists; post-reboot check uses `whoami`.
- NTP and timezone tasks in `packages.yml` are commented out pending an internal NTP server.
- Both `realise` and `maintenance` users are in the `docker` group.
