
#### DNS Enumeration
##### host

```bash
$host youtube.com
youtube.com has address <ip>
```

pour les records, utiliser l'option -t

```bash
$host -t mx megacorpone.com
megacorpone.com mail is handled by 10 fb.mail.gandi.net.
megacorpone.com mail is handled by 20 spool.mail.gandi.net.
megacorpone.com mail is handled by 50 mail.megacorpone.com.
megacorpone.com mail is handled by 60 mail2.megacorpone.com.


$host -t txt
megacorpone.com descriptive text "Try Harder"
megacorpone.com descriptive text "google-site-verification=U7B_b0HNeBtY4qYGQZNsEYXfCJ32hMNV3GtC0wWq5pA"
```

##### NsLookup

```bash
$ nslookup mail.megacorptwo.com

DNS request timed out.
    timeout was 2 seconds.
Server:  UnKnown
Address:  192.168.50.151

Name:    mail.megacorptwo.com
Address:  192.168.50.154
```


```bash
$ nslookup -type=TXT info.megacorptwo.com 192.168.50.151
Server:  UnKnown
Address:  192.168.50.151

info.megacorptwo.com    text =

        "greetings from the TXT record body"
```

#### Port énumération

##### nmap

```bash
-sV Services version
-O OS
-oG <fichier.txt> Met le résultat du scan dans le fichier spécifié
-sU udp scan
-sT tcp scan mais Force le 3 ways handshake donc scan long (SYN<->SYN+ACK<->ACK)
-sS tcp SYN scan donc scan tcp mais qui s arrete au premier SYN donc pas de 3 ways handshake = rapide
-sn ne scan pas les ports mais scan les machines présente dans le réseau
-Pn ne scan pas les machines mais scan les ports
-sC scan en utilisant les scripts nse de base 
-p- scan tout les ports
```

https://github.com/ekol-x9/nmap-cheatsheet


#### SMB Enumeration

```bash
nmap -v -p 139,445 <ip>

nmap -v -p 139,445 --script smb-os-discovery <ip>

sudo nbtscan -r 192.168.50.0/24
```

#### SMTP Enumeration

```bash
nc -nv <ip> 25

telnet <ip> 25
```

#### SNMP Enumeration

```bash
sudo nmap -sU --open -p 161 <ip> -oG open-snmp.txt
```