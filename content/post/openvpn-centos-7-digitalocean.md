+++
date = "2016-05-05T20:37:31-05:00"
tags = ["travel", "tech", "security", "devops", "privacy", "linux", "selinux", "servers", "vpn"]
title = "Setting up OpenVPN on CentOS 7 using DigitalOcean"
topics = ["security", "tech", "linux", "travel"]

+++

# Introduction
##### Why Bother?

As should be abundantly clear from my prior writings I am about to leave on a trip for a year.  During that time I'll likely be making use of numerous public Wi-Fi access points, not to mention whatever dodgy cellular providers are available in each location I travel to.  As part of my overall stance on privacy, its essential I take steps to secure my communication while traveling, the primary of which is using a VPN for basically everything on both my laptop and my phone.  To do this, I'm using a droplet from [DigitalOcean](https://www.digitalocean.com/) that's just $5/mo and doesn't have to be shared with anyone else (from an IP/network perspective anyway).

##### What does a VPN do?

A VPN, simply put, creates an encrypted tunnel between your computer (or phone) and the VPN server.  All the traffic that would typically go to the Internet gets redirected down this tunnel and reaches the Internet from the other side, as if it was originating on the server rather than on your computer or your phone.  What this means is nobody locally can snoop your traffic and you can appear logically on the Internet to come from anywhere you can get a VPN.  The downside of course is that there's some overhead which makes your connection somewhat slower and that you have to trust the server on the other side.  That second point is why we're going to set up our own VPN server rather than using some random VPN service.  Bonus points, after I'm done I'll be able to watch Netflix streams from anywhere in the world.

##### VPN Clients and You!

Of course to connect you need a VPN client.  On OS X I recommend using [Viscosity ($9)](https://www.sparklabs.com/viscosity/) or [Tunnelblick (FREE)](https://www.tunnelblick.net/).  On iOS I recommend [OpenVPN Connect](https://itunes.apple.com/us/app/openvpn-connect/id590379981?mt=8) which is the official iOS client for OpenVPN.  If I get around to writing the rest of it, I'll cover setting up Viscosity in Part 3 of my OS X Set Up Guide.

# Basic Server Setup

I'll skip over quite a bit here although [I've written on the topic of server security some before](https://tristor.ro/blog/2016/03/20/the-ops-approach-to-linux-server-security/).  In this case, the focus is on using separate instances in DigitalOcean for each service to provide separation and ensuring strong access controls.

##### 1. Create a new user on the server

I use ZSH, so its the first thing I install.  But this is optional and you are welcome to use `/bin/bash` instead.

```
useradd -U -G wheel -m -s /bin/zsh tristor
``` 

##### 2.  Harden the SSH service

Grab [my hardened SSH config](https://gist.github.com/Tristor/6d589939ee43e8956a94) and adapt it to your purposes.  It goes in `/etc/ssh/sshd_config`.  Afterwards restart the SSH server with `systemctl restart sshd`.  Since we're on CentOS make sure to change `UsePAM no` to `UsePAM yes` in the configuration otherwise the SSH server will complain because of Red Hat customizations.

By default my SSH config requires a public key + password to login to the system via SSH and does not allow root SSH logins.


##### 4. Enable IPv4 Forwarding

Edit /etc/sysctl.conf and add the following lines
```
# Packet forwarding
net.ipv4.ip_forward = 1
```
Save the file, then run `sysctl -p` to load the changes.

You can verify forward is enabled by doing `cat /proc/sys/net/ipv4/ip_forward` which should return `1` if its enabled.

##### 4. Put in to place basic firewall rules

First things first, we're going to ditch `firewalld` because its tragic and terrible.  I'll leave that one for a further analysis down the road.

```
systemctl stop firewalld
systemctl mask firewalld
yum install iptables iptables-services
```

Then we are going to deploy [basic firewall rules to block all inbound and outbound traffic except the following](https://gist.github.com/Tristor/ed0f6867d2b0fa4c1f80300af6e0e12e):

Inbound:
* 22 (SSH)
* 1194 (UDP OpenVPN)
* 443 (TCP OpenVPN) (commented out by default)

Outbound:
* 53 (DNS)
* 80 (HTTP)
* 443 (HTTPS)

With our hardened SSH config, I'm not really concerned further with SSH, but this prevents any service ports from being accessible in or out that aren't in use by things we care about on this instance.

That last line in the firewall rules is to enable NAT routing for the VPN server.  More on that in a moment.

##### 5. **Save** Your Firewall Rules and Enable the Firewall

```
systemctl enable iptables
systemctl start iptables
service iptables save
```

That last line is very important.  It saves a copy of your currently running firewall rules to `/etc/sysconfig/iptables` so they'll be loaded on reboot, otherwise your box will be wide open on a reboot.

# Install and Configure OpenVPN

##### 1. Set up EPEL

There are no decent packages for RHEL/CentOS from OpenVPN officially, so luckily [EPEL](https://fedoraproject.org/wiki/EPEL) exists.

```
rpm -Uvh http://download.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-6.noarch.rpm
```

This will install and set up the EPEL repository locally.

##### 2. Install OpenVPN packages

```
yum install openvpn easy-rsa
cp -R /usr/share/easy-rsa/2.0/ /etc/openvpn/easy-rsa/
```

##### 3. Configure the OpenVPN Server

I've taken the liberty of [providing my OpenVPN config](https://gist.github.com/Tristor/b939b830c6426926696a49d4789031e8) which you can put into `/etc/openvpn/server.conf`.  It's generic, has hardened settings, and is matched to the above firewall rules in IPtables (note that the subnet set in the `server` line must match that set for doing NAT in the firewall)

After putting the configuration in place you need to generate your Diffie-Hellman parameters and your OpenVPN PSK.  Within `/etc/openvpn/` run the following commands

```
openvpn --genkey --secret ta.key
openssl dhparam -out dh2048.pem 2048
```

##### 4. Create Your Certificate Authority (CA)

First things first, you need to change directories to `/usr/share/easy-rsa/2.0/`.  Then edit the vars file.  Main things you need to change are the following:

```
export KEY_COUNTRY="US"
export KEY_PROVINCE="CA"
export KEY_CITY="SanFrancisco"
export KEY_ORG="Fort-Funston"
export KEY_EMAIL="mail@domain"
export KEY_OU="VPN"
```

Set these to match your relevant details.  While you're in this file, a quick note about keysize.  In this guide I'm using 2048-bit RSA keys, 2048-bit DH params, and 256bit AES as the preferred TLS cipher.  I think this is a decent trade-off between performance and security and that bumping to 4096-bit RSA keys and DH params provides diminishing returns.  If you'd like to do that however, make sure to set `KEYSIZE=4096` in the easy-rsa vars file and make sure your generated and notated 4096 DH params above.


Now we get to do the fun part, which is build our CA, generate our server certificate, and make our dummy CRL

###### Build the CA
```
source vars
./clean-all
./build-ca
```

When prompted, the 'default' values should be what was provided in the vars file.  There should be no need to change any of them most likely.

###### Build the server key
```
./build-key-server server
```

###### Create and Revoke a dummy-client
```
./build-key dummy-client
./revoke-full dummy-client
```

###### Create the symlinks
Now we need to create symlinks to our configuration folder to make sure everything works.

```
ln -s /usr/share/easy-rsa/2.0/keys/ca.crt /etc/openvpn/ca.crt
ln -s /usr/share/easy-rsa/2.0/keys/server.crt /etc/openvpn/server.crt
ln -s /usr/share/easy-rsa/2.0/keys/server.key /etc/openvpn/server.key
ln -s /usr/share/easy-rsa/2.0/keys/crl.pem /etc/openvpn/crl.pem
```

The last thing to do before calling it quits with setting up the OpenVPN server is setting up a logrotate policy and making sure the service starts successfully.

```
systemctl enable openvpn@server
systemctl start openvpn@server
```

The above should also cause the log file at `/var/log/openvpn.log` to be created.

Create a new logrotate policy in `/etc/logrotate.d/openvpn` and put the following in it for the contents:

```
/var/log/openvpn.log {
 missingok
 notifempty
 copytruncate
 compress
 daily
 rotate 7
```

##### 5. Create Your Client Certificates and Configuration

Now back over in your easy-rsa folder, we need to create a client certificate for our first client and then build a configuration for them which includes those certificates.  You can do this for each client in turn.  Replace $CN with what you want the CN of the certificate to be, for instance 'tristor-laptop'

```
source vars
./build-key $CN
```

You can pretty much accept the defaults.  Once you've got the client keys, it's time to build the configuration.

I always create a directory in `/etc/openvpn/client-configs` to hold the client configurations and name them $CN.ovpn, this is optional.

A [stub client config is available here to start with](https://gist.github.com/Tristor/a7222c02ff9365fef9b741a1358d875a).  This was adapted from [a blog post here](https://2kswiki.wordpress.com/2015/06/17/hardened-openvpn-on-centos-7/).

Still from inside your easy-rsa folder, run the following to get the replacement string for the `verify-x509-name` parameter.

```
openssl x509 -in easy-rsa/keys/server.crt -text|grep Subject:|sed 's|/name=|, name=|g;s|/emailAddress=|, emailAddress=|g;s|.*Subject: ||g'
```

Then, simply replace the contents of each certificate/key field (where it has a '...') with the contents of the actual certificates or keys in question.

* CA = Your CA certificate `/etc/openvpn/ca.crt`
* Cert = Your Client Certificate `/usr/share/easy-rsa/2.0/keys/$CN.crt`
* Key = Your Client Key `/usr/share/easy-rsa/2.0/keys/$CN.key`
* TLS-Auth = Your PSK `/etc/openvpn/ta.key`

Of course, make sure you update the `remote` config parameter with the IP address of your VPN server.

##### 6. Distribute Your Client Certificates

The easiest way to distribute these for the system you're configuring the server from and anything that can connect to it is simply to cat out the completed configs and copy/paste from your SSH session into a text editor to save the .ovpn config files.  For getting a configuration onto your phone, using iTunes is much preferred to using email for obvious security reasons, although both methods are technically supported by the client.  I'm not going to cover client-side configuration beyond whats above, but in most cases if you import the config it should be as simple as 'connecting'.


# Further Reading

* [Hardening OpenVPN for Defcon](https://www.agwa.name/blog/post/hardening_openvpn_for_def_con)
* [Hardened OpenVPN on CentOS 7](https://2kswiki.wordpress.com/2015/06/17/hardened-openvpn-on-centos-7/)
* [OpenVPN Community Wiki: Hardening](https://community.openvpn.net/openvpn/wiki/Hardening)


# Outro

I know that many of you were expecting a write-up of configuring OpenVPN on my Ubiquiti EdgeRouter Lite (ER3L).  I decided instead to do it on a DigitalOcean cloud server so that I wouldn't be limited to only my VPN to my home network while traveling.  In this case, OpenVPN is my back-up, but also the one I built out first.  I will be doing a follow-up article about setting up L2TP w/ IPSEC client-VPNs w/ cert-based auth on the Ubiquiti EdgeRouter Lite in the coming weeks, as that will be my primary.  L2TP w/ IPSEC has the advantage of being natively supported on iOS, OS X, and Windows, where OpenVPN requires an additional client.  I can also configure [SideStep](http://chetansurpur.com/projects/sidestep/) to auto-connect to L2TP based VPNs whenever I connect to a public Wi-Fi network.

Hope this helps out somebody though.  Cheers.