# ⏱ Модуль 2 — Быстрый готовый конфиг с командами без лишнего 🔥

## Заточен под реальный стенд на Proxmox в колледже

### 🐧 ISP

```
vim /etc/chrony.conf
local stratum 5
allow 0.0.0.0/0
systemctl enable --now chronyd
```

### 🍃 BR-RTR

```
ntp server 172.16.5.1
show ntp status

security-profile 0
rule 0 permit tcp any any eq 22
security 0

write memory
```

### 🍃 HQ-RTR

```
security-profile 0
rule 0 permit tcp any any eq 22
security 0

write memory
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

apt-get install nfs-server -y
systemctl enable --now nfs-server
mkdir -p /raid/nfs
chmod 777 /raid/nfs
echo "/raid/nfs 192.168.2.0/28(rw,sync,no_subtree_check)" >> /etc/exports
exportfs -rav
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
apt-get update
apt-get install chrony -y
vim /etc/chrony.conf
#pool pool.ntp.org iburst
server 172.16.5.1 iburst
systemctl enable --now chronyd

ssh sshuser@192.168.1.10 -p 2026
ssh user@192.168.2.10

apt-get update
apt-get install ansible sshpass -y
vim /etc/ansible/hosts
[hq]
HQ-SRV ansible_port=2026 ansible_host=192.168.1.10 ansible_user=sshuser ansible_ssh_pass=P@ssw0rd ansible_python_interpreter=/usr/bin/python3
HQ-CLI ansible_host=192.168.2.10 ansible_user=user ansible_ssh_pass=resu ansible_python_interpreter=/usr/bin/python3

[routers]
HQ-RTR ansible_host=192.168.1.1 ansible_user=admin ansible_password=admin ansible_connection=network_cli ansible_network_os=ios ansible_python_interpreter=/usr/bin/python3
BR-RTR ansible_host=192.168.3.1 ansible_user=admin ansible_password=admin ansible_connection=network_cli ansible_network_os=ios ansible_python_interpreter=/usr/bin/python3
ansible all -m ping
```




