# Global transparent proxy by nftables and v2raya (A web control panel used for switching v2ray nodes)

## This guide will help you to config global transparent proxy on Debian 11.

## Other distributions are NOT tested and have NO guarantees.

## To begin with, you should ensure that you have got root access in order to do the following steps.

### 1. Install required softwares

#### 0. Install necessary packages

Debian:

```bash
apt update -y&&apt install wget curl nftables -y
```

If you need to use pppoe, install `pppoeconf`

```bash
apt install pppoeconf -y
```

Remove iptables by this command:

```bash
apt purge iptables* -y
```

#### 1. Install v2ray via official install script 

```bash
bash <(curl -L https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh)
```

#### 2. Install v2raya

You may read this guide `https://v2raya.org/docs/prologue/installation/debian/` to install v2raya, but DO NOT install v2ray kernel. Only v2raya is required.

#### 3. Edit the systemd unit of v2raya in order to prevent v2raya from breaking iptables and sysctl.

```bash
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

#### 4. Enable ipv4 forward in kernel and reload systemd

```bash
echo 'net.ipv4.ip_forward = 1' >>/etc/sysctl.conf &&sysctl -p&&systemctl daemon-reload
```

#### 5. An example of nftables config.

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
    tcp dport {80,443,22,3389,853,8080} redirect to :1234
  }
chain postrouting {
    type nat hook postrouting priority srcnat;policy accept;
    oifname {enp4s0,ppp0} masquerade
 }
}
```

For this configuration, the `{enp4s0,ppp0}` object should be replaced by the name of the WAN interface on your device, the `enp3s0` object should be replaced by the name of the LAN interface on your device, the `192.168.2.1/24` object should be replaced by your LAN IP CIDR, and the`{ssh,https,http,1080}` object is list of ports that you want to enable on your WAN interface.

Please notice that only often used ports will go through proxy for the configuration above. If you want all ports to go through proxy, just replace `tcp dport {80,443,22,3389,853,8080} redirect to :1234` with `ip protocol tcp redirect to :1234`.

The UDP traffic won't go through proxy (In my opinion, the best way to relay UDP is L2 VPN instead. There are lots of problems of v2ray UDP relay).

For example,you can use wireguard over v2ray to proxy UDP and ICMP,just like this

``` json
//this is a part of v2ray inbound        
{
            "listen": "127.0.0.1",
            "port": 12480,
            "protocol": "dokodemo-door",
            "settings": {
                "address": "127.0.0.1",//please replace this to your wireguard server ip
                "port": 80,//please replace this to your wireguard endpoint port
                "network": "udp"
            },
            "tag": "wireguard"
        }
```

now you can use `127.0.0.1:12480` as wireguard endpoint instead of YOURSERVERIP:80 

The file `/etc/iprules/bypass.nft` is the IP list you want to bypass (IP list of China).

An example of the bypass list:

```
define bypassd = {
10.0.0.0/8,
172.16.0.0/12,
192.168.0.0/16,
}
```

In most cases, this list should contain CHN IP list and RFC1918 defined addresses.

you may get list from [ipip.net](https://github.com/17mon/china_ip_list) or from [clang.cn](https://ispip.clang.cn/all_cn_cidr.txt)

If this device don't offer NAT services , delete `oifname {enp4s0,ppp0} masquerade`.

#### 6. An example of v2ray configuration

You may replace `/usr/local/etc/v2ray/config.json` with the config below:

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

#### 7. Install and config `AdGuardHome`

If you don't need DNS and DHCP services, ignore it.

Official guide to install AdGuardHome: https://github.com/AdguardTeam/AdGuardHome

Or you may run this script:

```bash
curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v
```

An example of DNS upstream config guide: https://github.com/fernvenue/adguardhome-upstream

Enable DHCP server in `AdGuardHome` panel if needed.

#### 8. Start and enjoy

```bash
systemctl enable --now v2ray
systemctl enable --now nftables
systemctl enable --now v2raya
```

Now you can access http://SERVERADDR:2017 to access v2raya panel to add v2ray nodes.
