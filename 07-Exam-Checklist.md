# 07 — Exam Checklist — Jour J

## Avant de commencer

- [ ] Connexion VPN établie et stable
- [ ] Proctoring lancé et validé
- [ ] Dossier de notes ouvert (Obsidian)
- [ ] Template de rapport ouvert
- [ ] Listener nc ouvert sur port 4444
- [ ] Serveur HTTP Python prêt (`python3 -m http.server 80`)
- [ ] Outils transférables prêts (winPEAS, linPEAS, nc.exe, GodPotato...)

---

## Stratégie de démarrage (15 premières minutes)

```bash
# 1. Lancer les scans sur TOUTES les machines en parallèle
nmap -p- --min-rate 5000 <IP1> <IP2> <IP3> <DC-IP> -oA scans/fast_all

# 2. Pendant que les scans tournent → lire le brief exam
# Identifier : quelle est la plage réseau ? Quelles sont les IPs ?
# Y a-t-il un set AD ? (souvent précisé)

# 3. Scans détaillés sur les ports trouvés
nmap -sC -sV -p <ports> <IP> -oA scans/<IP>_detailed
```

---

## Ordre d'attaque recommandé

```
1. Set AD (40 pts) → Priorité absolue
   └── Foothold machine 1
   └── Pivot machine 2
   └── Domain Controller = 40 pts

2. Machine standalone 20 pts
3. Machine standalone 20 pts
4. Machine standalone 10 pts (si temps restant)
```

> 💡 Avec le set AD complet (40 pts) + 1 machine 20 pts + local.txt d'une autre = 70 pts → PASS

---

## Checklist par machine

### Énumération initiale
- [ ] Scan tous ports (`nmap -p-`)
- [ ] Scan détaillé avec scripts (`nmap -sC -sV`)
- [ ] Scan UDP si rien trouvé
- [ ] Énumération web si port 80/443/8080
- [ ] Énumération SMB si port 445
- [ ] Recherche CVE/exploit pour chaque service + version

### Foothold
- [ ] CVE identifiée → adaptée et testée
- [ ] Credentials par défaut testés
- [ ] Upload de fichier tenté (si web)
- [ ] Shell obtenu et stabilisé

### Post-exploitation / Privesc
- [ ] `whoami /all` ou `id` exécuté
- [ ] WinPEAS / LinPEAS lancé
- [ ] `sudo -l` vérifié (Linux)
- [ ] SUID vérifié (Linux)
- [ ] SeImpersonatePrivilege vérifié (Windows)
- [ ] Credentials dans les fichiers cherchés
- [ ] Privesc réussi

### Proof
- [ ] Screenshot : `whoami && hostname && ipconfig/ip a`
- [ ] Screenshot : contenu de `proof.txt`
- [ ] Chemin complet noté : `C:\Users\Administrator\Desktop\proof.txt`
- [ ] Toutes les commandes notées dans le rapport

---

## Rapport — Ce qui doit y être pour chaque machine

Pour chaque machine :
1. **IP et nom de la machine**
2. **Screenshot du scan nmap**
3. **Étapes d'exploitation** (commandes exactes + outputs + screenshots)
4. **Privesc** (commandes exactes + outputs + screenshots)
5. **Screenshot proof.txt + whoami + hostname**
6. **Scripts/exploits utilisés** (inclus en annexe texte)

> ⚠️ Sans screenshot de proof.txt avec whoami et hostname = 0 point pour la machine

---

## Règles à ne pas oublier

| Règle | Détail |
|---|---|
| Metasploit | **1 seule cible** max sur tout l'exam |
| Meterpreter | Compte comme usage Metasploit |
| SQLmap | **Interdit** |
| Nessus/OpenVAS | **Interdit** |
| IA / ChatGPT | **Interdit** |
| Demander de l'aide | **Interdit** (Discord inclus) |
| Machines hors scope | **Interdit** d'attaquer |

---

## Gestion du temps

| Heure | Objectif |
|---|---|
| H+0:15 | Scans lancés sur tout, brief lu |
| H+4:00 | Set AD foothold + machine 2 |
| H+7:00 | DC compromis → 40 pts |
| H+10:00 | Machine standalone 1 (20 pts) |
| H+14:00 | Machine standalone 2 (20 pts) |
| H+18:00 | Machine standalone 3 (10 pts) si temps |
| H+20:00 | Rapport finalisé |
| H+23:45 | Fin exam |

---

## Si tu bloques

```
1. Attends 45 min → si bloqué → passe à une autre machine
2. Reviens plus tard avec un regard neuf
3. Re-énumère depuis zéro (port oublié ? vhost ? creds testés partout ?)
4. Vérifie les ports UDP
5. Vérifie les credentials trouvés sur d'autres machines
6. Ne panique pas — la solution est dans l'énumération
```

---

## Après l'exam (24h pour le rapport)

- [ ] Rapport en PDF
- [ ] Nom du fichier : `OSCP-OS-XXXXX-Exam-Report.pdf`
- [ ] Archivé en `.7z` : `OSCP-OS-XXXXX-Exam-Report.7z`
- [ ] Taille < 200MB
- [ ] Upload sur https://upload.offsec.com

---

## Tags
#oscp #exam #checklist
