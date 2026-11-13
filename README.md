# Ansible Server Hardening

Ansible playbooks that take a **fresh Ubuntu 22.04 VPS** from a default install to a
hardened, production-ready baseline: a non-root admin user, key-only SSH, a locked-down
firewall, fail2ban intrusion prevention, and automatic security updates.

> **Honest note:** this is a portfolio / lab project built to demonstrate Ansible and
> Linux hardening skills. It has been tested against disposable Ubuntu 22.04 servers.
> It is not a client engagement and makes no claims about production use by third parties.
> Review every variable before running it against a server you care about.

## What it does

- [x] Updates the APT cache and installs base packages (`curl`, `vim`, `htop`, `ufw`, `unattended-upgrades`)
- [x] Sets the server timezone
- [x] Creates a non-root **admin user** with sudo access and passwordless sudo
- [x] Installs your SSH public key for the admin user
- [x] Hardens SSH: key-only authentication, root login disabled, password auth disabled,
      configurable port, `MaxAuthTries 3`, idle-session timeouts
- [x] Configures **UFW**: deny incoming by default, allow outgoing, open only SSH/HTTP/HTTPS
- [x] Installs and configures **fail2ban** with an SSH jail (ban after repeated failures)
- [x] Enables **unattended-upgrades** so security patches are applied automatically

## Repository layout

```
.
├── ansible.cfg               # Project-wide Ansible defaults
├── site.yml                  # Main playbook — applies all roles in order
├── inventory/
│   └── hosts.ini             # Example inventory (edit with your server)
├── group_vars/
│   └── all.yml               # All tunable variables in one place
└── roles/
    ├── common/               # Base packages + timezone
    ├── users/                # Admin user, sudo, SSH key
    ├── ssh/                  # sshd_config hardening
    ├── firewall/             # UFW rules
    ├── fail2ban/             # SSH brute-force protection
    └── updates/              # Automatic security updates
```

## Requirements

- **Control node:** Ansible **2.15+** (`ansible --version`), Python 3, and the `community.general` + `ansible.posix` collections — install once with `ansible-galaxy collection install community.general ansible.posix`
- **Target:** a fresh **Ubuntu 22.04 LTS** server you can reach over SSH as root
  (or as a user with passwordless sudo) for the first run
- Your SSH **public key** on the control node (default: `~/.ssh/id_ed25519.pub`)

## Usage

1. Point the inventory at your server — edit `inventory/hosts.ini` and replace
   `vps.example.com` with your server's hostname or IP.

2. Adjust variables in `group_vars/all.yml` (admin username, SSH port, timezone, ports).

3. **Dry-run first** — see exactly what would change, change nothing:

   ```bash
   ansible-playbook -i inventory/hosts.ini site.yml --check --diff
   ```

4. Apply for real:

   ```bash
   ansible-playbook -i inventory/hosts.ini site.yml
   ```

5. After the run, log in as the new admin user **in a new terminal** and confirm
   sudo works *before* closing your original session:

   ```bash
   ssh -p 22 admin@your-server
   sudo whoami   # should print: root
   ```

## Variables you can change

All variables live in `group_vars/all.yml`:

| Variable | Default | Purpose |
|---|---|---|
| `admin_user` | `admin` | Name of the non-root admin account to create |
| `admin_ssh_public_key_file` | `~/.ssh/id_ed25519.pub` | Public key installed for the admin user |
| `ssh_port` | `22` | SSH listen port (change only if you update firewall rules too — the playbook does this for you via `ufw_allowed_ports`) |
| `server_timezone` | `UTC` | System timezone, e.g. `Asia/Karachi`, `Europe/London` |
| `ufw_allowed_ports` | SSH, 80, 443 | Incoming ports UFW will allow |
| `fail2ban_maxretry` | `5` | Failed logins before a ban |
| `fail2ban_bantime` | `3600` | Ban length in seconds |
| `fail2ban_findtime` | `600` | Window (seconds) in which failures are counted |

## ⚠️ Safety note — read before you run

This playbook **disables SSH password authentication and root login**. If your public
key is wrong or missing, you can lock yourself out of the server.

- **Keep a root / console session open** in a second terminal while testing any
  SSH changes, and only close it after you have verified key-based login as the
  admin user works in a *new* session.
- Always run with `--check` first, and test on a throwaway VPS before a real one.
- Make sure your hosting provider offers a web console / rescue mode as a fallback.
- The `sshd_config` is validated with `sshd -t` before it is installed, and Ansible
  will refuse to apply a config that fails validation — but a *valid* config with the
  *wrong key* will still lock you out. Double-check `admin_ssh_public_key_file`.

## Skills demonstrated

- Ansible: roles, handlers, templates (Jinja2), group variables, idempotent tasks,
  config validation (`validate:`), dry-run / check mode workflows
- Linux server hardening: SSH daemon configuration, sudoers management, UFW firewalling
- Intrusion prevention with fail2ban jails
- Unattended security patching with `unattended-upgrades`
- Safe change management for remote servers

## License

MIT — use it, fork it, adapt it for your own servers.
