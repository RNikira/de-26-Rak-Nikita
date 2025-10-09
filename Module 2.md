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
apt-get update && apt-get install fdisk -y
echo -e "n\n\n\n\n\nw" | fdisk /dev/md0
mkfs.ext4 /dev/md0p1
echo -e "/dev/md0p1    /raid  ext4  defaults      0      0" >> /etc/fstab
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
echo -e "192.168.1.10:/raid/nfs  /mnt/nfs      nfs    intr,soft,_netdev,x-systemd.automount    0      0" >> /etc/fstab
mount -a
mount -v
touch /mnt/nfs/test
```
---
NTP
---
- ISP
```
apt-get install chrony -y
echo -e "server 127.0.0.1 iburst prefer\nhwtimestamp *\nlocal stratum 5\nallow 0/0" >> /etc/chrony.conf
systemctl enable --now chronyd
systemctl restart chronyd
chronyc sources
chronyc tracking | grep Stratum
```
- HQ-SRV | CLI
```
apt-get install chrony -y
echo server 172.16.1.1 iburst prefer > /etc/chrony.conf
systemctl enable --now chronyd
systemctl restart chronyd
chronyc sources
timedatectl
```
- BR-SRV
```
apt-get update && apt-get install chrony -y
echo server 172.16.2.1 iburst prefer > /etc/chrony.conf
systemctl enable --now chronyd
systemctl restart chronyd
chronyc sources
timedatectl
```
---
ANSIBLE
---
- BR-SRV
```
apt-get install ansible -y
tee /etc/ansible/hosts > /dev/null << 'EOF'
Vhs:
 hosts:
  HQ-SRV:
   ansible_host: 192.168.1.10
   ansible_user: remote_user
   ansible_port: 2026
  HQ-CL1:
   ansible_host: 192.168.2.10
   ansible_user: remote_user
   ansible_port: 2026
  HQ-RTR:
   ansible_host: 192.168.1.1
   ansible_user: net_admin
   ansible_password: P@sswOrd
   ansible_connection: network_cli
   ansible_network_os: ios
  BR-RTR:
   ansible_host: 192.168.3.1
   ansible_user: net_admin
   ansible_password: P@sswOrd
   ansible_connection: network_cli
   ansible_network_os: ios
EOF

sed -i '/^\[defaults\]$/a ansible_python_interpreter=/usr/bin/python3\ninterpreter_python=auto_silent\nansible_host_key_checking=false' /etc/ansible/ansible.cfg

ssh-keygen -t rsa -b 2048 -f /root/.ssh/id_rsa -N ""
expect -c 'spawn ssh -p 2026 remote_user@192.168.1.10; expect "password:"; send "P@sswOrd\r"; interact'
expect -c 'spawn ssh -p 2026 remote_user@192.168.2.10; expect "password:"; send "P@sswOrd\r"; interact'
```
