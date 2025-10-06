- ISP
```
hostnamectl set-hostname ISP
mkdir /etc/net/ifaces/ens{20,21,22}
echo -e "TYPE=eth\nBOOTPROTO=dhcp\nDISABLED=no\nCONFIG_IPV4=yes" > /etc/net/ifaces/ens20/options
echo -e "TYPE=eth\nBOOTPROTO=static\nDISABLED=no\nCONFIG_IPV4=yes" > /etc/net/ifaces/ens21/options
echo -e "TYPE=eth\nBOOTPROTO=static\nDISABLED=no\nCONFIG_IPV4=yes" > /etc/net/ifaces/ens22/options
echo "172.16.1.1/28" > /etc/net/ifaces/ens21/ipv4address
echo "172.16.2.1/28" > /etc/net/ifaces/ens22/ipv4address
sed -i 's/net.ipv4.ip_forward = 0/net.ipv4.ip_forward = 1/g' /etc/net/sysctl.conf
sysctl -p
systemctl restart network
apt-get update
apt-get install iptables -y && apt-get reinstall tzdata
timedatectl set-timezone Asia/Yekaterinburg
iptables -t nat -A POSTROUTING -o ens20 -s 0/0 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables
exec bash
```
- HQ-R
```
en
conf t
hostname hq-rtr
ip domain-name au-team.irpo
ntp timezone utc+5
int int0
  ip add 172.16.1.2/28
  ip nat outside
  ex
int int1
  ip add 192.168.1.1/27
  ip nat inside
  ex
int int2
  ip add 192.168.2.1/28
  ip nat inside
  ex
int int3
  ip add 192.168.99.1/29
  ex
port te0
  service-instance te0/int0
  encapsulation untagged
  ex
int int0
  connect port te0 service-instance te0/int0
  ex
```


