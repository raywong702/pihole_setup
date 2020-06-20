# pihole_setup

## pi
* enable ssh in preferences
* update pi
```bash
sudo apt update
sudo apt full-upgrade
```

## wireguard

### install
```bash
sudo apt-get install raspberrypi-kernel-headers
echo "deb http://deb.debian.org/debian/ unstable main" | sudo tee --append /etc/apt/sources.list.d/unstable.list
sudo apt-get install dirmngr 
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 8B48AD6246925553
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 7638D0442B90D010
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 04EE7237B7D453EC
printf 'Package: *\nPin: release a=unstable\nPin-Priority: 150\n' | sudo tee --append /etc/apt/preferences.d/limit-unstable
sudo apt-get update
sudo apt-get install wireguard
```

### ipv4
```bash
sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/g' /etc/sysctl.conf
sudo sysctl -p
sudo echo 1 > /proc/sys/net/ipv4/ip_forward
sysctl net.ipv4.ip_forward
sudo reboot
```

### resolvconf
```bash
sudo apt install resolvconf -y
```

### server setup
```bash
mkdir wireguard
cd wireguard
umask 077
wg genkey | tee server_private_key | wg pubkey > server_public_key
wg genkey | tee s9_private_key | wg pubkey > s9_public_key
wg genkey | tee macbookair_private_key | wg pubkey > macbookair_public_key
sudo touch /etc/wireguard/wg0.conf
sudo chown -v root:root /etc/wireguard/wg0.conf
sudo chmod -v 600 /etc/wireguard/wg0.conf
sudo vi /etc/wireguard/wg0.conf
```

### wg0.conf
```
[Interface]
PrivateKey = <server private key>
Address = 192.168.99.1/24
ListenPort = 51900
SaveConfig = true

# Replace eth0 with the interface open to the internet (e.g might be wlan0 if wifi)
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o ens3 -j MASQUERADE; ip6tables -A FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -A POSTROUTING -o ens3 -j MASQUERADE

PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o ens3 -j MASQUERADE; ip6tables -D FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -D POSTROUTING -o ens3 -j MASQUERADE

[Peer]
# s9
PublicKey = <s9 client public key>
AllowedIPs = 192.168.99.2/32, 192.168.1.0/24

[Peer]
# macbookair
PublicKey = <macbookair client public key>
AllowedIPs = 192.168.99.3/32, 192.168.1.0/24
```

### s9.conf
```
[Interface]
PrivateKey = <s9 client private key>
Address = 192.168.99.2/32
DNS = 192.168.99.1

[Peer]
PublicKey = <server public key>
Endpoint = <your.publicdns.com>:51900
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
```

### macbookair.conf
```
[Interface]
PrivateKey = <macbookair client private key>
Address = 192.168.99.3/32
DNS = 192.168.99.1

[Peer]
PublicKey = <server public key>
Endpoint = <your.publicdns.com>:51900
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
```

### start wireguard
```bash
sudo modprobe wireguard
sudo wg-quick up wg0
sudo wg
sudo systemctl enable wg-quick@wg0
sudo systemctl status wg-quick@wg0
ifconfig wg0
```

### firewall
```
placeholder
```

## pihole install
Choose wg0 as the interface and 192.168.99.1/24 as the IP address and Google DNS as upstream
```bash
curl -sSL https://install.pi-hole.net | bash
```

## unbound

### unbound install
```bash
sudo apt update
sudo apt install unbound
curl -o /var/lib/unbound/root.hints https://www.internic.net/domain/named.root
vi /etc/unbound/unbound.conf.d/pi-hole.conf
```

### pi-hole.conf
```
server:
     # if no logfile is specified, syslog is used
     # logfile: "/var/log/unbound/unbound.log"
     verbosity: 1
     port: 5353

     do-ip4: yes
     do-udp: yes
     do-tcp: yes

     # may be set to yes if you have IPv6 connectivity
     do-ip6: no

     # use this only when you downloaded the list of primary root servers
     root-hints: "/var/lib/unbound/root.hints"

     # respond to DNS requests on all interfaces
     interface: 0.0.0.0
     max-udp-size: 3072

     # IPs authorised to access the DNS Server
     access-control: 0.0.0.0/0                 refuse
     access-control: 127.0.0.1                 allow
     access-control: 192.168.99.0/24           allow

     # hide DNS Server info
     hide-identity: yes
     hide-version: yes

     # limit DNS fraud and use DNSSEC
     harden-glue: yes
     harden-dnssec-stripped: yes
     harden-referral-path: yes

     # add an unwanted reply threshold to clean the cache and avoid, when possible, DNS poisoning
     unwanted-reply-threshold: 10000000

     # have the validator print validation failures to the log val-log-level: 1
     # don't use Capitalisation randomisation as it known to cause DNSSEC issues sometimes
     # see https://discourse.pi-hole.net/t/unbound-stubby-or-dnscrypt-proxy/9378 for further details
     use-caps-for-id: no

     # reduce EDNS reassembly buffer size
     # suggested by the unbound man page to reduce fragmentation reassembly problems
     edns-buffer-size: 1472

     # TTL bounds for cache
     cache-min-ttl: 3600
     cache-max-ttl: 86400

     # perform prefetching of close to expired message cache entries
     # this only applies to domains that have been frequently queried
     prefetch: yes
     prefetch-key: yes
     # one thread should be sufficient, can be increased on beefy machines
     num-threads: 1
     # ensure kernel buffer is large enough to not lose messages in traffic spikes
     so-rcvbuf: 1m

     # ensure privacy of local IP ranges
     private-address: 192.168.0.0/16
     private-address: 169.254.0.0/16
     private-address: 172.16.0.0/12
     private-address: 10.0.0.0/8
     private-address: fd00::/8
     private-address: fe80::/10
```

```bash
sudo service unbound start
dig pi-hole.net @127.0.0.1 -p 5353
dig sigfail.verteiltesysteme.net @127.0.0.1 -p 5353
```

## pihole setup
```
http://<pihole ip>/admin/
```

Settings > DNS
* Remove upstream
* Update ```Custom 1 (IPv4)``` with ```127.0.0.1#5353```
* Listen on all interfaces? or wg0?
* Use DNSSEC

Group Mangement > Adlists
Copy green ones from  ```https://firebog.net/```

YouTube block list
https://raw.githubusercontent.com/kboghdady/youTube_ads_4_pi-hole/master/black.list

Blacklist > RegEx filter
From ```https://raw.githubusercontent.com/mmotti/pihole-regex/master/regex.list```
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

Local DNS Records
Domain: pi.hole
IP Address: <your ip>
  
## crontab
```bash
* * * * * /root/startup.sh >/dev/null 2>&1
0 0 * * * /usr/local/bin/pihole -g >/dev/null 2>&1
```

### startup.sh
```bash
#!/bin/bash
#set -x

source /etc/profile
CHECK=/root/.check
UPTIME=$(/usr/bin/uptime | /usr/bin/perl -lne 'm/up (.*) min/ && print $1')
if [ ! -f "${CHECK}" ]; then
    if [ ! -z "${UPTIME}" ] && (( "${UPTIME}" <= 2 )) && (( "${UPTIME}" > 0 )); then
        touch "${CHECK}"
        /usr/local/bin/pihole restartdns
    fi
else
    if [ ! -z "${UPTIME}" ] && (( "${UPTIME}" > 2 )); then
        rm "${CHECK}"
    fi
fi

exit 0
```

restart
```bash
pihole restartdns
```

update gravity
```bash
pihole -g
```

backup in /root
```bash
pihole -a -t
```

## references
* https://www.sethenoka.com/build-your-own-wireguard-vpn-server-with-pi-hole-for-dns-level-ad-blocking/
* https://github.com/adrianmihalko/raspberrypiwireguard
* https://engineerworkshop.com/blog/how-to-set-up-wireguard-on-a-raspberry-pi/
* https://www.sigmdel.ca/michel/ha/wireguard/wireguard_02_en.html
* https://www.linode.com/community/questions/19346/wireguard-one-click-app-suddenly-does-not-work-rtnetlink-answers
