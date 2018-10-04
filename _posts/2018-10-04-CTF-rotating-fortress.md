---
layout: post-detail
title: CTF rotating fortress
date: 2018-10-04
categories: CTF
description: CTF rotation fortress, iso -> https://www.vulnhub.com/entry/rotating-fortress-101,248/
img-url: ''
comments: true
---

I don't have finish this vm for now, but i progress... slowly :)

# Iso

https://www.vulnhub.com/entry/rotating-fortress-101,248/

nmap -sS 192.168.2.0/24

```txt
Nmap scan report for 192.168.2.53
Host is up (0.00082s latency).
Not shown: 999 closed ports
PORT   STATE SERVICE
80/tcp open  http
MAC Address: 08:00:27:A7:B0:CB (Oracle VirtualBox virtual NIC)
```

nmap -sT -sV -A -p- 192.168.2.53 

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

Let's look the web server.

redirect to /Janus.php

```txt
You're not the Admin!
```

Ok. A look with burp-suite into Target\>Site map, Response\> Raw

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

On a web browser, open the web debugg (F12), go into Application\>Cookies\>http://192.168.2.53 and change the value isAdmin=1

Reload the page:

```txt
Welcome Back Admin Last edited file was: /LELv3FfpLrbX1S4Q2FHA1hRtIoQa38xF8dzc8O9z/home.html Flag: 1{7daLI]} ggez
```

go into http://192.168.2.53//LELv3FfpLrbX1S4Q2FHA1hRtIoQa38xF8dzc8O9z/home.html

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

We have an indice for the page `/news.html`. The caesar cipher, i'll write the script in ruby.

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
zeushereafteralongtimeofthinkingihavedecidedtoretirefromprojectrotatingfortresshoweveridonotwanttokilltheprojectwithmyretirementsoiampresentingyouallachallengeihavesetupapuzzleontheserverifyoucangetpastallpuzzlestheserverisyoursbythewayihaveremovedeverybodiesloginsfromtheserverexpectminesothiswontbeeasytakethisitmightbeusefule dvqyhwmfvqrducqjbzumysrwdgmfdht goodluck
```

An hidden key `dvqyhwmfvqrducqjbzumysrwdgmfdht`.

Second:

```txt
wewillberestrictingaccesstotheserveruntilfurthernoticeforaneventanthena
```

Last message:
```txt
Decoded -4 content : zeushasaskedmetomakeanencoderforourupdatesthisismetestingitoutifitworksiwillbesendingittotherestofyouaswellasadecodertir
```

The site has a cookie `wheel_code`, past the key into value. Into `/resources`, you will find a new video `wheel.mp4`

    $ wget http://192.168.2.53/LELv3FfpLrbX1S4Q2FHA1hRtIoQa38xF8dzc8O9z/resources/wheel.mp4
    $ wget http://192.168.2.53/LELv3FfpLrbX1S4Q2FHA1hRtIoQa38xF8dzc8O9z/resources/Harpocrates.gif

You remenber the code from `/home`, we can decrypt it with our script.

     ./dec 667176666162
     Decoded 7 content : INSIDE
     Decoded 39 content : inside

`inside`..? Here it become crazy.

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

Seriously, too viscious, we have a new code to decrypt, a link.

```txt
Decoded -6 content : pfychgdpvmxpupdkmcvctggquyfmgvbt
```

http://192.168.2.53/pfychgdpvmxpupdkmcvctggquyfmgvbt/

