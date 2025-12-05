# 🖥️ Writeup - Lower5

**Platform:** Vulnyx  
**Operating System:** Linux  

# INSTALLATION

We download the `zip` containing the `.ova` of the Lower5 machine, extract it, and import it into VirtualBox.

We configure the network interface of the Lower5 machine and run it alongside the attacker machine.

# HOST DISCOVERY

At this point, we still don’t know which `IP` address is assigned to Lower5, so we discover it as follows:

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
 10.0.4.31       08:00:27:33:a7:76      1      60  PCS Systemtechnik GmbH
 ```

 We identify with high confidence that the victim’s IP is `10.0.4.31`.

 # PORT SCANNING

Next, we perform a general scan to check which ports are open, followed by a more exhaustive scan to gather relevant service information.

```bash
nmap -n -Pn -sS -sV -p- --open --min-rate 5000 10.0.4.31
``` 

```bash
nmap -n -Pn -sCV -p22,80 --min-rate 5000 10.0.4.31
```
Info:
```
Starting Nmap 7.95 ( https://nmap.org ) at 2025-12-05 16:11 CET
Nmap scan report for 10.0.4.31
Host is up (0.00022s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u5 (protocol 2.0)
| ssh-hostkey: 
|   256 a9:a8:52:f3:cd:ec:0d:5b:5f:f3:af:5b:3c:db:76:b6 (ECDSA)
|_  256 73:f5:8e:44:0c:b9:0a:e0:e7:31:0c:04:ac:7e:ff:fd (ED25519)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-title: vTeam a Corporate Multipurpose Free Bootstrap Responsive template
|_http-server-header: Apache/2.4.62 (Debian)
MAC Address: 08:00:27:33:A7:76 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.68 seconds
```