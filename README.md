# Ansible Remote Workstation

[![Ansible](https://img.shields.io/badge/Ansible-2.14+-EE0000?style=flat&logo=ansible&logoColor=white)](https://www.ansible.com/)
[![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04-E95420?style=flat&logo=ubuntu&logoColor=white)](https://ubuntu.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Automate the provisioning of a **24/7 remote workstation** on Ubuntu VMs using Ansible. This project transforms a bare Ubuntu server into a fully functional desktop environment accessible via Chrome Remote Desktop — ideal for continuous workflows, remote development, or headless desktop stations.

## Features

- **XFCE4 Desktop Environment** — Lightweight, stable, and resource-efficient
- **Chrome Remote Desktop** — Secure, cross-platform remote access with per-user service
- **Google Chrome** — Pre-installed web browser
- **UFW Firewall** — Pre-configured with custom SSH port and IP whitelisting (IPv4/IPv6)
- **24/7 Always-On** — Sleep, screen lock, DPMS, and power management fully disabled
- **D-Bus & Polkit** — Properly configured for CRD service permissions
- **Idempotent Design** — Safe to run multiple times without side effects

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Ubuntu 24.04 VM                      │
│                                                         │
│  ┌─────────────┐  ┌──────────────┐                      │
│  │   LightDM   │─▶│   XFCE4      │                      │
│  │  (Display   │  │  (Desktop    │                      │
│  │   Manager)  │  │   Session)   │                      │
│  └─────────────┘  └──────────────┘                      │
│         ▲                                               │
│         │                                               │
│  ┌──────────────┐                                       │
│  │ Chrome Remote│  ◀── Remote access from any device   │
│  │   Desktop    │                                       │
│  └──────────────┘                                       │
│                                                         │
│  ┌──────────────┐  ┌──────────────────────────────────┐ │
│  │     UFW      │  │  Sleep/Screen Lock Disabled      │ │
│  │  (Firewall)  │  │  (24/7 Operation)                │ │
│  └──────────────┘  └──────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

## Prerequisites

| Requirement | Version / Details |
|-------------|-------------------|
| **Control Node** | Linux/macOS/WSL with Ansible 2.14+ |
| **Target Host** | Ubuntu 24.04 LTS (amd64) |
| **SSH Access** | Root or sudo user with passwordless SSH |
| **Python 3** | Pre-installed on Ubuntu 24.04 |
| **RAM** | Minimum 2 GB (4 GB recommended) |
| **Disk** | Minimum 20 GB free space |

## Quick Start

### 1. Install Dependencies

```bash
# Install Ansible collections
ansible-galaxy install -r requirements.yml
```

### 2. Configure Inventory

Edit [`inventory.ini`](inventory.ini) with your VM's connection details:

```ini
[work_station]
work-station ansible_host=YOUR_VM_IP ansible_port=YOUR_SSH_PORT ansible_user=YOUR_SSH_USER
```

### 3. Set Environment Variables

```bash
# Required: Worker user credentials
export VAR_user_worker=worker
export VAR_user_worker_password=your_secure_password

# Optional: SSH port for firewall rules (default: 2223)
export TF_VAR_vm4_ssh_port=2223

# Optional: Trusted IP networks for firewall (comma-separated, empty = allow any)
export TF_FW_SRC_IP4="1.2.3.4,5.6.7.8"
export TF_FW_SRC_IP6="2001:db8::1"

# Optional: System timezone (default: Europe/Dublin)
export TIMEZONE=Europe/Dublin
```

### 4. Run the Playbook

```bash
ansible-playbook site.yml
```

With override variables:

```bash
ansible-playbook site.yml \
  -e "VAR_user_worker=myuser" \
  -e "VAR_user_worker_password=supersecret"
```

## Configuration Reference

### Environment Variables

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `VAR_user_worker` | Non-root user to create | — | Yes |
| `VAR_user_worker_password` | Password for the worker user | `changeme` | Yes |
| `VAR_user_worker_uid` | UID for the worker user (for runtime dir) | `1001` | No |
| `TF_VAR_vm4_ssh_port` | SSH port for UFW rules | `2223` | No |
| `TF_FW_SRC_IP4` | Trusted IPv4 networks (comma-separated) | *(allow any)* | No |
| `TF_FW_SRC_IP6` | Trusted IPv6 networks (comma-separated) | *(allow any)* | No |
| `TIMEZONE` | System timezone | `Europe/Dublin` | No |

### What Gets Installed

| Component | Purpose |
|-----------|---------|
| `xfce4` + `xfce4-goodies` | Desktop environment |
| `lightdm` + `lightdm-gtk-greeter` | Display manager (no auto-login, guest disabled) |
| `google-chrome-stable` | Web browser |
| `chrome-remote-desktop` | Remote access service (per-user) |
| `ufw` | Uncomplicated Firewall |
| `xdg-utils`, `dbus-x11`, `dbus-user-session` | X11 utilities and D-Bus session |
| `fonts-liberation`, `fonts-noto-color-emoji` | Font packages |
| `at-spi2-core`, `polkitd`, `policykit-1` | Accessibility and policy management |
| `libpam-systemd`, `systemd-sysv`, `sudo` | System utilities |

### Chrome Remote Desktop Configuration

The playbook configures CRD to run as a per-user service with proper D-Bus and polkit permissions:

| Configuration | Purpose |
|---------------|---------|
| System-wide service | Disabled, stopped, and masked (replaced by per-user instance) |
| Per-user service template | `chrome-remote-desktop@<user>` enabled and started |
| D-Bus policy | Allows worker user to own `org.chromium.RemoteDesktop` |
| Polkit rule | Grants suspend/hibernate permissions for CRD |
| Session files | `.chrome-remote-desktop-session`, `.xsession`, `.xsessionrc` all launch XFCE |
| Systemd override | Sets `HOME`, `DISPLAY`, `XDG_*`, and `DBUS_SESSION_BUS_ADDRESS` |
| User linger | Enabled via `loginctl enable-linger` so services run without active login |
| cidata handling | Unmounts cloud-init cidata ISO and adds udev rule to prevent automount |

### Post-Setup Steps

After the playbook completes, register Chrome Remote Desktop:

1. Follow the instructions at [remotedesktop.google.com/headless](https://remotedesktop.google.com/headless) to generate and run the registration command on the VM.

2. Set a PIN when prompted (at least 6 digits).

3. Manage your remote workstation at [remotedesktop.google.com/access](https://remotedesktop.google.com/access).

> **To add a new host**, follow the instructions at [remotedesktop.google.com/headless](https://remotedesktop.google.com/headless).
> **To manage existing hosts**, visit [remotedesktop.google.com/access](https://remotedesktop.google.com/access).

## Project Structure

```
.
├── ansible.cfg                 # Ansible configuration
├── inventory.ini               # Host inventory
├── requirements.yml            # Galaxy collection dependencies
├── site.yml                    # Main playbook entry point
├── .env                        # Environment variables (gitignored)
├── .envrc                      # direnv configuration (optional)
├── .gitignore                  # Git ignore rules
└── roles/
    └── work_station/
        ├── tasks/
        │   └── main.yml        # All provisioning tasks
        └── handlers/
            └── main.yml        # Service restart handlers
```

### Task Breakdown

The [`roles/work_station/tasks/main.yml`](roles/work_station/tasks/main.yml) role executes in this order:

1. **Hostname** — Sets `work-station` as the system hostname, updates `/etc/hosts`, unmounts cidata ISO, and adds udev rule to prevent automount
2. **System Updates** — Updates apt cache, fixes interrupted dpkg state, and upgrades all packages (`dist` upgrade with autoremove/autoclean)
3. **Timezone** — Configures system timezone (notifies rsyslog restart)
4. **Worker User** — Creates a non-root user with sudo access (password required), sets password via `chpasswd`, and adds to `chrome-remote-desktop` group
5. **UFW Firewall** — Configures firewall with custom SSH port and optional IP whitelisting (IPv4/IPv6), default-deny incoming
6. **XFCE4 Desktop** — Installs desktop environment, supporting utilities, fonts, and accessibility tools
7. **LightDM Display Manager** — Installs and configures LightDM with GTK greeter (no auto-login, guest disabled, lock-screen disabled)
8. **Google Chrome** — Downloads and installs Chrome stable from Google's repository
9. **Chrome Remote Desktop** — Installs CRD, configures per-user systemd service, D-Bus policy, polkit rules, session files, and enables user linger
10. **Power Management** — Disables all sleep, suspend, screen lock, DPMS, and XFCE screensaver for 24/7 operation
11. **Reboot** — Reboots the system if `/var/run/reboot-required` exists

## Security Considerations

- **Firewall First**: UFW is enabled with default-deny incoming policy
- **Custom SSH Port**: Default port 22 is not used; configure your port via `TF_VAR_vm4_ssh_port`
- **IP Whitelisting**: Optionally restrict SSH access to specific IPs via `TF_FW_SRC_IP4`/`TF_FW_SRC_IP6`
- **No Auto-Login**: LightDM is configured without automatic login
- **Password Management**: Use strong passwords and consider integrating with a secrets manager (e.g., Ansible Vault)

### Using Ansible Vault

For production use, encrypt sensitive variables:

```bash
# Create encrypted vars file
ansible-vault create group_vars/all/vault.yml

# Run playbook with vault
ansible-playbook site.yml --ask-vault-pass
```

## Troubleshooting

### Chrome Remote Desktop won't start

```bash
# Check per-user service status
systemctl status chrome-remote-desktop@<worker_user>

# View logs
journalctl -u chrome-remote-desktop@<worker_user> -f

# Re-register the host
/opt/google/chrome-remote-desktop/start-host
```

### Desktop not accessible via CRD

```bash
# Verify XFCE session config
cat ~/.chrome-remote-desktop-session

# Ensure lightdm is running
systemctl status lightdm
```

### Firewall blocking connections

```bash
# Check UFW status
sudo ufw status verbose

# Review allowed rules
sudo ufw show added
```

### System still going to sleep

```bash
# Verify sleep targets are masked
systemctl status sleep.target suspend.target

# Check xfce power manager config
cat ~/.config/xfce4/xfconf/xfce-perchannel-xml/xfce4-power-manager.xml
```

## Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.

## Acknowledgments

- [Ansible](https://www.ansible.com/) for automation
- [XFCE](https://www.xfce.org/) for the lightweight desktop
- [Chrome Remote Desktop](https://remotedesktop.google.com/) for remote access
