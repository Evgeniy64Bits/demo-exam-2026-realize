# ⏱ Модуль 2 — Быстрый готовый конфиг с командами без лишнего 🔥

## Заточен под реальный стенд на Proxmox в колледже

### 🐧 ISP

```
vim /etc/chrony.conf
local stratum 5
allow 0.0.0.0/0
systemctl restart chronyd
```

### 🍃 BR-RTR

```
ntp server 172.16.2.1

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
apt-get update && apt-get install chrony nfs-server -y

vim /etc/chrony.conf
#pool pool.ntp.org iburst
server 172.16.1.1 iburst
systemctl restart chronyd

mdadm --create /dev/md0 -l0 -n 2 /dev/sdb /dev/sdc
mkfs.ext4 /dev/md0
mdadm --detail /dev/md0 > /etc/mdadm.conf
mkdir /raid
vim /etc/fstab
/dev/md0	/raid	ext4	defaults	0	0
mount -av

systemctl enable --now nfs-server
mkdir /raid/nfs
chmod 777 /raid/nfs
echo "/raid/nfs 192.168.1.32/27(rw,sync,no_subtree_check)" >> /etc/exports
exportfs -rav
```

### 🐧 HQ-CLI

```
apt-get update && apt-get install yandex-browser-stable -y

vim /etc/chrony.conf
#pool pool.ntp.org iburst
server 172.16.1.1 iburst
systemctl restart chronyd
chronyc sources

mkdir /mnt/nfs
mount 192.168.1.2:/raid/nfs /mnt/nfs
vim /etc/fstab
192.168.1.2:/raid/nfs  /mnt/nfs  nfs  defaults,_netdev  0  0
mount -av

systemctl enable --now sshd
```

### 🐧 BR-SRV

```
apt-get update && apt-get install chrony ansible sshpass -y
vim /etc/chrony.conf
#pool pool.ntp.org iburst
server 172.16.2.1 iburst
systemctl restart chronyd

ssh sshuser@192.168.1.2 -p 2026
ssh user@192.168.1.34

vim /etc/ansible/hosts
[hq]
hq-srv ansible_port=2026 ansible_host=192.168.1.2 ansible_user=sshuser ansible_ssh_pass=P@ssw0rd ansible_python_interpreter=/usr/bin/python3
hq-cli ansible_host=192.168.1.34 ansible_user=user ansible_ssh_pass=resu ansible_python_interpreter=/usr/bin/python3

[routers]
hq-rtr ansible_host=192.168.1.1 ansible_user=admin ansible_password=admin ansible_connection=network_cli ansible_network_os=ios ansible_python_interpreter=/usr/bin/python3
br-rtr ansible_host=192.168.2.1 ansible_user=admin ansible_password=admin ansible_connection=network_cli ansible_network_os=ios ansible_python_interpreter=/usr/bin/python3

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
    container_name: tespapp
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
docker compose config
docker compose up -d
docker ps -a
```

### 🐧 HQ-SRV

```
apt-get install httpd2 apache2-mod_php8.1 php8.1 php8.1-mysqlnd php8.1-mysqli -y
vim /etc/apt/sources.list.d/alt.list     (раскомментировать реп-ии)
rpm [p10] http://ftp.atllinux.org/pub/distributions/ALTLinux p10/branch/x86_64 classic
rpm [p10] http://ftp.atllinux.org/pub/distributions/ALTLinux p10/branch/noarch classic
rpm [p10] http://ftp.atllinux.org/pub/distributions/ALTLinux p10/branch/x86_64-i586 classic
apt-get update
apt-get install mariadb-server -y
systemctl enable --now httpd2
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
chmod -R 755 /var/www/html
vim /var/www/html/index.php
$servername = "localhost";
$username = "web";
$password = "P@ssw0rd";
$dbname = "webdb";
systemctl restart httpd2
```

### 🐧 HQ-CLI

```
http://192.168.1.2
http://192.168.2.2:8080
```

### 🍃 HQ-RTR

```
ip nat source static tcp 192.168.1.2 80 172.16.1.2 8080
ip nat source static tcp 192.168.1.2 2026 172.16.1.2 2026
write memory
```

### 🍃 BR-RTR

```
ip nat source static tcp 192.168.2.2 8080 172.16.2.2 8080
ip nat source static tcp 192.168.2.2 2026 172.16.2.2 2026
write memory
```

### 🐧 ISP

```
apt-get install curl -y
curl http://172.16.1.2:8080
curl http://172.16.2.2:8080

systemctl enable --now sshd
ssh sshuser@172.16.1.2 -p 2026
ssh sshuser@172.16.2.2 -p 2026
```

8 tasks done - normal
---
DNAT здесь исходя из смысла

Можно хоть вначале сделать, но если не успеть сделать приложения - DNAT бесполезен на половину

Проверку DNAT можно исключить в целях экономии времени

### 🐧 ISP

```
apt-get update && apt-get install nginx apache2-htpasswd nano -y
systemctl enable --now nginx
htpasswd -c /etc/nginx/.htpasswd WEB
P@ssw0rd
nano /etc/nginx/sites-enabled.d/reverse.conf
server {
    listen 80;
    server_name web.au-team.irpo;
    auth_basic "Restricted Access";
    auth_basic_user_file /etc/nginx/.htpasswd;

    location / {
        proxy_pass http://172.16.1.2:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

server {
    listen 80;
    server_name docker.au-team.irpo;

    location / {
        proxy_pass http://172.16.2.2:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
nginx -t
systemctl restart nginx
```

### 🐧 HQ-CLI

```
http://web.au-team.irpo
http://docker.au-team.irpo
```
10 tasks done - hard
---

### 🐧 HQ-CLI

```
vim /etc/resolv.conf
search au-team.irpo
nameserver 192.168.2.2
nameserver 192.168.1.2
```

### 🐧 HQ-SRV - not important i think, but perfectly

```
vim /etc/dnsmasq.conf
server=/au-team.irpo/192.168.2.2
systemctl restart dnsmasq
```

### 🐧 BR-SRV

```
# Part 1
vim /etc/hosts
127.0.0.1 localhost
192.168.2.2 br-srv.au-team.irpo br-srv

vim /etc/resolv.conf
search au-team.irpo
nameserver 192.168.2.2
nameserver 192.168.1.2

vim /etc/apt/sources.list.d/alt.list     (раскомментировать реп-ии)
rpm [p10] http://ftp.atllinux.org/pub/distributions/ALTLinux p10/branch/x86_64 classic
rpm [p10] http://ftp.atllinux.org/pub/distributions/ALTLinux p10/branch/noarch classic
rpm [p10] http://ftp.atllinux.org/pub/distributions/ALTLinux p10/branch/x86_64-i586 classic
apt-get update && apt-get install task-samba-dc task-auth-ad-sssd -y

rm -f /etc/samba/smb.conf
rm -rf /var/lib/samba
mkdir -p /var/lib/samba/sysvol

systemctl stop docker
samba-tool domain provision
realm - AU-TEAM.IRPO  (press Enter)
domain - AU-TEAM  (press Enter)
server-role - dc  (press Enter)
dns-backend - SAMBA_INTERNAL  (press Enter)
dns-forwarder - 192.168.1.2
adminpass - P@ssw0rd!

cp /var/lib/samba/private/krb5.conf /etc/krb5.conf

systemctl enable --now samba
systemctl start docker

host -t SRV _ldap._tcp.au-team.irpo
kinit administrator

#Part 2
samba-tool group add hq

for i in 1 2 3 4 5; do
samba-tool user create hquser$i P@ssw0rd$i
done

for i in 1 2 3 4 5; do
samba-tool group addmembers hq hquser$i
done

samba-tool group listmembers hq
```

### 🐧 HQ-CLI

```
# Part 3
system-auth write ad au-team.irpo hq-cli AU-TEAM administrator 'P@ssw0rd!'

# Part 4
vim /etc/security/access.conf
+:@hq:ALL
-:ALL:ALL

# Part 5
EDITOR=nano visudo
%hq ALL=(ALL) NOPASSWD: /bin/cat, /bin/grep, /usr/bin/id

reboot

su - hquser1
sudo id
sudo cat /etc/passwd
sudo bash
```
All 11 tasks done - very hard
---

### Принцип: сколько успел - столько успел и столько баллов получил за выполненные задания

### Поделил на условные сложности - отметки сколько успел выполнить заданий за 1 ч. 30 мин.
