fuzzlr
======

Easy-ish(?) way to set up a system for proxying netflix traffic to other countries, a-la unblock-us.

Requirements
============
- A VPS or other internet-connected device (dedicated server, raspberry pi, laptop, internet-connected dishwasher) capable of sending and receiving HTTP/TLS traffic on arbitrary ports, listening on arbitrary ports and, most importantly, running `sniproxy`.
- A device of some sort that routes traffic between your network and the internet, which provides DHCP services and allows for software installation, crontasks, simple NAT-based outbound and inbound port translation, and general manipulation of its filesystem and operations.

Assumptions Made
================
As I'm basing this off a setup I did on my own network, some assumptions are made.
- Your internet-connected oscillating fan or lightbulb is running Debian or a Debian-based Linux distribution.
- Your internet gateway device is an OpenWRT router capable of running `unbound` and doing basic packet rerouting via `iptables`' NAT table.
- Your OpenWRT router is running the LuCI web interface or is running `uhttpd` with similar configuration to what LuCI ships with.
- You have a dynamically-assigned WAN IP address because your ISP is a jerk.

The Guide (Don't panic)
=======================
Set-up is pretty easy and, with all the work done already, shouldn't take much time to get finished. It's split into two parts, remote and local, and doesn't have many steps.

Remote
------
For debian 7 or earlier users, you'll need to follow these steps first:
- `ssh` into your server, run `apt-get update` followed by `apt-get upgrade` to make sure your system is up-to-date.
- Run `apt-get install build-essential`. This will install a more or less complete compile environment.
- `mkdir udns` followed by `cd udns`, then `wget http://www.corpit.ru/mjt/udns/udns-0.4.tar.gz && wget http://ftp.de.debian.org/debian/pool/main/u/udns/udns_0.4-1.debian.tar.gz`. Then, `tar xfz udns-0.4.tar.gz && tar xfz udns_0.4-1.debian.tar.gz && mv debian udns-0.4/` and `cd udns-0.4`.
- Run `dpkg-buildpackage` then `cd ..` and run `dpkg -i *.deb` as root.
- `cd ..` and `rm -rf udns`

For ubuntu users, these steps should be followed instead:
- `ssh` into your server, run `apt-get update` followed by `apt-get upgrade` to make sure your system is up-to-date.
- Run `apt-get install build-essential libudns-dev`. This will install a more or less complete compile environment.

From then on, all steps apply to any debian-based distribution:
- Run `apt-get install autotools-dev cdbs debhelper dh-autoreconf dpkg-dev gettext git libev-dev libpcre3-dev pkg-config` followed by `git clone https://github.com/dlundquist/sniproxy.git`, then `cd sniproxy`
- Run `./autogen.sh` followed by `dpkg-buildpackage`, then `cd ..` and run `dpkg -i sniproxy_*.deb` as root.
- Run `rm -rf sniproxy`.
- Download `sniproxy.conf` from this git repository and install it at `/etc/sniproxy.conf` on your server. Then run `$EDITOR /etc/sniproxy.conf` and change whichever values needed to proxy the sites you want (edit what's in the http_table and tls_table blocks) and serve your proxy on the ports you want (change the ports on the listen lines). It is *strongly* recommended you do not use the default ports.
- Run `update-rc.d sniproxy defaults`, then `$EDITOR /etc/default/sniproxy` and set `ENABLED` to 1. Finally, `service sniproxy start`.

Local
-----
- First, run `opkg update` followed by `opkg list-upgradable`, and `opkg upgrade` any packages listed as upgradable.
- Run `opkg install unbound`. `unbound` pulls in openssl as a dependency so make sure your OpenWRT install has enough space free on `/overlay` before running this!
- Download `unbound.conf` from this git repository and install it to `/etc/unbound/unbound.conf`. Then, `vim /etc/unbound/unbound.conf` (replace vim with your editor of choice) and add `private-domain`, `local-zone` and `local-data` directives for all domains you want to proxy (do *not* alter the listener port unless you know what you're doing). It is also *strongly* recommended that you change the `interface` and `access-control` directives from `0.0.0.0` and `0.0.0.0/0` to your router's LAN IP address and your DHCP subnet, respectively. Then `/etc/init.d/unbound enable` and `/etc/init.d/unbound start`.
- Download `firewall.user` from this git repository and install it to `/etc/firewall.user`, then `vim /etc/firewall.user`.
- Change `10.1.2.3/24` to your local DHCP subnet. Change `169.254.0.1` to your remote server's IP address. Change `12345` to the first listener port you set in `sniproxy.conf`. Change `23456` to the second listener port you set.
- Run `/etc/init.d/firewall restart` or restart the firewall service in the LuCI interface.
- Download `dnsredirect-cgi` from this git repository and install it to `/www/cgi-bin/switchdns`. Then `vim /www/cgi-bin/switchdns`, look for the line beginning `[ -z "$(echo "$REMOTE_HOST"|grep` and change `10.1.2.` to your DHCP subnet, sans prefix length or the final octet. Then look for the two lines which start with `IPT_`, and change `10.254.254.254` to your router's LAN IP address.
- Run `mkdir -p /usr/local/bin` and then download `dnsredirect-cron` from this repository and install it to `/usr/local/bin/dnsredirect-cron`. Then `vim /usr/local/bin/dnsredirect-cron`, find the lines starting `UIPTRULE` or `TIPTRULE` and change both instances of `10.254.254.254` to your router's LAN IP address.
- Finally, run `crontab -e` and add the following on a new line: `*/5 * * * * /usr/local/bin/dnsredirect-cron`.

You're done!

Notes/Why Bother?
=================
The set-up process is a tad more complex than other documented solutions, but wins out due to allowing users to enable/disable the proxying on individual hosts at will, with automatic proxy rule expiry. This how-to is, however, written with Debian and OpenWRT in mind, and *will* need altered to work with other LAN setups or operating systems/linux distributions.
The local side of things obviously requires `uhttpd` configured to execute CGI, but may also require certain baseline local storage space and RAM, I'm unsure as I've only tested this on my own network.
If you're lucky enough to have a statically-assigned IP from your ISP, you can gleefully skip installing the `/etc/firewall.user` file and set your `sniproxy` ports to `80` and `443`, dropping the two `http_table` and `tls_table` tables for a single, nameless `table` block, inside which lies a single set of proxyable domains without explicit outbound ports (`:80` or `:443` respectively). Instead of all that noise, a simple `iptables` rule resembling the two following will suffice: `iptables -I INPUT -p tcp ! -s Your.Static.Home.IP --dport 80 -j REJECT` and `iptables -I INPUT -p tcp ! -s Your.Static.Home.IP --dport 443 -j REJECT`.
Note that if your remote server does not accept inbound traffic by default, you'll need to remove the exclamation marks and change `-j REJECT` to `-j ACCEPT`.

Security Notes/Disclaimer
=========================
The scripts and configuration files provided in this git repository are provided as-is, with no guarantee of security. You're expected to know how to prevent external abuse of the systems, and by using this guide/how-to/the files provided, you acknowledge that you are fully liable for anything that may happen (or may not happen).
