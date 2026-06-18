# Playbook IR — Attaque Brute Force SSH
**Référence :** IR-2026-002  
**Scénario :** Tentatives d'authentification SSH par force brute  
**MITRE ATT&CK :** T1110.001 — Brute Force: Password Guessing  
**Sévérité :** 🔴 Critique (Level 10)  
**Auteur :** Abraham Diaco | Cybersecurity Specialist | CEH · ISO 27001 LA  

---

## Métriques réelles du lab

| Métrique | Valeur |
|----------|--------|
| Outil d'attaque | Hydra v9.5 |
| Source | Kali Linux (agent ID 002) |
| Cible | userver — 192.168.120.128 |
| Alertes générées | **1,858** |
| Temps de détection | **< 2 minutes** |
| Rule ID principal | **5763** — SSH brute force |
| Level max déclenché | **10 / 15** |

---

## Règles Wazuh déclenchées

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

## Flux de détection

```
[Kali Linux] ──hydra──► [userver:22/SSH]
                               │
                    Multiples échecs auth
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
                        ALERTE LEVEL 10
                               │
                        Dashboard Wazuh
                        (< 2 minutes)
```

---

## Phase 1 — Identification

### Indicateurs de compromission (IoC)

- Multiples échecs SSH sur un compte en quelques secondes
- Rule 5763 déclenchée : `sshd: brute force trying to get access`
- Rule 40111 déclenchée : `Multiple authentication failures`
- Level 10 sur plusieurs règles simultanées

### Commandes de vérification

```bash
# Vérifier les tentatives SSH en cours
sudo grep "Failed password" /var/log/auth.log | tail -20

# Identifier l'IP source
sudo grep "Failed password" /var/log/auth.log | awk '{print $11}' | sort | uniq -c | sort -rn

# Voir les alertes Wazuh en temps réel
sudo tail -f /var/ossec/logs/alerts/alerts.log | grep -E "5763|40111|5758"
```

### Questions clés

- [ ] L'IP source est-elle connue / légitime ?
- [ ] Y a-t-il eu une connexion réussie après les échecs ?
- [ ] Le compte ciblé existe-t-il sur le système ?
- [ ] L'attaque est-elle toujours en cours ?

---

## Phase 2 — Confinement

### Blocage immédiat de l'IP source

```bash
# Identifier l'IP attaquante
ATTACKER_IP=$(sudo grep "Failed password" /var/log/auth.log | awk '{print $11}' | sort | uniq -c | sort -rn | head -1 | awk '{print $2}')

echo "IP attaquante : $ATTACKER_IP"

# Bloquer via iptables
sudo iptables -A INPUT -s $ATTACKER_IP -j DROP

# Vérifier le blocage
sudo iptables -L INPUT -n | grep $ATTACKER_IP
```

### Bloquer via fail2ban (si installé)

```bash
sudo fail2ban-client set sshd banip $ATTACKER_IP
```

### Vérifier si une session est ouverte

```bash
# Sessions SSH actives
who
w
last | head -10

# Connexions réseau actives
ss -tnp | grep :22
```

---

## Phase 3 — Analyse

### Vérifier si l'attaque a réussi

```bash
# Chercher une connexion réussie après les échecs
sudo grep "Accepted password\|Accepted publickey" /var/log/auth.log

# Vérifier les connexions suspectes dans Wazuh
sudo grep "authentication_success" /var/ossec/logs/alerts/alerts.log
```

### Analyser l'étendue

```bash
# Compter les tentatives par compte ciblé
sudo grep "Failed password" /var/log/auth.log | awk '{print $9}' | sort | uniq -c | sort -rn

# Durée de l'attaque
sudo grep "Failed password" /var/log/auth.log | awk '{print $1,$2,$3}' | head -1
sudo grep "Failed password" /var/log/auth.log | awk '{print $1,$2,$3}' | tail -1
```

---

## Phase 4 — Éradication

```bash
# Désactiver l'authentification par mot de passe SSH (clé uniquement)
sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config

# Changer le port SSH (optionnel)
sudo sed -i 's/#Port 22/Port 2222/' /etc/ssh/sshd_config

# Redémarrer SSH
sudo systemctl restart sshd

# Installer et configurer fail2ban
sudo apt install fail2ban -y
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

---

## Phase 5 — Rétablissement

```bash
# Vérifier la configuration SSH
sudo sshd -t

# Tester la connexion SSH depuis un client autorisé
ssh -p 2222 user@192.168.120.128

# Vérifier fail2ban actif
sudo fail2ban-client status sshd
```

---

## Phase 6 — Rapport d'incident

| Champ | Valeur |
|-------|--------|
| Date/Heure début | 2026-06-18T20:53:18 UTC |
| Date/Heure détection | 2026-06-18T20:53:26 UTC |
| Agent ciblé | userver (ID: 000) |
| IP source | Kali Linux (ID: 002) |
| Outil utilisé | Hydra v9.5 |
| Compte ciblé | root |
| Connexion réussie | ❌ Non |
| Alertes générées | 1,858 |
| Rule principale | 5763 — Level 10 |
| Temps de détection | < 2 minutes |
| Statut | ✅ Contenu et éradiqué |

---

## Matrice d'escalade

| Condition | Action |
|-----------|--------|
| > 100 tentatives en 1 min | Bloquer IP immédiatement |
| Connexion réussie détectée | Escalader au SOC Manager |
| Multiples IPs sources | Possible DDoS — escalader RSSI |
| Compte admin ciblé | Notification direction immédiate |

---

## Leçons apprises

1. **fail2ban** doit être installé et configuré sur tous les serveurs exposés SSH
2. **L'authentification par mot de passe** doit être désactivée — utiliser uniquement les clés SSH
3. **Le port 22** par défaut est une cible systématique — envisager un port non standard
4. Wazuh détecte l'attaque en **< 2 minutes** avec 1,858 alertes générées

---

*Playbook généré à partir d'un lab réel — Abraham Diaco · Cybersecurity Specialist | CEH · ISO 27001 LA*
