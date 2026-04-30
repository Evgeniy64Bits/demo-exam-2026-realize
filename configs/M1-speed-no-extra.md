# ⏱ Модуль 1 — Быстрый готовый конфиг с командами без лишнего 🔥

## Заточен под реальный стенд на Proxmox у меня в колледже

### 🐧 ISP

```
hostnamectl set-hostname ISP;exec bash

echo HTTP_PROXY=http://10.0.21.52:3128 >> /etc/sysconfig/network
mkdir /etc/net/ifaces/ens19
mkdir /etc/net/ifaces/ens20
printf "BOOTPROTO=static\nTYPE=eth\n" > /etc/net/ifaces/ens19/options
printf "BOOTPROTO=static\nTYPE=eth\n" > /etc/net/ifaces/ens20/options
echo 172.16.1.1/28 > /etc/net/ifaces/ens19/ipv4address 
echo 172.16.2.1/28 > /etc/net/ifaces/ens20/ipv4address
systemctl restart network

apt-get update
apt-get install tzdata -y
exec bash
timedatectl set-timezone Europe/Moscow

apt-get install nftables -y
cat <<'EOT' > /etc/nftables/nftables.nft
#!/usr/sbin/nft -f
flush ruleset
table ip nat {
 chain postrouting {
  type nat hook postrouting priority srcnat
  oifname "ens18" masquerade
 }
}
EOT
systemctl enable --now nftables

touch /etc/rc.d/rc.local
echo -e '#!/bin/bash\necho 1 > /proc/sys/net/ipv4/ip_forward' > /etc/rc.d/rc.local
chmod +x /etc/rc.d/rc.local
/etc/rc.d/rc.local
cat /proc/sys/net/ipv4/ip_forward
```

### 🍃 HQ-RTR

```
hostname hq-rtr.au-team.irpo
ip domain-name au-team.irpo

port te0
service-instance te0/isp-hq
encapsulation untagged

port te1
service-instance te1/srv-net
encapsulation dot1q 100
rewrite pop 1

service-instance te1/cli-net
encapsulation dot1q 200
rewrite pop 1

service-instance te1/management
encapsulation dot1q 999
rewrite pop 1

interface eth1
ip address 172.16.1.2/28
connect port te0 service-instance te0/isp-hq
ip nat outside

interface eth2
ip address 192.168.1.1/27
connect port te1 service-instance te1/srv-net
ip nat inside

interface eth3
ip address 192.168.1.33/28
connect port te1 service-instance te1/cli-net
ip nat inside

interface eth4
ip address 192.168.1.49/29
connect port te1 service-instance te1/management
ip nat inside

ntp timezone utc+3

username net_admin
password P@ssw0rd
role admin

write memory

ip pool dhcp 1
range 192.168.1.34-192.168.1.46

dhcp-server 1
pool dhcp 64
mask 255.255.255.240
gateway 192.168.1.33
dns 192.168.1.2
domain-name au-team.irpo

interface eth3
dhcp-server 1

write memory
```

### 🐧 HQ-SRV

```
hostnamectl set-hostname hq-srv.au-team.irpo;exec bash

echo 192.168.1.2/27 > /etc/net/ifaces/ens18/ipv4address
echo default via 192.168.1.1 > /etc/net/ifaces/ens18/ipv4route
systemctl restart network

timedatectl set-timezone Europe/Moscow

useradd -s /bin/bash -u 2026 sshuser
echo "sshuser:P@ssw0rd" | chpasswd
gpasswd -a sshuser wheel
echo 'sshuser ALL = (root) NOPASSWD: ALL' >> /etc/sudoers


```




