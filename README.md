# Ansible Remote Workstation

[![Ansible](https://img.shields.io/badge/Ansible-2.14+-EE0000?style=flat&logo=ansible&logoColor=white)](https://www.ansible.com/)
[![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04-E95420?style=flat&logo=ubuntu&logoColor=white)](https://ubuntu.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Automate the provisioning of a **24/7 remote workstation** on Ubuntu VMs using Ansible. This project transforms a bare Ubuntu server into a fully functional desktop environment accessible via Chrome Remote Desktop — ideal for AI-assisted research, continuous workflows, or headless development stations.

## Features

- **XFCE4 Desktop Environment** — Lightweight, stable, and resource-efficient
- **Chrome Remote Desktop** — Secure, cross-platform remote access
- **Google Chrome** — Pre-installed with auto-launch configuration
- **UFW Firewall** — Pre-configured with custom SSH port and IP whitelisting
- **24/7 Always-On** — Sleep, screen lock, and power management fully disabled
- **Auto-Launch Apps** — Chrome windows open automatically on boot (configurable URLs)
- **Idempotent Design** — Safe to run multiple times without side effects

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Ubuntu 24.04 VM                      │
│                                                         │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────┐ │
│  │   LightDM   │─▶│   XFCE4      │─▶│  Chrome Windows│ │
│  │  (Display   │  │  (Desktop    │  │  (Auto-launch) │ │
│  │   Manager)  │  │   Session)   │  │                │ │
│  └─────────────┘  └──────────────┘  └────────────────┘ │
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
| `TF_VAR_vm4_ssh_port` | SSH port for UFW rules | `2223` | No |
| `TF_FW_SRC_IP4` | Trusted IPv4 networks (comma-separated) | *(allow any)* | No |
| `TF_FW_SRC_IP6` | Trusted IPv6 networks (comma-separated) | *(allow any)* | No |
| `TIMEZONE` | System timezone | `Europe/Dublin` | No |

### What Gets Installed

| Component | Purpose |
|-----------|---------|
| `xfce4` + `xfce4-goodies` | Desktop environment |
| `lightdm` + `lightdm-gtk-greeter` | Display manager |
| `google-chrome-stable` | Web browser |
| `chrome-remote-desktop` | Remote access service |
| `ufw` | Uncomplicated Firewall |
| `xdg-utils`, `dbus-x11`, `fonts-*` | Supporting utilities |

### Post-Setup Steps

After the playbook completes, you must **register Chrome Remote Desktop**:

1. Connect to the VM via SSH as the worker user:
   ```bash
   ssh -p YOUR_SSH_PORT worker@YOUR_VM_IP
   ```

2. Start the CRD registration:
   ```bash
   /opt/google/chrome-remote-desktop/start-host
   ```

3. Follow the on-screen instructions to generate and paste an authorization URL.

4. Once registered, connect via [remotedesktop.google.com](https://remotedesktop.google.com/).

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

1. **Hostname** — Sets `work-station` as the system hostname
2. **System Updates** — Updates apt cache and upgrades all packages
3. **Timezone** — Configures system timezone
4. **Worker User** — Creates a non-root user with passwordless sudo
5. **UFW Firewall** — Configures firewall with custom SSH port and IP rules
6. **XFCE4 Desktop** — Installs desktop environment and display manager
7. **Google Chrome** — Downloads and installs Chrome stable
8. **Chrome Remote Desktop** — Installs and configures CRD service
9. **Power Management** — Disables all sleep, suspend, and screen lock features
10. **Auto-Launch** — Configures Chrome windows to open on boot
11. **Reboot** — Reboots the system if required after updates

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
# Check service status
systemctl status chrome-remote-desktop

# View logs
journalctl -u chrome-remote-desktop -f

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
