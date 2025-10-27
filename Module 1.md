- ISP
```
hostnamectl set-hostname ISP
mkdir /etc/net/ifaces/ens{20,21,22}
echo -e "TYPE=eth\nBOOTPROTO=dhcp\nDISABLED=no\nCONFIG_IPV4=yes" > /etc/net/ifaces/ens20/options
echo -e "TYPE=eth\nBOOTPROTO=static\nDISABLED=no\nCONFIG_IPV4=yes" > /etc/net/ifaces/ens21/options
echo -e "TYPE=eth\nBOOTPROTO=static\nDISABLED=no\nCONFIG_IPV4=yes" > /etc/net/ifaces/ens22/options
echo 172.16.1.1/28 > /etc/net/ifaces/ens21/ipv4address
echo 172.16.2.1/28 > /etc/net/ifaces/ens22/ipv4address
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
- HQ-RTR
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
port te1
  service-instance te1/int1
    encapsulation dot1q 100
    rewrite pop 1
    ex
  service-instance te1/int2
    encapsulation dot1q 200
    rewrite pop 1
    ex
  service-instance te1/int3
    encapsulation dot1q 999
    rewrite pop 1
    ex
int int1
  connect port te1 service-instance te1/int1
  ex
int int2
  connect port te1 service-instance te1/int2
  ex
int int3
  connect port te1 service-instance te1/int3
  ex
ip route 0.0.0.0 0.0.0.0 172.16.1.1
username net_admin
  password P@ssw0rd
  role admin
  ex
int tunnel.0
  ip add 172.16.0.1/30
  ip mtu 1400
  ip tunnel 172.16.1.2 172.16.2.2 mode gre
  ip ospf authentication-key ecorouter
  ex
router ospf 1
  network 172.16.0.0/30 area 0
  network 192.168.1.0/27 area 0
  network 192.168.2.0/28 area 0
  passive-interface default
  no passive-interface tunnel.0
  area 0 authentication
  ex
ip nat pool NAT_POOL 192.168.1.1-192.168.1.254,192.168.2.1-192.168.2.254
ip nat source dynamic inside-to-outside pool NAT_POOL overload interface int0
ip pool cli_pool 192.168.2.10-192.168.2.10
dhcp-server 1
  pool cli_pool 1
    mask 255.255.255.240
    gateway 192.168.2.1
    dns 192.168.1.10
    domain-name au-team.irpo
    exit
  exit
int int2
  dhcp-server 1
  exit
write
```
- BR-RTR
```
en
conf t
hostname br-rtr
ip domain-name au-team.irpo
ntp timezone utc +5
int int0
  ip add 172.16.2.2/28
  ip nat outside
  ex
int int1
  ip add 192.168.3.1/28
  ip nat inside
  ex
port te0
  service-instance te0/int0
  encapsulation untagged
  ex
int int0
  connect port te0 service-instance te0/int0
  ex
port te1
  service-instance te1/int1
  encapsulation untagged
  ex
int int1
  connect port te1 service-instance te1/int1
  ex
ip route 0.0.0.0 0.0.0.0 172.16.2.1
username net_admin
  role admin
  password P@ssw0rd
  ex
int tunnel.0
  ip add 172.16.0.2/30
  ip mtu 1400
  ip tunnel 172.16.2.2 172.16.1.2 mode gre
  ip ospf authentication-key ecorouter
  ex
router ospf 1
  network 172.16.0.0/30 area 0
	network 192.168.3.0/28 area 0
	passive-interface default
	no passive-interface tunnel.0
	area 0 authentication
	ex
ip nat pool NAT_POOL 192.168.3.1-192.168.3.254
ip nat source dynamic inside-to-outside pool NAT_POOL overload interface int0
write
```
- HQ-SRV
```
hostnamectl hostname hq-srv.au-team.irpo
timedatectl set-timezone Asia/Yekaterinburg
mkdir /etc/net/ifaces/ens20
echo -e "TYPE=eth\nBOOTPROTO=static\nDISABLED=no\nCONFIG_IPV4=yes" > /etc/net/ifaces/ens20/options
echo 192.168.1.10/27 > /etc/net/ifaces/ens20/ipv4address
echo default via 192.168.1.1 > /etc/net/ifaces/ens20/ipv4route
echo nameserver 8.8.8.8 > /etc/resolv.conf
systemctl restart network
apt-get update && apt-get install dnsmasq -y
systemctl enable --now dnsmasq
useradd sshuser -u 2026
echo "sshuser:P@ssw0rd" | chpasswd
sed -i 's/# WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL/ WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL/g' /etc/sudoers
gpasswd -a "sshuser" wheel
sed -i "1iPort 2026\nAllowUsers sshuser\nMaxAuthTries 2\nPasswordAuthentication yes\nBanner /etc/openssh/banner" /etc/openssh/sshd_config
echo "Authorized access only" > /etc/openssh/banner
systemctl restart sshd
sed -i "1i\no-resolv\ndomain=au-team.irpo\nserver=8.8.8.8\ninterface=*\naddress=/hq-rtr.au-team.irpo/192.168.1.1\nptr-record=1.1.168.192.in-addr.arpa,hq-rtr.au-team.irpo\naddress=/docker.au-team.irpo/172.16.1.1\naddress=/web.au-team.irpo/172.16.2.1\naddress=/br-rtr.au-team.irpo/192.168.3.1\naddress=/hq-srv.au-team.irpo/192.168.1.10\nptr-record=10.1.168.192.in-addr.arpa,hq-srv.au-team.irpo\naddress=/hq-cli.au-team.irpo/192.168.2.10\nptr-record=10.2.168.192.in-addr.arpa,hq-cli.au-team.irpo\naddress=/br-srv.au-team.irpo/192.168.3.10" /etc/dnsmasq.conf
echo "192.168.1.1	hq-rtr.au-team.irpo" >> /etc/hosts
systemctl restart dnsmasq
exec bash
```
- BR-SRV
```
hostnamectl hostname br-srv.au-team.irpo
timedatectl set-timezone Asia/Yekaterinburg
mkdir /etc/net/ifaces/ens20
echo -e "TYPE=eth\nBOOTPROTO=static\nDISABLED=no\nCONFIG_IPV4=yes" > /etc/net/ifaces/ens20/options
echo 192.168.3.10/28 > /etc/net/ifaces/ens20/ipv4address
echo default via 192.168.3.1 > /etc/net/ifaces/ens20/ipv4route
echo nameserver 8.8.8.8 > /etc/resolv.conf
systemctl restart network
useradd sshuser -u 2026
echo "sshuser:P@ssw0rd" | chpasswd
sed -i 's/# WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL/ WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL/g' /etc/sudoers
gpasswd -a "sshuser" wheel
sed -i "1iPort 2026\nAllowUsers sshuser\nMaxAuthTries 2\nPasswordAuthentication yes\nBanner /etc/openssh/banner" /etc/openssh/sshd_config
echo "Authorized access only" > /etc/openssh/banner
systemctl restart sshd
exec bash
```
- CLI
```
hostnamectl hostname hq-cli.au-team.irpo
timedatectl set-timezone Asia/Yekaterinburg
mkdir /etc/net/ifaces/ens20
echo -e "TYPE=eth\nBOOTPROTO=dhcp\nDISABLED=no\nCONFIG_IPV4=yes" > /etc/net/ifaces/ens20/options
echo nameserver 8.8.8.8 > /etc/resolv.conf
systemctl restart network
dig docker.au-team.irpo
dig web.au-team.irpo
exec bash
```


