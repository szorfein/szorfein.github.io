---
layout: post-detail
title: Build a secure GPG key
date: 2017-08-08
categories: gpg
description: How build a secure gpg key.
img-url: https://i.imgur.com/IEhmnGkm.jpg
comments: true
---

Start build new keypair.

For this purpose, it's generaly recommanded to boot on [tails](https://tails.boum.org) or [kodachi](https://www.digi77.com/linux-kodachi/) and stay offline.

```sh

$ gpg --expert --full-generate-key

    8 - Choose RSA (set your own capabilities)
        
    S) Toggle the sign capability
    E) Toggle the encrypt capability
    A) Toggle the authentificate capability
    Q) Finished
    Keysize do you want? 4096
    Is valid for? 5y
    Is this correct? y

```

After write you name, email and make a strong password, valid your key.
Now, we will create subkey for signing (S), encrypt (E) and authentificate (A).

```sh

$ gpg --expert --edit-key <YourID>

> addkey
    (8) RSA (set your own capabilities)
    Current allowed actions: Sign Encrypt 

    (S) Toggle the encrypt capability
    (a) Toggle Authentificate capability
    'Current allowed actions: Sign'
    (q) Finished
    What keysize do you want? 4096 
    Key is valid for? 6m
    (y)

> addkay
    (8) RSA (set your own capabilities)
    (s) Toggle Sign Capability
    Current allowed actions: Encrypt  
    (q) Finished
    What keysize do you want? 4096
    Key is valid for? 6m
    (y) 

> addkey (last for authentification)
    (s) Toggle the sign capability
    (e) Toggle the encrypt capability
    (a) Toggle the authentificate capability
    Current allowed actions: Authenticate
    (q) Finished
    What keysize do you want? 4096
    Key is valid for? 6m
    (y)

> save

```

Your key has been create, we will create a revocation certificate, export key for backup on another device (CD-R or USB key). 

```sh
$ gpg -a --export-secret-keys <NEWID> > secret_key.gpg
$ gpg -a --generate-revocation <NEWID> > revocation_cert.gpg
    
    Please select the reason for the revocation:
    Your desicion? 1 (Key has been compromised)

$ gpg -a --export <NEWID > public_key.gpg
$ gpg -a --export-secret-subkeys <NEWID> > secret_subkey.gpg

```

Delete original key.

```sh
$ gpg --delete-secret-keys <NEWID>
```

Press Delete key for each subkey. Verify than output of gpg -K is empty and re-import subkey.

```sh
$ gpg -K
$ gpg --import secret_subkeys.gpg

```

Re-export key for use on laptop.

```sh 
$ gpg -a --export-secret-keys <NEWID> > laptop_key_secret.gpg
$ gpg -a --export <NEWID> > laptop_key_public.gpg

```

Move alls *.gpg files on secure device and boot on your OS.
Import key with:

```sh 
$ gpg -a --import laptop_key_pub.gpg
$ gpg -a --import laptop_keys_secret.gpg
```

Trust ultimate your keys

```sh 
$ gpg --edit-key <NEWID>
> trust
> 5 = I trust ultmately
> save
```

Look your imported key, it look like:

```sh
$ gpg -K

    sec#  rsa4096/0x0FF123FF123FF123 2017-02-25 [C] [expire : 2019-02-25]
    Fingerprint of key  = 346E BDED 037B 1949 013D  3576 0F15 D984 5548 7B76
    uid                  [  ultimate ] Blabla <blabla@proton.com>
    ssb   rsa4096/0x123F234555FFF3FA 2017-02-25 [S] [expire : 2017-08-24]
    ssb   rsa4096/0x65FFBB38E24C1BFB 2017-02-25 [E] [expire : 2017-08-24]
    ssb   rsa4096/0x45FD80474C2BB4FC 2017-02-25 [A] [expire : 2017-08-24]

```

The first line with '#' after sec means that the secret key is not usable (you cannot create revocation certificate with this key and other things).
Set a gpg.conf with fresh keypair.

```sh
$ gpg -k

    note ID of our subkey with [S] and [E], we need for configure gpg.conf
    in this case:

    ssb   rsa4096/0x123F234555FFF3FA 2017-02-25 [S] [expire : 2017-08-24]
    ssb   rsa4096/0x65FFBB38E24C1BFB 2017-02-25 [E] [expire : 2017-08-24]

$ vim ~/.gnupg/gpg.conf

    default-key 0x123F234555FFF3FA (flag [S])
    default-recipient 0x65FFBB38E24C1BFB (flag [E])
```

If you need share key with friend, generate an a file like this:

```sh
$ gpg --armor --output "key.txt" --export <NEWID>
```

### When subkeys expire.

When key expire, you import the real secret key from flash device and enhance time of 6 month again,
The procedure seem like bellow, first, you remove expirate key:

```sh
$ gpg --delete-keys <UUID>
$ gpg -K
```
gpg -K for control than our keys has been delete.
Import secret key from flash device.

```sh
$ gpg --import secret.gpg
$ gpg --edit-key UUID
    > key 1
    > key 2
    > key 3
    > expire
    > 6m
    > save
```

Now, export new fresh subkey.

```sh
$ gpg -a --export-secret-keys UUID > laptop_key_secret.gpg
$ gpg -a --export UUID > laptop_keys_public.gpg
```

Delete the real secret key again...

```sh
$ gpg --delete-keys UUID
```

And import laptop key for our pc.

```sh
$ gpg --import laptop_keys_public.gpg
$ gpg --import laptop_keys_secret.gpg
$ gpg --edit-key UUID
    > trust
    > 5 ultime
    > save
```

And we have finished, I'll add article for mutt & other tools later :)
