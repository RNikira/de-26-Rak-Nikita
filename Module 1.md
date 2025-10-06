- ISP
```
hostnamectl set-hostname ISP
mkdir /net/ifaces/ens{20,21,22}
echo "DISABLED=no\nTYPE=eth\nCONFIG_IPV4=yes\nBOOTPROTO=dhcp" > /etc/net/ifaces/ens20/options
```
