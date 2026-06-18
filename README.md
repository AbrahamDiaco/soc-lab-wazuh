# 🛡️ SOC Lab — Threat Detection with Wazuh

**Author:** Abraham Diaco | Cybersecurity Specialist | CEH · ISO 27001 LA  
**Environment:** VirtualBox · Ubuntu 22.04 LTS · Windows 10 · Kali Linux  
**Wazuh version:** 4.14.5  
**Status:** 🟢 Active

---

## Lab Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     HOST MACHINE                        │
│                                                         │
│  ┌──────────────────┐    ┌────────────┐  ┌───────────┐  │
│  │   VM 1 — userver │    │   VM 2     │  │   VM 3    │  │
│  │ Wazuh Manager    │◄───│  Agent     │  │  Agent    │  │
│  │ + Indexer        │    │  Windows   │  │  Kali     │  │
│  │ + Dashboard      │◄───│  DESKTOP-  │  │  Linux    │  │
│  │ Ubuntu 22.04     │    │  VJ217GT   │  │           │  │
│  │ Wazuh v4.14.5    │    │            │  │           │  │
│  └──────────────────┘    └────────────┘  └───────────┘  │
│      192.168.120.128                                     │
└─────────────────────────────────────────────────────────┘
```

**Agents connectés :** 3 (userver · DESKTOP-VJ217GT · kali)  
**Réseau :** Host-Only + NAT

---

## Scénarios couverts

| # | Scénario | MITRE ATT&CK | Rule ID | Détection | Statut |
|---|----------|-------------|---------|-----------|--------|
| 1 | FIM — Fichier suspect dans /etc | T1565.001 | 554 | < 60s | ✅ Documenté |
| 2 | Brute Force SSH depuis Kali | T1110.001 | 5763 | < 2 min | ✅ Documenté |
| 3 | Active Response — Blocage IP auto | T1110.001 | 5763 | **< 1 sec** | ✅ Documenté |

---

## 📊 Métriques réelles du lab

| Scénario | Alertes | Temps de détection | Rule principale | Level | Intervention humaine |
|----------|---------|--------------------|----------------|-------|---------------------|
| FIM — Fichier suspect | 1 | < 60 secondes | 554 | 5 | — |
| Brute Force SSH | 1,858 | < 2 minutes | 5763 | 10 🔴 | — |
| Active Response | — | **< 1 seconde** ⚡ | 5763 | 10 🔴 | ❌ Aucune |

---

## Structure du repo

```
soc-lab-wazuh/
├── README.md
├── setup/
│   └── install-wazuh.md
├── scenarios/
│   ├── FIM-malware/
│   │   ├── playbook-FR.md
│   │   ├── playbook-EN.md
│   │   └── screenshots/
│   │       ├── 01-fim-alert-list.png
│   │       ├── 02-fim-alert-detail.png
│   │       └── 03-overview-dashboard.png
│   ├── bruteforce-ssh/
│   │   ├── playbook-FR.md
│   │   ├── playbook-EN.md
│   │   └── screenshots/
│   │       ├── 04-bruteforce-alert-list.png
│   │       └── 05-bruteforce-alert-detail.png
│   └── active-response/
│       ├── playbook-FR.md
│       ├── playbook-EN.md
│       └── screenshots/
│           ├── 06-active-response-log.png
│           ├── 06-active-response-iptables.png
│           └── 07-active-response-dashboard.png
└── rules/
    └── custom-rules.xml
```

---

## Scénario 1 — FIM · Fichier suspect

**Simulation :** Création de `malware-test.sh` dans `/etc`  
**Détection :** Rule 554 déclenchée en < 60 secondes  
**MITRE :** T1565.001 — Stored Data Manipulation

```
[Attaquant] → touch /etc/malware-test.sh
     ↓
[Wazuh syscheckd] → scan FIM
     ↓
[Rule 554] → "New file added to the system"
     ↓
[Dashboard] → Alerte en < 60s
```

📄 [Playbook FR](scenarios/FIM-malware/playbook-FR.md) · [Playbook EN](scenarios/FIM-malware/playbook-EN.md)

---

## Scénario 2 — Brute Force SSH

**Simulation :** Attaque Hydra v9.5 depuis Kali Linux sur userver  
**Détection :** 1,858 alertes · Rule 5763 Level 10 en < 2 minutes  
**MITRE :** T1110.001 — Brute Force: Password Guessing

```
[Kali] ──hydra──► [userver:22]
     ↓
[Wazuh PAM/sshd]
     ↓
[Rule 5763] → "SSH brute force" Level 10 🔴
     ↓
[Dashboard] → 1,858 alertes en < 2 min
```

📄 [Playbook FR](scenarios/bruteforce-ssh/playbook-FR.md) · [Playbook EN](scenarios/bruteforce-ssh/playbook-EN.md)

---

## Scénario 3 — Active Response · Blocage automatique

**Simulation :** Wazuh bloque automatiquement l'IP de Kali (192.168.120.130) via iptables  
**Réaction :** < 1 seconde · zéro intervention humaine  
**MITRE :** T1110.001 — Brute Force: Password Guessing

```
[Kali] ──hydra──► [userver:22]
     ↓
[Rule 5763 — Level 10]
     ↓
[wazuh-execd] → firewall-drop
     ↓
[iptables DROP 192.168.120.130]
     ↓
⚡ Bloqué en < 1 seconde · sans intervention humaine
```

📄 [Playbook FR](scenarios/active-response/playbook-FR.md) · [Playbook EN](scenarios/active-response/playbook-EN.md)

---

## Technologies utilisées

![Wazuh](https://img.shields.io/badge/Wazuh-4.14.5-blue)
![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04_LTS-orange)
![Kali](https://img.shields.io/badge/Kali-Linux-purple)
![Windows](https://img.shields.io/badge/Windows-10-blue)
![MITRE](https://img.shields.io/badge/MITRE-ATT%26CK-red)
![Hydra](https://img.shields.io/badge/Hydra-v9.5-darkred)
![iptables](https://img.shields.io/badge/iptables-Active_Response-green)

---

## Références

- [Documentation Wazuh](https://documentation.wazuh.com)
- [MITRE ATT&CK T1565.001](https://attack.mitre.org/techniques/T1565/001/)
- [MITRE ATT&CK T1110.001](https://attack.mitre.org/techniques/T1110/001/)
- [Guide d'installation](setup/install-wazuh.md)

---

*Projet éducatif — Abraham Diaco · Cybersecurity Specialist | CEH · ISO 27001 LA*
