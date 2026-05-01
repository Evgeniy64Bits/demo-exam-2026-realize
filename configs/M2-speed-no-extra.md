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
5 tasks done - easy
---

### 🐧 BR-SRV

```
apt-get install -y docker-engine docker-compose
systemctl enable --now docker
mkdir /mnt/cdrom
mount /dev/sr0 /mnt/cdrom
ls /mnt/cdrom/docker
cd /mnt/cdrom/docker
docker load -i site_latest.tar
docker load -i mariadb_latest.tar
mkdir /usr/docker
vim /usr/docker/docker-compose.yml
services:
  testapp:
    image: site:latest
    container_name: testapp
    restart: always
    ports:
      - "8080:8000"
    environment:
      DB_TYPE: maria
      DB_HOST: db
      DB_NAME: testdb
      DB_PORT: 3306
      DB_USER: test
      DB_PASS: P@ssw0rd
    depends_on:
      - db

  db:
    image: mariadb:10.11
    container_name: db
    restart: always
    environment:
      MARIADB_DATABASE: testdb
      MARIADB_USER: test
      MARIADB_PASSWORD: P@ssw0rd
      MARIADB_ROOT_PASSWORD: rootpass
cd /usr/docker
docker compose up -d
docker ps -a
```

### 🐧 HQ-SRV

```
apt-get install httpd2 apache2-mod_php8.1 php8.1 php8.1-mysqlnd php8.1-mysqli -y
systemctl enable --now httpd2
apt-get install mariadb-server -y
systemctl enable --now mariadb
mysql -u root
CREATE DATABASE webdb;
CREATE USER 'web'@'localhost' IDENTIFIED BY 'P@ssw0rd';
GRANT ALL PRIVILEGES ON webdb.* TO 'web'@'localhost';
FLUSH PRIVILEGES;
mkdir /mnt/cdrom
mount /dev/sr0 /mnt/cdrom
ls /mnt/cdrom/web
mysql -u root webdb < /mnt/cdrom/web/dump.sql
cp /mnt/cdrom/web/index.php /var/www/html/
cp /mnt/cdrom/web/logo.png /var/www/html/
chown -R apache2:apache2 /var/www/html
chmod -R 777 /var/www/html
vim /var/www/html/index.php
$servername = "localhost";
$username = "web";
$password = "P@ssw0rd";
$dbname = "webdb";
systemctl restart httpd2
```

### 🐧 HQ-CLI

```
http://192.168.1.10
http://192.168.3.10:8080
```

### 🍃 HQ-RTR

```
ip nat source static tcp 192.168.1.10 80 172.16.4.4 8080
ip nat source static tcp 192.168.1.10 2026 172.16.4.4 2026
write memory
```

### 🍃 BR-RTR

```
(config)#ip nat source static tcp 192.168.3.10 8080 172.16.5.5 8080
(config)#ip nat source static tcp 192.168.3.10 2026 172.16.5.5 2026
write memory
```
8 tasks done - normal. DNAT здесь исходя из смысла

Можно хоть вначале сделать, но не успел сделать приложения

---


