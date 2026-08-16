# 01 — Méthodologie Réseau

## 0. Scans initiaux — Lancer en parallèle immédiatement

```bash
# Tous les ports rapidement
nmap -p- --min-rate 5000 -oA scans/fast_all <scope>

# Détaillé sur les ports ouverts
nmap -sC -sV -p <ports> -oA scans/detailed <IP>

# Détection DC (AD)
nmap -p 88,389,445,636,3268 <scope>

# UDP si rien trouvé
nmap -sU --top-ports 20 <IP>
```

---

## 1. Identifier l'environnement

| Port ouvert | Signification |
|---|---|
| 88 (Kerberos) | Domain Controller |
| 389/636 (LDAP) | Active Directory |
| 445 (SMB) | Windows / partages |
| 3268 (Global Catalog) | AD |
| 3389 (RDP) | Windows remote |
| 5985/5986 (WinRM) | Windows remote management |
| 80/443/8080 | Web → voir [[05-Web]] |

---

## 2. Énumération par service

### SMB
```bash
nxc smb <IP> -u '' -p ''                          # anonymous
nxc smb <IP> -u 'guest' -p ''
nxc smb <IP> -u <user> -p <pass> --shares
nxc smb <IP> -u <user> -p <pass> --users
nxc smb <IP> -u <user> -p <pass> --rid-brute
smbclient -L \\\\<IP>\\ -N
smbclient \\\\<IP>\\<share> -N
```

### FTP
```bash
nxc ftp <IP>                                       # test anonymous
ftp <IP>                                           # user: anonymous
```

### SSH
```bash
nxc ssh <IP> -u <user> -p <pass>
ssh -i id_rsa <user>@<IP>
```

### WinRM
```bash
nxc winrm <IP> -u <user> -p <pass>
evil-winrm -i <IP> -u <user> -p <pass>
evil-winrm -i <IP> -u <user> -H <hash>
```

### RDP
```bash
nxc rdp <IP> -u <user> -p <pass>
xfreerdp /u:<user> /p:<pass> /v:<IP>
```

### SNMP
```bash
snmpwalk -c public -v1 <IP>
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt <IP>
```

---

## 3. Recherche d'exploit

```bash
searchsploit <service> <version>
searchsploit -m <EDB-ID>                           # copie l'exploit localement

# Toujours vérifier aussi
# Google : "<service> <version> exploit github"
# https://www.exploit-db.com
```

---

## 4. Credentials trouvés → Tester partout

```bash
nxc smb <scope> -u <user> -p <pass> --continue-on-success
nxc ssh <scope> -u <user> -p <pass> --continue-on-success
nxc winrm <scope> -u <user> -p <pass> --continue-on-success
nxc ftp <scope> -u <user> -p <pass> --continue-on-success
```

> ⚠️ **Règle d'or** : credentials trouvés = les tester sur TOUTES les machines du scope

---

## 5. Règle si tu bloques

```
Bloqué depuis 45 min ?
→ Reprends l'énumération depuis zéro
→ Port oublié ? vhost ? paramètre web non testé ?
→ Les creds trouvés → testés partout ?
→ UDP scanné ?
La réponse est TOUJOURS dans l'énumération
```

---

## Tags
#oscp #methodologie #network #enumeration
