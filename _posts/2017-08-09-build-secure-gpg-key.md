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

For this purpose, i recommanded you to boot on [tails](https://tails.boum.org) and stay offline, tails linux contain many good thing like [onion share](https://onionshare.org/), a persistant encrypted volume, and many other things.

## Gen new keys (with only C|capability flag)

    $ gpg --expert --full-generate-key

You can choose to create a RSA, ECC keys, procedure is the same, for this wiki, i create an ECC key.

```txt
Please select what kind of key you want:
  (11) ECC (set your own capabilities)
Your selection? 11
```

Next we remove Sign and Authenticate capability, keep only Certify.

```txt
 (S) Toggle the sign capability

Your selection? S
Current allowed actions: Certify

 (Q) Finished

Your selection? Q
```
Next we select a strong cipher, Curve 25519 or RSA 4096.

```txt
Please select which elliptic curve you want:
  (1) Curve 25519
Your selection? 1
Please specify how long the key should be valid.
Key is valid for? (0) 5y
Is this correct? (y/N) y
```

And choose bellow an email address active with your name.

```txt
GnuPG needs to construct a user ID to identify your key.
Real name: Alice
Email address: alice@protonmail.com
Comment:
You selected this USER-ID:
    "Alice <alice@protonmail.com>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? O
You need a Passphrase to protect your secret key
    Enter passphrase: <Alice's long passphrase>
    Repeat passphrase: <Alice's long passphrase>
```
If need help to create a secure password, try the [diceware](https://theintercept.com/2015/03/26/passphrases-can-memorize-attackers-cant-guess/) method.

## Change ciphers

Use a strong cipher preferences:

    $ gpg --edit-key alice

```txt
gpg> setpref SHA512 SHA384 SHA256 SHA224 AES256 AES192 AES CAST5 ZLIB BZIP2 ZIP
Set preference list to:
  Cipher: AES256, AES192, AES, CAST5, 3DES
  Digest: SHA512, SHA384, SHA256, SHA224, SHA1
  Compression: ZLIB, BZIP2, ZIP, Uncompressed
  Features: MDC, Keyserver no-modify
Really update the preferences? (y/N) y

You need a passphrase to unlock the secret key for
  user: "Alice <alice@domain.com>"

  Enter passphrase: <Alice long passphrase>
gpg> save
```

## Add subkeys (S|E|A)

Now, we will create subkey for signing (S), encrypt (E) and authentificate (A).

    $ gpg --expert --edit-key alice

```
gpg> addkey
  (11) ECC (set your own capabilities)
Your selection? 11
Current allowed actions: Sign Encrypt 

  (S) Toggle the encrypt capability
  (A) Toggle Authentificate capability

Current allowed actions: Sign
Your selection? Q
Please select which elliptic curve you want:
  (1) Curve 25519
Your selection? 1
Key is valid for? 6m
Is this correct? y
Really create? y

> addkey
  (12) ECC (encrypt only)
Your selection? 12
Please select which elliptic curve you want:
  (1) Curve 25519
Your selection? 1
Key is valid for? 6m
Is this correct? y
Really create? y 

> addkey (last for authentification)
  (11) ECC (set your own capabilities)
Your selection? 11
Current allowed actions: Sign
  
  (S) Toggle the sign authenticate capability

Your selection? S
Current allowed actions: 

  (A) Toggle the authenticate capability

Your selection? A
Current allowed actions:  Authenticate

   (Q) Finished

Your selection? Q
Please select which elliptic curve you want:
  (1) Curve 25519
Your selection? 1
Key is valid for? 6m
Is this correct? y
Really create? y
```

In resume, you have one key for each action:

```txt
sec ed25519/0x<new key id>
    created: 2018-12-13 expire: 2023-12-12 usage: C
    trust: ultimate validity: ultimate
ssb ed25519/0x<new key id>
    created: 2018-12-13 expires: 2019-06-11 usage: S
    trust: ultimate validity: ultimate
ssb cv25519/0x<new key id>
    created: 2018-12-13 expires: 2019-06-11 usage: E
    trust: ultimate validity: ultimate
ssb ed25519/0x<new key id>
    created: 2018-12-13 expires: 2019-06-11 usage: A
    trust: ultimate validity: ultimate
[ultimate] (1). alice <alice@protonmail.com>
```

Save our work.

```txt
gpg> save
```

## Revocation cert

Your key has been create, we will create a revocation certificate in case where key is compromised.

    $ gpg --generate-revocation alice > revocation.cert

```
Create a revocation certificate for this key? (y/N) y
Please select the reason for the revocation:
  0 = No reason specified
Your decision? 0
Enter an optional description; end it with an empty line:
>
Reason for revocation: No reason specified
(No description given)
Is this okay? (y/N) y
ASCII armored output forced.
Revocation certificate created.
```
## Backups into Tar archive

Backup your fresh key.

    $ gpg --armor --export-secret-keys alice > alice-secret.key
    $ gpg --armor --export alice > alice-public.key

Make a tar file

    $ tar -cf alice-master-keys.tar alice*.key revocation.cert

Delete useless file.

    $ shred -u alice*.key revocation.cert

Export all subkeys too temporary, we reimport soon.

    $ gpg --export-secret-subkeys alice > subkeys

Now, we have all necessary files, delete original key from actual system.

    $ gpg --delete-secret-keys alice
    $ gpg --delete-keys alice

Press *Delete key* for each subkeys. 
Control than output of `gpg -K` and `gpg -k` do not containt our keys.

    $ gpg -K
    $ gpg -k

Now, we will create a lesser keys with less privilege by reimport our `subkeys` file.

    $ gpg --import subkeys

```txt
gpg: key <key id>: public key "alice <alice@protonmail.com" imported
gpg: key <key id>: secret key imported
gpg: Total number processed: 1
gpg: new signatures: 1
gpg: secret keys read: 1
gpg: secret keys imported: 1
```

Clean the `subkeys` file.

    $ shred -u subkeys

Verify than the master signing key is missing (contain `sec#`):

    $ gpg -K

```
sec#  ed25519/0x<key id> 2018-12-13 [C] [expire : 2023-12-12]
Fingerprint of key  = 346E BDED 037B 1949 013D  3576 0F15 D984 5548 7B76
uid                  [ unknown ] Alice <alice@protonmail.com>
ssb   ed25519/0x<key id> 2018-12-13 [S] [expire : 2019-06-11]
ssb   cv25519/0x<key id> 2018-12-13 [E] [expire : 2019-06-11]
ssb   ed25519/0x<key id> 2018-12-13 [A] [expire : 2019-06-11]
```

The first line with 'sec#' means that the secret key is not usable (you cannot create revocation certificate, change password with this key and other things).

Export this lesser key which will be used on all other devices.

    $ gpg --armor --export-secret-keys alice > alice-secret-lesser.key
    $ gpg --armor --export alice > alice-public-lesser.key

You can make an archive too.

    $ tar -cf alice_lesser_keys.tar alice*.key

## Import lesser keys

If you are on tails linux, you can send this archive with [onion share](https://onionshare.org/), it's will generate an url with `.onion`. The service will automaticaly stop when archive is downloaded. To do this on tails:

Go to Places -> Home, right click on `alice-lesser-keys.tar` and select `Share with OnionShare`.  
So, OnionShare is open, click on `Start sharing`, it generate an url and rename archive.

To import lesser key on other devices:

    $ tar xvf alice_lesser_keys.tar
    $ gpg -a --import alice-secret-lesser.key
    $ gpg -a --import alice-public-lesser.key
    $ shred -u alice*.key

Trust ultimate on your keys

    $ gpg --edit-key alice

```
> trust
> 5 = I trust ultmately
> save
```

# Set our primary key

To use our key, we have to edit `gpg.conf`.

    $ gpg -k

note ID of our subkey with [S] and [E], we need for configure gpg.conf, in this example:

```
...
ssb   rsa4096/0x123F234555FFF3FA 2017-02-25 [S] [expire : 2017-08-24]
ssb   rsa4096/0x65FFBB38E24C1BFB 2017-02-25 [E] [expire : 2017-08-24]
...
```
    $ vim ~/.gnupg/gpg.conf

```
# flag S
default-key 0x123F234555FFF3FA
# flag E
default-recipient 0x65FFBB38E24C1BFB
```

If you need share key with friend or post on the web, generate a file like this:

    $ gpg --armor --output key.txt --export alice

## Send key to `keyserver.sks`

You have to search your key ID, this is a last 8 fingerprint character, `55487B76` here.

    $ gpg -k
      Fingerprint of key  = 346E BDED 037B 1949 013D  3576 0F15 D984 5548 7B76

    $ gpg --keyserver sks.keyservers.net --send-keys 55487B76

When people search your key, they type that:

    $ gpg -keyserver sks.keyservers.net --search-keys alice@protonmail.com

If the list contains many key, you have to compare the fingerprint.


## When you are compromised

If we have steal your device, we must send a certificate because of course, we are the only ones to have the secret key.

So, you have to boot on tails.

    $ tar xvf gpg_master_keys.tar
    $ gpg --import alice*.key
    $ gpg --edit-key alice

```
> list
> key 1
> key 2
> key 3
> revkey
Do you really want to revoke the selected subkeys? (y/N) y
Please select the reason for the revocation:
  0 = No reason specified
  1 = Key has been compromised
  2 = Key is superseded
  3 = Key is no longer used
  Q = Cancel
Your decision? 1
Enter an optional description; end it with an empty line:
>
Reason for revocation: Key has been compromised
(No description given)
Is this okay? (y/N) y
...
> save
```

Export certificate for our other device.

    $ gpg --armor --export > revoked_keys.asc

And on other devices which use your key.

    $ gpg --import revoked_keys.asc
    $ gpg --edit-key alice

```
...
pub  rsa4096/0x0FF123FF123FF123 2017-02-25 [C] [expire : 2019-02-25]
...
This key was revoked ...
sub   rsa4096/0x123F234555FFF3FA 2017-02-25 [S] [expire : 2017-08-24]
This key was revoked ...
sub   rsa4096/0x65FFBB38E24C1BFB 2017-02-25 [E] [expire : 2017-08-24]
This key was revoked ...
sub   rsa4096/0x45FD80474C2BB4FC 2017-02-25 [A] [expire : 2017-08-24]
> quit
```

You have to send each subkeys to `sks.keyserver.net`

    $ gpg --keyserver sks.keyservers.net --send-keys 123F234555FFF3FA
    $ gpg --keyserver sks.keyservers.net --send-keys 65FFBB38E24C1BFB
    $ gpg --keyserver sks.keyservers.net --send-keys 45FD80474C2BB4FC
    $ rm revoked_keys.asc

That's it.

## When subkeys expire.

When key expire, boot on `tails`, import the real secret key and enhance time of 6 month again,
The procedure is a bit repetitive...

    $ tar -xvf alice_master_keys.tar 
    $ gpg --armor --import alice_secret.key
    $ gpg --armor --import alice_public.key
    $ gpg --edit-key alice
    
```
> key 1
> key 2
> key 3
> expire
> 6m
> save
```

And re-do the same thing than before...(yaaa)

    $ gpg -a --export-secret-keys --armor alice > alice_secret.key
    $ gpg -a --export --armor alice > alice_public.key
    $ tar -cf alice_master_keys.tar alice*.key revocation.cert
    $ shred -u alice*.key revocation.cert
    $ gpg -a --export-secret-subkeys alice > subkeys
    $ gpg --delete-secret-keys alice
    $ gpg --import subkeys
    $ shred -u subkeys
    $ gpg --armor --export-secret-keys alice > alice_secret_lesser.key
    $ gpg --armor --export alice > alice_public_lesser.key
    $ tar -cf alice_lesser_keys.tar alice*.key

On your laptop, you have to remove older key and reimport the new.

    $ gpg --delete-keys alice
    $ tar xvf alice_lesser_keys.tar
    $ gpg -a --import alice_secret_lesser.key
    $ gpg -a --import alice_public_lesser.key
    $ shred -u alice*.key

Trust:
    
    $ gpg --edit-key alice

```
> trust
> 5 ultime
> save
```

### Troubleshooting

Post an issue to [github](https://github.com/szorfein/szorfein.github.io/issues)
