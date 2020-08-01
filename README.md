> ___Set-up instructions to get [Pi-Hole](https://pi-hole.net/), [Recursive local DNS](https://docs.pi-hole.net/guides/unbound/), and a [WireGaurd](https://www.wireguard.com/) running on a Rasberry Pi Zero w___

# Basic Pi-Hole Installation (via headless SSH install)
1. Download [Raspbery Pi OS Lite](https://www.raspberrypi.org/downloads/raspberry-pi-os/)
1. Download [Etcher](https://www.balena.io/etcher/)
1. Flash OS to miroSD with Etcher
1. Enable SSH on boot by creating a blank `ssh` file in the root SSD dir
    ```basg
    cd /Volumes/BOOT
    touch ssh
    ```
1. Enable WIFI on boot
    ```bash
    cd /Volumes/BOOT
    vi wpa_supplicant.conf
    ```
    ```bash
    country=US
    ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
    update_config=1

    network={
        ssid="YOUR-WIFI-SSID"
        psk="YOUR-WIFI-PASSWORD"
    }
    ```
1. Turn on the Rasberry PI and update it
    ```bash
    ssh pi@raspberrypi.local
    sudo apt-get update && sudo apt-get upgrade -y 
    
    # configure local and timezone
    sudo raspi-config raspi-config

    # change the password
    passwd
    ```
1. Install Pi-Hole
    ```bash
    curl -sSL https://install.pi-hole.net | bash
    # Be sure to write down the password at the end
    ```
1. Turn on DHCP on pi-hole
    - Disable both `Enable IPv6 support (SLAAC + RA)` and `Enable DHCP rapid commit (fast address assignment)`
    - Change local domain lookups to `local`
1. Turn off DHCP & DNS on router
1. Add additional block lists for kids devices
    - https://github.com/StevenBlack/hosts < use this one
    - https://firebog.net < maybe for later?
#### Resources
[Setting up Pi-Hole on a Raspberry Pi Zero W using SSH](https://lukesingham.com/pi-hole/)

[GETTING STARTED WITH THE RASPBERRY PI ZERO W WITHOUT A MONITOR](https://www.losant.com/blog/getting-started-with-the-raspberry-pi-zero-w-without-a-monitor?q=%20&hPP=20&idx=production_BLOG&p=0&is_v=1)

# Configure Unbound for local recursive DNS
1. Follow [Pi-hole as All-Around DNS Solution](https://docs.pi-hole.net/guides/unbound/)
1. Customiz PI-Hole to limit disk writes
    ```bash
    sudo nano /etc/pihole/pihole-FTL.conf

    # Make FTL only analyze A and AAAA queries (true or false)
    ANALYZE_ONLY_A_AND_AAAA=true
    # TTL (90-days) entries for FTL database (saves on space)
    MAXDBDAYS=90
    # only write to disk every 15mins
    DBINTERVAL=15
    ...
    sudo service pihole-FTL restart
    ```
1. Use Unbound cache instead of pi-hole's
    ```bash
    sudo nano /etc/dnsmasq.d/01-pihole.conf
    #Make this zero, so that pi-hole always forwards non-blocked lookups
    cache-size=0
    ...
    pihole restartdns
    ```
1. Turn up the unboud caching
    ```bash
    sudo nano /etc/unbound/unbound.conf

    prefetch: yes
    cache-min-ttl: 0
    serve-expired: yes
    msg-cache-size: 128m
    rrset-cache-size: 256m
    ```
1. Get the named list at [01:05 on day-of-month 15 in every 3rd month](https://crontab.guru/#05_01_15_*/3_*)
    ```bash
    sudo crontab -e

    # send an email on completion (if configured below)
    MAILTO=YOUR_EMAIL_ADDRESS
    #Updates Internic servers for unbound
    05 01 15 */3 * wget -O /var/lib/unbound/root.hints https://www.internic.net/domain/named.root
    ...
    service unbound restart
    ```
1. Configure a time sync fall back so that DNSSEC will continue to work after a power outage
    ```bash
    sudo nano /etc/systemd/timesyncd.conf
    FallbackNTP=pool.ntp.org 64.142.54.12 64.22.253.15
    ```
1. ✅**Make sure `Use DNSSEC` is off in pi-hole [DNS settings](http://pi.hole/admin/settings.php?tab=dns)** or you'll be doing an unecessary double DNSEC lookup

7. [Use DNS Perf Test](https://github.com/cleanbrowsing/dnsperftest/) to make sure DNS is fast - should be fast or faster then Google and Cloudflare (~40ms)
    - Initial before/after went from `58.70ms to 41.90ms` after the changes above

#### Resources
[Into the Pi-Hole you should go - 8 months later](https://www.reddit.com/r/pihole/comments/dezyvy/into_the_pihole_you_should_go_8_months_later/)

[Unbound as recursive DNS server - Slow performance](https://www.reddit.com/r/pihole/comments/d9j1z6/unbound_as_recursive_dns_server_slow_performance/)

# Configure Less Disk Writes (saves SSD)
It's a little unlcear based on all the random internet postings whether this is a real concern or not, but based on the number of steps in this doc, i'd just assume not have to do this setup to frequently. This might be overkill, but we have the RAM so why not. 

1. Install [Log2Ram](https://github.com/azlux/log2ram/blob/master/README.md) to limit SSD writes for logs and pi-hole logging 

# Allow for email notifications and auto updates

1. Install [UnattendedUpgrades](https://wiki.debian.org/UnattendedUpgrades) for automatic upgrades
    - these steps are a bit of a gues, check the docs as I instlled a lot of this automtically during pivpn installation

    ```bash
    apt-get install unattended-upgrades apt-listchanges

    # use Vi so you can / search for "MAIL"
    sudo vi /etc/apt/apt.conf.d/50unattended-upgrades

    #UPDATE
    Unattended-Upgrade::Mail "YOUR-EMAIL-ADDRESS";
    
    ..

    sudo nano  /etc/apt/apt.conf.d/20auto-upgrades

    APT::Periodic::Update-Package-Lists "1";
    APT::Periodic::Unattended-Upgrade "1";
   
    ..
    # get apt-listchange notifications
    sudo nano /etc/apt/listchanges.conf

    [apt]
    frontend=pager
    which=news
    email_address=YOUR-EMAIL-ADDRESS
    email_format=text
    confirm=false
    headers=false
    reverse=false
    save_seen=/var/lib/apt/listchanges.db

    ```
1. Set up gmail to allow pi to [send emails](https://medium.com/swlh/setting-up-gmail-and-other-email-on-a-raspberry-pi-6f7e3ad3d0e)
    ```bash
    sudo apt install postfix libsasl2-modules

    # choose 'Internet Site' for type during install
    ```
1. Log into Google Account management: https://myaccount.google.com
1. Select Security
1. App Password
    - App > Mail
    - Device > the name given in the above postfix settings, probably 'rasberrypi'
    - copy the password given
1. Give mail the app password
    ```bash
    sudo nano -B /etc/postfix/sasl/sasl_passwd

    # update name and pass below
    [smtp.gmail.com]:587 username@gmail.com:password
    ..
    
    # protect clear text pass above
    sudo chmod u=rw,go= /etc/postfix/sasl/sasl_passwd
    # give password hash to postfix
    sudo postmap /etc/postfix/sasl/sasl_passwd
    # make sure password db permissions are correct
    sudo chmod u=rw,go= /etc/postfix/sasl/sasl_passwd.db

    #update postfix
    sudo cp /etc/postfix/main.cf !#$.dist
    ```
1. finalize postfix settings
    ```bash
    sudo nano /etc/postfix/main.cf
    
    relayhost = [smtp.gmail.com]:587

    # Enable authentication using SASL.
    smtp_sasl_auth_enable = yes
    # Use transport layer security (TLS) encryption.
    smtp_tls_security_level = encrypt
    # Do not allow anonymous authentication.
    smtp_sasl_security_options = noanonymous
    # Specify where to find the login information.
    smtp_sasl_password_maps = hash:/etc/postfix/sasl/sasl_passwd
    # Where to find the certificate authority (CA) certificates.
    smtp_tls_CAfile = /etc/ssl/certs/ca-certificates.crt

    ..
    sudo systemctl restart postfix
    ```
1. Check that it worked
    ```bash
    echo 'message' | mail -s "It Worked!" username@gmail.com
    ```
1. Need to troubleshoot? [check here](https://medium.com/swlh/setting-up-gmail-and-other-email-on-a-raspberry-pi-6f7e3ad3d0e)

# Configure VPN (WireGaurd)
1. Install [PiVPN](https://www.pivpn.io/)
    ```bash
    curl -L https://install.pivpn.io | bash

    # Add a user
    pivpn add

    # get a quick QR setup code for iPHone config
    pivpn -qr
    ```
1. Forward UDP port `51820` from your router

## ...All done! 🎉