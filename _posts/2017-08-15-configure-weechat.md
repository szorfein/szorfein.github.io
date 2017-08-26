---
layout: post-detail
title: weechat + Tor + SASL
date: 2017-08-19
categories: weechat tor
description: Configure weechat with tor+SASL & secure password
img-url: https://i.imgur.com/XZOKWVZm.png
comments: true
---

### First create a set of secure data with weechat !
---
you can use `pwgen -sy 24 1` for generate password, thing than u need this password all time you start weechat (copy/paste :)).
Use you real email, it will be use if you forget password.

```sh
/secure passphrase 58W*$C#tjWA}F"DPL%&i5|&[
/secure set freenodePass J*:-V{~>'2z4fpnFGKT`u6Z
/secure set freenodeUser newUsername
/secure set freenodeMail newUserMail@example.com
```

### Create freenode server for weechat and create an account. 
---
Into weechat.
```
/server add freenode chat.freenode.net/6697 -ssl -autoconnect
/connect freenode
/msg NickServ REGISTER "${sec.data.freenodePass}" "${sec.data.freenodeMail}"
/msg NickServ IDENTIFY "${sec.data.freenodeUser}" "${sec.data.freenodePass}"
/msg NickServ GROUP
```

### SASL auth with [ecdsatool](https://github.com/kaniini/ecdsatool).
---
For SASL authentification, we will create ecdsa cert, more info [here](https://github.com/weechat/weechat/issues/251)

```
$ openssl ecparam -genkey -name prime256v1 >~/.weechat/ecdsa.pem
```

Find fingerprint with command bellow, we need after.
```
$ openssl ec -noout -text -conv_form compressed -in ~/.weechat/ecdsa.pem | \
grep '^pub:' -A 3 | tail -n 3 | tr -d ' \n:' | xxd -r -p | base64
```

In weechat, (we don't need define variable sasl_password here)
```
/msg nickserv set pubkey Av8k1FOGetUDq7sPMBfufSIZ5c2I/QYWgiwHtNXkVe
/set irc.server.freenode.sasl_mechanism ecdsa-nist256p-challenge
/set irc.server.freenode.sasl_username "${sec.data.freenodeUser}"
/set irc.server.freenode.sasl_key "%h/ecdsa.pem"
/reconnect freenode
```

### Connect to freenode over TLS.
---

Make an ssl cert [ref](https://freenode.net/kb/answer/chat)
```sh
$ mkdir ~/.weechat/ssl
$ cd ~/.weechat/ssl
$ openssl req -x509 -sha256 -new -newkey rsa:4096 -days 1000 -nodes -out freenode.pem -keyout freenode.pem
```

Into weechat 
```sh
/set irc.server.freenode.ssl_cert "%h/ssl/freenode.pem"
```
find fingerprint with:
```sh
$ openssl x509 -sha1 -noout -fingerprint -in .weechat/ssl/freenode.pem | sed -e 's/^.*=//;s/://g;y/ABCDEF/abcdef/'
```

Into weechat.
```
/msg nickserv cert add e084219c214d391a8fd75cdbb891b5b966515db7
/set irc.server.freenode.ssl_priorities "NORMAL:-VERS-TLS-ALL:+VERS-TLS1.0:+VERS-SSL3.0:%COMPAT"
/reconnect freenode
```

#### Tor
---
Finally, for use tor (i considerate than tor is alrealy run & work).
```sh
/set irc.server.freenode.addresses "freenodeok2gncmy.onion/7000"
/set irc.server.freenode.ssl on
/proxy add tor socks5 127.0.0.1 9050
/set irc.server.freenode.proxy "tor"
/reconnect freenode
```

#### for enhance privacy (optionnal)
---

Add somes settings bellow to weechat. detail from [faq](https://weechat.org/files/doc/weechat_faq.en.html#security)

```
/set irc.server_default.msg_part ""
/set irc.server_default.msg_quit ""
/set irc.ctcp.clientinfo ""
/set irc.ctcp.finger ""
/set irc.ctcp.source ""
/set irc.ctcp.time ""
/set irc.ctcp.userinfo ""
/set irc.ctcp.version ""
/set irc.ctcp.ping ""
/plugin unload xfer
/set weechat.plugin.autoload "*,!xfer"
```

### For any trouble, plz, post an issue to [github](https://github.com/szorfein/szorfein.github.io)
