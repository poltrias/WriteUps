# 🖥️ Writeup - Ximai 

**Platform:** HackMyVM  
**Operating System:** Linux  

# INSTALLATION

We download the `zip` containing the `.ova` of the Ximai machine, extract it, and import it into VirtualBox.

We configure the network interface of the Ximai machine and run it alongside the attacker machine.

# HOST DISCOVERY

At this point, we still don’t know which `IP` address is assigned to Ximai, so we discover it as follows:

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
 10.0.4.3        08:00:27:05:fa:12      1      60  PCS Systemtechnik GmbH      
 10.0.4.26       08:00:27:ab:56:1b      1      60  PCS Systemtechnik GmbH
 ```

 We identify with high confidence that the victim’s IP is `10.0.4.26`.

 # PORT SCANNING

Next, we perform a general scan to check which ports are open, followed by a more exhaustive scan to gather relevant service information.

```bash
nmap -n -Pn -sS -sV -p- --open --min-rate 5000 10.0.4.26
``` 

```bash
nmap -n -Pn -sCV -p22,80,3306,8000 --min-rate 5000 10.0.4.26
```
Info:
```
Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-25 17:51 CEST
Nmap scan report for 10.0.4.26
Host is up (0.00014s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey: 
|   3072 f6:a3:b6:78:c4:62:af:44:bb:1a:a0:0c:08:6b:98:f7 (RSA)
|   256 bb:e8:a2:31:d4:05:a9:c9:31:ff:62:f6:32:84:21:9d (ECDSA)
|_  256 3b:ae:34:64:4f:a5:75:b9:4a:b9:81:f9:89:76:99:eb (ED25519)
80/tcp   open  http    Apache httpd 2.4.62 ((Debian))
|_http-server-header: Apache/2.4.62 (Debian)
|_http-title: Apache2 Ubuntu Default Page: It works
3306/tcp open  mysql   MariaDB 10.3.23 or earlier (unauthorized)
8000/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-generator: WordPress 6.8.2
|_http-open-proxy: Proxy might be redirecting requests
|_http-server-header: Apache/2.4.62 (Debian)
|_http-title: NeonGrid Solutions
MAC Address: 08:00:27:AB:56:1B (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.59 seconds
```




```bash
gobuster dir -u http://10.0.4.26 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x html,zip,php,txt,bak,sh -b 403,404 -t 60
```

Info:
```
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.0.4.26
[+] Method:                  GET
[+] Threads:                 60
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
[+] Negative Status codes:   403,404
[+] User Agent:              gobuster/3.8
[+] Extensions:              html,zip,php,txt,bak,sh
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 10938]
/info.php             (Status: 200) [Size: 85802]
/reminder.php         (Status: 200) [Size: 3163]
Progress: 1183407 / 1543899 (76.65%)
```