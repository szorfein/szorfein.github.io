---
layout: post-detail
title: Build a secure GPG key
date: 2017-08-08
categories: gpg
comments: true
---

# Start build new keypair.

for this purpose, it's generaly recommanded to boot on Tails Linux or Kodachi.

```sh

$ gpg --expert --gen-full-key

    8 - Choose RSA
    (Q) Finished
    RSA4096 (4096)
    key expire in (5y)

```

Now we will create subkey for signing (S), encrypt (E) and authentificate (A).

```sh

$ gpg --expert --edit-key <YourID> | <MailAddr>

> addkey
(8) RSA 'own capability'
(e) Toggle Encrypt capability
(a) Toggle Authentificate capability
'Current allowed actions: Sign'
(q) Finished
(4096) 
(1y) Expire Time
(y) for signing key

> addkay
(8) RSA 'own capability'
(s) Toggle Sign Capability
'Current Allowed Actions: Encrypt'
(q) Finished
(4096)
(1y)
(y) for Encrypt Key

> addkey (last for authentification)
(s) Toggle sign Capability
(e) Toggle Encrypt Capability
(a) Toggle Authentification Capability
(4096)
(1y)
(y)
> save

```

Now, it's time for export key on another device !

