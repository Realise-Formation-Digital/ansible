# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is an Ansible repository for managing Ubuntu-based Linux hosts. It targets servers in a homelab/self-hosted environment (Proxmox) and assumes SSH access with privilege escalation (`become: true`). Playbook execution is automated via **Semaphore UI** using the `maintenance` user.

## Setup

Install required Ansible collections before running any playbook:

```bash
ansible-galaxy collection install -r requirements.yml
```

All sensitive variables live in `group_vars/all/vault.yml` (auto-loaded for all hosts). Fill in the values and encrypt before use:

```bash
ansible-vault encrypt group_vars/all/vault.yml
ansible-vault edit group_vars/all/vault.yml   # edit after encryption
```

## Running Playbooks (CLI)

```bash
# Full first installation on new hosts
ansible-playbook first-installation/main.yml --ask-vault-pass

# Routine system updates
ansible-playbook updates/main.yml --ask-vault-pass

# Reboot UniFi switches
ansible-playbook unifi/reboot.yml -i inventory/localhost.yml --ask-vault-pass

# Dry run
ansible-playbook first-installation/main.yml --check --ask-vault-pass

# Lint
ansible-lint
```

## Semaphore UI Setup

### 1. Key Store

Create these entries under **Key Store**:

| Name | Type | Value |
|------|------|-------|
| `maintenance-ssh-key` | SSH Key | Private key paired with `vault_maintenance_ssh_public_key` in vault.yml |
| `vault-password` | Login with password | Password used to encrypt `group_vars/all/vault.yml` |

### 2. Repository

Point Semaphore to this git repository (SSH or HTTPS). Branch: `main`.

### 3. Inventory

Create two inventory entries:

| Name | Type | Path |
|------|------|------|
| `hosts` | File | `inventory/hosts.yml` |
| `localhost` | File | `inventory/localhost.yml` |

### 4. Task Templates

| Template name | Playbook | Inventory | SSH Key | Vault Key |
|---|---|---|---|---|
| Update Systems | `updates/main.yml` | `hosts` | `maintenance-ssh-key` | `vault-password` |
| First Installation | `first-installation/main.yml` | `hosts` | `maintenance-ssh-key` | `vault-password` |
| UniFi Reboot | `unifi/reboot.yml` | `localhost` | *(none)* | `vault-password` |

## Architecture

Two main entry points for managed hosts:

- **`first-installation/main.yml`** — Full setup for new hosts, in order:
  1. `updates/updates.yml` — apt upgrade + conditional reboot
  2. `packages/docker.yml` — Docker CE + Portainer Agent (port 9001)
  3. `packages/packages.yml` — monitoring, diagnostics, and utility tools
  4. `security/security.yml` — ufw firewall, fail2ban, unattended-upgrades
  5. `users/users.yml` — `maintenance` user + SSH key + SSH hardening

- **`updates/main.yml`** — Routine patching only, scheduled via Semaphore.

- **`unifi/reboot.yml`** — Reboots UniFi switches across 3 sites via REST API. Runs on localhost, no SSH needed. Uses `inventory/localhost.yml`.

### Key notes

- All secrets in `group_vars/all/vault.yml` — one vault file, one password for Semaphore.
- ufw open ports: 22 (SSH), 3000 (Grafana), 9001 (Portainer Agent), 9090 (Prometheus), 9100 (node exporter); outgoing 514 UDP/TCP (syslog).
- `maintenance` user: passwordless sudo, in `docker` group, SSH key from vault.
- Portainer Agent pinned to `2.21.0` in `docker.yml`.
- NTP and timezone in `packages.yml` are commented out pending internal NTP server.
