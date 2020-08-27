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
