# ⏱ Модуль 2 — Быстрый готовый конфиг с командами без лишнего 🔥

## Заточен под реальный стенд на Proxmox в колледже

### 🐧 ISP

```
vim /etc/chrony.conf
local stratum 5
allow 0.0.0.0/0
systemctl enable --now chronyd
```

### 🐧 HQ-SRV

```
mdadm --create /dev/md0 -l0 -n 2 /dev/sdb /dev/sdc
mkfs.ext4 /dev/md0
echo "DEVICE partitions" > /etc/mdadm.conf
mdadm --detail --scan >> /etc/mdadm.conf
mkdir /raid
vim /etc/fstab
/dev/md0	/raid	ext4	defaults	0	0
mount -av

apt-get update
apt-get install chrony nfs-server -y
vim /etc/chrony.conf
#pool pool.ntp.org iburst
server 172.16.4.1 iburst
systemctl enable --now chronyd


```

### 🐧 HQ-CLI

```
apt-get update
apt-get install yandex-browser-stable -y
yandex-browser-stable

vim /etc/chrony.conf
#pool pool.ntp.org iburst
server 172.16.4.1 iburst
systemctl enable --now chronyd
chronyc sources

mkdir /mnt/nfs
mount 192.168.1.10:/raid/nfs /mnt/nfs
vim /etc/fstab
192.168.1.10:/raid/nfs  /mnt/nfs  nfs  defaults,_netdev  0  0
mount -av
```










### 🐧 BR-SRV

```

```







### 🍃 HQ-RTR

