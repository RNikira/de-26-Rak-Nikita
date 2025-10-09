SAMBA
---
- BR-SRV 
```
apt-get update && apt-get install task-samba-dc -y
echo nameserver 192.168.1.10 > /etc/resolv.conf
rm -rf /etc/samba/smb.conf
echo -e "192.168.3.10  br-srv.au-team.irpo" >> /etc/hosts
```
samba-tool domain provision
```
mv -f /var/lib/samba/private/krb5.conf /etc/krb5.conf
systemctl enable --now samba
samba-tool user add hquser1 P@ssw0rd
samba-tool user add hquser2 P@ssw0rd
samba-tool user add hquser3 P@ssw0rd
samba-tool user add hquser4 P@ssw0rd
samba-tool user add hquser5 P@ssw0rd
samba-tool group add hq
samba-tool group addmembers hq hquser1
samba-tool group addmembers hq hquser2
samba-tool group addmembers hq hquser3
samba-tool group addmembers hq hquser4
samba-tool group addmembers hq hquser5
apt-repo add rpm  http://altrepo.ru/local-
```
---
RAID
---
- HQ-SRV 
```
mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sd[b-c]
mdadm --detail -scan --verbose >> /etc/mdadm.conf
fdisk /dev/md0
printf ' n\n \n \n \n \n \nw
mkfs.ext4 /dev/md0p1
echo -e "/dev/md0p1  /raid  ext4  defaults  0  0" >> /etc/fstab
mkdir /raid
mount -a
apt-get install nfs-server -y
mkdir /raid/nfs
chown 99:99 /raid/nfs
chmod 777 /raid/nfs
echo -e "/raid/nfs  192.168.2.0/28(rw,sync,no_subtree_check)" >> /etc/exports
exportfs -a
exportfs -v
systemctl enable nfs
systemctl restart nfs
```
- HQ-CLI
```
apt-get update && apt-get install nfs-clients -y
mkdir -p /mnt/nfs
echo -e "192.168.1.10:/raid/nfs  /mnt/nfs  nfs  intr,soft,_netdex,x-systemd.automount  0  0" >> /etc/fstab
mount -a
mount -v
touch /mnt/nfs/test
```
