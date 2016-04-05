+++
date = "2016-03-23T22:04:23-05:00"
tags = ["security", "privacy", "osx", "apple", "devops", "tech"]
title = "Setting Up a Macbook for an OpSec Focused Developer - Part 1"
topics = ["security", "tech", "devops", "apple"]

+++


# Introduction

That time has come again, and I have acquired a new Macbook Pro.  In
this case its primarily in preparation for my trip so that I can edit
photos effectively on the go.  It replaces my aged 2011 Macbook Air
(which has served me well).  It seems an opportune time then to write up
my process for setting up a Macbook, and with a particular focus on
security.

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

# Part 1
## Unboxing & Setup Assistant

The first thing you'll be greeted with after powering on your shiny new
Macbook is the Apple Setup Assistant.  This process walks you through
the basics of setting up the system.  Unfortunately, if you're
privacy/security-minded there are some pitfalls here.  My recommendation
is to do the following:

1. Choose your language
2. Choose your region
3. Skip network configuration
4. Skip setting up iCloud
5. Create your user account with a relatively simple password for the
   moment.
6. Once setup completes, before you do anything else, Enable the Apple
   Firewall in System Preferences -> Security/Privacy

## System Preferences & iCloud

1. Open up System Preferences, everything following is within.
2. Under General, Disable Handoff
3. Under Mission Control -> Hot Corners, set a hot corner to put the display to sleep (equivalence to locking)
4. Under Security & Privacy -> General, set it to require a password IMMEDIATELY after sleep begins.
5. Under Spotlight -> Privacy, set any folders you don't want Spotlight to index.  For example, I set my Downloads folder because it tends to be cluttered.
6. Under Display, make sure Airplay Display is set to "Off"
7. Under Trackpad, turn the tracking speed to maximum unless you /love/ slow mice. Disable Force Click and enable Tap to Click unless you prefer otherwise.
8. Under Bluetooth, turn Bluetooth off unless you are using a wireless mouse/trackpad/keyboard or intending to do so.
9. Under Users & Groups, click the lock, enter your password, and then disable the Guest User account.
10. At this time, change your password to something strong.  I recommend a good password combined with the output of a slot on a Yubikey configured with a randomized static string. This effectively provides two-factor authentication for system login and disk decryption (which we will enable momentarily)
11. Under Security & Privacy -> FileVault, enable FileVault and follow the instructions, wait for it to complete before continuing.
12. Under Security & Privacy -> Firewall, enable the Apple Firewall.  Leave this enabled, we will install and configure Murus and Little Snitch later in this guide on top of this for stronger security.
13. Under Network, turn on Wi-Fi if it was not already enabled.  Connect to your preferred wireless network.
14. Still under your Network preferences, click Advanced and check the boxes for requiring administrative authorization to:
  - Create computer to computer (adhoc) networks
  - Change networks
  - Turn Wi-Fi on or off
15. Under iCloud, sign-in to your iCloud account.  Enable the minimum subset of iCloud services you need.  In my case I sync contacts, calendar, and use the iCloud keychain for relationship between my iPhone and Macbook.  I also enable Find My Mac w/ Location Services to help recover the laptop if stolen.

## Web Browser Configuration (Firefox)

1. Open Safari and Browse to [https://mozilla.org/](https://mozilla.org) to download Firefox.
2. Close Safari, install Firefox, and open it setting it as your default browser.
3. Open up the hamburger menu, then customize.  Drag Pocket and Hello out of your menu.
4. Open up [about:config](about:config) and change the following:
  - loop.enabled = false
  - browser.pocket.enabled = false
  - browser.pocket.api = '' (empty string)
  - browser.pocket.oAuthConsumerKey = ''
  - browser.pocket.site = ''
  - browser.pocket.enabledLocales = ''
  - privacy.trackingprotection.enabled = true
  - geo.enabled = false
  - dom.event.clipboardevents.enabled = false
  - browser.send_pings = false
  - webgl.disabled = true
  - dom.battery.enabled = false
5. Make a new entry in about:config
  - noscript.httpsDefWhitelist = false
6. Open a new tab.  Click "Got It", and then click the gear icon and select "Show Blank Page"
7. Open up preferences.  Within that context do the following:
8. General -> When Firefox Starts set "Show a blank page"
9. Search -> Delete every search engine except DuckDuckGo (and possibly Wikipedia/Google if you are okay with that)
10. Search -> Uncheck 'Provide search suggestions'
11. Privacy -> History, Firefox will 'Use custom settings for history', then uncheck all boxes for History except 'Accept cookies from sites'
12. Privacy -> History, set Accept third-party cookies 'Never'
13. Privacy -> Location Bar, uncheck all boxes
14. Security -> Logins, uncheck 'Remember logins for sites'
15. Advanced -> Data Choices, uncheck both boxes
16. Close Preferences
17. Open up Add-ons.  Install the following:
  - uBlock Origin
  - NoScript
  - HTTPS Everywhere
  - Disconnect
  - Decentraleyes
18. Ensure you configure uBlock Origin to use Advanced mode, and enable Web RTC leak protection
19. Otherwise configure the above addons as you desire.  See [Privacy Tools](https://privacytools.io) for more suggestions.


## Set a Boot Selection Password in Open Firmware

Why might you want to do this?  Well it prevents a malicious actor who gets physical access to your device from running a bootable software intended to bypass FileVault2 FDE, clone your disk, or to reinstall the OS so it can bypass Find My Mac.  In any case, your should be able to have control over who can run software on your system, and this gives you a bit more control for minimal to no inconvenience as the password will only be necessary if you boot off something other than your normal startup disk (which is protected via FileVault2 FDE).

1. Shut down your Mac completely.
2. Startup into Recovery Mode. `Command + R` during bootup, pretty much as soon as the screen backlight comes on or when you here the OS X chime.
3. Choose your language
4. Choose the Firmware Password Utility from the Utilities menu at the top.
5. Enter your new password and then again to verify.  **DO NOT FORGET THIS PASSWORD!**  You will be unable to boot off any disk except the startup disk, including performing a recovery, restoration from backup, or reinstall without this password (which is where the security benefit comes from).
6. Click Set Password
7. Choose `Restart` from the Apple menu.

See [Apple's official documentation for using a firmware password]
(https://support.apple.com/en-us/HT204455) for more details if you desire.

# [Part 2 is now available](https://tristor.ro/blog/2016/04/02/setting-up-a-macbook-for-an-opsec-focused-developer---part-2/)

Thanks for reading!
