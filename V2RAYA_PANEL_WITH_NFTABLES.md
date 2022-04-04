``` bash
#this is the file of nftables config
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

the bypass is chn list



this is the config of v2ray

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

and you need edit the systemd file of v2raya to this(debian )

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

just enjoy it

