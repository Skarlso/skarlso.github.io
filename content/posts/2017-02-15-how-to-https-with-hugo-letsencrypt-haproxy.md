+++
author = "hannibal"
categories = ["Hugo", "DevOps", "Haproxy", "LetsEncrypt", "HTTPS"]
date = "2017-02-15T19:20:00+01:00"
title = "How to HTTPS with Hugo LetsEncrypt and HAProxy"
url = "/2017/02/15/how-to-https-with-hugo-letsencrypt-haproxy"

+++

# Intro

Hi folks.

Today, I would like to write about how to do HTTPS for a website, without the need to buy a certificate and set it up via your DNS provider. Let's begin.

## Abstract

What you will achieve by the end of this post:
- Every call to HTTP will be redirected to HTTPS via [haproxy](https://www.haproxy.com).
- HTTPS will be served with Haproxy and [LetsEncrypt](https://letsencrypt.org) as the Certificate provider.
- Automatically update the certificate before its expiration.
- No need for IPTable rules to route 8080 to 80.
- Traffic to and from your page will be encrypted.
- This all will cost you nothing.

I will use a static website generator for this called [Hugo](https://gohugo.io) which, if you know me, is my favorite generator tool. These instructions
are for haproxy and hugo, if you wish to use apache and nginx for example, you'll have to dig for the corresponding settings for letsencrypt and certbot.

# What You Will Need

## Hugo

You will need hugo, which can be downloaded from here: [Hugo](https://gohugo.io). A simple website will be enough. For themes, you can take a look
at the humongous list located here: [HugoThemes](http://themes.gohugo.io/).

## Haproxy

Haproxy can be found here: [Haproxy](https://www.haproxy.com). There are a number of options to install haproxy. I chose a simple
`apt-get install haproxy`.

## Let's Encrypt

Information about Let's Encrypt can be found on their website here: [Let's Encrypt](https://letsencrypt.org).
Let's Encrypt's client is now called [Certbot](https://certbot.eff.org/) which is used to generate the certificates. To get the latest code
you either clone the repository [Certbot](https://github.com/certbot/certbot), or use an auto downloader:

~~~bash
user@webserver:~$ wget https://dl.eff.org/certbot-auto
user@webserver:~$ chmod a+x ./certbot-auto
user@webserver:~$ ./certbot-auto --help
~~~

Either way, I'm using the current latest version: *v0.11.1*.

## Sudo

This goes without saying, but that these operations will require you to have sudo privileges. I suggest staying in sudo for ease of use.
This means that the commands, I'll write here, will assume you are in `sudo su` mode thus no `sudo` prefix will be used.

## Portforwarding

In order for your website to work under https this guide assumes that you have port *80* and *443* open on your router / network security group.

# Setup

## Single Server Environment

It is possible for haproxy, certbot and your website to run on designated servers. Haproxy's abilities allows to define multiple server sources.
In this guide, my haproxy, website and certbot will all run on the same server; thus redirecting to 127.0.0.1 and local ips. This is more
convenient, because otherwise the haproxy IP would have to be a permanent local/remote ip. Or an automated script would have to be setup which is
notified upon IP change and updates the ip records.

## Creating a Certificate

Diving in, the first thing you will require is a certificate. A certificate will allow for encrypted traffic and an authenticated website.
Let's Encrypt which is basically functioning as an independent, free, automated CA (Certificate Authority). Usually,
the process would be to pay a CA to give you a signed, generated certificate for your website, and you would have to set that up with your DNS
provider. Let's Encrypt has that all automated, and free of any charge. Neat.

### Certbot

So let's get started. Clone the repository into `/opt/letsencrypt` for further usage.

~~~bash
git clone https://github.com/certbot/certbot /opt/letsencrypt
~~~

### Generating the certificate

Make sure that there is nothing listening on ports: 80, 443. To list usage:

~~~bash
netstat -nlt | grep ':80\s'
netstat -nlt | grep ':443\s'
~~~

Kill everything that might be on these ports, like apache2 and httpd. These will be used by haproxy and certbot for challenges
and redirecting traffic.

You will be creating a [standalone](https://certbot.eff.org/docs/using.html#standalone) certificate. This is the reason we need port 80 and 443 open.
Run certbot by defining the `certonly` and `--standalone` flags. For domain validation you are going to use port 443, tls-sni-01 challenge.
The whole command looks like this:

~~~bash
cd /opt/letsencrypt
./certbot-auto certonly --standalone -d example.com -d www.example.com
~~~

If this displays something like, "couldn't connect" you probably still have something running on a port it tries to use. The
generated certificate will be located under `/etc/letsencrypt/archive` and `/etc/letsencrypt/keys` while `/etc/letsencrypt/live` is
a symlink to the latest version of the cert. It's wise to not copy these away from here, since the live link is always updated to the latest version.
Our script will handle haproxy, which requires one cert file made from privkey + fullchain|.pem files.

### Setup Auto-Renewal

Let's Encrypt issues short lived certificates (90 days). In order to not have to do this procedure every 89 days, certbot provides a nifty
command called `renew`. However, for the cert to be generated, the port 443 has to be open. This means, haproxy needs to be stopped before
doing the renew. Now, you COULD write a script which stops it, and after the certificate has been renewed, starts it again, but certbot has
you covered again in that department. It provides hooks called `pre-hook` and `post-hook`. Thus, all you have to write is the following:

~~~bash
#!/bin/bash

cd /opt/letsencrypt
./certbot-auto renew --pre-hook "service haproxy stop" --post-hook "service haproxy start"
DOMAIN='example.com' sudo -E bash -c 'cat /etc/letsencrypt/live/$DOMAIN/fullchain.pem /etc/letsencrypt/live/$DOMAIN/privkey.pem > /etc/haproxy/certs/$DOMAIN.pem'
~~~

If you would like to test it first, just include the switch `--dry-run`.

In case of success you should see something like this:

~~~bash
root@raspberrypi:/opt/letsencrypt# ./certbot-auto renew --pre-hook "service haproxy stop" --post-hook "service haproxy start" --dry-run
Saving debug log to /var/log/letsencrypt/letsencrypt.log

-------------------------------------------------------------------------------
Processing /etc/letsencrypt/renewal/example.com.conf
-------------------------------------------------------------------------------
Cert not due for renewal, but simulating renewal for dry run
Running pre-hook command: service haproxy stop
Renewing an existing certificate
Performing the following challenges:
tls-sni-01 challenge for example.com
Waiting for verification...
Cleaning up challenges
Generating key (2048 bits): /etc/letsencrypt/keys/0002_key-certbot.pem
Creating CSR: /etc/letsencrypt/csr/0002_csr-certbot.pem
** DRY RUN: simulating 'certbot renew' close to cert expiry
**          (The test certificates below have not been saved.)

Congratulations, all renewals succeeded. The following certs have been renewed:
  /etc/letsencrypt/live/example.com/fullchain.pem (success)
** DRY RUN: simulating 'certbot renew' close to cert expiry
**          (The test certificates above have not been saved.)
Running post-hook command: service haproxy start
~~~

Put this script into a crontab to run every 89 days like this:

~~~bash
crontab -e
# Open crontab for edit and paste in this line
* * */89 * * /root/renew-cert.sh
~~~

And you should be all set. Now we move on the configure haproxy to redirect and to use our newly generated certificate.

## Haproxy

Like I said, haproxy requires a single file certificate in order to encrypt traffic to and from the website. To do this, we need to combine
`privkey.pem` and `fullchain.pem`. As of this writing, there are a couple of solutions to automate this via a post hook on renewal. And also,
there is an open ticket with certbot to implement a simpler solution located here: https://github.com/certbot/certbot/issues/1201. I, for now,
have chosen to simply concatenate the two files together with `cat` like this:

~~~bash
DOMAIN='example.com' sudo -E bash -c 'cat /etc/letsencrypt/live/$DOMAIN/fullchain.pem /etc/letsencrypt/live/$DOMAIN/privkey.pem > /etc/haproxy/certs/$DOMAIN.pem'
~~~

It will create a combined cert under `/etc/haproxy/certs/example.com.pem`.

### Haproxy configuration

If haproxy happens to be running, stop it with `service haproxy stop`.

First, save the default configuration file: `cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.old`. Now, overwrite the old one with this
new one (comments about what each setting does, are in-lined; they are safe to copy):

~~~bash
global
    daemon
    # Set this to your desired maximum connection count.
    maxconn 2048
    # https://cbonte.github.io/haproxy-dconv/configuration-1.5.html#3.2-tune.ssl.default-dh-param
    # bit setting for Diffie - Hellman key size.
    tune.ssl.default-dh-param 2048

defaults
    option forwardfor
    option http-server-close

    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http

# In case it's a simple http call, we redirect to the basic backend server
# which in turn, if it isn't an SSL call, will redirect to HTTPS that is
# handled by the frontend setting called 'www-https'.
frontend www-http
    # Redirect HTTP to HTTPS
    bind *:80
    # Adds http header to end of end of the HTTP request
    reqadd X-Forwarded-Proto:\ http
    # Sets the default backend to use which is defined below with name 'www-backend'
    default_backend www-backend

# If the call is HTTPS we set a challenge to letsencrypt backend which
# verifies our certificate and than direct traffic to the backend server
# which is the running hugo site that is served under https if the challenge succeeds.
frontend www-https
    # Bind 443 with the generated letsencrypt cert.
    bind *:443 ssl crt /etc/haproxy/certs/skarlso.com.pem
    # set x-forward to https
    reqadd X-Forwarded-Proto:\ https
    # set X-SSL in case of ssl_fc <- explained below
    http-request set-header X-SSL %[ssl_fc]
    # Select a Challenge
    acl letsencrypt-acl path_beg /.well-known/acme-challenge/
    # Use the challenge backend if the challenge is set
    use_backend letsencrypt-backend if letsencrypt-acl
    default_backend www-backend

backend www-backend
   # Redirect with code 301 so the browser understands it is a redirect. If it's not SSL_FC.
   # ssl_fc: Returns true when the front connection was made via an SSL/TLS transport
   # layer and is locally deciphered. This means it has matched a socket declared
   # with a "bind" line having the "ssl" option.
   redirect scheme https code 301 if !{ ssl_fc }
   # Server for the running hugo site.
   server www-1 192.168.0.17:8080 check

backend letsencrypt-backend
   # Lets encrypt backend server
   server letsencrypt 127.0.0.1:54321
~~~

Save this, and start haproxy with `services haproxy start`. If you did everything right, it should say nothing.
If, however, there went something wrong with starting the proxy, it usually displays something like this:

~~~bash
Job for haproxy.service failed. See 'systemctl status haproxy.service' and 'journalctl -xn' for details.
~~~

You can also gather some more information on what went wrong from `less /var/log/haproxy.log`.

# Starting the Server

Everything should be ready to go. Hugo has the concept of a baseUrl. Everything that it loads, and tries to access
will be prefixed with it. You can either set it through it's `config.yaml` file, or from the command line.

To start the server, call this from the site's root folder:

~~~bash
hugo server --bind=192.168.x.x --port=8080 --baseUrl=https://example.com --appendPort=false
~~~

Interesting thing here to note is `https` and the port. The IP could be 127.0.0.1 as well. I experienced problems though
with not binding to network IP when I was debugging the site from a different laptop on the same network.

Once the server is started, you should be able to open up your website from a different browser, not on your local network,
and see that it has a valid certificate installed. In Chrome you should see a green icon telling you that the cert is valid.

# Last Words

And that is all. The site should be up and running and the proxy should auto-renew your site's certificate. If you happened to
change DNS or change the server, you'll have to reissue the certificate.

Thanks for reading!
Any questions or trouble setting something up, please feel free to leave a comment.

Cheers,
Gergely.
