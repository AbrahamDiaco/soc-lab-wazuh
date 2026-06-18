# IR Playbook — Active Response · Automatic IP Block
**Reference:** IR-2026-003  
**Scenario:** Automatic IP blocking via Wazuh Active Response  
**MITRE ATT&CK:** T1110.001 — Brute Force: Password Guessing  
**Severity:** 🔴 Critical (Level 10)  
**Author:** Abraham Diaco | Cybersecurity Specialist | CEH · ISO 27001 LA  

---

## Real Lab Metrics

| Metric | Value |
|--------|-------|
| Blocked IP | **192.168.120.130** (Kali Linux) |
| Trigger | Rule 5763 — Level 10 |
| Response time | **< 1 second** |
| Block duration | **600 seconds (10 min)** |
| Blocking method | iptables DROP |
| Human intervention | ❌ None |

---

## Active Response Configuration

```xml
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>5763,40111</rules_id>
  <timeout>600</timeout>
</active-response>
```

| Parameter | Value | Description |
|-----------|-------|-------------|
| command | firewall-drop | Block via iptables |
| location | local | Execute on manager |
| rules_id | 5763, 40111 | Rules triggering the block |
| timeout | 600 | Block duration in seconds |

---

## Detection and Response Flow

```
[Kali 192.168.120.130] ──hydra──► [userver:22]
                                        │
                             8+ SSH failures in < 30s
                                        │
                              [Wazuh analysisd]
                                        │
                               Rule 5763 — Level 10
                          "SSH brute force detected"
                                        │
                              [wazuh-execd]
                                        │
                        active-response/bin/firewall-drop
                                        │
                    ┌───────────────────┼───────────────────┐
                    ▼                   ▼                   ▼
             check_keys          iptables DROP         LOG entry
          192.168.120.130     192.168.120.130      active-responses.log
                                        │
                                < 1 SECOND ⚡
                                        │
                          Kali blocked automatically
                          with zero human intervention ✅
```

---

## Phase 1 — Detection

### Trigger Indicators

- Rule 5763: `sshd: brute force trying to get access` — Level 10
- Rule 40111: `Multiple authentication failures` — Level 10
- Frequency: 8+ SSH attempts in less than 30 seconds

### Detection Verification

```bash
# Monitor brute force alerts in real time
sudo tail -f /var/ossec/logs/alerts/alerts.log | grep 5763

# Check in Dashboard
# Threat Hunting → rule.id: 5763
```

---

## Phase 2 — Automatic Response

### What Wazuh does automatically

```bash
# Wazuh automatically executes:
/var/ossec/active-response/bin/firewall-drop add - 192.168.120.130

# Equivalent to:
sudo iptables -A INPUT -s 192.168.120.130 -j DROP
```

### Block verification

```bash
# Confirm active iptables rule
sudo iptables -L INPUT -n | grep 192.168.120.130

# Expected output:
# DROP  all  --  192.168.120.130  0.0.0.0/0

# Check Active Response log
sudo cat /var/ossec/logs/active-responses.log | grep "firewall-drop"
```

---

## Phase 3 — Post-Incident Analysis

```bash
# Count attempts from blocked IP
sudo grep "192.168.120.130" /var/log/auth.log | grep "Failed" | wc -l

# Check if attack succeeded before block
sudo grep "192.168.120.130" /var/log/auth.log | grep "Accepted"

# Attack duration before block
sudo grep "192.168.120.130" /var/log/auth.log | grep "Failed" | awk '{print $1,$2,$3}' | head -1
sudo grep "192.168.120.130" /var/log/auth.log | grep "Failed" | awk '{print $1,$2,$3}' | tail -1
```

---

## Phase 4 — Manual Unblock (if needed)

```bash
# Unblock IP before timeout expires
sudo iptables -D INPUT -s 192.168.120.130 -j DROP

# Or via Wazuh Active Response
sudo /var/ossec/bin/agent_control -b 192.168.120.130 -f firewall-drop -u 000
```

---

## Phase 5 — Post-Incident Hardening

```bash
# Permanent protection via fail2ban
sudo apt install fail2ban -y

# fail2ban SSH config
sudo tee /etc/fail2ban/jail.local << 'EOF'
[sshd]
enabled = true
port = ssh
maxretry = 5
findtime = 300
bantime = 3600
EOF

sudo systemctl restart fail2ban
sudo fail2ban-client status sshd

# Disable password authentication
sudo sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart sshd
```

---

## Incident Report

| Field | Value |
|-------|-------|
| Attack date/time | 2026-06-18T23:10:00 UTC |
| Block date/time | 2026-06-18T23:10:02 UTC |
| Blocked source IP | 192.168.120.130 (Kali Linux) |
| Triggered rule | 5763 — Level 10 |
| Blocking method | iptables DROP (firewall-drop) |
| Block duration | 600 seconds |
| Successful login | ❌ No |
| Human intervention | ❌ None |
| Response time | **< 1 second** ⚡ |
| Status | ✅ Automatically blocked |

---

## Escalation Matrix

| Condition | Action |
|-----------|--------|
| Blocked IP attempts bypass | Permanent block + alert SOC |
| Multiple source IPs | Possible DDoS — escalate to CISO |
| Successful login before block | Major incident — immediate isolation |
| Recurring block same IP | Permanent blacklist + report |

---

## Lessons Learned

1. Active Response reduces reaction time to **< 1 second** vs manual intervention
2. 600s timeout is a balance — too short = attacker retries, too long = risk of blocking legitimate users
3. Combine Active Response + fail2ban for defense in depth
4. Always verify via iptables that the block is effective

---

*Playbook generated from a real lab environment — Abraham Diaco · Cybersecurity Specialist | CEH · ISO 27001 LA*
