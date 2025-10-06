- ISP
```
hostnamectl set-hostname ISP
mkdir /etc/net/ifaces/ens{20,21,22}
echo -e "TYPE=eth\nBOOTPROTO=dhcp\nDISABLED=no\nCONFIG_IPV4=yes" > /etc/net/ifaces/ens20/options
echo -e "TYPE=eth\nBOOTPROTO=static\nDISABLED=no\nCONFIG_IPV4=yes" > /etc/net/ifaces/ens21/options
echo -e "TYPE=eth\nBOOTPROTO=static\nDISABLED=no\nCONFIG_IPV4=yes" > /etc/net/ifaces/ens22/options
echo "172.16.1.1/28" > /etc/net/ifacces/ens21/ipv4address
echo "172.16.2.1/28" > /etc/net/ifacces/ens22/ipv4address
sed -i 's/net.ipv4.ip_forward = 0/net.ipv4.ip_forward = 1/g' /etc/net/sysctl.conf
sysctl -p
systemctl restart network
apt-get update && apt-get install iptables -y && apt-get reinstall tzdata
timedatectl set-timezone Asia/Yekaterinburg
iptables -t nat -A POSTROUTING -o ens20 -s 0/0 -j MASQUERADE
iptables-save > /et/sysconfig/iptables
systemctl enable --now iptables
exec bash
```
