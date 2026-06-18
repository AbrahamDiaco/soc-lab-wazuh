# 🛡️ SOC Lab — Threat Detection with Wazuh

**Author:** Abraham Diaco   
**Environment:** VirtualBox · Ubuntu 22.04 LTS  
**Status:** 🟢 Active  

> A hands-on home lab simulating real-world SOC scenarios using Wazuh as SIEM/XDR. Each scenario includes a bilingual (FR/EN) incident response playbook mapped to MITRE ATT&CK.

---

## 🏗️ Lab Architecture

```
┌─────────────────────────────────────────────────┐
│                  Host Machine                   │
│              VirtualBox (NAT Network)           │
└────────────┬──────────────────┬─────────────────┘
             │                  │
    ┌────────▼────────┐ ┌───────▼────────┐
    │   VM 1 — Manager│ │ VM 2 — Agent   │
    │  Ubuntu 22.04   │ │ Ubuntu 22.04   │
    │                 │ │                │
    │ Wazuh Manager   │ │ Wazuh Agent    │
    │ Wazuh Indexer   │ │ (attack target)│
    │ Wazuh Dashboard │ │                │
    │ IP: 192.168.x.10│ │ IP: 192.168.x.20│
    └─────────────────┘ └────────────────┘
```

| VM | Role | OS | RAM |
|---|---|---|---|
| VM 1 | Wazuh Manager + Dashboard | Ubuntu 22.04 LTS | 4 GB |
| VM 2 | Wazuh Agent (target) | Ubuntu 22.04 LTS | 2 GB |

---

## 📁 Repository Structure

```
soc-lab-wazuh/
├── README.md                        ← This file
├── setup/
│   └── install-wazuh.md            ← Step-by-step installation guide
├── scenarios/
│   └── FIM-malware/
│       ├── playbook-FR.md          ← IR Playbook (French)
│       ├── playbook-EN.md          ← IR Playbook (English)
│       └── screenshots/            ← Wazuh alert screenshots
└── rules/
    └── custom-rules.xml            ← Custom Wazuh detection rules
```

---

## 🎯 Scenarios Covered

| # | Scenario | MITRE ATT&CK | Playbook | Status |
|---|---|---|---|---|
| 01 | Suspicious File / Malware (FIM) | T1565.001 | [FR](scenarios/FIM-malware/playbook-FR.md) · [EN](scenarios/FIM-malware/playbook-EN.md) | ✅ Done |
| 02 | SSH Brute Force | T1110.001 | Coming soon | 🔄 Planned |
| 03 | Linux Privilege Escalation | T1548.001 | Coming soon | 🔄 Planned |
| 04 | Network Reconnaissance (Nmap) | T1046 | Coming soon | 🔄 Planned |

---

## 🔧 Tools & Technologies

| Category | Tools |
|---|---|
| SIEM / XDR | Wazuh 4.x |
| Virtualization | VirtualBox |
| OS | Ubuntu 22.04 LTS |
| Threat Intel | VirusTotal (Wazuh integration) |
| Framework | MITRE ATT&CK |
| Documentation | Markdown (FR/EN) |

---

## 📋 Playbook Structure

Each scenario follows the **NIST SP 800-61** incident response lifecycle:

```
Identification → Containment → Analysis → Eradication → Recovery → Reporting
```

Every playbook includes:
- ✅ Detection flow diagram
- ✅ Step-by-step response checklist
- ✅ Ready-to-use bash commands
- ✅ MITRE ATT&CK mapping
- ✅ Escalation matrix
- ✅ Incident report template
- ✅ Bilingual (French / English)

---

## 🚀 Getting Started

```bash
# Clone this repository
git clone https://github.com/AbrahamDiaco/soc-lab-wazuh.git

# Start with the installation guide
cat setup/install-wazuh.md

# Browse the first scenario
cat scenarios/FIM-malware/playbook-EN.md
```

See [setup/install-wazuh.md](setup/install-wazuh.md) for the full Wazuh deployment guide.

---

## 📊 Lab Metrics (Scenario 01 — FIM)

| Metric | Value |
|---|---|
| Detection time | < 60 seconds |
| Alert rule triggered | Wazuh rule 554 |
| MITRE technique | T1565.001 |
| VirusTotal engines (test file) | 0 / 72 (simulated benign) |
| False positives | 0 |

---

## 🌍 Languages

All playbooks are available in **French and English** to support international SOC environments and multilingual teams.

---

## 📬 Contact

**Abraham Diaco** — Cybersecurity Specialist | CEH · ISO/IEC 27001 LA   
💼 [LinkedIn](https://linkedin.com/in/abrahamdiaco)  
📧 tanguy.diaco@gmail.com  

---

*This lab is part of my SOC Analyst portfolio. All scenarios are performed in an isolated virtual environment for educational purposes only.*
