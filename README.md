# Network Automation Lab: Ansible & GNS3

> Automated Cisco IOS configuration backup using Ansible on a GNS3 lab environment.  
> Dự án tự động hóa backup cấu hình Cisco IOS bằng Ansible trên môi trường giả lập GNS3.

![CI](https://github.com/hoangdonguit/network-automation-lab/actions/workflows/ci.yml/badge.svg)

---

## 📐 Network Topology / Sơ đồ Lab
```
                    Ubuntu 24.04 (Control Node)
                         172.17.0.1 (Docker bridge)
                               |
                         172.17.0.11
                             [R1]
                    10.0.0.1 /
                            /
                       [R2] 10.0.0.2
                  10.0.0.5 \
                            \
                         10.0.0.6
                             [R3]
```

| Device | Interface      | IP Address   | Role              |
|--------|----------------|--------------|-------------------|
| R1     | Fa0/0 (Cloud)  | 172.17.0.11  | Gateway to host   |
| R1     | Fa1/0          | 10.0.0.1     | Link to R2        |
| R2     | Fa0/0          | 10.0.0.5     | Link to R3        |
| R2     | Fa1/0          | 10.0.0.2     | Link to R1        |
| R3     | Fa0/0          | 10.0.0.6     | Link to R2        |

> Ansible only needs to reach **R1** (`172.17.0.11`) directly.  
> R2 and R3 are accessed via R1 as the jump host through static routing.

---

## ✨ Features / Tính năng

- **Multi-device backup** — Backs up R1, R2, R3 simultaneously in one playbook run
- **Dynamic directory** — Auto-creates `backups/<hostname>/` per device
- **Timestamped files** — Each backup file includes `YYYYMMDD_HHMMSS` so history is never overwritten
- **Credential security** — `hosts.yml` is gitignored; a safe `.example` template is provided instead
- **SSH compatibility fix** — Resolves KEX/cipher mismatch between OpenSSH 9.x (Ubuntu 24.04) and legacy Cisco IOS
- **CI pipeline** — GitHub Actions runs syntax check and lint on every push

---

## 🛠️ Tech Stack

| Layer | Tool |
|---|---|
| Control Node OS | Ubuntu 24.04 LTS |
| Network Emulator | GNS3 (Cisco c7200 IOS) |
| Automation | Ansible + ansible-pylibssh |
| Version Control | Git + GitHub Actions |

---

## 🚀 Quick Start

**Prerequisites:** GNS3 lab running, Ubuntu can ping `172.17.0.11`
```bash
# 1. Clone the repo
git clone https://github.com/hoangdonguit/network-automation-lab.git
cd network-automation-lab

# 2. Set up inventory
cp inventory/hosts.yml.example inventory/hosts.yml
# Edit hosts.yml — replace YOUR_PASSWORD_HERE with real credentials

# 3. Run backup
ansible-playbook -i inventory/hosts.yml playbooks/backup_ios.yml
```

Backup files will appear in `backups/R1/`, `backups/R2/`, `backups/R3/`.

---

## 🧩 Challenges & Solutions

### Challenge 1: SSH Algorithm Incompatibility

**Problem:** Ubuntu 24.04 ships with OpenSSH 9.x which disables legacy algorithms by default.
Cisco IOS c7200 only supports older KEX (`diffie-hellman-group1-sha1`) and ciphers
(`aes128-cbc`, `3des-cbc`) that modern OpenSSH refuses to negotiate.

**Symptom:**
```
fatal: [R1]: FAILED! => {"msg": "ssh connection failed: no matching key exchange method found"}
```

**Solution:** Added a project-scoped `ansible.cfg` with explicit SSH overrides — without
touching system-wide `~/.ssh/config`:
```ini
[defaults]
host_key_checking = False

[ssh_connection]
ssh_args = -o KexAlgorithms=+diffie-hellman-group1-sha1
           -o HostKeyAlgorithms=+ssh-rsa
           -o Ciphers=+aes128-cbc,3des-cbc
```

**Lesson:** Always scope legacy SSH fixes to the project level to avoid security regression
on other connections.

---

### Challenge 2: Reaching R2 and R3 via R1

**Problem:** Only R1 (`172.17.0.11`) is directly reachable from Ubuntu via the Docker bridge.
R2 (`10.0.0.2`) and R3 (`10.0.0.6`) are on internal GNS3 subnets with no direct route from the host.

**Solution:** Configured static routes on the Ubuntu host to forward `10.0.0.0/8` traffic
through R1, and added static routes on R1 to reach R2/R3:
```bash
# On Ubuntu host
sudo ip route add 10.0.0.0/8 via 172.17.0.11
```
```cisco
! On R1
ip route 10.0.0.4 255.255.255.252 10.0.0.2
```

---

## 🎯 Proof of Work

Ansible successfully backing up all 3 routers — including SSH cipher negotiation override:

![Proof of Work](image.png)

---

## 📁 Repository Structure
```
network-automation-lab/
├── .github/
│   └── workflows/
│       └── ci.yml              # Syntax check + lint on every push
├── backups/
│   ├── R1/                     # running-config_YYYYMMDD_HHMMSS.txt
│   ├── R2/
│   └── R3/
├── inventory/
│   ├── hosts.yml               # gitignored — never committed
│   └── hosts.yml.example       # safe template for public viewing
├── playbooks/
│   └── backup_ios.yml
├── ansible.cfg
├── .gitignore
└── README.md
```
