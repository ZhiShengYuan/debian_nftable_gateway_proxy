>``` text
>#!/usr/sbin/nft -f
>flush ruleset
>include "/etc/iprules/bypass.nft"
>table inet filter
>delete table inet filter
>table inet filter {
>       chain input {
>               type filter hook input priority 0;policy drop;
>               ct state established,related accept
>               iifname lo accept
>               iifname enp3s0 accept
>               icmp type echo-request accept
>               ip saddr 192.168.1.0/24 accept
>               tcp dport {ssh,https,http,5443,1080,23,24,5201,53,1194} accept
>               udp dport {1080,5201,53} accept
>       }
>       chain forward {
>               type filter hook forward priority 0;policy drop;
>               iifname enp3s0 oifname {enp4s0,ppp0} accept
>               iifname {enp4s0,ppp0} oifname enp3s0 ct state {related,established} accept
>       }
>       chain output {
>               type filter hook output priority 0;policy accept;
>      }
>}
>table ip nat {
>  chain proxy {
>    ip daddr $bypassd return
>    tcp dport {80,443,22,3389,853} redirect to :1234
>    udp dport 53 redirect to :53
>    return
>}
>chain prerouting {
>    type nat hook prerouting priority 0; policy accept;
>    goto proxy
>  }
>chain postrouting {
>    type nat hook postrouting priority srcnat;policy accept;
>    oifname {enp4s0,ppp0} masquerade
> }
>}
>```
>
>

also,enable ipv4 && ipv6 forward in sysctl.conf

>
>
>```text
>net.ipv4.ip_forward=1
>net.ipv6.conf.all.disable_ipv6 = 0
>net.ipv6.conf.default.disable_ipv6 = 0
>net.ipv6.conf.lo.disable_ipv6 = 0
>net.ipv6.conf.all.forwarding=1
>net.ipv6.conf.default.forwarding=1
>net.ipv6.conf.default.accept_ra=2
>net.ipv6.conf.default.use_tempaddr=1
>```

and use `wide-dhcpv6-client` to assignment ipv6 address (as pd), `miniupnpd` also can use to offer upnpd server(iptables required)
