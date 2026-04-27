# ansible-vm-hardening

An Ansible playbook collection for automating Linux VM hardening, compliance enforcement, and monitoring agent deployment across an enterprise VM estate.

Built to automate real infrastructure tasks: patch management, SSH hardening, firewall enforcement, centralised syslog forwarding, and Zabbix agent deployment — the kind of repetitive work that should never be done manually across 100+ VMs.

---

## Playbooks

| # | Playbook | What it does |
|---|---|---|
| 1 | `01_patch_management.yml` | Applies security patches (apt/yum), reboots if kernel updated |
| 2 | `02_ssh_hardening.yml` | Enforces secure `sshd_config` — disables root login, password auth, sets timeouts |
| 3 | `03_user_account_audit.yml` | Finds and locks idle accounts, reports accounts with no password |
| 4 | `04_firewall_enforcement.yml` | Installs and configures `ufw` with a port whitelist |
| 5 | `05_syslog_forwarding.yml` | Configures `rsyslog` to forward all logs to a central syslog server |
| 6 | `06_zabbix_agent_deploy.yml` | Installs and configures the Zabbix agent, opens firewall port |

All playbooks can be run individually or together via `site.yml`.

---

## Requirements

### Control node (your machine)
- Python 3.10+
- Ansible 2.14+
- `community.general` collection (for `ufw` module)

```bash
pip install ansible
ansible-galaxy collection install community.general
```

### Target hosts
- Ubuntu 20.04/22.04 LTS **or** RHEL/CentOS/Rocky 8/9
- SSH key-based authentication configured
- The Ansible user must have passwordless `sudo`

---

## Quick start

**1. Clone the repo**
```bash
git clone https://github.com/yourusername/ansible-vm-hardening.git
cd ansible-vm-hardening
```

**2. Edit the inventory**
```bash
nano inventory/hosts.ini
```
Replace the example IPs with actual hosts. Group them as `production`, `monitoring`, or `dmz`.

**3. Edit variables**
```bash
nano inventory/group_vars/all.yml
```
Key values to update:
- `syslog_server_ip` — IP of centralised syslog server
- `zabbix_server_ip` — IP of Zabbix server
- `ufw_allowed_tcp_ports` — ports to whitelist on all VMs
- `ssh_allowed_users` — restrict SSH to specific usernames (leave empty to allow all)

**4. Test connectivity**
```bash
ansible all_vms -i inventory/hosts.ini -m ping
```

Expected output:
```
192.168.1.10 | SUCCESS => { "ping": "pong" }
192.168.1.11 | SUCCESS => { "ping": "pong" }
...
```

**5. Dry run (check mode — no changes made)**
```bash
ansible-playbook -i inventory/hosts.ini site.yml --check --diff
```

**6. Run the full pipeline**
```bash
ansible-playbook -i inventory/hosts.ini site.yml
```

---

## Running individual playbooks

```bash
# Patch only
ansible-playbook -i inventory/hosts.ini playbooks/01_patch_management.yml

# SSH hardening on production only
ansible-playbook -i inventory/hosts.ini playbooks/02_ssh_hardening.yml --limit production

# User audit — check mode, no changes
ansible-playbook -i inventory/hosts.ini playbooks/03_user_account_audit.yml --check

# Firewall on DMZ hosts only
ansible-playbook -i inventory/hosts.ini playbooks/04_firewall_enforcement.yml --limit dmz

# Syslog forwarding
ansible-playbook -i inventory/hosts.ini playbooks/05_syslog_forwarding.yml

# Deploy Zabbix agent
ansible-playbook -i inventory/hosts.ini playbooks/06_zabbix_agent_deploy.yml
```

---

## Running by tag

Tags let you run a subset of the full pipeline:

```bash
# Run only hardening-related playbooks (SSH + firewall)
ansible-playbook -i inventory/hosts.ini site.yml --tags hardening

# Run only monitoring setup
ansible-playbook -i inventory/hosts.ini site.yml --tags monitoring

# Run patching and logging
ansible-playbook -i inventory/hosts.ini site.yml --tags patch,logging
```

Available tags: `patch` · `ssh` · `hardening` · `users` · `audit` · `firewall` · `syslog` · `logging` · `zabbix` · `monitoring`

---

## Repository structure

```
ansible-vm-hardening/
├── ansible.cfg                        # Ansible configuration (auto-loaded)
├── site.yml                           # Master playbook — runs all playbooks in order
├── inventory/
│   ├── hosts.ini                      # Host inventory with groups
│   └── group_vars/
│       └── all.yml                    # Shared variables for all hosts
└── playbooks/
    ├── 01_patch_management.yml
    ├── 02_ssh_hardening.yml
    ├── 03_user_account_audit.yml
    ├── 04_firewall_enforcement.yml
    ├── 05_syslog_forwarding.yml
    └── 06_zabbix_agent_deploy.yml
```

---

## Example output

```
PLAY [Patch Management — Apply Security Updates] ******************************

TASK [Update apt cache] *******************************************************
ok: [192.168.1.10]
ok: [192.168.1.11]

TASK [Apply all security upgrades (apt)] **************************************
changed: [192.168.1.10]
ok: [192.168.1.11]

TASK [Check if reboot is required (Debian/Ubuntu)] ****************************
ok: [192.168.1.10]
ok: [192.168.1.11]

TASK [Reboot host if required] ************************************************
changed: [192.168.1.10]
skipping: [192.168.1.11]

PLAY RECAP ********************************************************************
192.168.1.10               : ok=4  changed=2  unreachable=0  failed=0
192.168.1.11               : ok=3  changed=0  unreachable=0  failed=0
```

---

## Safety notes

- Always run with `--check --diff` first on production hosts to preview changes before applying them
- `02_ssh_hardening.yml` validates `sshd_config` with `sshd -t` before restarting — it will not lock you out
- `03_user_account_audit.yml` **locks** idle accounts by default (`password_lock: true`), not deletes — set `account_lock_instead_of_delete: false` with caution
- `04_firewall_enforcement.yml` resets `ufw` rules before applying the whitelist — ensure port 22 is in `ufw_allowed_tcp_ports` before running or you will lose SSH access

---

## Author

Gautam Dinesh — [linkedin.com/in/gdinesh2001](https://linkedin.com/in/gdinesh2001)
