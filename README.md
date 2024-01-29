# pihole_setup

## pi

### change password & enable ssh
```bash
sudo raspi-config
```

### setup static ip in your router for pi

### port forward 51820 to pi for wireguard

### port forward 32400 to pi for plex

### update pi
```bash
sudo apt update
sudo apt -y full-upgrade
```

## unbound

### unbound install

https://docs.pi-hole.net/guides/dns/unbound/#setting-up-pi-hole-as-a-recursive-dns-server-solution

```bash
sudo apt update
sudo apt -y install unbound dnsutils
sudo wget https://www.internic.net/domain/named.root -qO- | sudo tee /var/lib/unbound/root.hints
sudo vi /etc/unbound/unbound.conf.d/pi-hole.conf
```

### pi-hole.conf

```
sudo vi /etc/unbound/unbound.conf.d/pi-hole.conf
```

```
server:
    # If no logfile is specified, syslog is used
    # logfile: "/var/log/unbound/unbound.log"
    verbosity: 0

    interface: 127.0.0.1
    port: 5335
    do-ip4: yes
    do-udp: yes
    do-tcp: yes

    # May be set to yes if you have IPv6 connectivity
    do-ip6: no

    # You want to leave this to no unless you have *native* IPv6. With 6to4 and
    # Terredo tunnels your web browser should favor IPv4 for the same reasons
    prefer-ip6: no

    # Use this only when you downloaded the list of primary root servers!
    # If you use the default dns-root-data package, unbound will find it automatically
    #root-hints: "/var/lib/unbound/root.hints"

    # Trust glue only if it is within the server's authority
    harden-glue: yes

    # Require DNSSEC data for trust-anchored zones, if such data is absent, the zone becomes BOGUS
    harden-dnssec-stripped: yes

    # Don't use Capitalization randomization as it known to cause DNSSEC issues sometimes
    # see https://discourse.pi-hole.net/t/unbound-stubby-or-dnscrypt-proxy/9378 for further details
    use-caps-for-id: no

    # Reduce EDNS reassembly buffer size.
    # IP fragmentation is unreliable on the Internet today, and can cause
    # transmission failures when large DNS messages are sent via UDP. Even
    # when fragmentation does work, it may not be secure; it is theoretically
    # possible to spoof parts of a fragmented DNS message, without easy
    # detection at the receiving end. Recently, there was an excellent study
    # >>> Defragmenting DNS - Determining the optimal maximum UDP response size for DNS <<<
    # by Axel Koolhaas, and Tjeerd Slokker (https://indico.dns-oarc.net/event/36/contributions/776/)
    # in collaboration with NLnet Labs explored DNS using real world data from the
    # the RIPE Atlas probes and the researchers suggested different values for
    # IPv4 and IPv6 and in different scenarios. They advise that servers should
    # be configured to limit DNS messages sent over UDP to a size that will not
    # trigger fragmentation on typical network links. DNS servers can switch
    # from UDP to TCP when a DNS response is too big to fit in this limited
    # buffer size. This value has also been suggested in DNS Flag Day 2020.
    edns-buffer-size: 1232

    # Perform prefetching of close to expired message cache entries
    # This only applies to domains that have been frequently queried
    prefetch: yes

    # One thread should be sufficient, can be increased on beefy machines. In reality for most users running on small networks or on a single machine, it should be unnecessary to seek performance enhancement by increasing num-threads above 1.
    num-threads: 1

    # Ensure kernel buffer is large enough to not lose messages in traffic spikes
    so-rcvbuf: 1m

    # Ensure privacy of local IP ranges
    private-address: 192.168.0.0/16
    private-address: 169.254.0.0/16
    private-address: 172.16.0.0/12
    private-address: 10.0.0.0/8
    private-address: fd00::/8
    private-address: fe80::/10
```

```
sudo service unbound restart
dig pi-hole.net @127.0.0.1 -p 5335
dig fail01.dnssec.works @127.0.0.1 -p 5335
dig dnssec.works @127.0.0.1 -p 5335
```

```
systemctl is-active unbound-resolvconf.service
sudo systemctl disable --now unbound-resolvconf.service
sudo sed -Ei 's/^unbound_conf=/#unbound_conf=/' /etc/resolvconf.conf
sudo rm /etc/unbound/unbound.conf.d/resolvconf_resolvers.conf
sudo service unbound restart
```

## pihole

### pihole install
Choose eth0 as the interface and 192.168.x.x as the IP address and Google DNS as upstream
```bash
curl -sSL https://install.pi-hole.net | bash
```

### reset password
```bash
pihole -a -p
```

### configure
```
http://<pihole ip>/admin/
```

Settings > DNS
* Remove upstream
* Update ```Custom 1 (IPv4)``` with ```127.0.0.1#5335```

Group Mangement > Adlists
Copy green ones from https://firebog.net/

YouTube block list
https://raw.githubusercontent.com/kboghdady/youTube_ads_4_pi-hole/master/black.list

Blacklist > RegEx filter
From https://raw.githubusercontent.com/mmotti/pihole-regex/master/regex.list
```
^ad([sxv]?[0-9]*|system)[_.-]([^.[:space:]]+\.){1,}|^.+[_.-]ad([sxv]?[0-9]*|system)[_.-]
^(.+[_.-])?adse?rv(er?|ice)?s?[0-9]*[_.-]
^(.+[_.-])?telemetry[_.-]
^adim(age|g)s?[0-9]*[_.-]
^adtrack(er|ing)?[0-9]*[_.-]
^advert(s|is(ing|ements?))?[0-9]*[_.-]
^aff(iliat(es?|ion))?[_.-]
^analytics?[_.-]
^banners?[_.-]
^beacons?[0-9]*[_.-]
^count(ers?)?[0-9]*[_.-]
^mads\.
^pixels?[-.]
^stat(s|istics)?[0-9]*[_.-]
^track(ing)?[0-9]*[_.-]
```

Whitelist
```
thepiratebay.org
mobile.pipe.aria.microsoft.com
link.patch.com
syndication.twitter.com
costco.com
```

### update gravity
```bash
pihole -g
```

### backup in home directory
```bash
pihole -a -t
```

## wireguard with pivpn

### wireguard install
```bash
curl -L https://install.pivpn.io | bash
```

### hotfix

https://github.com/pivpn/pivpn/issues/920#issuecomment-578651638

```
sudo -s
source /usr/src/wireguard-*/dkms.conf
dkms uninstall wireguard/$PACKAGE_VERSION
dkms remove wireguard/$PACKAGE_VERSION
dkms add wireguard/$PACKAGE_VERSION
dkms build wireguard/$PACKAGE_VERSION
dkms install wireguard/$PACKAGE_VERSION
exit
pivpn debug
```

### add clients
```
# For full tunnel use 0.0.0.0/0, ::/0 and for split tunnel use 192.168.1.0/24
AllowedIPs = 10.6.0.1/32, 192.168.1.0/24
```
```bash
pivpn add
```

### status
```bash
systemctl status wg-quick@wg0

systemctl status unbound

pihole status
```

### backup and transfer clients
```bash
scp pi-user@ip-of-your-raspberry:configs/whatever.conf
```

## crontab
```bash
0 0 * * * /usr/local/bin/pihole -g >/dev/null 2>&1
```

## update router's DNS to pihole's ip address

if you need 2 ips, and do not have 2 piholes, use ethernet and wireless or junk ip.

## install wireguard clients
https://play.google.com/store/apps/details?id=com.wireguard.android&hl=en_US

https://apps.apple.com/us/app/wireguard/id1441195209

## restart pihole dns after reboot

```bash
pihole restartdns
pihole arpflush
```

## references
* https://www.sethenoka.com/build-your-own-wireguard-vpn-server-with-pi-hole-for-dns-level-ad-blocking/
* https://github.com/adrianmihalko/raspberrypiwireguard
* https://engineerworkshop.com/blog/how-to-set-up-wireguard-on-a-raspberry-pi/
* https://www.sigmdel.ca/michel/ha/wireguard/wireguard_02_en.html
* https://www.linode.com/community/questions/19346/wireguard-one-click-app-suddenly-does-not-work-rtnetlink-answers
* https://github.com/notasausage/pi-hole-unbound-wireguard
* https://davidshomelab.com/access-your-home-network-from-anywhere-with-wireguard-vpn/
* https://pimylifeup.com/raspberry-pi-plex-server/
* https://www.synology.com/en-uk/knowledgebase/DSM/tutorial/File_Sharing/How_to_access_files_on_Synology_NAS_within_the_local_network_NFS
