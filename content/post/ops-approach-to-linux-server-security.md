+++
date = "2016-03-20T14:48:50-05:00"
tags = ["devops", "security", "privacy", "servers", "tech", "linux"]
title = "The Ops Approach to Linux Server Security"
topics = ["devops", "security", "tech", "linux"]

+++

This post was originally a response to a question I received from a
friend via email, with some additions.  I'm not going to try to get very
in-depth here, this is more of a high-level overview of what you should
be doing to secure a server running Linux.  This is mainly focused on a
business environment where you have multiple users and multiple servers
(and are hopefully using configuration management software).

### 1. Strong Authentication

* Use good, **unique**, root passwords for each server.
  - You might want to use a service like Vault, or software like
    1Password, Keypass, LastPass, etc. to store these to make the unique
    nature less onerous.
* Root SSH is disabled.
* 2FA *and* key-based authentication should both be required.
  - I implement this by using a Yubikey slot configured to output a
    static string which was randomly generated and append this to my
    passwords.  In addition, I enforce key-based auth + password.  See
    example config later in this section for details.
* Users are required to have strong passwords
* Users are required to password protect the private key portion of
   their SSH keypair.
* Users are required to use a password manager and have unique passwords
  for all services.
* [Example SSH Server Hardened
  Config](https://gist.github.com/Tristor/6d589939ee43e8956a94)


### 2. Strong Authorization

* User accounts are centrally managed
  - Chef data_bags
  - LDAP/AD
  - Etc.
* Users are granted access to systems/services based on group
  membership.
  - Follow the principle of least privilege here.
* Your sudoers file (/etc/sudoers) is locked down to a handful of groups
  - You should be mapping specific commands that are allowed per group
    as much as possible.
  - Only the ops team should have unrestricted sudo access.

### 3. Services are users too!

* Use jails for services when possible.
* Always make sure services that start as root drop privileges after
  starting.
* Implement SELinux to [enforce](http://stopdisablingselinux.com/) filesystem and socket access policies via Mandatory Access Controls (MAC)
* Ensure you strictly manage UNIX file mode permissions.
  - For instance if a service only needs to read its config, but not
    write to it.  The configuration should be set to 0400 and possibly
    chattr +i
* Use hardened service configs whenever possible.
  - [Mozilla's guide for Nginx, Apache, and
    HAProxy](https://wiki.mozilla.org/Security/Server_Side_TLS)
  - [Duraconf](https://github.com/ioerror/duraconf)
  - [My SecureMail Site](http://securemail.tristor.ro/#!index.md) (needs
    updating)

### 4. Information Leaks **Are** Security Breaches

* Encrypt all the things
  - Use TLS with the strongest crypto possible for your application
  - Keep abreast of any security vulnerabilities in your SSL libraries
    or implementation
  - If possible, use TLS 1.2, disable all previous versions
  - I currently recommend using only AES256-GCM-SHA384 based
    ciphersuites.
* Encrypt data at rest
  - Use EBS encrypted volumes (if you're on AWS)
  - Use EncFS where prudent and performance isn't impacted
  - If you control the physical hardware, use GELI/LUKS to encrypt
    disks.
* Encrypt all data in-flight as far as possible, end-to-end.  
  - Terminate SSL at the app server, not at the LB.  Or configure the LB
    to re-encrypt with a different certificate pair.
* Set custom banners/service headers.  Don't provide an attacker with
  fingerprinting information about your service(s).
* Security through obscurity is not security, but it can be a layer.
  Don't provide any information to an attacker you can avoid.

### 5. Firewalls are Your Friend, but are not a Panacaea

* Block all *INGRESS* and *EGRESS* traffic that is not strictly
  necessary
* Every device should have a local firewall configured
* Every network should have an edge firewall configured
* Rules can be managed in a semi-automated fashion using configuration
  management to maintain consistency in the behavior of your systems.
* If a service only needs to be accessed from other specific systems,
  disallow access from anywhere else at the firewall.
  - For instance, a DB server should whitelist access to the DB port
    from the app server IPs, but otherwise should deny access.
* If unencrypted and encrypted communications are served on different
  ports for a particular service and its feasible, block unencrypted
communications entirely.
* If a service doesn't need to be available publicly, keep it entirely
  behind your edge firewall and require users to VPN in to gain access
to that service.

### 6. Audits May not be Fun, but Audit Controls are Awesome

* Ultimately some person or group of people have the 'keys to the
  kingdom'.  You should know when they're being used and why.
* Deploy centralized logging
  - The classic is the ELK stack, but I find Logstash to be very
    unstable.  I recommend instead using RSyslog -> Kafka -> Flume ->
    ES + other output options (Hadoop?). 
* Automatic privilege use detection (e.g. ThreatStack)
* Command logging (e.g. auditd, psacct, etc.)
* Destructive actions on a system should trigger alerts
* User access should be gated by an external process

### 7. Mitigate Software Vulnerabilities Before They're Found

* Build [grsecurity](https://grsecurity.net/) patched kernels
  - Enable PaX
  - Enable kASLR
* Build software with [hardened GCC
  flags](https://wiki.gentoo.org/wiki/Hardened/FAQ#Do_I_need_to_pass_any_flags_to_LDFLAGS.2FCFLAGS_in_order_to_turn_on_hardened_building.3F)
where possible.
  - PIE
  - SSP
  - Stack Protection
* Focus on building important packages against more secure libraries if
  available.
  - E.g. use LibreSSL instead of OpenSSL
* Avoid using unsigned packages or public package repositories.
  - Resign packages and provide your own internal repos for vendor
    packages
  - Build non-vendored software as custom vendor packages
  - Avoid pip, gems, npm, etc.
* Build your own package mirrors (relates to above)
  - Katello or SpaceWalk can be a boon here
* Deploy your own package signing key and configure systems to verify
  signatures before installing packages and to reject packages not
  signed by your key. 


That's my general take on things.  Some of that might be a bit overkill
for a single web server or something, but is a pretty general stance
that can apply to a handful of servers or a few thousand with equal
success and gain benefit from scale (the ROI goes up as you scale up).
It's definitely not a complete list, but it should give anyone getting
into deploying more than a few servers an idea of what to think about
when doing so.

Thanks for reading.
