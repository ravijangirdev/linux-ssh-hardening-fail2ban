# Linux Server Security Hardening: SSH & Fail2Ban Intrusion Prevention

## 📌 Project Overview
Exposing Linux servers to the public internet attracts automated scanners and relentless brute-force attacks targeting standard authentication ports. Relying on default configurations leaves servers vulnerable to credential stuffing, dictionary attacks, and denial of service.

This project implements an enterprise-grade **Host-Level Access Hardening & Intrusion Prevention Architecture** on an **AWS EC2 (Ubuntu Linux)** production instance:
* **Modular SSH Hardening:** Utilized drop-in configuration overrides (`/etc/ssh/sshd_config.d/`) to enforce strict authentication controls, enforce non-root administrative sessions, and configure idle timeouts.
* **Intrusion Prevention Engine (Fail2Ban):** Deployed Fail2Ban with persistent custom overrides (`jail.local`), decoupling configurations from upstream package updates.
* **Firewall Layer Integration:** Configured Fail2Ban ban actions to pipe directly into the system's Uncomplicated Firewall (`ufw`), dropping malicious packets at the network layer.
* **Live Brute-Force & Recovery Drill:** Executed simulated failed authentication loops, confirmed automated jail triggering and log generation, and demonstrated production unban procedures for false-positive mitigation.

---

## 🏗️ Security Architecture & Incident Flow

```text
[ Malicious Actor / Botnet ]
             |
             v (Repeated SSH Auth Failures)
[ Linux SSH Daemon (sshd) ]
             |
             v (Writes event stream)
[ Systemd Journal / /var/log/auth.log ]
             |
             v (Monitors & matches regex filters)
[ Fail2Ban Daemon (sshd jail) ]
             |
     (3 failures within 10m window)
             v
[ Automated Ban Triggered ] ---> [ Write Audit to /var/log/fail2ban.log ]
             |
             v
[ UFW Firewall: Packet Drop ] ---> (Attacker Blocked for 10m)
```

---

## ⚙️ Configuration Standards

### 1. SSH Hardening (`/etc/ssh/sshd_config.d/99-hardening.conf`)
Drop-in configuration enforcing zero-trust access controls:
```text
# Disable administrative root logins
PermitRootLogin no

# Enforce strict retry caps to hinder dictionary attacks
MaxAuthTries 3

# Prune inactive connections to avoid idle hijacking
ClientAliveInterval 300
ClientAliveCountMax 2

# Reject unauthenticated blank credentials
PermitEmptyPasswords no
```

*Pre-flight syntax validation and zero-downtime reload:*
```bash
sudo sshd -t
sudo systemctl reload ssh
```

### 2. Fail2Ban Jail Profile (`/etc/fail2ban/jail.local`)
Custom override file isolating production tuning parameters:
```ini
[DEFAULT]
bantime  = 10m
findtime = 10m
maxretry = 3
banaction = ufw

[sshd]
enabled = true
port    = ssh
logpath = %(sshd_log)s
backend = systemd
```

---

## 🔍 Validation & Live Penetration Drill

To verify runtime efficacy, a controlled intrusion scenario was simulated against localhost.

### Step 1: Simulate Credential Brute-Force
```bash
for i in {1..4}; do ssh -o StrictHostKeyChecking=no -o ConnectTimeout=2 fakeuser@127.0.0.1 -p 22; done
```

### Step 2: Jail Detection & Ban Audit
Inspecting the active jail filters and runtime counters:
```bash
sudo fail2ban-client status sshd
```
*Output Evidence:*
```text
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     4
|  `- File list:        systemd
`- Actions
   |- Currently banned: 1
   |- Total banned:     1
   `- Banned IP list:   127.0.0.1
```

Inspecting system security logs:
```bash
sudo tail -n 20 /var/log/fail2ban.log
```
*Log confirmed: `[sshd] Ban 127.0.0.1` executed via UFW rules.*

### Step 3: Production Unban Drill (Mitigating False Positives)
Demonstrated standard operating procedure for client/admin recovery:
```bash
# Execute immediate unban
sudo fail2ban-client set sshd unbanip 127.0.0.1

# Confirm filter reclamation
sudo fail2ban-client status sshd
```
*Result: Banned IP list cleared; host connectivity restored.*

---

## 🚀 Key Takeaways & Sysadmin Skills Demonstrated
* **Access Control & Hardening:** Implementation of Principle of Least Privilege across remote connectivity layers.
* **Log-Based Intrusion Prevention:** Dynamic regex matching on authentication logs and proactive packet filtering.
* **Firewall Automation:** Coordinating daemon actions across `fail2ban` and `ufw` subsystems.
* **Security Operations & Incident Handling:** Diagnosing ban events, parsing security audit trails, and executing manual operational remediation.
