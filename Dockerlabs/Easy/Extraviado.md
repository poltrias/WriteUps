---
icon: linux
---

**Plataforma:** Dockerlabs\
**Sistema Operativo:** Linux

> **Tags:** `Linux` `Web` `Base64` `Information Leakage` `Lateral Movement` `Weak Credentials`

## INSTALACIÓN

Descargamos el `.zip` de la máquina desde DockerLabs a nuestro entorno y seguimos los siguientes pasos.

```bash
unzip extraviado.zip
```

La máquina ya está descomprimida y solo falta montarla.

```bash
sudo bash auto_deploy.sh extraviado.tar
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
Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-12 17:57 CEST
Nmap scan report for 172.17.0.2
Host is up (0.000031s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 cc:d2:9b:60:14:16:27:b3:b9:f8:79:10:df:a1:f3:24 (ECDSA)
|_  256 37:a2:b2:b2:26:f2:07:d1:83:7a:ff:98:8d:91:77:37 (ED25519)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.73 seconds
```

Tenemos abiertos los puertos `22` y `80`.

Accedemos por `HTTP` y nos encontramos con una página por defecto de `Apache2`.

Al inspeccionar el código fuente, encontramos un comentario al final:

```
#.........................................................................................................ZGFuaWVsYQ== : Zm9jYXJvamE=
```

Parecen ser un usuario y una contraseña codificados en `Base64`, así que los decodificamos por separado.

```bash
echo "ZGFuaWVsYQ==" | base64 -d
daniela
```

```bash
echo "Zm9jYXJvamE=" | base64 -d
focaroja
```

Obtenemos credenciales para el usuario `daniela` : `focaroja`.

Intentamos acceder por `SSH` con estas credenciales.

```bash
ssh daniela@172.17.0.2
```

Funciona!

## ESCALADA DE PRIVILEGIOS

Una vez dentro, comprobamos permisos `sudo`, `SUID`, `Capabilities`.

Pero no encontramos ninguna vía de escalada.

Buscando entre los archivos del sistema, localizamos en el directorio `/home/daniela/` una carpeta oculta llamada `/.secreto`.

Dentro encontramos un archivo llamado `passdiego`, que contiene un string en `Base64`.

```
YmFsbGVuYW5lZ3Jh
```

Lo decodificamos.

```bash
echo "YmFsbGVuYW5lZ3Jh" | base64 -d
```

Info:

```
ballenanegra
```

Deducimos que probablemente se trate de la contraseña de un usuario llamado `diego`.

Comprobamos en el archivo `/etc/passwd` que este usuario existe, así que podemos pivotar hacia él.

```bash
su diego
```

Una vez como usuario `diego`, encontramos en `/home/diego/.local/share` un archivo oculto con el siguiente acertijo.

```
password de root

En un mundo de hielo, me muevo sin prisa,
con un pelaje que brilla, como la brisa.
No soy un rey, pero en cuentos soy fiel,
de un color inusual, como el cielo y el mar
tambien.
Soy amigo de los ni~nos, en historias de
ensue~no.
Quien soy, que en el frio encuentro mi due~no?
```

La respuesta es `osoazul`, por lo que la utilizamos para autenticarnos directamente como usuario `root`.

```bash
su root
```

```
root@dockerlabs:/home/diego/.local/share# whoami
root
root@dockerlabs:/home/diego/.local/share#
```

Ya somos root!
