---
layout: post-detail
title: Configure Chrome
date: 2018-02-07
categories: chrome
description: How configure web browser based on chrome, work for chromium, vivaldi...
img-url: https://www.freepnglogos.com/uploads/google-chrome-png-logo/google-chrome-review-ebooks-png-logo-34.png
comments: true
---

We configure web browser based on chrome to enforce privacy and security with tor.  
Once installing, go to URL bellow:  

+ chrome -> chrome://settings/content
+ vivaldi -> vivaldi://settings/content

It will open thing like this: 
![content](/assets/imgs/vivaldi_01.jpg "chrome setting 1")

# Disable everything

Next, you block every options, like the both screenshots.
![content](/assets/imgs/vivaldi_02.jpg "chorme settings 2")

Concerning javascript, you have too allow only site than you like visit.
![content](/assets/imgs/vivaldi_03.jpg "chrome settings javascript")

For cookies, we remove them when we close the browser:
![content](/assets/imgs/vivaldi_05.jpg "chrome cookie")

Security & privacy, only enable `Do not track`:
![content](/assets/imgs/vivaldi_06.jpg "security & privacy")

# Script to start

To start your web browser, you can write a little script like bellow:

    $ vim vivaldi-sec

```
#!/bin/sh

agentsList=(
    "Opera/12.80 (Windows NT 5.1; U; en) Presto/2.10.289 Version/12.02"
    "Opera/9.80 (Macintosh; Intel Mac OS X 10.6.8; U; fr) Presto/2.9.168 Version/11.52"
    "Mozilla/5.0 (Windows; U; Windows NT 6.1; rv:2.2) Gecko/20110201"
    "Mozilla/5.0 (X11; U; Linux i686; en-US; rv:1.9a3pre) Gecko/20070330"
    "Mozilla/5.0 (Windows NT 6.1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/41.0.2228.0 Safari/537.36"
    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_10_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/41.0.2227.1 Safari/537.36"
    "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/41.0.2227.0 Safari/537.36"
)

RANDOM=$$$(date +%s)
rand=$[$RANDOM % ${#agentsList[@]}]
agent="${agentsList[$rand]}"

echo -e start vivaldi...

# -no-sandbox is require to start vivaldi with firejail --seccomp
firejail /opt/vivaldi/vivaldi -no-sandbox --proxy-server="${HTTP_PROXY}" --host-resolver-rules="MAP * ~NOTFOUND , EXCLUDE 127.0.0.1" \
        --disk-cache-dir=/tmp/cache -user-agent="${agent}" --incognito
```

What do the script? 
+ He select a random user agent in the list, you can enhance the list too, replace extension for change agent.
+ Use your local DNS with a program like dnscrypt-proxy.
+ Make the cache directory to /tmp, so you don't need to use bleachbit or another tool to remove the cache.
+ Start vivaldi with firejail, you have to install it.

For `--proxy-server` option, you have two choice: 
+ join on TOR directly by `socks5://127.0.0.1:9050` 
+ or use an http proxy like privoxy `http://127.0.0.1:8118`.

# Extensions

The tor project recommand do not install many extension, so the tor browser use noscript and https-everywhere, the tails project has add ublock-origin.  
So for chrome, you can add [https-everywhere](https://www.eff.org/https-everywhere), [ublock-origin](https://github.com/gorhill/uBlock) and [scriptsafe](https://github.com/andryou/scriptsafe).

### Troubleshooting

Post an issue to [github](https://github.com/szorfein/szorfein.github.io/issue).
