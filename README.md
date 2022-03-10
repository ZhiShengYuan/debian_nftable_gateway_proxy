# debian_nftable_gateway_proxy

### 1.install debian and nftables,uninstall iptables

### 2.install curl and something in need(like unzip)

### 3.exec this code,please replace 192.168.1.0/24 to the lan cidr,enp4s0 to the wan ifname,enp3s0 to lan ifname,and custome the `tcp dport {ssh,https,http} accept` to the port you want allow wan to access

``` bash
#!/bin/bash
# please replace 1234 to the v2ray dokodemo port
mkdir /etc/nftables;
curl 'http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest' > /tmp/raw;
echo "define chnroute_list = {" > /etc/nftables/chnroute.nft;
cat /tmp/raw| grep ipv4 | grep CN | awk -F\| '{ printf("%s/%d\n", $4, 32-log($5)/log(2)) }' | sed s/$/,/g >> /etc/nftables/chnroute.nft;
echo "}" >> /etc/nftables/chnroute.nft;
cat >>/etc/nftables/private.nft<<EOF
define private_list = {
    0.0.0.0/8,
    10.0.0.0/8,
    127.0.0.0/8,
    169.254.0.0/16,
    172.16.0.0/12,
    192.168.0.0/16,
    224.0.0.0/4,
    240.0.0.0/4,
}
EOF
echo  -e "\nnet.ipv4.ip_forward=1\n">>/etc/sysctl.conf&&sysctl -p;
echo '
#!/usr/sbin/nft -f
flush ruleset
include "/etc/nftables/chnroute.nft"
include "/etc/nftables/private.nft"
table inet filter
delete table inet filter
table inet filter {
       chain input {
               type filter hook input priority 0;policy drop;
               ct state established,related accept
               iifname lo accept
               iifname enp3s0 accept
               icmp type echo-request accept
               ip saddr 192.168.1.0/24 accept
               tcp dport {ssh,https,http} accept
       }
       chain forward {
               type filter hook forward priority 0;policy drop;
               iifname enp3s0 oifname enp4s0 accept
               iifname enp4s0 oifname enp3s0 ct state {related,established} accept
       }
       chain output {
               type filter hook output priority 0;
      }
}
table ip ipv4-nat
delete table ip ipv4-nat
table ip ipv4-nat {
    chain postrouting {
        type nat hook postrouting priority srcnat;policy accept;
        oifname {enp4s0} masquerade
    }
}
table ip nat {
  chain proxy {
    ip daddr $private_list return
    ip daddr $chnroute_list return
    ip protocol tcp redirect to :1234
}
chain prerouting {
    type nat hook prerouting priority 0; policy accept;
    goto proxy
  }
}
'>/etc/nftables.conf
systemctl restart nftables.service && systemctl enable nftables.service

```

### 4.config v2ray

``` json
{
    "inbounds": [
        {
      "port": "1234",
      "protocol": "dokodemo-door",
      "settings": {
        "network": "tcp,udp",
        "followRedirect": true
      },
      "sniffing": {
        "enabled": true,
        "destOverride": [ "http", "tls" ]
      },
      "tag": "dkd"
    }
        ]
}
```
