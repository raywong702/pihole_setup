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
```bash
sudo apt update
sudo apt -y install unbound dnsutils
sudo curl -o /var/lib/unbound/root.hints https://www.internic.net/domain/named.root
sudo vi /etc/unbound/unbound.conf.d/pi-hole.conf
```

### pi-hole.conf

update 192.168.x in pi-hole.conf

https://github.com/notasausage/pi-hole-unbound-wireguard/blob/master/pi-hole.conf

```bash
sudo service unbound start
dig pi-hole.net @127.0.0.1 -p 5353
dig sigfail.verteiltesysteme.net @127.0.0.1 -p 5353
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
* Update ```Custom 1 (IPv4)``` with ```127.0.0.1#5353```

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
lcprd1.samsungcloudsolution.net
```

### update gravity
```bash
pihole -g
```

### backup in /root
```bash
pihole -a -t
```

## wireguard with pivpn

### wireguard install
```bash
curl -L https://install.pivpn.io | bash
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
