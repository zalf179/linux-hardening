# Linux Hardening with Ansible

An Ansible playbook that hardens an Ubuntu server (tested on Ubuntu 24.04 running in a VMware VM). Built as a learning project for infrastructure automation and Linux security.

**Result:** Lynis hardening index went from **67 → 80** after running the playbook.

## What it does

| Area | Changes |
|------|---------|
| **SSH** | Disables root login and password auth, key-only login, `MaxAuthTries 3`, `MaxSessions 2`, disables X11/TCP/agent forwarding, `LogLevel VERBOSE`, idle timeout |
| **Firewall** | Enables `ufw` with default deny incoming, SSH allowed with rate limiting |
| **fail2ban** | Installed and enabled, `jail.local` created so package updates don't overwrite config |
| **Kernel** | sysctl hardening (SYN cookies, rp_filter, no redirects/source routing, martian logging, `dmesg_restrict`, `kptr_restrict`, `suid_dumpable`) |
| **Modules** | Blacklists `dccp`, `sctp`, `rds`, `tipc` and `usb-storage` |
| **Auditing** | `auditd` enabled with rules for identity files, sudoers, SSH config and auth log |
| **Integrity** | AIDE database initialized for file integrity monitoring |
| **Malware scan** | `rkhunter` installed |
| **Passwords** | `libpam-pwquality`, `PASS_MAX_DAYS 365`, `PASS_MIN_DAYS 1`, `UMASK 027` |
| **Misc** | Core dumps disabled, legal login banners, unattended security upgrades |

## Requirements

**Control node** (where you run Ansible, e.g. Fedora or any Linux/WSL2):

```bash
sudo dnf install -y ansible        # Fedora
# sudo apt install -y ansible      # Debian/Ubuntu
ansible-galaxy collection install community.general ansible.posix
```

**Target server:** Ubuntu with SSH access using a key, and a user with sudo rights.

> Ansible does not run natively on Windows as a control node. Use WSL2 or a Linux machine.

## Setup

1. Set up SSH key login from the control node to the target:

   ```bash
   ssh-keygen -t ed25519
   ssh-copy-id user@TARGET_IP
   ssh user@TARGET_IP    # must log in without a password
   ```

2. Copy the example inventory and edit it:

   ```bash
   cp inventory.ini.example inventory.ini
   ```

   ```ini
   [ubuntu]
   192.168.x.x ansible_user=youruser ansible_ssh_private_key_file=~/.ssh/id_ed25519
   ```

3. Test connectivity:

   ```bash
   ansible -i inventory.ini ubuntu -m ping
   ```

## Usage

**Take a VM snapshot first and keep a second SSH session open**, so you can recover if a change locks you out.

```bash
# Dry run (some service tasks may be skipped because packages aren't installed in check mode)
ansible-playbook -i inventory.ini hardening.yml --check --diff -K

# Real run
ansible-playbook -i inventory.ini hardening.yml -K
```

`-K` prompts for the sudo password of the remote user. To avoid the prompt in a lab, configure passwordless sudo on the VM (not recommended for production).

The playbook is idempotent: running it again only changes what has drifted.

## Verify

```bash
# From the control node: password login should be rejected
ssh -o PubkeyAuthentication=no user@TARGET_IP      # Permission denied (publickey)

# On the server
sudo ufw status verbose
sudo systemctl status fail2ban auditd
sudo sshd -T | grep -Ei "passwordauth|permitroot|allowtcp"
sudo aide --check --config /etc/aide/aide.conf
sudo rkhunter --propupd
sudo lynis audit system
```

## Things to know

- **SSH tunnels are blocked** (`AllowTcpForwarding no`). Remove it from the playbook if you need port forwarding.
- **USB storage is blocked.** Remove the `usb-storage` line if you use USB passthrough in your hypervisor.
- **`MaxSessions 2`** can break tools that open many sessions over one connection. Raise it if needed.
- **`PASS_MAX_DAYS` only applies to new users.** For existing users run `sudo chage -M 365 username`.
- **AIDE** reports legitimate changes (updates, config edits) as differences. Re-initialize or update its database after planned changes.
- **Intentionally skipped** (high risk or low value for a lab VM): GRUB password, separate `/home` `/tmp` `/var` partitions, changing the SSH port, remote log host.
- `install_recommends: false` is used for the extra tools so `rkhunter` doesn't pull in a mail server.

## Project structure

```
.
├── hardening.yml           # Main playbook
├── inventory.ini.example   # Example inventory (copy to inventory.ini)
├── .gitignore
└── README.md
```

## Disclaimer

For learning and lab use. Review every task before running it on a real server, and test on a VM or staging machine first. Some settings may break applications that rely on the services or features they disable.
