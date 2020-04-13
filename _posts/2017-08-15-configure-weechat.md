---
layout: post-detail
title: weechat + Tor + SASL
date: 2017-08-19
categories: weechat tor
description: Configure weechat with tor+SASL & secure password
img-url: https://i.imgur.com/vOCpUA9m.png
comments: true
---

Once weechat is install, launch it.

    $ weechat

# Add a Freenode server

Add a freenode server without SSL, we enable it later.

    /server add freenode chat.freenode.net/6667 -autoconnect

Change the nickname by default, it's used by freenode to create your account...

    /set irc.server.freenode.nicks ninja

Connect to freenode...

    /connect freenode

# Create your freenode account

You have to create an account, this is a restriction to use TOR. And yes, anonyma is take a hit...
You can create a password with `pwgen` like this: `pwgen -sy 24 1`.

    /msg NickServ REGISTER password ninja@ninja.co

Keep your email address private:

    /msg NickServ SET HIDE EMAIL ON

You will receive in your mail box, a command line to enter bellow like:

    /msg NickServ VERIFY REGISTER ninja ijgimopaoijv

Next, to enable TOR, we will using the `SASL EXTERNAL` method.

# Enable SASL EXTERNAL

Create a new certificate TLS. [ref](https://freenode.net/kb/answer/certfp)

```sh
$ mkdir ~/.weechat/certs
$ cd ~/.weechat/certs
$ openssl req -x509 -new -newkey rsa:4096 -sha256 -days 1000 -nodes -out freenode.pem -keyout freenode.pem
```

Find sha1sum fingerprint.

    $ openssl x509 -in freenode.pem -outform der | sha1sum -b | cut -d' ' -f1
      e084219c214d391a8fd75cdbb891b5b966515db7

Into weechat, we switch to ssl.

```sh
/msg nickserv cert add e084219c214d391a8fd75cdbb891b5b966515db7
/set irc.server.freenode.ssl_priorities "NORMAL:-VERS-TLS-ALL:+VERS-TLS1.0:+VERS-SSL3.0:%COMPAT"
/set irc.server.freenode.ssl_cert "%h/certs/freenode.pem"
/set irc.server.freenode.sasl_mechanism external
/set irc.server.freenode.ssl on
/set irc.server.freenode.addresses "chat.freenode.net/6697"
/reconnect freenode
```

You should be reconnect with your username.

# Tor

Finally, to use tor. (tor should run)

```sh
/set irc.server.freenode.addresses "freenodeok2gncmy.onion/7000"
/proxy add tor socks5 127.0.0.1 9050
/set irc.server.freenode.proxy "tor"
```

You have to disable `ssl_verify` who doesn't work with TOR.
    
    /set irc.server.freenode.ssl_verify off
    /reconnect freenode

# Enhance your privacy

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

And save all our works:

    /save

Reconnect to freenode as a ninja :)

    /reconnect freenode

## Troubleshooting

Please, post an issue to [github](https://github.com/szorfein/szorfein.github.io).
