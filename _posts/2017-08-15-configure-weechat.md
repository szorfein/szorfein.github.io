---
layout: post-detail
title: weechat + Tor + SASL
date: 2017-08-19
categories: weechat tor
description: Configure weechat with tor+SASL & secure password
img-url: https://i.imgur.com/vOCpUA9m.png
comments: true
---

Once weechat install, launch it.

    $ weechat

## Add Freenode server

Add a freenode server without SSL, we enable it later.

```
/server add freenode chat.freenode.net/6667 -autoconnect
/connect freenode
```

## Create a set of secure data

You can use `pwgen -sy 24 1` to generate password, thing than you need this password all time you start weechat (copy/paste :)).

    /secure passphrase 58W*$C#tjWA}F"DPL%&i5|&[

After, we create a password and email (gmail, protonmail or what you want).

```sh
/secure set fn_pwd J*:-V{~>'2z4fpnFGKT`u6Z
/secure set fn_mail alice@protonmail.com
```

## Register an account

You have to register an account now, this is a restriction to use TOR. And yes, anonymat takes a hit...

```
/msg NickServ REGISTER "${sec.data.fn_pwd}" "${sec.data.fn_mail}"
/msg NickServ SET PRIVATE ON
/msg NickServ IDENTIFY alice "${sec.data.fn_pwd}"
/msg NickServ GROUP
```

You will receive a verification mail. follow instruction.

    /msg NickServ identify "${sec.data.fn_pwd}"

Next, to enable TOR, we must choose between `ECDSA-NIST256P-CHALLENGE` or using `SASL EXTERNAL`.

## By using ECDSA-NIST256P-CHALLENGE

We going to create ecdsa cert, more info [here](https://www.weechat.org/files/doc/stable/weechat_user.en.html#irc_sasl_ecdsa_nist256p_challenge)

```
$ mkdir ~/.weechat/certs
$ cd ~/.weechat/certs
$ openssl ecparam -genkey -name prime256v1 > ecdsa.pem
```

Find fingerprint with command bellow, we need after.

```
$ openssl ec -noout -text -conv_form compressed -in ~/.weechat/certs/ecdsa.pem | \
grep '^pub:' -A 3 | tail -n 3 | tr -d ' \n:' | xxd -r -p | base64
```

return this for me.

    Ao15ByPN8oYy26dzARLyrFLCEbBDS40lk3e1rLJ0yBnR

In weechat, (we don't need use sasl_password here)

```
/msg nickserv identify "${sec.data.fn_pwd}"
/msg nickserv set pubkey Ao15ByPN8oYy26dzARLyrFLCEbBDS40lk3e1rLJ0yBnR
/set irc.server.freenode.sasl_mechanism ecdsa-nist256p-challenge
/set irc.server.freenode.sasl_username "alice"
/set irc.server.freenode.sasl_key "%h/certs/ecdsa.pem"
/reconnect freenode
```

## By using SASL EXTERNAL

Make an tls cert. [ref](https://freenode.net/kb/answer/certfp)

```sh
$ cd ~/.weechat/certs
$ openssl req -x509 -new -newkey rsa:4096 -sha256 -days 1000 -nodes -out freenode.pem -keyout freenode.pem
```

Find sha1sum fingerprint.

    $ openssl x509 -in freenode.pem -outform der | sha1sum -b | cut -d' ' -f1
      e084219c214d391a8fd75cdbb891b5b966515db7

Into weechat 

```sh
/msg nickserv cert add e084219c214d391a8fd75cdbb891b5b966515db7
/set irc.server.freenode.ssl_priorities "NORMAL:-VERS-TLS-ALL:+VERS-TLS1.0:+VERS-SSL3.0:%COMPAT"
/set irc.server.freenode.ssl_cert "%h/certs/freenode.pem"
/set irc.server.freenode.sasl_mechanism external

/reconnect freenode
```

## Tor

Finally, to use tor. (tor should run)

```sh
/set irc.server.freenode.addresses "freenodeok2gncmy.onion/7000"
/set irc.server.freenode.ssl on
/proxy add tor socks5 127.0.0.1 9050
/set irc.server.freenode.proxy "tor"
```

You have to disable `ssl_verify` who doesn't work with TOR.
    
    /set irc.server.freenode.ssl_verify off
    /reconnect freenode

## Enhance privacy (optionnal)

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

### Troubleshooting

Please, post an issue to [github](https://github.com/szorfein/szorfein.github.io).
