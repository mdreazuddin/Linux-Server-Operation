# To achieve high availability with Virtual IP failover between DC and DR servers of SUSE Linux.

### Goal Recap
- VIP: 192.168.50.100
- If DC (192.168.50.50) is healthy, VIP should be assigned here.
- If DC fails, then DR (192.168.50.60) takes over the VIP.
- When DC recovers, it should reclaim the VIP.

### Requirements

- Root privilege on both servers.
- SSH key exchange (optional) for communication.
- ip, ping, and arping must be installed (default on most systems).


## Below Steps will execute for DC and DR both.

**Step 1: Create Common Config File**

/etc/ha_config.conf – create on both servers:

```
ROLE="DC"  # Change to DR on the second server
VIP="192.168.50.100"
INTERFACE="eth0"
PEER_IP="192.168.50.60"  # DR peer (change to 192.168.50.50 for DR)
CHECK_IP="8.8.8.8"        # Or your internal service to check
```

**Step 2: Bash Script – /usr/local/bin/vip_ha.sh**

```
#!/bin/bash

source /etc/ha_config.conf

function has_vip() {
    ip addr show $INTERFACE | grep -q "$VIP"
}

function add_vip() {
    ip addr add $VIP/24 dev $INTERFACE
    arping -q -A -c 3 -I $INTERFACE $VIP
    echo "$(date) - VIP added to $INTERFACE"
}

function remove_vip() {
    ip addr del $VIP/24 dev $INTERFACE
    echo "$(date) - VIP removed from $INTERFACE"
}

function check_health() {
    ping -c 3 -W 1 $CHECK_IP &>/dev/null
}

if [[ "$ROLE" == "DC" ]]; then
    if check_health; then
        if ! has_vip; then
            add_vip
        fi
    else
        if has_vip; then
            remove_vip
        fi
    fi
elif [[ "$ROLE" == "DR" ]]; then
    ping -c 2 -W 1 $PEER_IP &>/dev/null
    if [[ $? -ne 0 ]]; then
        if ! has_vip; then
            add_vip
        fi
    else
        if has_vip; then
            remove_vip
        fi
    fi
fi
```

**Step 3: Make It Executable**

`chmod +x /usr/local/bin/vip_ha.sh`

**Step 4: Setup Cron Job to Run Every 10 Seconds**

`crontab -e`

***Add:***

```
* * * * * /usr/local/bin/vip_ha.sh
* * * * * sleep 10 && /usr/local/bin/vip_ha.sh
* * * * * sleep 20 && /usr/local/bin/vip_ha.sh
* * * * * sleep 30 && /usr/local/bin/vip_ha.sh
* * * * * sleep 40 && /usr/local/bin/vip_ha.sh
* * * * * sleep 50 && /usr/local/bin/vip_ha.sh
```

**Optinal** But recommended
Run the following commands on both servers (DC and DR):

adding this to /etc/sysctl.conf:

```
# Required for Virtual IP ARP behavior
net.ipv4.conf.eth0.arp_ignore = 1
net.ipv4.conf.eth0.arp_announce = 2
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
net.ipv4.conf.eth0.proxy_arp = 1
```

**Apply sysctl changes:**

`sysctl -p`

***What Happens Internally***
- DC checks internet/local health (CHECK_IP).
- If healthy → ensures VIP is bound.
- If unhealthy → removes VIP.
- DR checks DC's health by pinging it.
- If unreachable → DR takes VIP and ARP broadcasts.
- If DC is back → DR drops VIP, DC will reassign.

***Test Cases or Verify***

`ping 192.168.50.100`
`arp -an | grep 192.168.50.100`
`curl http://192.168.50.100`

- Start both servers, DC gets the VIP.
- Shutdown DC → DR gets VIP after few seconds.
- Start DC again → DC gets VIP, DR removes.
