# IR Playbook — SSH Brute Force Attack
**Reference:** IR-2026-002  
**Scenario:** SSH authentication brute force attempts  
**MITRE ATT&CK:** T1110.001 — Brute Force: Password Guessing  
**Severity:** 🔴 Critical (Level 10)  
**Author:** Abraham Diaco | Cybersecurity Specialist | CEH · ISO 27001 LA  

---

## Real Lab Metrics

| Metric | Value |
|--------|-------|
| Attack tool | Hydra v9.5 |
| Source | Kali Linux (agent ID 002) |
| Target | userver — 192.168.120.128 |
| Alerts generated | **1,858** |
| Detection time | **< 2 minutes** |
| Main rule ID | **5763** — SSH brute force |
| Max level triggered | **10 / 15** |

---

## Wazuh Rules Triggered

| Rule ID | Description | Level |
|---------|-------------|-------|
| 5763 | sshd: brute force trying to get access | 10 🔴 |
| 40111 | Multiple authentication failures | 10 🔴 |
| 5551 | PAM: Multiple failed logins in a small period | 10 🔴 |
| 2502 | User missed password more than one time | 10 🔴 |
| 5758 | Maximum authentication attempts exceeded | 8 🟠 |
| 5760 | sshd: authentication failed | 5 🟡 |
| 5557 | unix_chkpwd: Password check failed | 5 🟡 |
| 5503 | PAM: User login failed | 5 🟡 |

---

## Detection Flow

```
[Kali Linux] ──hydra──► [userver:22/SSH]
                               │
                    Multiple auth failures
                               │
                    [Wazuh syscheckd / PAM]
                               │
              ┌────────────────┼─────────────────┐
              ▼                ▼                  ▼
         Rule 5760        Rule 5758          Rule 5763
     auth failed (x∞)   max attempts      BRUTE FORCE
                               │
                          Rule 40111
                     Multiple auth failures
                               │
                        LEVEL 10 ALERT
                               │
                        Wazuh Dashboard
                        (< 2 minutes)
```

---

## Phase 1 — Identification

### Indicators of Compromise (IoC)

- Multiple SSH failures on a single account within seconds
- Rule 5763 triggered: `sshd: brute force trying to get access`
- Rule 40111 triggered: `Multiple authentication failures`
- Level 10 on multiple simultaneous rules

### Verification Commands

```bash
# Check ongoing SSH attempts
sudo grep "Failed password" /var/log/auth.log | tail -20

# Identify source IP
sudo grep "Failed password" /var/log/auth.log | awk '{print $11}' | sort | uniq -c | sort -rn

# Real-time Wazuh alerts
sudo tail -f /var/ossec/logs/alerts/alerts.log | grep -E "5763|40111|5758"
```

### Key Questions

- [ ] Is the source IP known / legitimate?
- [ ] Was there a successful login after the failures?
- [ ] Does the targeted account exist on the system?
- [ ] Is the attack still ongoing?

---

## Phase 2 — Containment

### Immediate IP blocking

```bash
# Identify attacking IP
ATTACKER_IP=$(sudo grep "Failed password" /var/log/auth.log | awk '{print $11}' | sort | uniq -c | sort -rn | head -1 | awk '{print $2}')

echo "Attacker IP: $ATTACKER_IP"

# Block via iptables
sudo iptables -A INPUT -s $ATTACKER_IP -j DROP

# Verify block
sudo iptables -L INPUT -n | grep $ATTACKER_IP
```

### Block via fail2ban (if installed)

```bash
sudo fail2ban-client set sshd banip $ATTACKER_IP
```

### Check for active sessions

```bash
# Active SSH sessions
who
w
last | head -10

# Active network connections
ss -tnp | grep :22
```

---

## Phase 3 — Analysis

### Check if attack succeeded

```bash
# Look for successful login after failures
sudo grep "Accepted password\|Accepted publickey" /var/log/auth.log

# Check Wazuh for successful auth events
sudo grep "authentication_success" /var/ossec/logs/alerts/alerts.log
```

### Assess scope

```bash
# Count attempts per targeted account
sudo grep "Failed password" /var/log/auth.log | awk '{print $9}' | sort | uniq -c | sort -rn

# Attack duration
sudo grep "Failed password" /var/log/auth.log | awk '{print $1,$2,$3}' | head -1
sudo grep "Failed password" /var/log/auth.log | awk '{print $1,$2,$3}' | tail -1
```

---

## Phase 4 — Eradication

```bash
# Disable password authentication (keys only)
sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config

# Change SSH port (optional)
sudo sed -i 's/#Port 22/Port 2222/' /etc/ssh/sshd_config

# Restart SSH
sudo systemctl restart sshd

# Install and configure fail2ban
sudo apt install fail2ban -y
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

---

## Phase 5 — Recovery

```bash
# Verify SSH configuration
sudo sshd -t

# Test SSH connection from authorized client
ssh -p 2222 user@192.168.120.128

# Verify fail2ban is active
sudo fail2ban-client status sshd
```

---

## Phase 6 — Incident Report

| Field | Value |
|-------|-------|
| Start date/time | 2026-06-18T20:53:18 UTC |
| Detection date/time | 2026-06-18T20:53:26 UTC |
| Targeted agent | userver (ID: 000) |
| Source IP | Kali Linux (ID: 002) |
| Tool used | Hydra v9.5 |
| Targeted account | root |
| Successful login | ❌ No |
| Alerts generated | 1,858 |
| Main rule | 5763 — Level 10 |
| Detection time | < 2 minutes |
| Status | ✅ Contained and eradicated |

---

## Escalation Matrix

| Condition | Action |
|-----------|--------|
| > 100 attempts in 1 min | Block IP immediately |
| Successful login detected | Escalate to SOC Manager |
| Multiple source IPs | Possible DDoS — escalate to CISO |
| Admin account targeted | Immediate executive notification |

---

## Lessons Learned

1. **fail2ban** must be installed and configured on all SSH-exposed servers
2. **Password authentication** should be disabled — use SSH keys only
3. **Default port 22** is a systematic target — consider non-standard port
4. Wazuh detected the attack in **< 2 minutes** with 1,858 alerts generated

---

*Playbook generated from a real lab environment — Abraham Diaco · Cybersecurity Specialist | CEH · ISO 27001 LA*
