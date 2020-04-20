---
layout: post-detail
title: weechat + Tor + SASL
date: 2020-04-21
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

You will receive in your mail box, a command line to enter bellow like:

    /msg NickServ VERIFY REGISTER ninja ijgimopaoijv

Next, to enable TOR, we will using the `SASL` method.

# Enable SASL authentication 

Create the new key. [ref](https://www.weechat.org/files/doc/stable/weechat_user.en.html#irc_sasl_ecdsa_nist256p_challenge) 

```sh
$ mkdir ~/.weechat/certs
$ cd ~/.weechat/certs
$ openssl ecparam -genkey -name prime256v1 -out ~/.weechat/certs/ecdsa.pem
```

Find the fingerprint.

    $ openssl ec -noout -text -conv_form compressed -in ~/.weechat/certs/ecdsa.pem | grep '^pub:' -A 3 | tail -n 3 | tr -d ' \n:' | xxd -r -p | base64
      e084219c214d391a8fd75cdbb891b5b966515db7

Into weechat, we enable SASL.

```sh
/msg nickserv set pubkey e084219c214d391a8fd75cdbb891b5b966515db7
/set irc.server.freenode.sasl_mechanism ecdsa-nist256p-challenge
/set irc.server.freenode.sasl_username "ninja"
/set irc.server.freenode.sasl_key "%h/certs/ecdsa.pem"

/reconnect freenode
```

You should be reconnect with your username.

# Tor

Finally, to use tor. (tor should run) [ref](https://www.weechat.org/files/doc/stable/weechat_user.en.html#irc_tor_freenode)

```sh
/set irc.server.freenode.addresses "ajnvpgl6prmkb7yktvue6im5wiedlz2w32uhcwaamdiecdrfpwwgnlqd.onion"
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
