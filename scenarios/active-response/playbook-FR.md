# Playbook IR — Active Response · Blocage automatique IP
**Référence :** IR-2026-003  
**Scénario :** Blocage automatique d'une IP attaquante via Wazuh Active Response  
**MITRE ATT&CK :** T1110.001 — Brute Force: Password Guessing  
**Sévérité :** 🔴 Critique (Level 10)  
**Auteur :** Abraham Diaco | Cybersecurity Specialist | CEH · ISO 27001 LA  

---

## Métriques réelles du lab

| Métrique | Valeur |
|----------|--------|
| IP bloquée | **192.168.120.130** (Kali Linux) |
| Déclencheur | Rule 5763 — Level 10 |
| Temps de réaction | **< 1 seconde** |
| Durée du blocage | **600 secondes (10 min)** |
| Méthode de blocage | iptables DROP |
| Intervention humaine | ❌ Aucune |

---

## Configuration Active Response

```xml
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>5763,40111</rules_id>
  <timeout>600</timeout>
</active-response>
```

| Paramètre | Valeur | Description |
|-----------|--------|-------------|
| command | firewall-drop | Commande de blocage via iptables |
| location | local | Exécution sur le manager |
| rules_id | 5763, 40111 | Règles déclenchant le blocage |
| timeout | 600 | Durée du blocage en secondes |

---

## Flux de détection et réponse

```
[Kali 192.168.120.130] ──hydra──► [userver:22]
                                        │
                             8+ échecs SSH en < 30s
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
                               < 1 SECONDE ⚡
                                        │
                          Kali bloquée automatiquement
                          sans intervention humaine ✅
```

---

## Phase 1 — Détection

### Indicateurs déclencheurs

- Rule 5763 : `sshd: brute force trying to get access` — Level 10
- Rule 40111 : `Multiple authentication failures` — Level 10
- Fréquence : 8+ tentatives SSH en moins de 30 secondes

### Vérification de la détection

```bash
# Voir les alertes brute force en temps réel
sudo tail -f /var/ossec/logs/alerts/alerts.log | grep 5763

# Vérifier dans le Dashboard
# Threat Hunting → rule.id: 5763
```

---

## Phase 2 — Réponse automatique

### Ce que Wazuh fait automatiquement

```bash
# Wazuh exécute automatiquement :
/var/ossec/active-response/bin/firewall-drop add - 192.168.120.130

# Ce qui revient à :
sudo iptables -A INPUT -s 192.168.120.130 -j DROP
```

### Vérification du blocage

```bash
# Confirmer la règle iptables active
sudo iptables -L INPUT -n | grep 192.168.120.130

# Résultat attendu :
# DROP  all  --  192.168.120.130  0.0.0.0/0

# Vérifier le log Active Response
sudo cat /var/ossec/logs/active-responses.log | grep "firewall-drop"
```

---

## Phase 3 — Analyse post-incident

```bash
# Compter les tentatives de l'IP bloquée
sudo grep "192.168.120.130" /var/log/auth.log | grep "Failed" | wc -l

# Vérifier si l'attaque a réussi avant le blocage
sudo grep "192.168.120.130" /var/log/auth.log | grep "Accepted"

# Durée de l'attaque avant blocage
sudo grep "192.168.120.130" /var/log/auth.log | grep "Failed" | awk '{print $1,$2,$3}' | head -1
sudo grep "192.168.120.130" /var/log/auth.log | grep "Failed" | awk '{print $1,$2,$3}' | tail -1
```

---

## Phase 4 — Déblocage manuel (si nécessaire)

```bash
# Débloquer une IP avant expiration du timeout
sudo iptables -D INPUT -s 192.168.120.130 -j DROP

# Ou via Wazuh Active Response
sudo /var/ossec/bin/agent_control -b 192.168.120.130 -f firewall-drop -u 000
```

---

## Phase 5 — Durcissement post-incident

```bash
# Bloquer définitivement via fail2ban
sudo apt install fail2ban -y

# Config fail2ban SSH
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

# Désactiver l'auth par mot de passe SSH
sudo sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart sshd
```

---

## Rapport d'incident

| Champ | Valeur |
|-------|--------|
| Date/Heure attaque | 2026-06-18T23:10:00 UTC |
| Date/Heure blocage | 2026-06-18T23:10:02 UTC |
| IP source bloquée | 192.168.120.130 (Kali Linux) |
| Règle déclenchée | 5763 — Level 10 |
| Méthode de blocage | iptables DROP (firewall-drop) |
| Durée du blocage | 600 secondes |
| Connexion réussie | ❌ Non |
| Intervention humaine | ❌ Aucune |
| Temps de réaction | **< 1 seconde** ⚡ |
| Statut | ✅ Bloqué automatiquement |

---

## Matrice d'escalade

| Condition | Action |
|-----------|--------|
| IP bloquée tente de contourner | Bloquer définitivement + alerter SOC |
| Multiples IPs sources | Possible DDoS — escalader RSSI |
| Connexion réussie avant blocage | Incident majeur — isolation immédiate |
| Blocage récurrent même IP | Blacklist permanente + rapport |

---

## Leçons apprises

1. L'Active Response réduit le temps de réaction à **< 1 seconde** vs intervention manuelle
2. Le timeout de 600s est un équilibre — trop court = attaquant réessaie, trop long = risque de blocage légitime
3. Combiner Active Response + fail2ban pour une protection en profondeur
4. Toujours vérifier via iptables que le blocage est effectif

---

*Playbook généré à partir d'un lab réel — Abraham Diaco · Cybersecurity Specialist | CEH · ISO 27001 LA*
