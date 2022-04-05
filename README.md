## Please ensure that you have got root privilege to do the following steps.

### 1. Install v2ray using this script:
```bash
bash <(curl -L https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh)
```

### 2. Install nftables (if possible,remove iptables)
Debian/Ubuntu:
```bash
apt install nftables -y
```
CentOS:
```bash
yum install nftables -y
```

### 3. Install v2raya from `https://github.com/v2rayA/v2rayA`

### 4. Install `miniupnpd` `dnsmasq` `AdGuardHome` 

### 5. Config `AdGuardHome` (Example: `https://github.com/fernvenue/adguardhome-upstream`)

## An example of nftables config.

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

Please remember to replace `{enp4s0,ppp0}` to your own WAN interface name(ppp0 for example, which is the default interface name created by pppoeconf), `enp3s0` to your LAN interface, `192.168.31.0/24` to your LAN IP CIDR, and`{ssh,https,http,1080}`to the port you want to enable on WAN.

### 6. Example of dnsmasq config

```bash
interface = enp3s0
port = 9053
dhcp-range = 192.168.31.2,192.168.31.254,30m
dhcp-option = option:router,192.168.31.1
dhcp-option = option:dns-server,192.168.31.1
dhcp-leasefile = /var/log/dhcp.leases
```

### 7.Example of v2ray config. (v2ray and v2raya are enabled simultaneously)

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

### 8. Change the service object of v2raya to prevent it from breaking the sysctl configure and iptables.

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
ExecStart=/usr/bin/v2raya --log-disable-timestamp --lite
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### 9. Enable ipv4 forward in your kernel
```bash
net.ipv4.ip_forward = 1
```

### 10.start

```
systemctl daemon-reload
systemctl enable --now nftables 
systemctl enable --now v2ray
systemctl enable --now v2raya
sysctl -p
```

