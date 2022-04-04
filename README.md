### 1.install v2ray via `bash <(curl -L https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh)`

### 2.install nftables (if possible,remove iptables),and v2raya `https://github.com/v2rayA/v2rayA`

### 3.install `miniupnpd` `dnsmasq` `AdGuardHome` 

### 4.config `AdGuardHome` via `https://github.com/fernvenue/adguardhome-upstream`

### 5.config nftables like this

```bash
#!/usr/sbin/nft -f
flush ruleset
include "/etc/iprules/bypass.nft"
table inet filter {
       chain input {
               type filter hook input priority 0;policy drop;
               ct state {established,related} accept
               iifname lo accept
               iifname enp3s0 accept
               icmp type echo-request accept
               ip saddr 192.168.31.0/24 accept
               tcp dport {ssh,https,http,1080} accept
               udp dport {53} accept
       }
       chain forward {
               type filter hook forward priority 0;policy drop;
               iifname enp3s0 oifname {enp4s0,ppp0} accept
               iifname {enp4s0,ppp0} oifname enp3s0 ct state {related,established} accept
       }
       chain output {
               type filter hook output priority 0;policy accept;
      }
}
table ip nat {
chain prerouting {
    type nat hook prerouting priority 0; policy accept;
    ip daddr $bypassd return
    tcp dport {80,443,22,3389,853} redirect to :1234
  }
chain postrouting {
    type nat hook postrouting priority srcnat;policy accept;
    oifname {enp4s0,ppp0} ip ttl set 64
    oifname {enp4s0,ppp0} masquerade
 }
}
```

**the /etc/iprules/bypass.nft is chn list** 

also remember replace `{enp4s0,ppp0}` to your wan interface name(ppp0 often be the pppoe interface name),and `enp3s0` to LAN interface, `192.168.31.0/24` to your lan ip cidr,`{ssh,https,http,1080}`to the port you want open to wan 

### 6.config dnsmasq

```bash
interface = enp3s0
port = 9053
dhcp-range = 192.168.31.2,192.168.31.254,30m
dhcp-option = option:router,192.168.31.1
dhcp-option = option:dns-server,192.168.31.1
dhcp-leasefile = /var/log/dhcp.leases
```

write it as you like

### 7.config v2ray(yes,you need run v2ray and v2raya in the same time)

``` json
{
  "log": {

  },
  "inbounds": [
    {
      "tag": "transparent",
      "port": 1234,
      "protocol": "dokodemo-door",
      "settings": {
        "network": "tcp,udp",
        "followRedirect": true
      },
      "sniffing": {
        "enabled": true,
        "destOverride": [
          "http",
          "tls"
        ]
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "socks",
      "settings": {
        "servers": [
          {
            "address": "127.0.0.1",
            "port": 20170
          }
        ]
      },
      "tag": "proxy"
    }
  ]
}
```

### 8.replace v2ray systemd unit to prevent change the sysctl configure and iptables

``` 
# /etc/systemd/system/v2raya.service
[Unit]
Description=v2rayA Service
Documentation=https://github.com/v2rayA/v2rayA/wiki
After=network.target nss-lookup.target iptables.service ip6tables.service
Wants=network.target

[Service]
Type=simple
User=root
LimitNPROC=500
LimitNOFILE=1000000
ExecStart=/usr/bin/v2raya --log-disable-timestamp --lite --v2ray-confdir /usr/local/etc/v2ray/v2raya
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### 9. enable ipv4 forward 

### 10.start

```
systemctl enable --now nftables 
systemctl enable --now v2ray
systemctl enable --now v2raya
sysctl -p
```

