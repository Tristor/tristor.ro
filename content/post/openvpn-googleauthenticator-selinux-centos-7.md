+++
date = "2016-04-21T17:51:05-05:00"
tags = ["tech", "linux", "devops", "security", "selinux", "vpn"]
title = "OpenVPN + Google Authenticator + SELinux on CentOS 7"
slug = "openvpn-google-authenticator-selinux-on-centos-7"
topics = ["tech", "linux", "devops", "security"]

+++

Just a quick post to share this with anyone else that needs it.  I spent hours using Google and reading posts from random people on the net, including bug comments from Dan Walsh on a never solved Fedora bug specifically related to this.  The conclusion I came to was that hardly anyone uses SELinux and the ones that do just hack around the problem rather than solving it.

In this particular case, the fault is really with the terrible implementation of Google Authenticator, which I found out during the course of this by reading through the source code.  Long story short, it creates a new file named `$HOME/.google_authenticator~` and renames it to `$HOME/.google_authenticator`.  This of course plays havoc with SELinux.

There's several folks who've written decent instructions about dealing with this for SSH (long story short, put .google_authenticator inside of .ssh/).  Nobody really covered the OpenVPN case though.  Well, below all your questions will be answered, and all your hopes and dreams will be fulfilled.

openvpn-googleauthenticator.te
```
policy_module(openvpn-googleauthenticator, 0.0.1)

require {
    type auth_home_t;
    type openvpn_t;
    type user_home_dir_t;
    class dir { search getattr open };
    class file { rename write getattr read create unlink open };
}

#================ openvpn_t ===========
allow openvpn_t auth_home_t:file { write getattr read create unlink open };
allow openvpn_t user_home_dir_t:dir { write remove_name add_name };
allow openvpn_t user_home_dir_t:file { rename write getattr read create unlink open };
#========= because PAM is dumb ===========
filetrans_pattern(openvpn_t, user_home_dir_t, auth_home_t, file, ".google_authenticator")
```

Use the above SELinux policy module wisely and good luck to you.