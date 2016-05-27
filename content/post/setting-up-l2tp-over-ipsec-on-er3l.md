+++
draft = true
date = "2016-05-22T21:30:09-05:00"
tags = ["travel", "tech", "security", "devops", "privacy", "linux", "ubiquiti", "vpn", "vyatta"]
title = "Setting Up L2TP over IPSEC VPN on Ubiquiti EdgeRouter Lite (ER3L)"
topics = ["security", "tech", "linux", "travel", "ubiquiti"]

+++

As I mentioned previously I'd be writing a guide about setting up L2TP over IPSEC for client VPN on an [Ubiquiti EdgeRouter Lite (ER3L)](https://www.ubnt.com/edgemax/edgerouter-lite/).  The process is pretty straight-forward, but I did run into one gotcha that wasn't covered in the [either of the](https://help.ubnt.com/hc/en-us/articles/204959404-EdgeMAX-Set-up-L2TP-over-IPsec-VPN-server) [official guides](https://help.ubnt.com/hc/en-us/articles/204950294-EdgeMAX-L2TP-Server).  It had to do with the IKE and ESP key exchange methods that were set in the Vyatta scripts.  This guide was written with version 

## Gotcha Fix

In `/opt/vyatta/share/perl5/Vyatta` there is a file named `L2TPConfig.pm`.  Near line 426 in this file you'll find the templated connection block.  You need to set the following for the `ike=` and `esp=` strings.

`ike=aes256-sha512-modp4096,aes256-sha512-modp2048,aes256-sha256-modp4096,aes256-sha256-modp2048,aes256-sha1-modp2048,aes256-sha1!`

`esp=aes256gcm16-ecp384,aes128gcm16-ecp256,aes256-sha384-modp4096,aes256-sha384-modp2048,aes256-sha256-modp4096,aes256-sha256-modp2048,aes128-sha384-modp4096,aes128-sha384-modp2048,aes128-sha256-modp4096,aes128-sha256-modp2048,aes256gcm16,aes128gcm16,aes256-sha384,aes256-sha256,aes256-sha1,aes128-sha384,aes128-sha256,aes128-sha1!`

By changing these you are disabling 3DES and enabling several modes for ESP and IKE for AES256 and AES128, preference ordered to use the most secure modes first.  The SHA1 modes are still left in place purely to support Windows clients, but if you are not using Windows clients you can omit them and only use the SHA2 (sha256,sha384,sha512) modes.

This is required for any modern version of OS X, iOS, or Windows to connect, otherwise it won't work and your logs will be full of messages like:

```
May 23 02:13:32 nullroute pluto[5963]: "remote-access-mac-zzz"[2] xx.xx.xx.xx #2: Oakley Transform [AES_CBC (256), HMAC_SHA2_256, MODP_2048] refused due to strict flag
May 23 02:13:32 nullroute pluto[5963]: "remote-access-mac-zzz"[2] xx.xx.xx.xx #2: Oakley Transform [AES_CBC (256), HMAC_SHA1, MODP_2048] refused due to strict flag
May 23 02:13:32 nullroute pluto[5963]: "remote-access-mac-zzz"[2] xx.xx.xx.xx #2: Oakley Transform [AES_CBC (256), HMAC_MD5, MODP_2048] refused due to strict flag
May 23 02:13:32 nullroute pluto[5963]: "remote-access-mac-zzz"[2] xx.xx.xx.xx #2: Oakley Transform [AES_CBC (256), HMAC_SHA2_512, MODP_2048] refused due to strict flag
May 23 02:13:32 nullroute pluto[5963]: "remote-access-mac-zzz"[2] xx.xx.xx.xx #2: Oakley Transform [AES_CBC (256), HMAC_SHA2_256, MODP_1536] refused due to strict flag
```

Thanks to [hceuterpe1 in the forums](https://community.ubnt.com/t5/EdgeMAX/L2TP-IPSec-works-from-iPhone-but-not-Windows-8/td-p/501453) for guiding me along the right path.

## VPN Config Steps

This is pretty much identical to what's in the official guide, but I'll reproduce it here for your convenience.

1. SSH to your ER3L
2. Run `configure`
3. Set up IPSec (replace eth1 with your WAN interface)

```
set vpn ipsec ipsec-interfaces interface eth1

```