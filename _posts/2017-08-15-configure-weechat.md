---
layout: post-detail
title: configure weechat
date: 2017-08-14
categories: irc tor ssl
description: how configure weechat
img-url: 
---

# basic options

```
/set weechat.plugin.autoload "*,!lua,!tcl,!ruby,!fifo,!logger,!relay"
/set weechat.startup.display_logo off 
/mouse enable
/set weechat.network.gnutls_ca_file "/etc/ssl/certs/ca-certificates.crt"
```

## set a passphrase for save confidential data

Considerate than you password is mybestpassphrase, you do something like it.

```
/secure passphrase mybestpassphrase
/secure set secureUser typeYourConfidentialUsername mybestpassphrase
/secure set securePasswd typeYourConfidentialPassword mybestpassphrase
/set irc.server.freenode.sasl_username "${sec.data.secureUser}"
/set irc.server.freenode.sasl_password "${sec.data.securePasswd}"
```

## good plugins

```
/script install buffers.pl
/script install buddylist
/iset
```

## options for paranoids user and memory.

Look on [faq](https://weechat.org/files/doc/weechat_faq.en.html#general) for detail of each options.
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
```

## set a proxy with Tor.

```
/proxy add tor socks5 127.0.0.1 9050
```

## register account to freenode (freenode block tor traffic else...)

```
/server add freenode chat.freenode.net/6697 -ssl -autoconnect
/connect freenode
/msg NickServ REGISTER password [email protected]
```

## Configure Freenode with Tor

Create ssl key:

```sh
$ openssl ecparam -genkey -name prime256v1 > ~/.weechat/ecdsa.pem
$ openssl ec -noout -text -conv_form compressed -in ~/.weechat/ecdsa.pem | \
grep '^pub:' -A 3 | tail -n 3 | tr -d ' \n:' | xxd -r -p | base64

/msg NickServ identify password (without SASL)
/msg nickserv set pubkey Av8k1FOGetUDq7sPMBfufSIZ5c2I/QYWgiwHtNXkVe

```

