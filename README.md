# pihole_setup

## pi
* enable ssh in preferences
* update pi
```bash
sudo apt update
sudo apt full-upgrade
```

## openvpn

### install openvpn
```bash
wget https://git.io/vpn -O openvpn-install.sh
chmod 755 openvpn-install.sh
./openvpn-install.sh
```

### setup openvpn
```bash
vi /etc/openvpn/server/server.conf
```

comment out to only use vpn for dns lookup
```
#push "redirect-gateway def1 bypass-dhcp"
```

add tun ip
```
push "dhcp-option DNS 10.8.0.1"
```

add logging
```
log /var/log/openvpn.log
verb 3
```

restart openvpn
```bash
systemctl restart openvpn-server@server
```

create more client .opvn files
```bash
./openvpn-install.sh
```

update .ovpn files
```bash
;remote 98.109.40.122 1194
remote thewongguy.ddns.net 1194
```

### unbound
```bash
sudo apt update
sudo apt install unbound
wget -O root.hints https://www.internic.net/domain/named.root
sudo mv root.hints /var/lib/unbound/
```
```bash
vi /etc/unbound/unbound.conf.d/pi-hole.conf
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
    root-hints: "/var/lib/unbound/root.hints"

    # Trust glue only if it is within the server's authority
    harden-glue: yes

    # Require DNSSEC data for trust-anchored zones, if such data is absent, the zone becomes BOGUS
    harden-dnssec-stripped: yes

    # Don't use Capitalization randomization as it known to cause DNSSEC issues sometimes
    # see https://discourse.pi-hole.net/t/unbound-stubby-or-dnscrypt-proxy/9378 for further details
    use-caps-for-id: no

    # Reduce EDNS reassembly buffer size.
    # Suggested by the unbound man page to reduce fragmentation reassembly problems
    edns-buffer-size: 1472

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
```bash
sudo service unbound start
dig pi-hole.net @127.0.0.1 -p 5335
```

### pihole
Choose tun0 as the interface and 10.8.0.1/24 as the IP address
```bash
curl -sSL https://install.pi-hole.net | bash
```

```
http://192.168.1.28/admin/
```

Settings > DNS
* Remove upstream
* Update ```Custom 1 (IPv4)``` with ```127.0.0.1#5335```
* Listen on all interfaces
* Use DNSSEC

Group Mangement > Adlists
Copy green ones from  ```https://firebog.net/```

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
  
### crontab
```bash
* * * * * /root/startup.sh >/dev/null 2>&1
0 0 * * * /usr/local/bin/pihole -g >/dev/null 2>&1
```

startup.sh
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

Restart
```bash
pihole restartdns
```

Update gravity
```bash
pihole -g
```

Backup in /root
```bash
pihole -a -t
```
