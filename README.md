# global transparent proxy by nftables with v2raya as wen panel

this essay is the guide to config global transparent proxy in Debian

### the software we need

`nftables` `v2ray` `v2raya` `AdGuardHome` ,if you need use pppoe to access internet,you may install `pppoeconf`,all command below should run as root

install necessary package

```bash
apt update -y&&apt install wget curl -y
```

1.install nftables

``` bash
apt install nftables -y
```

2.install v2ray via official install script 

```bash
bash <(curl -L https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh)
```

3.install v2raya

please refer this link `https://v2raya.org/docs/prologue/installation/debian/`,but don't install v2ray kernel,only install v2raya

4.edit v2raya systemd unit to prevent v2raya change iptables and sysctl

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

5.enable ipv4 forward and reload systemd

```bash
echo 'net.ipv4.ip_forward = 1' >>/etc/sysctl.conf &&sysctl -p&&systemctl daemon-reload
```

6.config nftables

```
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
               ip saddr 192.168.2.1/24 accept
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
    oifname {enp4s0,ppp0} masquerade
 }
}
```

in this configure file,the `{enp4s0,ppp0}` should replace by the WAN interface name in your device,and `enp3s0` to the LAN interface, `192.168.2.1/24` to the network of your LAN,`{ssh,https,http,1080}` to the port you want to open to WAN

please attention that this configure file only proxy often use port,if you want proxy all port,just replace `tcp dport {80,443,22,3389,853} redirect to :1234` to `ip protocol tcp redirect to :1234`

the UDP traffic won't be proxy(in my option,the best way to relay UDP is L2 VPN instead this,there is lots of problems of v2ray UDP relay)

the `/etc/iprules/bypass.nft` is the IP list you want to bypass

the format of this file like this

```
define bypassd = {
10.0.0.0/8,
172.16.0.0/12,
192.168.0.0/16,
}
```

in most scene,this list should be chn ip list and RFC1918 defined address

if this device needn't offer Nat server ,just delete `oifname {enp4s0,ppp0} masquerade`

7.config v2ray

just replace `/usr/local/etc/v2ray/config.json` to below 

```json
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

8.install and config `AdGuardHome`

If you needn't DNS server and DHCP server,please ignore it

please refer this to install https://github.com/AdguardTeam/AdGuardHome

or just run this script `curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v`

please refer this repo to config

https://github.com/fernvenue/adguardhome-upstream

and enable DHCP server in `AdGuardHome` panel

you can refer this 

9.start and enjoy

```
systemctl enable --now v2ray
systemctl enable --now nftables
systemctl enable --now v2raya
```

and now you can access http://SERVERADDR:2017 to access v2raya panel to add node

