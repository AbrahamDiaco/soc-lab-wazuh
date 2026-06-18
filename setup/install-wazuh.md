# 🛠️ Wazuh Installation Guide — Home Lab Setup

**File:** `setup/install-wazuh.md`  
**Environment:** VirtualBox / VMware · Ubuntu 22.04 LTS  
**Wazuh version:** 4.x (current stable)  
**Author:** Abraham Diaco | Cybersecurity Specialist  

---

## Lab Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     HOST MACHINE                        │
│                                                         │
│  ┌──────────────────┐    ┌────────────┐  ┌───────────┐  │
│  │   VM 1           │    │   VM 2     │  │   VM 3    │  │
│  │ Wazuh Manager    │◄───│  Agent     │  │  Agent    │  │
│  │ + Indexer        │    │  Windows   │  │  Linux    │  │
│  │ + Dashboard      │◄───│  10/11     │  │  Ubuntu   │  │
│  │ Ubuntu 22.04     │    │            │  │  (target) │  │
│  │ RAM: 4 Go min    │    │ RAM: 2 Go  │  │ RAM: 1 Go │  │
│  └──────────────────┘    └────────────┘  └───────────┘  │
│         ↑                                               │
│   IP: 192.168.56.10  ← Réseau Host-Only                 │
└─────────────────────────────────────────────────────────┘
```

**Réseau recommandé :** Adaptateur 1 = NAT (accès internet) · Adaptateur 2 = Host-Only (communication inter-VMs)

---

## Prérequis

| Composant | Requis |
|-----------|--------|
| OS Manager | Ubuntu 22.04 LTS (64-bit) |
| RAM Manager | 4 Go minimum (8 Go recommandé) |
| Disque Manager | 50 Go minimum |
| Accès internet | Requis pour téléchargement |
| Droits | sudo/root |

---

## Étape 1 — Préparation de la VM Manager (Ubuntu 22.04)

### 1.1 Mise à jour du système

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl wget gnupg -y
```

### 1.2 Désactiver le swap (recommandé pour Wazuh Indexer)

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/' /etc/fstab
```

### 1.3 Vérifier le hostname

```bash
hostnamectl set-hostname wazuh-manager
echo "192.168.56.10 wazuh-manager" | sudo tee -a /etc/hosts
```

---

## Étape 2 — Installation Wazuh (All-in-One)

> ℹ️ L'installation all-in-one installe automatiquement : Wazuh Manager + Wazuh Indexer (OpenSearch) + Wazuh Dashboard

### 2.1 Télécharger le script officiel

```bash
curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh
curl -sO https://packages.wazuh.com/4.x/config.yml
```

### 2.2 Générer les certificats et lancer l'installation

```bash
sudo bash wazuh-install.sh -a
```

> ⏱️ Durée estimée : 10 à 20 minutes selon la connexion internet

### 2.3 Récupérer les credentials admin (affichés en fin d'install)

```bash
# Les credentials sont aussi sauvegardés ici :
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

**⚠️ Notez impérativement le mot de passe admin affiché — il n'est pas récupérable facilement.**

---

## Étape 3 — Accès au Dashboard

### 3.1 Vérification des services

```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
```

Tous les services doivent afficher : `Active: active (running)`

### 3.2 Connexion au Dashboard

Ouvrir depuis le navigateur de l'hôte :

```
https://192.168.56.10
```

| Champ | Valeur |
|-------|--------|
| URL | `https://<IP_VM_MANAGER>` |
| Username | `admin` |
| Password | *(affiché lors de l'install)* |

> 🔒 Accepter le certificat auto-signé (normal en lab)

---

## Étape 4 — Déploiement de l'Agent Linux (VM 3)

### 4.1 Sur le Manager — Générer la commande d'enrôlement

Depuis le Dashboard Wazuh :  
`Agents` → `Deploy new agent` → Choisir Linux/Ubuntu → Copier la commande générée

### 4.2 Sur VM 3 (Ubuntu cible)

```bash
# Coller la commande générée par le Dashboard, exemple :
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --dearmor -o /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update && sudo apt install wazuh-agent -y

# Configurer le Manager
sudo sed -i 's/MANAGER_IP/192.168.56.10/' /var/ossec/etc/ossec.conf

# Démarrer et activer l'agent
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

### 4.3 Vérifier la connexion depuis le Manager

```bash
sudo /var/ossec/bin/agent_control -l
```

L'agent doit apparaître avec le statut `Active`.

---

## Étape 5 — Déploiement de l'Agent Windows (VM 2)

### 5.1 Depuis le Dashboard

`Agents` → `Deploy new agent` → Windows → Copier la commande PowerShell générée

### 5.2 Sur VM 2 (Windows 10/11) — PowerShell en administrateur

```powershell
# Coller la commande du Dashboard, exemple :
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.x.x-1.msi -OutFile wazuh-agent.msi
msiexec /i wazuh-agent.msi /q WAZUH_MANAGER="192.168.56.10"

# Démarrer le service
NET START WazuhSvc
```

---

## Étape 6 — Vérification Globale

### 6.1 Tous les agents connectés

```bash
# Sur le Manager
sudo /var/ossec/bin/agent_control -l

# Résultat attendu :
# ID: 001, Name: ubuntu-target, IP: 192.168.56.x, Status: Active
# ID: 002, Name: windows-target, IP: 192.168.56.x, Status: Active
```

### 6.2 Test d'alerte basique (FIM)

```bash
# Sur VM 3 (agent Linux) — modifier un fichier critique surveillé
sudo touch /etc/test-wazuh-alert.tmp
```

Vérifier dans le Dashboard → `Security Events` — une alerte FIM doit apparaître dans les 30 secondes.

---

## Troubleshooting courant

| Problème | Cause probable | Solution |
|----------|---------------|----------|
| Dashboard inaccessible | Service pas démarré | `sudo systemctl restart wazuh-dashboard` |
| Agent ne se connecte pas | Pare-feu bloque port 1514 | `sudo ufw allow 1514/tcp` sur le Manager |
| Indexer lent | RAM insuffisante | Allouer 4 Go minimum à la VM Manager |
| Certificat expiré | Lab arrêté longtemps | Regénérer via `wazuh-certs-tool.sh` |

---

## Ports importants à connaître

| Port | Protocole | Usage |
|------|-----------|-------|
| 1514 | TCP/UDP | Communication Manager ↔ Agents |
| 1515 | TCP | Enrôlement des agents |
| 55000 | TCP | API REST Wazuh |
| 443 | HTTPS | Dashboard |
| 9200 | HTTPS | Wazuh Indexer (OpenSearch) |

---

## Références

- [Documentation officielle Wazuh](https://documentation.wazuh.com)
- [Wazuh GitHub](https://github.com/wazuh/wazuh)
- [MITRE ATT&CK](https://attack.mitre.org)

---

*Lab déployé à des fins éducatives — Abraham Diaco · Cybersecurity Specialist | CEH · ISO 27001 LA*
