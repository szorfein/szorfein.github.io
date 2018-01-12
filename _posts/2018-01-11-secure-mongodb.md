---
layout: post-detail
title: secure mongodb
date: 2018-01-11
categories: mongodb
description: Secure mongodb with TLS/SSL, auth, roles, ...
img-url: https://webassets.mongodb.com/_com_assets/cms/mongodb-logo-rgb-j6w271g1xn.jpg
comments: true
---

Ok, we start on a fresh install.  
After start the service, `mongodb` is ready to start on port 27017 per default, test your install.

    $ sudo systemctl start mongodb
    $ mongo
      > exit

# Create the root account

Log on mongo.

    $ mongo

```
> use admin
> db.createUser({
    user: 'rootUser',
    pwd: '1234',
    roles: [
        { role: 'root', db: 'admin' }
    ],
    authenticationRestrictions: [{
        clientSource: ['127.0.0.1'],
        serverAddress: ['127.0.0.1']
    }]
})
> exit
```

# Add collection if need

Auth on mongo.

    $ mongo -u rootUser -p 1234 --authentication admin
    
You can create somes collections now.

```
> db.createCollection('otherCollection')
> exit
```

# Enable TLS/SSL

We going to enable TLL/SSL. Here a custom script to generate a self-signed certificate.

    $ vim gen-certs.sh

Past content bellow.

```sh

#!/bin/sh

## ref: https://stackoverflow.com/questions/35790287/self-signed-ssl-connection-using-pymongo#35967188

# HOST is the domain name, for a local server (127.0.0.1), don't touch 'uname -n'
HOST=$(uname -n)
MAIL=""
COUNTRY=""
RANDOM_2DIGIT=""
PREFIX="mongo"
ORG_UNIT="dev1"

# boolean
ENCRYPT=false

if ( $ENCRYPT ) then
    openssl req -out "${PREFIX}-ca.pem" -keyout "${PREFIX}-privkey.pem" -new -x509 -days 365 -subj "/C=${COUNTRY:-AU}/ST=NSW/O=Organisation/CN=root/emailAddress=${MAIL:?}"
    wait
else
    openssl req -out "${PREFIX}-ca.pem" -keyout "${PREFIX}-privkey.pem" -new -x509 -nodes -days 365 -subj "/C=${COUNTRY:-AU}/ST=NSW/O=Organisation/CN=root/emailAddress=${MAIL:?}"
fi

echo ${RANDOM\_2DIGIT:-00} > "${PREFIX}-file.srl"

#
# SERVER
#

openssl genrsa -out "${PREFIX}-server.key" 2048 || exit 1

openssl req -key "${PREFIX}-server.key" -new -out "${PREFIX}-server.req" -subj  "/C=${COUNTRY:-AU}/ST=NSW/O=Organisation/CN=server1/CN=${HOST:?}/emailAddress=${MAIL:?}" || exit 1

openssl x509 -req -in "${PREFIX}-server.req" -CA "${PREFIX}-ca.pem" -CAkey "${PREFIX}-privkey.pem" -CAserial "${PREFIX}-file.srl" -out "${PREFIX}-server.crt" -days 365 || exit 1

cat ${PREFIX}-server.key ${PREFIX}-server.crt > "${PREFIX}-server.pem"

openssl verify -CAfile "${PREFIX}-ca.pem" "${PREFIX}-server.pem"

#
# CLIENT
#

openssl genrsa -out ${PREFIX}-client.key 2048

openssl req -key ${PREFIX}-client.key -new -out ${PREFIX}-client.req -subj "/C=${COUNTRY:-AU}/ST=NSW/O=Organisation/OU=${ORG_UNIT:-Organisation-client}/CN=client1/emailAddress=${MAIL:?}"

openssl x509 -req -in ${PREFIX}-client.req -CA ${PREFIX}-ca.pem -CAkey "${PREFIX}-privkey.pem" -CAserial ${PREFIX}-file.srl -out ${PREFIX}-client.crt -days 365

cat ${PREFIX}-client.key ${PREFIX}-client.crt > ${PREFIX}-client.pem

openssl verify -CAfile ${PREFIX}-ca.pem ${PREFIX}-client.pem

## clean
#rm ${PREFIX}-*
```

As you look, the script contain somes variables than you can customize:

+ `HOST=$(uname -n)`, is better than `localhost` or `127.0.0.1`, for a non-local server, put a domain name.
+ `MAIL=""`
+ `COUNTRY=""`, default is `AU`, two letter here, FR, EN, US...
+ `RANDOM_2DIGIT=""`, default is `00`
+ `PREFIX="mongo"`, all created file are prefix with mongo.
+ `ENCRYPT=true`, if enable, program will call for a password, recommanded on production env.
+ `ORG_UNIT=""`, or `OU` default is `dev1`. value must be different for each user and from `mongo-ca.pem` cert.

Once ready, launch that:

    $ chmod +x gen-certs.sh
    $ ./gen-certs.sh

The program have generate 11 files.

The program server `mongod` need `mongo-server.pem` and `mongo-ca.pem`.
Program client need `mongo-client.pem` and `mongo-ca.pem`.

Edit `/etc/mongod.conf`

```
# vim /etc/mongod.conf

net:
    ssl: 
        mode: requireSSL
        PEMKeyFile: <full-path-mongo-server.pem>
        CAFile: <full-path-mongo-ca.pem>
security:
    clusterAuthMode: x509
```

Restart the daemon:

    $ sudo systemctl restart mongodb

To log in with client: 

    $ mongo --ssl --sslPEMKeyFile ~/mongo-client.pem --sslCAFile ~/mongo-ca.pem --host <server hostname>

If you have set `ENCRYPT="true"`, you need to add an additionnal argument for passphrase `--sslPEMKeyPassword "yourpassword"`

# Create user accounts with x509 certs

Retrieve the subject string from a client cert with this command:

    $ openssl x509 -in mongo-client.pem -inform PEM -subject -nameopt RFC2253
    
return:

      subject= emailAddress=alice@protonmail.com,CN=client1,OU=dev1,O=Organisation,ST=NSW,C=FR

You have to past this without space character in a mongo shell (drop `subject= `).

    $ mongo --ssl --sslPEMKeyFile ~/mongo-client.pem --sslCAFile ~/mongo-ca.pem --host <server hostname> -u rootUser -p 1234 --authenticationDatabase admin

```
> db.getSiblingDB("$external").runCommand({
    createUser: "emailAddress=alice@protonmail.com,CN=client1,OU=dev1,O=Organisation,ST=NSW,C=FR",
    roles: [
        { role: 'readWrite', db: 'otherCollection' },
        { role: 'userAdminAnyDatabase', db: 'admin' }
    ],
    writeConcern: { w: "majority" , wtimeout: 5000 }
})
> exit
```

To connect with x509 cert:

    $ mongo --ssl --sslPEMKeyFile ~/.certs/mongo-client.pem --sslCAFile ~/.certs/mongo-ca.pem --host phaeton

And to perform an authentification:

```
> db.getSiblingDB("$external").auth({
    mechanism: "MONGODB-X509",
    user: "emailAddress=alice@protonmail.com,CN=client1,OU=dev1,O=Organisation,ST=NSW,C=FR",
})
```

Article will evolve later...

### Troubleshooting

Post an issue to [github](https://github.com/szorfein/szorfein.github.io/issue).
