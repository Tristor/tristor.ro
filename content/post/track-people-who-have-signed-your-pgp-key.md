+++
date = "2016-05-15T18:56:49-05:00"
tags = ["tech", "security", "cryptography"]
title = "How to Track People Who've Signed Your PGP Key in Keybase"
topics = ["tech", "security", "cryptography"]

+++

This is going to be a short article, but I thought this might be useful to someone else.  As many of you may already know, I use a service called [Keybase](https://keybase.io/tristor).  This service provides a number of features:

* Prove ownership of your social media identities via cryptographic cross-verification
* Prove your identify for your PGP key via cross-verificatin to your known social identities
* Prove ownership of your devices cryptographically
* Share encrypted files between your devices seamlessly using KBFS
* Track other Keybase users and encrypt messages to them simply


Of course as should be obvious upon setting up your Keybase account, it has no relation to the existing OpenPGP Web of Trust (WoT), and therefore no easy way to connect the two.

Pardon my absurd levels of BASH jankery, but the following set of commands should provide you an easy list of Keybase usernames to track so that you can follow anyone who has signed your public key.

First create an environment variable named $YOURKEYID containing your OpenPGP KeyID from your email address you used as your uid

```
export YOURKEYID="$(gpg --list-key youremail@example.com | awk 'match($0,/0x[A-F0-9]{16}/) { print substr($0,RSTART,RLENGTH);exit;}')"
```

This only works if you have the public key of each key that's signed yours within your keyring.

If you haven't yet done this, here is a dirty one-liner to import all the public keys which have signed your own key.

```
gpg --list-sigs $YOURKEYID | grep 'ID not found' | perl -nwe '/([0-9A-F]{8})/ && print "$1\n"' | xargs gpg --recv-keys
```

Then we generate a list of keyids that have signed your key  (note on ZSH you must first touch the files you are appending to if they aren't yet created)

```
gpg --list-sigs $YOURKEYID | awk 'match($0,/0x[A-F0-9]{16}/) {print substr($0,RSTART,RLENGTH)}' | sort | uniq >> keyid
```

Then from the list of keyids we retrieve the key fingerprints. 

```
for keyid in $(cat keyid); do gpg --fingerprint $keyid | grep -E '^\s+Key'| sed -e 's/ //g' -e 's/Keyfingerprint=//g' >> fingerprints; done;
```

Next we retrieve the [Keybase](https://keybase.io/) username for each key fingerprint if one exists by using the Keybase API.  Note, there is a way to speed this up by doing it in parallel, but doing so triggers server-side rate limits, so its best to leave this alone so things are done serially.

```
for fp in $(cat fingerprints); do curl https://keybase.io/_/api/1.0/user/discover.json\?key_fingerprint=$fp | jq '.matches.key_fingerprint | .[0][0] | .username' >> kbnames; done;
```

Almost done.  Next we remove any `null` values from the output and limit it to unique usernames by running

```
sed -n '/null/!p' kbnames | sort | uniq > kbnames2
```

Finally we track all the users who have signed our key in order, following the normal process with a simple `for` loop.  This of course expects you to have the `keybase` CLI client installed and logged in already on the device you're running this on.

```
for user in $(cat kbnames2); do keybase track $user; done;
```


Hope this helps somebody else.  I'd package this up into a tidy little BASH script, but it seems like something I won't have to run very often so it wasn't really worth the trouble.  This could probably be done far cleaner in a single Ruby script versus using BASH, but meh.

Cheers.