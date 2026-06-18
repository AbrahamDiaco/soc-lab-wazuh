# 🛡️ SOC Lab — Threat Detection with Wazuh

**Author:** Abraham Diaco | Cybersecurity Specialist | CEH · ISO 27001 LA  
**Environment:** VirtualBox · Ubuntu 22.04 LTS · Windows 10  
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

| # | Scénario | MITRE ATT&CK | Rule ID | Statut |
|---|----------|-------------|---------|--------|
| 1 | FIM — Fichier suspect créé dans /etc | T1565.001 | 554 | ✅ Documenté |
| 2 | Brute Force SSH | T1110.001 | 5710 | 🔜 En cours |
| 3 | Active Response — Blocage IP | T1059 | — | 🔜 Planifié |

---

## 📊 Métriques réelles du lab

| Métrique | Valeur |
|----------|--------|
| Temps de détection FIM | **< 60 secondes** |
| Rule ID déclenché | **554** — New file added to the system |
| Alertes générées | **1** alerte ciblée, 0 faux positif |
| Agents monitorés | **3** (Linux × 2, Windows × 1) |
| Version Wazuh | **4.14.5** |

---

## Structure du repo

```
soc-lab-wazuh/
├── README.md
├── setup/
│   └── install-wazuh.md          # Guide d'installation complet
├── scenarios/
│   └── FIM-malware/
│       ├── playbook-FR.md         # Playbook IR en français
│       ├── playbook-EN.md         # Playbook IR en anglais
│       └── screenshots/
│           ├── 01-fim-alert-list.png
│           ├── 02-fim-alert-detail.png
│           └── 03-overview-dashboard.png
└── rules/
    └── custom-rules.xml           # Règles personnalisées (à venir)
```

---

## Scénario FIM — Résultats

**Simulation :** Création d'un fichier suspect (`malware-test.sh`) dans `/etc`  
**Détection :** Wazuh a déclenché la Rule 554 en moins de 60 secondes  
**Mapping MITRE :** T1565.001 — Stored Data Manipulation

```
[Attaquant] → touch /etc/malware-test.sh
     ↓
[Wazuh syscheckd] → scan FIM déclenché
     ↓
[Rule 554] → "New file added to the system"
     ↓
[Dashboard] → Alerte visible en < 60s
```

---

## Technologies utilisées

![Wazuh](https://img.shields.io/badge/Wazuh-4.14.5-blue)
![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04_LTS-orange)
![Windows](https://img.shields.io/badge/Windows-10-blue)
![Kali](https://img.shields.io/badge/Kali-Linux-purple)
![MITRE](https://img.shields.io/badge/MITRE-ATT%26CK-red)

---

## Références

- [Documentation Wazuh](https://documentation.wazuh.com)
- [MITRE ATT&CK T1565.001](https://attack.mitre.org/techniques/T1565/001/)
- [Playbook FR](scenarios/FIM-malware/playbook-FR.md)
- [Playbook EN](scenarios/FIM-malware/playbook-EN.md)

---

*Projet éducatif — Abraham Diaco · Cybersecurity Specialist | CEH · ISO 27001 LA*
