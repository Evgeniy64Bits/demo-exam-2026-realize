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
echo 172.16.4.1/28 > /etc/net/ifaces/ens19/ipv4address 
echo 172.16.5.1/28 > /etc/net/ifaces/ens20/ipv4address
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

```
