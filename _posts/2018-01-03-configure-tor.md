---
layout: post-detail
title: Configure tor
date: 2018-01-03
categories: tor
description: Configure tor with commandline.
img-url: https://i.imgur.com/J2l0SMcm.png
comments: true
---

# Dependencies

You have to install `tor` and `obfs4proxy`.

# Tor with bridge

To configure tor with [bridge](https://www.torproject.org/docs/bridges), you have to get bridge [here](https://bridges.torproject.org).

Choose 2 - Get Bridges
Advanced Option -> obfs4 and click on Get Bridges

After fill the little captcha, you receive a list like this:

```
obfs4 12.11.33.133:9443 65B5BCEIJFIJIE333JIJ3I3J38C20A252F2ABBB2 cert=iJMOEGJiej+jgoijeEIGJi/gjII/gjiOIJRGjmozjmogjzoiOIJMOIZJGBOZIGOZIOOIii iat-mode=0
obfs4 211.255.244.77:9443 01OIJFOJ1OI444O2I4O24IO4I22O2O2O2J3JO2I0 cert=E1/GieoigIEigjiEIJGIEOIoemieiogOMIHZHGIHihgh/oijmIJIEELEL+THIELljil/qa iat-mode=0
obfs4 33.123.14.133:33746 1OI1OIJ4IJFOIJGOI2JOGJ2OII4OGJ4O2JGO4IO2 cert=AIojoirjgIIRJGORIJGOppijg+jgOIJoirgiJOIj/ijgorjOJGRJirmgjmahabbaOAII21 iat-mode=1
```

Edit the file `/etc/tor/torrc`, you add content above with prefix each line by `Bridge`:

```
Bridge obfs4 12.11.33.133:9443 65B5BCEIJFIJIE333JIJ3I3J38C20A252F2ABBB2 cert=iJMOEGJiej+jgoijeEIGJi/gjII/gjiOIJRGjmozjmogjzoiOIJMOIZJGBOZIGOZIOOIii iat-mode=0
Bridge obfs4 211.255.244.77:9443 01OIJFOJ1OI444O2I4O24IO4I22O2O2O2J3JO2I0 cert=E1/GieoigIEigjiEIJGIEOIoemieiogOMIHZHGIHihgh/oijmIJIEELEL+THIELljil/qa iat-mode=0
Bridge obfs4 33.123.14.133:33746 1OI1OIJ4IJFOIJGOI2JOGJ2OII4OGJ4O2JGO4IO2 cert=AIojoirjgIIRJGORIJGOppijg+jgOIJoirgiJOIj/ijgorjOJGRJirmgjmahabbaOAII21 iat-mode=1
```

And add somes line to use these bridge:

```
UseBridges 1
ClientTransportPlugin obfs4 exec /usr/bin/obfs4proxy
Sandbox 0 # don't work with ClientTransportPlugin
```

Restart tor and it's finish:

    sudo systemctl restart tor

# Tor as bridge server

To create a bridge relay, private or not, edit `/etc/tor/torrc`.

```
BridgeRelay 1
ORPort auto
ExtORPort auto
ServerTransportPlugin obfs4 exec /usr/bin/obfs4proxy
ClientTransportPlugin obfs4 exec /usr/bin/obfs4proxy
ExitPolicy reject *:*
```

If you want a private bridge, add this:

    PublishServerDescriptor 0

And restart tor:

    sudo systemctl restart tor

## Monitor tor with [nyx](https://nyx.torproject.org/index.html)

Nyx need enable `ControlPort` to work. So we add the default port `9051` at `/etc/tor/torrc`.

    ControlPort 9051

`ControlPort` can be secure a bit by create a password with tor.

    $ tor --hash-password "password"
      16:8CD24F19A72985D76030682916C57D238A0393206C55D2D2B26B9813E5

Always in `torrc`:

```
HashedControlPassword 16:8CD24F19A72985D76030682916C57D238A0393206C55D2D2B26B9813E5
CookieAuthentication 1
```

Restart tor...

    $ sudo systemctl restart tor

Nyx will be able to contact tor.

    $ nyx

Enter your password and we'll done :)

### Troubleshoot

Post an issue at [github](https://github.com/szorfein/szorfein.github.io/issues)
