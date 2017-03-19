+++
date = "2017-02-21T10:37:28-05:00"
title = "Setting up Matrix Synapse with TURN on Debian Jessie and FreeBSD 11"
topics = ["tech", "security", "devops", "privacy", "linux", "servers", "bsd"]
tags = ["security", "tech", "linux", "bsd"]
draft = true

+++

# Introduction

#### What is Matrix Synapse?

[Matrix](https://matrix.org/) is a rethinking of how text and media based communications work.  Taking a decentralized federated approach, it provides a set of APIs that support sending text, images, audio, and video between clients.  With the [Riot IM client](https://riot.im/) you get a similar interface and UX to Slack which sits on top of the Matrix Synapse server.  Since Matrix is an open standard protocol, there are several servers and clients, but Synapse is the reference design written in Python using the Twisted framework.  This is what we'll be using today for this how-to, however there are alternative servers which are very interesting.  In particular, there's a Matrix server written in Rust with an emphasis on security and correctness called [Ruma](https://www.ruma.io/).

#### What is coTURN?

[coTURN](https://github.com/coturn/coturn) is an evolution of the reference TURN server project for RFC5766.  It adds additional advanced features to TURN, including for example the ability to work with other services via a REST API.  It's internals are built on top of libevent2 ensuring that its high performance.  In our case, we'll be using TURN/STUN to support relaying video and VOIP calls through Matrix.  For more information about what coTURN can do see its project page. 

### Security Considerations

The Synapse homeserver and Riot client while being the reference design is still in early Beta stages and may have security flaws.  Additionally, end-to-end encryption while now enabled by default is still in development stages.  End-to-end encryption is an optional feature in the protocol.  Given the nature of this, and that the majority of channels on the Matrix network are bound to IRC through IRC - Matrix bridges, please be aware that there are significant security considerations to communicating across Matrix, just as with any other group chat tool. 

For the highest communications security, I currently recommend that people use [Signal](https://whispersystems.org/).

#### Hardware Requirements

It goes somewhat without saying that hardware requirements vary based on the load you expect your server to run.  If it's just you and a handful of friends, probably not a lot required.  If you're setting this up for a larger organization that needs to be taken into account.  This is especially true of TURN which provides the VOIP and video relay capabilities.  For the purposes of this howto and my personal setup, I expect no more than maybe 10 chat clients connected to my homeserver with registered users and no more than 2 participants in a single 1-to-1 video chat to occur.  Thus here is what I used:

Matrix Synapse server:
* Debian Jessie 8.7x64
* 1GB RAM/1 vCPU droplet on [DigitalOcean (referral link)](https://m.do.co/c/8d8fbc814bfd)

TURN server:
* FreeBSD 11.0x64 ZFS root
* 1GB RAM/1 vCPU droplet on [DigitalOcean (referral link)](https://m.do.co/c/8d8fbc814bfd)


Note on the referral links above.  This is the first time I've posted a referral link, however I'd like to continue experimenting and writing how-tos.  Clicking the link does not make me money, what it does is get me hosting credit with DigitalOcean to help cover part of my bills for running these and other servers.

Another note on server setup.  _**Don't enable IPv6**_.  There's a bug in Twisted 16 (now fixed in 17.10) that causes IPv6 DNS resolution to fail and will break federation.  Thanks to @Shell in the official Matrix channel #matrix:matrix.org for helping me figure this out.  As you'll see from my configuration, I did have IPv6 enabled and while I was able to get it working you'll save yourself many headaches for the moment by sticking to IPv4.

# Basic Server Setup - Linux

I'll skip over quite a bit here although [I've written on the topic of server security some before](https://tristor.ro/blog/2016/03/20/the-ops-approach-to-linux-server-security/).  In this case, the focus is on using separate instances in DigitalOcean for each service to provide separation and ensuring strong access controls.

##### 0. Ensure the server is up to date

`apt-get update && apt-get upgrade -y`

##### 1. Create a new user on the server

I use ZSH, so its the first thing I install.  But this is optional and you are welcome to use `/bin/bash` instead.

```
useradd -U -m -s /bin/zsh -G sudo tristor
passwd tristor
``` 

After installing the user, you may need to change the default SSH configuration manually to set `PasswordAuthentication yes` in order to add your public key to the new user via `ssh-copy-id`.  This is very important because the next step will ONLY allow your user to login via SSH via public key + password from here on out.

##### 2.  Harden the SSH service

Grab [my hardened SSH config](https://gist.github.com/Tristor/6d589939ee43e8956a94) and adapt it to your purposes.  It goes in `/etc/ssh/sshd_config`.  Afterwards restart the SSH server with `systemctl restart sshd`.

By default my SSH config requires a public key + password to login to the system via SSH and does not allow root SSH logins.


##### 3. Put in to place basic firewall rules

Since we're on Debian for this server we can take advantage of the great IPtables wrapper `ufw`.  If it's not already installed, install it now with `apt-get install ufw`.

We'll be using a default deny policy for both incoming and outgoing, to make sure you don't lock yourself out of the server allow SSH before you enable the firewall.

```
ufw allow ssh
ufw allow out ssh
ufw logging on
ufw default deny outgoing
ufw enable
ufw status verbose
```

You should see the following now when you run `ufw status verbose`:

```
Status: active
Logging: on (low)
Default: deny (incoming), deny (outgoing)
New profiles: skip

To                         Action      From
--                         ------      ----
22                         ALLOW IN    Anywhere
22                         ALLOW IN    Anywhere (v6)


22                         ALLOW OUT   Anywhere
22                         ALLOW OUT   Anywhere (v6)
```

Now we're going to add the rules for services which normally need out.  The services I consider essential to a basic system is DNS, NTP, HTTP, and HTTPS.  This allows you to use package managers and sync to time services.

```
ufw allow out ntp
ufw allow out domain
ufw allow out http
ufw allow out https
```

Next we'll allow outgoing and incoming ports for Matrix Synapse Server and Nginx.

```
ufw allow http
ufw allow https
ufw allow 8448
ufw allow out 8448
```

One nice feature of both `pf` (which we'll use in the next section) and `ufw` is that they support using service names instead of port numbers by parsing the port to service mappings in `/etc/services`.

# Basic Server Setup - FreeBSD

Most of the same things which apply in my advice about Linux also apply to FreeBSD.  While it's a different operating system, there are many similarities as they're both POSIX compliant and are interacted with using similar or the same shell.

##### 0. Ensure the server is up to date

`pkg update && pkg upgrade`

##### 1. Create a new user on the server

As before, if you want to use ZSH ensure you install it first.  On FreeBSD it'll be in `/usr/local/bin/zsh` instead of `/bin/zsh`.  The process of adding a user is a bit different, it'll walk you through a series of prompts including letting you set the password, so you just run `adduser`.  Ensure you add your new user to the `wheel` group.

Finally, run `visudo` and uncomment the line
```
%wheel ALL=(ALL) ALL
```

As before, you may need to temporarily set `PasswordAuthentication yes` in your `/etc/ssh/sshd_config` so that you can run `ssh-copy-id` against your new user as after the next step you'll no longer be able to login without using a public key + password and that user account

##### 2. Harden the SSH service

As before grab [my hardened SSH config](https://gist.github.com/Tristor/6d589939ee43e8956a94) and adapt it to your purposes.  It goes in `/etc/ssh/sshd_config`.  Afterwards restart the SSH server with `/etc/rc.d/sshd restart`.

By default my SSH config requires a public key + password to login to the system via SSH and does not allow root SSH logins.

##### 3. Put in to place basic firewall ruels

For FreeBSD we get access to the beautiful and high-performance firewall `pf`.  `pf` is configured entirely by file, so there's no need to run a bunch of commands here.  I've taken the liberty of creating [a recommended pf configuration for the TURN server](https://gist.github.com/Tristor/04b6bb4a268fb50f5196392a2ce4c58e).  You can grab this file and put it in `/etc/pf.conf` and then run the following:

```
pfctl -n -v -f /etc/pf.conf
```

This parses the file and verifies syntax but doesn't change anything.  Assuming everything checks out we next need to load the kernel modules for `pf` and then ensure these modules and the service load on boot.


```
kldload pf
kldload pflog
cat <<'EOF' >> /boot/loader.conf.local
# pf
pf_load="YES"
pflog_load="YES"
EOF

cat << 'EOF' >> /etc/rc.conf
# pf
pf_enable="YES"
pf_rules="/etc/pf.conf"
pflog_enable="YES"
pflog_logfile="/var/log/pflog"
EOF
```

Now we can enable the `pf` firewall with `pfctl -ef /etc/pf.conf`

When you enable `pf` the first time it will cause you to lose your SSH connection.  Don't panic, just reconnect and all should be well.

# Install and Configure Nginx - Linux

Next we're going to install the Nginx web server on our Debian instance.  This will act as a reverse proxy in front of Matrix Synapse so that clients can connect on port 443, and so we can enforce some advanced TLS features.

##### 1. Add the Repository
Add the Nginx mainline repository to your apt sources
```
cat << 'EOF' >> /etc/apt/sources.list.d/nginx.list
deb http://nginx.org/packages/mainline/debian/ jessie nginx
deb-src http://nginx.org/packages/mainline/debian/ jessie nginx
EOF
```

Add the Nginx signing key, then run apt-get update and install Nginx
```
apt-key adv --fetch-keys https://nginx.org/keys/nginx_signing.key
apt-get update
apt-get install -y nginx
```

##### 2. Set up the Configuration Split
The official Nginx packages do not default to the `sites-available`/`sites-enabled` split that I prefer for managing configurations, and is a typical Debianism.  So we can set this up manually very easily by editing the file `/etc/nginx/nginx.conf`

```
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
```

Then run the following:

```
mkdir -pv /etc/nginx/sites-{available,enabled}
mv /etc/nginx/conf.d/default.conf /etc/nginx/sites-available/
ln -s /etc/nginx/sites-available/default.conf /etc/nginx/sites-enabled/default.conf
systemctl restart nginx
```

If the above works, you should still see the default Nginx page when visiting the web address.  From here, you can start putting/editing configs in `/etc/nginx/sites-available` and then linking them to `/etc/nginx/sites-enabled` when you want to use load those configurations.

##### 3. Create a configuration for CertBot

###### A. Install CertBot and Create Web Root
I decided I would put the web content into `/var/www/matrix.tristor.ro/public/`.  You can choose any path you'd like, but whichever path you choose needs to be chowned to `www-data:www-data` and then you need to add the user `nginx` to the `www-data` group, as below.

```
mkdir -pv /var/www/matrix.tristor.ro/public/
chown -R www-data:www-data /var/www/
usermod -G www-data -a nginx
```

Finally, we need to install and run [CertBot](https://certbot.eff.org/).  [Thanks to the work by the EFF](https://supporters.eff.org/donate/support-work-on-certbot), we're able to easily automate [Let's Encrypt](https://letsencrypt.org/) actions like generating or renewing certificates.  In order to provide the domain validation for granting a certificate, Let's Encrypt needs to verify you control the domain in some way.  One of those ways is by modifying content in a well-known path during the validation process in a short period of time.  This is how CertBot works with the "webroot" mode we'll be using here.  The `certbot` package is part of Jessie Backports.

```
cat << 'EOF' >> /etc/apt/sources.list.d/backports.list
deb http://ftp.debian.org/debian jessie-backports main
EOF

apt-get update
apt-get install -y -t jessie-backports certbot
```

###### B. Configure Nginx and Run CertBot
Once the package is finished installing, we'll make a configuration in Nginx for CertBot and perform a webroot based certificate generation.  You can find [the configuration here](https://gist.github.com/Tristor/003d46b3202a087a8718f62eb36418bc) then put it into `/etc/nginx/sites-available/certbot.conf`.  From there it's a simple matter to link it and restart Nginx after removing the default site.

```
rm -f /etc/nginx/sites-enabled/default.conf
ln -s /etc/nginx/sites-available/certbot.conf /etc/nginx/sites-enabled/certbot.conf
systemctl restart nginx
```

Now we can run CertBot to get our certificate which should succeed. Obviously you should specify your webroot path and domain name, respecitively.

`certbot certonly --agree-tos --webroot -w /var/www/matrix.tristor.ro/public/ -d matrix.tristor.ro`

You may be asked for your email address as part of the first run.  After this completes you should see a message that tells you the location of your new certificate. Something like ` /etc/letsencrypt/live/matrix.tristor.ro/fullchain.pem (success)`

The CertBot package will automatically put a cron job to automate renewals in `/etc/cron.d/certbot`.  You can modify this if you need to.  You can test a renewal at any time by running `certbot renew --dry-run` to see the result.

##### 4. Link Certificates, Generate DH Params, Create OCSP Bundle, and Set up Reverse Proxy

So now that we've got certificates from Let's Encrypt, we need to make sure Nginx knows about them and that we've taken the other necessary steps for having a secure TLS configuration.

```
mkdir -pv /etc/nginx/ssl
openssl dhparam -out /etc/nginx/ssl/dhparam.pem 2048
ln -s /etc/letsencrypt/live/matrix.tristor.ro/fullchain.pem /etc/nginx/ssl/matrix.tristor.ro.crt
ln -s /etc/letsencrypt/live/matrix.tristor.ro/privkey.pem /etc/nginx/ssl/matrix.tristor.ro.key
``` 

For the OCSP bundle, you can do one of two things:

###### 1. (More Secure) Get the CA and Intermediate certificates yourself and generate your bundle with `cat`

1. Retrieve the [Let's Encrypt Cross-Signed Intermediate X3](https://letsencrypt.org/certificates/) and the [IdenTrust DST CA X3](https://www.identrust.com/certificates/trustid/root-download-x3.html) into separate files on your server.

2. Concatenate these together into an OCSP Bundle.  Order is Root -> Intermediate.  `cat identrust-ca.crt letsencrypt.crt > ocsp-bundle.crt`

3. Place the bundle in your Nginx SSL directory `cp ocsp-bundle.crt /etc/nginx/ssl/ocsp-bundle.crt`

###### 2. (Less Secure) Grab my pre-generated OCSP Bundle over HTTPS.

```
cd /etc/nginx/ssl/
curl -o ocsp-bundle.crt https://tristor.ro/files/letsencrypt-ocspbundle.crt
```

Finally we set up our reverse proxy.  Matrix Synapse uses port 8448 for federation between servers, and can either have clients connect on the same port or you can proxy it to 443 as we're going to do.  The [reverse proxy configuration I provide](https://gist.github.com/Tristor/8bb758c9e8ad67afffb1eb6e9d69c8cf) will need to be modified to point to the correct certificate paths and web roots.  Additionally, if you followed my instructions above and didn't enabled IPv6 on your droplet, you should remove the listen lines containing `[::]:` as these are for binding IPv6 sockets.

Finally, link your configuration, removing the certbot configuration, and restart Nginx.
```
rm -f /etc/nginx/sites-enabled/certbot.conf
ln -s /etc/nginx/sites-available/matrix-synapse.conf /etc/nginx/sites-enabled/matrix-synapse.conf
systemctl restart nginx
```

At this point, Nginx is configured correctly, and we can now move on to working on the Matrix Synapse server instance itself.

# Install and Configure Matrix Synapse - Linux

##### 1. Add the Repository and Install Synapse
Add the official Matrix repository to your apt sources
```
cat << 'EOF' >> /etc/apt/sources.list.d/nginx.list
deb http://matrix.org/packages/debian/ jessie main
deb-src http://matrix.org/packages/debian/ jessie main
EOF
```

Add the Nginx signing key, then run apt-get update and install Nginx
```
apt-key adv --fetch-keys https://matrix.org/packages/debian/repo-key.asc
apt-get update
apt-get install -y matrix-synapse
```

When prompted by `dpkg` to enter a domain for your Matrix installation, be aware that the domain should correspond to your root domain as we'll fix things later by adding an `SRV` record in DNS.  For instance, for me, my Matrix domain is `tristor.ro` and my homeserver is running on `matrix.tristor.ro`, but I have an SRV record which allows federation to work.  The domain is where users will be registered, so I am `@Tristor:tristor.ro` in Matrix.

##### 2. Configure Your Synapse HomeServer

Before you can do much with Synapse, you need to let it generate the server signing key.  To do this, run ``