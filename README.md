# 🔍 Network Vulnerability Assessment — DNS & DHCP Server (VM-4)

![Kali Linux](https://img.shields.io/badge/Kali_Linux-557C94?style=for-the-badge&logo=kali-linux&logoColor=white)
![Nmap](https://img.shields.io/badge/Nmap-7.95-4EAA25?style=for-the-badge)
![Ubuntu](https://img.shields.io/badge/Ubuntu_24.04-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Module](https://img.shields.io/badge/Module-Communications%20%26%20Networking%20Security-blue?style=for-the-badge)

> **Module:** Communications and Networking Security (B9CY103) | Dublin Business School  
> **Client:** Technovia Solutions | **Target:** VM-4 DNS & DHCP Server (192.168.233.128)  
> A professional penetration test and vulnerability assessment identifying 20 vulnerabilities across a Linux server — including 5 Critical severity findings.

---

## 📌 Overview

This project is a full internal vulnerability assessment of a Linux-based server (Ubuntu 24.04 LTS) designated as a DNS & DHCP server for a simulated enterprise client, Technovia Solutions.

The assessment combined **external Nmap scanning from a Kali Linux attacker machine** with **authorised SSH access** for configuration analysis — mirroring real-world grey-box penetration testing methodology.

**Key outcome:** The server designated for DNS and DHCP was found to be completely failing its primary function (both services were not running), while simultaneously exposing over 40 unnecessary open ports — creating a massive and unnecessary attack surface.

---

## 🎯 Scope

| Item | Detail |
|------|--------|
| Target | VM-4 — Ubuntu 24.04 LTS |
| IP Address | 192.168.233.128 |
| Role | DNS & DHCP Server |
| Client | Technovia Solutions |
| Assessment Type | Grey-box (external scan + authorised system access) |
| Attacker Machine | Kali Linux |
| Date | 22 April 2025 |

---

## 🔬 Methodology

A five-phase approach was used, progressing from broad discovery to deep configuration analysis:

```
Phase 1 — Discovery
└── nmap -sn 192.168.233.0/24
    Found 5 live hosts including VM-4 at 192.168.233.128

Phase 2 — Port Scanning
└── nmap -sS -p- 192.168.233.128
    Revealed 40+ open TCP ports (65507 closed)

Phase 3 — Service & OS Detection
└── nmap -sV 192.168.233.128   → OpenSSH 9.6p1, Samba smbd 4
└── nmap -O  192.168.233.128   → Linux 4.x/5.x kernel confirmed

Phase 4 — Vulnerability Analysis
└── nmap -A                    → Full script automation scan
└── nmap --script vuln         → Known CVE checks (SMB, IRC)
└── nmap --script dns          → DNS zone transfer, blacklist tests
└── nmap --script smb-enum-shares,smb-enum-users
└── nmap --script broadcast-dhcp-discover

Phase 5 — Configuration Review (authorised SSH access)
└── ssh arturo@192.168.233.128
└── systemctl status bind9, isc-dhcp-serve
└── ls -la /etc/bind/ and /etc/dhcp/
└── cat /etc/bind/named.conf.local
└── systemctl list-units --type=service --state=running
└── dpkg -l | grep -E 'bind|dhcp|dns'
```

---

## 🚨 Findings Summary

| Severity | Count |
|----------|-------|
| 🔴 Critical | 5 |
| 🟠 High | 9 |
| 🟡 Medium | 4 |
| 🟢 Low | 2 |
| **Total** | **20** |

---

## 🔴 Critical Findings

### VULN-001 — DNS Service Not Running
**Port:** 53 | **Severity:** Critical

The server's primary function — DNS — was completely non-operational despite BIND9 being installed.

**Evidence:**
```bash
arturo@vm:~$ systemctl status bind9
Unit bind9.service could not be found.
```
The `/etc/bind/` directory contained all configuration files saved as `.dpkg-new` (unactivated package defaults), meaning they were never properly applied.

**Impact:** Complete failure of DNS functionality — clients cannot resolve names across the network.

---

### VULN-002 — DHCP Service Not Running
**Port:** 67/68 | **Severity:** Critical

The DHCP service was similarly non-operational despite the package being installed and a configuration file existing at `/etc/dhcp/dhcpd.conf`.

**Evidence:**
```bash
arturo@vm:~$ systemctl status isc-dhcp-serve
Unit isc-dhcp-serve.service could not be found.
```

**Impact:** Network clients cannot obtain IP addresses automatically, causing connectivity failures across the organisation.

---

### VULN-004 — Excessive Open Ports (40+ services)
**Severity:** Critical

A full port scan revealed over 40 open TCP ports on a server that should only expose ports 53 (DNS) and 67/68 (DHCP).

**Evidence (Nmap output):**
```
22/tcp    open  ssh
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
2222/tcp  open  EtherNet/IP-1
3333/tcp  open  dec-notes
4444/tcp  open  krb524
4701/tcp  open  netxms-mgmt
4702/tcp  open  netxms-sync
5555/tcp  open  freeciv
6666/tcp  open  irc
... and 30+ more
```

**Impact:** Massively expanded attack surface with numerous entry points for attackers.

---

### VULN-011 — Industrial Protocol Exposed (EtherNet/IP)
**Port:** 2222 | **Severity:** Critical

An EtherNet/IP industrial control protocol was exposed on port 2222 — a protocol designed for Operational Technology (OT) environments, with no business justification on a DNS/DHCP server.

**Evidence:** `nmap -sS` confirmed `2222/tcp open EtherNet/IP-1`

**Impact:** Industrial protocol exposure could allow attackers to interact with or pivot to control systems.

---

### VULN-018 — Role-Based Architecture Violation
**Severity:** Critical

The server was running a combination of DNS, DHCP, database (PostgreSQL, Redis), IRC, Samba, Kerberos, gaming (FreeCiv), and network monitoring (NetXMS) services simultaneously — a fundamental violation of server role isolation principles.

**Evidence:** `systemctl list-units --type=service --state=running` confirmed 19 active services including:
- `postgresql@16-main.service`
- `redis-server.service`
- `nmbd.service` / `smbd.service` (Samba)
- `xinetd.service`

**Impact:** Complete violation of least privilege and security best practices — cross-contamination between security domains.

---

## 🟠 High Severity Findings

| ID | Service | Finding |
|----|---------|---------|
| VULN-003 | DNS (53) | Zone transfer config allows `any` — exposes full internal network topology |
| VULN-006 | SMB (139/445) | Message signing enabled but not required — MITM risk |
| VULN-007 | IRC (6666) | IRC service running on a DNS/DHCP server — unauthorized communication channel |
| VULN-008 | Config | Multiple `.dpkg-new` files — configuration changes never properly applied |
| VULN-010 | Kerberos (4444) | KRB524 running without proper configuration |
| VULN-012 | NetXMS (4701-4705) | Network monitoring services exposed without access controls |
| VULN-014 | Auth | Xinetd running — legacy insecure service wrapper, brute force risk |
| VULN-017 | Files | Insecure permissions on `/etc/bind/` config files |
| VULN-019 | DHCP | DHCP config exists but improperly scoped — no security controls |
| VULN-020 | Network | Critical and non-critical services mixed — cross-contamination risk |

---

## 🟡 Medium Severity Findings

| ID | Service | Finding |
|----|---------|---------|
| VULN-005 | SSH (22) | OpenSSH 9.6p1 version banner exposed — targeted exploit risk |
| VULN-009 | DB Services | PostgreSQL and Redis running unnecessarily |
| VULN-013 | Port 5555 | FreeCiv gaming service on a production server |
| VULN-015 | System | System minimized without security hardening applied |
| VULN-016 | DNS | DNS blacklist configuration exists but not implemented |

---

## 🛡️ Mitigation Plan

### Fix the Primary Services (VULN-001, VULN-002)
```bash
# Activate DNS configuration files
mv /etc/bind/named.conf.options.dpkg-new /etc/bind/named.conf.options
mv /etc/bind/named.conf.local.dpkg-new /etc/bind/named.conf.local

# Enable and start services
systemctl enable --now bind9
systemctl enable --now isc-dhcp-server
```

### Restrict DNS Zone Transfers (VULN-003)
```bash
# In /etc/bind/named.conf.local — replace:
# allow-transfer { any; };
# With:
allow-transfer { 192.168.1.10; };  # Only authorised secondary DNS server
```

### Lock Down the Firewall (VULN-004)
```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow 53          # DNS
ufw allow 67/udp      # DHCP
ufw allow 68/udp      # DHCP client
ufw allow 22          # SSH (restrict to management subnet if possible)
ufw enable
```

### Remove Unnecessary Services (VULN-007, VULN-009, VULN-013)
```bash
systemctl disable --now irc
systemctl disable --now redis-server
systemctl disable --now postgresql
apt remove freeciv-server xinetd
```

### Fix SMB Signing (VULN-006)
```bash
# In /etc/samba/smb.conf under [global]:
server signing = mandatory
```

### Fix File Permissions (VULN-017)
```bash
chown root:bind /etc/bind/*
chmod 640 /etc/bind/named.conf*
chmod 644 /etc/bind/db.*
```

### Enable SSH Hardening (VULN-005)
```bash
# In /etc/ssh/sshd_config:
DebianBanner no
PasswordAuthentication no
PermitRootLogin no
```

### Block Industrial Protocol Port (VULN-011)
```bash
ufw deny 2222
systemctl disable EtherNet/IP-related-service
```

---

## 📋 Future Recommendations

- Implement role-based server architecture — one server, one purpose
- Use infrastructure-as-code (Ansible/Terraform) for consistent, auditable configs
- Establish regular vulnerability scanning schedule
- Implement DNSSEC for DNS integrity protection
- Apply CIS Benchmark hardening for Ubuntu server
- Set up fail2ban and auditd for intrusion detection and logging
- Provide security awareness training for technical staff on server hardening

---

## 🔧 Tools Used

| Tool | Version | Purpose |
|------|---------|---------|
| Nmap | 7.95 | Port scanning, service detection, OS fingerprinting, vuln scripts |
| Kali Linux | 2024.x | Attacker machine |
| VMware Workstation | — | Virtualised test environment |
| Bash / systemctl | — | Server-side configuration analysis via SSH |

---

## 📚 References

- Nmap (2025) — Nmap Reference Guide. https://nmap.org/book/man.html
- ISC (2025) — DHCP Server Documentation. https://www.isc.org/dhcp/
- OWASP (2024) — Web Security Testing Guide. https://owasp.org/www-project-web-security-testing-guide/
- NIST (2024) — SP 800-123: Guide to General Server Security. https://csrc.nist.gov/publications/detail/sp/800-123/final

---

## 👤 Author

**Anshio Renin Micheal Antony Xavier Soosammal**  
MSc Cybersecurity | Dublin Business School  
Student No: 20036753  
🔗 [LinkedIn](https://linkedin.com/in/anshio-renin-ms) | CC ISC2 Certified | Open to Work in Ireland
