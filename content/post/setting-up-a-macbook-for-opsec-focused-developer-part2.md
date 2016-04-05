+++
date = "2016-04-02T16:43:34-05:00"
tags = ["security", "privacy", "osx", "apple", "devops", "tech"]
title = "Setting Up a Macbook for an OpSec Focused Developer - Part 2"
topics = ["security", "tech", "devops", "apple"]

+++

# Introduction

My apologies for the delay in posting part 2.  I encountered a few chicken-and-egg problems in that I wanted to write this update from my new Macbook but needed complete the remainder of the setup in order to have a comfortable and secure environment to do so from.  Without further ado, on to the meat of it.

# Organization

I'm breaking this article up into several parts to both assist me in the
process of writing it and to make it easier to digest.  I'm taking some
steps out of order, but am making an effort to organize them into the
most logical order possible.

* Part 1: [Unboxing, Setup Assistant, iCloud, and System Preferences & Web Browser Configuration (Firefox)](https://tristor.ro/blog/2016/03/23/setting-up-a-macbook-for-an-opsec-focused-developer---part-1/)
* Part 2: [Setting up a Basic Development Environment](https://tristor.ro/blog/2016/04/02/setting-up-a-macbook-for-an-opsec-focused-developer---part-2/)
* Part 3: Additional Applications to Install
* Part 4: Configure Backups and Firewalls.
* Part 5: Wrapping Up

# Part 2
## Security Consciousness Warning - READ THIS

At several points in the following instructions I am going to have you pipe cURL to BASH.  Unfortunately this is commonplace and in many cases is the only supported method to use many projects targeted at developers that are currently online.  For many reasons, this practice is downright stupid.  The ones I'm most concerned with however are the security considerations.  In general you should NEVER run untrusted code on your local system, especially with elevated privileges.  It is for this reason why I manually download and execute the scripts used for installation after reading them.  I recommend you do the same.  You have been warned!

## Install iTerm2

First we're going to download and install [iTerm2](https://iterm2.com/).  While I do think Terminal.App has gotten better lately, iTerm2 is still far more flexible and has some nice features that Terminal.app lacks.  If you prefer to stay as vanilla as possible, you can skip this step and move onto the next one.  Either way, when you're ready open up a terminal as the next few items will be performed on the command-line.

## Install Homebrew

Next we're going to install Homebrew.  Homebrew is a package manager for OS X that provides an easy way to install and keep up to date lots of open source software, much of which you might be familiar with from the land of Linux.  It's absolutely essential for any developer or ops person, as it gives you access to tools which are otherwise painful to impossible to get working, and helps keep your tools up to date to avoid security issues and get access to the latest features.

Without further ado, run:
```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

Then afterwards, you should run `brew doctor` and ensure it returns back the line "Your system is ready to brew."  If you see anything else, it should provide instructions on how to fix the issue.  It's useful to run `brew doctor` any time you encounter issues with Homebrew as it will help diagnose the cause.

Before we proceed you need to add the following "taps", which are essentially additional package repositories for Homebrew.

```bash
brew tap caskroom/cask
brew tap homebrew/completions
brew tap homebrew/core
brew tap homebrew/dupes
brew tap homebrew/nginx
brew tap homebrew/python
brew tap homebrew/services
brew tap homebrew/versions
```

Before you can go too much farther, you'll need to have XCode installed along with the developer tools.  This might have been mentioned above by `brew doctor`.  You can do this by running `xcode-select --install`.

## Install Initial Required Brews

We're going to install several packages.  I'm going to go in a specific order to enable you to chain custom options in dependent packages for the best experience.

First, install `openssl` and `libressl` with `brew install`.  Then

* `curl` with `--with-c-ares --with-libidn --with-libressl --with-nghttp2 --with-libssh2`
* `git` with `--with-blk-sha1 --with-gettext --with-pcre --with-persistent-https --with-brewed-openssl --with-brewed-curl`
	- Note: for /some/ services you may run into certificate issues by doing this.  If you want an easier time of it and aren't as concerned about having the /latest/ SSL library for Git, remove `--with-brewed-curl --with-brewed-openssl` from the above.
* `openssh` with `--with-libressl --without-openssl`
	- Note: using the offical OpenSSH sources via Homebrew will break OSX keychain support, but this will become irrelevant if you use ZSH w/ my fork of Prezto as noted below.
* `python3`
* `vim` with `--override-system-vi --with-lua --with-python3`
* `macvim` with `--with-python3 --with-lua`
* `zsh`
* `bash`
* `bash-completion`
* `git-flow-avh`
* `mercurial`
* `subversion`
* `ssh-copy-id`
* `rsync`
* `wget`
* `archey`
	- Note: is used in the `.zlogin` in my fork of Prezto
* `jq`
* `pigz`
* `p7zip`
* `lbzip2`
* `keybase`

## Create an SSH Keypair

If you don't already have an SSH keypair, now is a good time to make one since you'll be able to add it to your .zpreztorc in the next step.  The most common algorithm used for SSH keypairs is RSA, and you should be using a minimum of a 4096-bit key if you do choose RSA.  A newly supported set of Elliptical Curve algorithms are now available though and I highly recommend using ed25519 keypairs if you can.  RSA has some compatability advantages since its been around longer, and is supported by older versions of OpenSSH, and if you use something like AWS you may find out soon that only RSA keys are supported.  Of course, you're welcome to do what I do and create two keypairs, one which is RSA and one using Elliptical Curve.

In either case, we'll use the tool `ssh-keygen`.

For RSA:  
`ssh-keygen -t rsa -b 4096`

For Elliptical Curve:  
`ssh-keygen -t ed25519`

While you're at it, you should harden your local SSH configuration by editing `~/.ssh/config` with Vim or your favorite installed editor.

This configuration is a good starting place, cribbed from [Mozilla's OpenSSH Security Guidelines](https://wiki.mozilla.org/Security/Guidelines/OpenSSH)

```
# Ensure KnownHosts are unreadable if leaked - it is otherwise easier to know which hosts your keys have access to.
HashKnownHosts yes
# Host keys the client accepts - order here is honored by OpenSSH
HostKeyAlgorithms ssh-ed25519-cert-v01@openssh.com,ssh-rsa-cert-v01@openssh.com,ssh-ed25519,ssh-rsa,ecdsa-sha2-nistp521-cert-v01@openssh.com,ecdsa-sha2-nistp384-cert-v01@openssh.com,ecdsa-sha2-nistp256-cert-v01@openssh.com,ecdsa-sha2-nistp521,ecdsa-sha2-nistp384,ecdsa-sha2-nistp256
# Set the crypto algorithms used. 
KexAlgorithms curve25519-sha256@libssh.org,ecdh-sha2-nistp521,ecdh-sha2-nistp256,ecdh-sha2-nistp384,diffie-hellman-group-exchange-sha256
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,umac-128-etm@openssh.com,hmac-sha2-512,hmac-sha2-256,umac-128@openssh.com
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr
```

## Install Prezto

Prezto is a fork of the popular Oh-My-ZSH, which is massively improved, follows idomatic ZSH scripting styles, and is much much more performant.  I have forked it to fix some OS X specific bugs in several of the plugins, and unfortunately my PRs to the project have been denied so I maintain my own fork for that purpose.  My fork also includes some opinionated configuration which might be helpful getting you started.

* [Official Prezto](https://github.com/sorin-ionescu/prezto)
* [My Fork](https://github.com/Tristor/prezto)

Finally, proceed by following the install instructions as laid out here for my fork:

 1. Launch Zsh:

        zsh

 2. Clone the repository:

        git clone https://github.com/Tristor/prezto.git --recursive  "${ZDOTDIR:-$HOME}/.zprezto"

 3. Create a new Zsh configuration by copying the Zsh configuration files
     provided:

        setopt EXTENDED_GLOB
        for rcfile in "${ZDOTDIR:-$HOME}"/.zprezto/runcoms/^README.md(.N); do
          ln -s "$rcfile" "${ZDOTDIR:-$HOME}/.${rcfile:t}"
        done

 4. Set Zsh as your default shell:

        vim /etc/shells (add /usr/local/bin/zsh)
		chsh -s /usr/local/bin/zsh

 5. Open a new Zsh terminal window or tab.

After the installation, make sure you edit `.zpreztorc` to put your SSH key in the listing for the SSH plugin.  In my fork it has my keys listed, in the upstream it is an empty list.

## Install Janus

You use Vim, right?  If not, about high time you learned.  Either way, you might want a jump start and that's where [Janus](https://github.com/carlhuda/janus) comes in.  It provides a sane set of plugins and tweaks to make using Vim an enjoyable experience without preventing you from customizing to your hearts desire.  To install run:

```
curl -L https://bit.ly/janus-bootstrap | bash
```

## Install Textmate2

We all love Vim, but sometimes you just want more.  Enter [Textmate2](http://macromates.com/download) which is an OS X specific text-editor, with a native UI, and very nice feature set.  You can use Sublime Text if you prefer, but know that Sublime and several other editors started out as attempts to clone Textmate for non-Mac OSes.  Might as well give the original inspiration a try and find out why it helped make OS X a development platform of choice.

## OS X Dev-Focused Tuning

These are adapted from [Mathias Bynens' excellent dotfiles](https://github.com/mathiasbynens/dotfiles).  You can basically copy/paste this into a terminal and type your password for the things which require sudo when prompted.  I've filtered through these and selected the ones I think are relevant in the most general case and do the most benefit for improving security, privacy, and usefulness of the system to a developer.  You are welcome to review Mathias' original suggestions and modify to suit your own needs.  Below is a documented simple script and the same but ready to be copy/pasted into the terminal.

{{< gist d3c699d16f6c1bbeec8f4c9d647a1f24 >}}

## Configure Git

## Languages/Managers

Before you get too much farther along, you should set up a projects folder to work in.  I suggest following the [Go developer conventions](https://golang.org/doc/code.html) because they're pretty sane.  In fact, my opionated configurations from my fork of Prezto assume that you will have a folder in your home directory named `projects` and inside you will have `go` as your `$GOPATH`, which will follow the conventions in the link, and everything else will be git cloned inside of `~/projects`.

### Install RVM/Ruby

Create a `.gemspec` file in your home directory that contains `gem: --no-ri --no-rdoc` in order to prevent generating local documentation and to speed up gem installation.  If you have Internet access and/or install Dash (mentioned in Part 3) you definitely won't need the local docs.

RVM is the "Ruby Version Manager" and helps you make a bit of sense out of the complete and utter mess that is managing Ruby environments.  Good luck Rubyists!

Install RVM w/ latest stable Ruby:
```
gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3
\curl -sSL https://get.rvm.io | bash -s stable --ruby
```

Once this has been completed, I recommend you install the following gems to make your life easier:
```
gem install bundler
gem install pry
gem install capistrano
gem install rake
```

### Install NVM/Node.JS

Love it or hate it, you're probably going to need it.  NVM is "Node Version Manager" and serves a similar purpose to RVM, except for the often mired world of Node.JS.  

Install NVM w/ latest (as of this writing) stable Node.JS:
```
curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.31.0/install.sh | bash
nvm install v5.10.0
nvm alias default v5.10.0
```

I'd make a joke here about left-pad being a useful npm module, but I don't think JS devs deserve anymore ribbing than they already get.

### Install Golang

This one is pretty simple.

```
brew install go
```

Yep, you're done assuming you previously set your $GOPATH and/or you're using my recommend workspace layout and my fork of Prezto.

### Install Haskell/Cabal-Install

This one is a bit trickier, be prepared for some compilation time

Install latest cabal-install and GHC
```
brew install cabal-install
cabal update
cabal install cabal-install
```

### Install Python2.7/Python3

Note the `pip` commands below should instead be `syspip` if you're using my fork of Prezto as I have set an environment variable that prevents pip from running outside a virtualenv for environment cleanliness unless you invoke it with `syspip`

```
brew install python
brew install python3
pip -U pip
pip3 -U pip
pip install virtualenv
```

### Install ChefDK (optional)

If you're an ops person who uses Chef, you need to install the ChefDK, because otherwise its a nightmare getting everything working.

Grab it [on the official ChefDK download page](https://downloads.chef.io/chef-dk/) and follow the instructions there for install.  Good luck.

## Install VirtualBox/Vagrant

Grab and install the latest [Virtualbox and Virtualbox Extensions](https://www.virtualbox.org/wiki/Downloads) and install them as usual for such things.  This will provide you a basic FOSS hypervisor that will back Vagrant, Docker-Machine, and Kitchen-CI (if you're using Chef).  After Virtualbox is installed, install [Vagrant from the official download page](https://www.vagrantup.com/downloads.html).

## Install Docker/Docker-Machine

Replace `$name` below with whatever you want to call your docker-machine instance.  I named mine `build` because I primarily use Docker in build script wrappers to essentially act as a build chroot environment.

```
brew install docker
brew install docker-machine
docker-machine create --driver virtualbox $name
eval $(docker-machine env $name)
```

If you want to automatically have this environment configured when you start up, make sure that your .zshrc has the proper start and eval statements for `docker-machine`.

## Install Veertu (optional)

Veertu is AFAIK the only OS X hypervisor application which is entirely native to OS X and works within App Store sandboxing.  You can [get it on the Mac App Store](https://itunes.apple.com/us/app/veertu-native-virtualization/id1024069033) for free.  It costs $40 if you want to run Windows with it, but it lets you run an unlimited number of Linux VMs for free which might be useful for having a VM-based Linux desktop environment.  I mostly use Chef + Kitchen-CI + Vagrant + Virtualbox for testing, so I rarely use Veertu, but considering its free it doesn't hurt to install.

## Install Programmer Fonts

Visit the [guide to programmer fonts](http://www.lowing.org/fonts/) and pick your favorite or search around for other options.  Install it and configure iTerm2 and your preferred text editor(s) to use it.  That's pretty much the gist.  I use [Inconsolata](https://github.com/google/fonts/tree/master/ofl/inconsolata) because its hinting works very well with the Retina displays.


# Stay tuned for Part 3!

Thanks for reading.