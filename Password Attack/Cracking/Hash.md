## Hasher un mot

```bash
echo -n "secret" | sha256sum
```

#### Compter le nombre de caractères

```bash
echo -n "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789" | wc -c
```

#### Benchmark de Hashcat

```bash
hashcat -b

METAL API (Metal 370.64.2)
==========================
* Device #01: Apple M4 Pro, skipped

OpenCL API (OpenCL 1.2 (Nov  8 2025 19:10:43)) - Platform #1 [Apple]
====================================================================
* Device #02: Apple M4 Pro, GPU, 9093/18186 MB (1704 MB allocatable), 16MCU

Benchmark relevant options:
===========================
* --backend-devices-virtmulti=1
* --backend-devices-virthost=1
* --optimized-kernel-enable

---------------------
* Hash-Mode 900 (MD4)
---------------------

Speed.#02........: 15261.6 MH/s (2.73ms) @ Accel:416 Loops:1024 Thr:256 Vec:1

-------------------
* Hash-Mode 0 (MD5)
-------------------

Speed.#02........:  8619.4 MH/s (4.84ms) @ Accel:416 Loops:1024 Thr:256 Vec:1

----------------------
* Hash-Mode 100 (SHA1)
----------------------

Speed.#02........:  3966.6 MH/s (10.54ms) @ Accel:416 Loops:1024 Thr:256 Vec:1

---------------------------
* Hash-Mode 1400 (SHA2-256)
---------------------------

Speed.#02........:  1425.3 MH/s (29.34ms) @ Accel:416 Loops:1024 Thr:256 Vec:1
``` 

#### Hash de password manager

```bash
hashcat --help | grep -i "KeePass"
13400 | KeePass 1 (AES/Twofish) and KeePass 2 (AES)         | Password Manager
```
```bash
hashcat -m 13400 keepass.hash /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/rockyou-30000.rule --force
```




