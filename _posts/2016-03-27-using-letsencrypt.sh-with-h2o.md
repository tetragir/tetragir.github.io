---
layout:	post
title:	Using letsencrypt.sh with h2o
date: 2016-03-27 16:00:00
category: freebsd/www
tags: [freebsd h2o letsencrypt letsencrypt.sh webserver]
type: it
platform: FreeBSD
---

This will be a really short guide about how to set up the [H2O](https://h2o.examp1e.net/) webserver with letsencrypt, and how to automate it. I've read a nice tutorial about letsencrypt and nginx on [Peter Wemm's site](https://blog.crashed.org/switching-ssl-certs-to-letsencrypt/). This guide is similar, but for an h2o webserver. I intend to write another guide about www/h2o later once v2.0 is released, this is just a short tutorial about letsencrypt.

* TOC
{:toc}

## Requirements

You only need the [www/h2o](https://www.freshports.org/www/h2o/) webserver (obviously) and the [security/letsencrypt.sh](https://www.freshports.org/security/letsencrypt.sh/).

## Preparation

First, install security/letsencrypt.sh from ports (with portmaster):

~~~
portmaster security/letsencrypth.sh
~~~

Or from packages:

~~~
pkg install security/letsencrypt.sh
~~~

Then configure h2o to redirect the domain validation request to the right folder.

~~~
paths:
    "/.well-known/acme-challenge":
        file.dir: "/usr/local/etc/letsencrypt.sh/.acme-challenges"
~~~

The above needs to be in the part where h2o is configured to listen on port 80. The other important thing is to place this **before** the redirection to https (if any).
Then restart h2o.

~~~
service h2o restart
~~~

## Generate the certificates

Create a *config.sh* file containing the contact email address, and a *domains.txt* file with the domain (both with and without "www.") and request a certificate. Replace *tetragir.com* with the actual domain. The options that can be configured are explained in the */usr/local/etc/letsencrypt.sh/config.sh.example* file.

~~~
# cd /usr/local/etc/letsencrypt.sh
# echo 'CONTACT_EMAIL=your@email.address' > config.sh
# echo 'tetragir.com www.tetragir.com' > domains.txt
# letsencrypt.sh
~~~

Once you are done, the result should be the following:

~~~
 # INFO: Using main config file /usr/local/etc/letsencrypt.sh/config.sh
Processing tetragir.com
 + Signing domains...
 + Generating private key...
 + Generating signing request...
 + Requesting challenge for tetragir.com...
 + Responding to challenge for tetragir.com...
 + Challenge is valid!
 + Requesting certificate...
 + Checking certificate...
 + Done!
 + Creating fullchain.pem...
 + Done!
~~~

If everything went right, the certificate can be found in the */usr/local/etc/letsencrypt.sh/certs/tetragir.com/* folder.

## Configure h2o to the newly created certificates

H2O needs to be told where these certificates actually are, so the following lines need to be placed in the part where h2o is configured to listen on port 443.

~~~
certificate-file: /usr/local/etc/letsencrypt.sh/certs/tetragir.com/fullchain.pem
key-file: /usr/local/etc/letsencrypt.sh/certs/tetragir.com/privkey.pem
~~~

Naturally you need to replace the folder names with the actual path. Then restart h2o.

~~~
service h2o restart
~~~

## Automating letsencrypt

The certificates from letsencrypt are only valid for 90 days and therefore it is advisable to automate the process. It is possible to run letsencrypt.sh with cron, but I like the "periodic" solution more. In order to make it work, place the following line in */etc/periodic.conf*:

~~~
weekly_letsencrypt_enable="YES"
~~~

This way the server will check for certificate renewals every week and will renew it when necessary (that is, when remaining validity time is shorter than 30 days). Have fun!
