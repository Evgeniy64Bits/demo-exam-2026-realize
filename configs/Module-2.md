# 📘 Модуль 2 — Пояснения к сетевому администрированию

Покажу этот модуль на Proxmox, отличия с VMware Workstation будут только в интерфейсах (ens18/19, te0/1, ...)

Если вы не ознакомились с особенностью в 1-ом модуле - ознакомьтесь в пункте "2. Настройка IP-адресов"

1. [Установка Яндекс Браузера на HQ-CLI](#1-установка-яндекс-браузера-на-hq-cli)
2. [Настройка службы сетевого времени (chrony) на ISP](#2-настройка-службы-сетевого-времени-chrony-на-isp)
3. [Конфигурация файлового хранилища на HQ-SRV](#3-конфигурация-файлового-хранилища-на-hq-srv)
4. [Настройка NFS-сервера на HQ-SRV](#4-настройка-nfs-сервера-на-hq-srv)
7. [Развёртывание веб-приложения на HQ-SRV](#5-развёртывание-веб-приложения-на-hq-srv)
8. [Настройка Ansible на BR-SRV](#6-настройка-ansible-на-br-srv)
9. [Развёртывание веб-приложения в Docker на BR-SRV](#7-развёртывание-веб-приложения-в-docker-на-br-srv)
10. [Настройка статической трансляции портов на маршрутизаторах](#8-настройка-статической-трансляции-портов-на-маршрутизаторах)
11. [Настройка nginx как обратного прокси на ISP](#9-настройка-nginx-как-обратного-прокси-на-isp)
12. [Настройка web-based аутентификации на ISP](#10-настройка-web-based-аутентификации-на-isp)
13. [Настройка Samba DC на BR-SRV](#11-настройка-samba-dc-на-br-srv)

Ниже представлены IP'шники у ВМ'ок в Proxmox. Реальная нумерация интерфейсов в колледже - ens18, ens19, ens20. У меня такая - ens19, ens20, ens21. C первым модулем такая же история.

![ip-in-proxmox](../pictures-m2/ip-proxmox.png)

## 1. Установка Яндекс Браузера на HQ-CLI

> [!NOTE]
> Установим простым способом через официальные репозитории ALT

### 🐧 HQ-CLI

```
apt-get update
apt-get install yandex-browser-stable -y
yandex-browser-stable    #запускаем под обычным пользователем (не root)
```

![zadanie-1](../pictures-m2/1-hq-cli-yandex-browser.png)

## 2. Настройка службы сетевого времени (chrony) на ISP

> [!NOTE]
> Установку chrony производим на ISP, HQ-SRV, HQ-CLI и BR-SRV.
> 
> pool pool.ntp.org iburst - это вышестоящий внешний сервер NTP по умолчанию, его и оставим
> 
> local stratum 5 - это позиция нашего ISP в иерархии NTP-серверов
>
> По правильному мы должны вписать 5 наших сетей, попадающих под использование NTP:
> 
> allow 172.16.4.0/28
>
> allow 172.16.5.0/28
>
> allow 192.168.1.0/26
>
> allow 192.168.2.0/28
>
> allow 192.168.50.0/27
>
> Покажу по простому варианту - указать все сети: allow 0.0.0.0/0
>
> chronyc tracking показывает у меня 3 из-за отсутствия ограничений выхода ко внешним NTP-серверам - это нормально. В сети колледжа выхода к ним не будет, stratum покажет 5. Также может наблюдаться расхождение во времени на колледжском стенде, ничего страшного - это не наша проблема
>
> Устанавливать chrony приходится только на HQ-SRV и BR-SRV. У ISP и HQ-CLI chrony предустановлен

### 🐧 ISP

```
# chrony уже предустановлен, проверить rpm -qa | grep chrony
vim /etc/chrony.conf
# под строчкой pool pool.ntp.org iburst пишем:
local stratum 5
allow 0.0.0.0/0

systemctl enable --now chronyd
systemctl restart chronyd

chronyc tracking  #проверка
```

![zadanie-2](../pictures-m2/2-isp-chrony-conf.png)

![zadanie-2](../pictures-m2/2-isp-chrony-systemctl-tracking.png)

### 🐧 HQ-SRV, HQ-CLI

```
apt-get update
apt-get install chrony -y
vim /etc/chrony.conf
# закомментируем строчку:
pool pool.ntp.org iburst
# добавим:
server 172.16.4.1 iburst

systemctl restart chronyd
systemctl enable --now chronyd

# проверки:
chronyc sources
date
```

![zadanie-2](../pictures-m2/2-hq-chrony-conf.png)

### 🐧 BR-SRV

```
apt-get update
apt-get install chrony -y
vim /etc/chrony.conf
# закомментируем строчку:
pool pool.ntp.org iburst
# добавим:
server 172.16.5.1 iburst

systemctl restart chronyd
systemctl enable --now chronyd

# проверки:
chronyc sources
date
```

![zadanie-2](../pictures-m2/2-br-srv-chrony-conf.png)

### 🍃 BR-RTR

```
(config)#ntp server 172.16.5.1
exit

#write memory

# проверки:
show ntp status
show ntp date
```

![zadanie-2](../pictures-m2/2-br-rtr-ntp.png)

## 3. Конфигурация файлового хранилища на HQ-SRV

> [!NOTE]
> Создаём массив RAID 0 из двух дисков (они уже подключены к ВМ). Форматируем под файловую систему ext4. Автоматически монтируем массив в каталог /raid
>
> mdadm предустановлен на HQ-SRV. Проверить: rpm -qa | grep mdadm

### 🐧 HQ-SRV

```
mdadm --create /dev/md0 -l0 -n 2 /dev/sdb /dev/sdc
mkfs.ext4 /dev/md0

# добавляем информацию о RAID-массиве:
echo "DEVICE partitions" > /etc/mdadm.conf
mdadm --detail --scan >> /etc/mdadm.conf

mkdir /raid
vim /etc/fstab
# добавляем строчку (жмём tab после каждого параметра):
/dev/md0	/raid	ext4	defaults	0	0
mount -av

# проверки:
df -h
lsblk
```

![zadanie-3](../pictures-m2/3-hq-srv-mdadm-raid.png)

![zadanie-3](../pictures-m2/3-hq-srv-fstab.png)

![zadanie-3](../pictures-m2/3-hq-srv-auto-mount-check.png)

## 4. Настройка NFS-сервера на HQ-SRV

> [!NOTE]
> NFS (Network File System) — это протокол распределённой файловой системы, который позволяет компьютерам в сети обмениваться файлами так, будто они расположены локально, хотя физически находятся на сервере (физическом или виртуальном). С помощью NFS можно монтировать удалённую директорию на локальном компьютере, делая её доступной как локальные файлы.
>
> На HQ-SRV создаём каталог nfs на недавно созданном raid массиве. В /etc/export определяем директорию, которая будет доступна удалённым клиентам (всеи из сети HQ-CLI), и задаём параметры доступа.
>
> rw — разрешает клиенту чтение и запись в экспортируемую директорию
>
> sync — отвечать на следующие запросы только тогда, когда данные будут сохранены на диск
>
> no_subtree_check — отключить проверку обращения к экспортированной папке
>
> На клиенте монтируем сетевой каталог в локальную точку монтирования и обеспечиваем автомонтирование
>
> Конечная проверка. На HQ-CLI создать файл: touch /mnt/nfs/testfile
>
> На HQ-SRV проверить что каталог отобразился и тем самым понять, что NFS успешно настроен: ls /raid/nfs

### 🐧 HQ-SRV

```
apt-get update
apt-get install nfs-server -y
systemctl enable --now nfs-server

mkdir -p /raid/nfs
chmod 777 /raid/nfs

# добавляем строку в /etc/exports:
echo "/raid/nfs 192.168.2.0/28(rw,sync,no_subtree_check)" >> /etc/exports

# экспортируем каталоги:
exportfs -rav
```

![zadanie-4](../pictures-m2/4-hq-srv-nfs-install.png)

![zadanie-4](../pictures-m2/4-hq-srv-exports-raid-nfs.png)

### 🐧 HQ-CLI

```
mkdir /mnt/nfs
mount 192.168.1.10:/raid/nfs /mnt/nfs

vim /etc/fstab
# добавляем строку (жмём tab после каждого параметра):
192.168.1.10:/raid/nfs  /mnt/nfs  nfs  defaults,_netdev  0  0
mount -av
```

![zadanie-4](../pictures-m2/4-hq-cli-mount-network-dir.png)

![zadanie-4](../pictures-m2/4-hq-cli-fstab.png)

![zadanie-4](../pictures-m2/4-hq-cli-auto-mount-dir-and-check.png)

![zadanie-4](../pictures-m2/4-hq-srv-nfs-works-check.png)

## 5. Развёртывание веб-приложения на HQ-SRV

> [!NOTE]
> Поднимаем простенькое веб-приложение с базой данных, с помощью apache HTTP Server и mariadb server.
>
> Используем подготовленные файлы из каталога "web" Additional.iso
>
> MariaDB — форк MySQL с открытым исходным кодом. 
>
> Apache — веб-сервер с открытым исходным кодом. Выступает посредником между файлами сайта, которые хранятся на файловой системе сервера, и пользователями, которые хотят на сайт зайти.
> 
> MariaDB в связке с Apache — это классическая конфигурация для создания веб-серверов, известная как LAMP-стек (Linux, Apache, MariaDB, PHP). Такая установка позволяет размещать динамические веб-сайты и приложения, где Apache обрабатывает HTTP-запросы, MariaDB хранит данные, а PHP выполняет обработку динамического контента.
> 
> Весь процесс можно разделить на 7 этапов:
>
> 1) Установка Apache и PHP
>
> 2) Установка MariaDB
>
> 3) Создание базы данных
>
> 4) Импорт базы данных
>
> 5) Копирование файлов сайта
>
> 6) Настройка подключения к БД
>
> 7) Открытие веб-приложения с клиента

Подключаем Additional.iso к HQ-SRV и BR-SRV (для дальнейшего задания - развёртывание веб-приложения в Docker). Перед подключением выключаем виртуалки.

![zadanie-5](../pictures-m2/5-srv-add-additional-in-proxmox.png)

После upload'а на хранилище, добавляем CD/DVD устройство и подключаем к нему Additional.iso

![zadanie-5](../pictures-m2/5-srv-add-cd-dvd-to-vm.png)

![zadanie-5](../pictures-m2/5-srv-add-additional-to-vm.png)

Запускаем обратно виртуалки и готово. Проверить - lsblk

### 🐧 HQ-SRV

```
apt-get update
apt-get install httpd2 apache2-mod_php8.1 php8.1 php8.1-mysqlnd php8.1-mysqli -y
systemctl enable --now httpd2

apt-get install mariadb-server -y
systemctl enable --now mariadb

mysql -u root
CREATE DATABASE webdb;
CREATE USER 'web'@'localhost' IDENTIFIED BY 'P@ssw0rd';
GRANT ALL PRIVILEGES ON webdb.* TO 'web'@'localhost';
FLUSH PRIVILEGES;
exit

mkdir /mnt/cdrom
mount /dev/sr0 /mnt/cdrom
ls /mnt/cdrom/web
mysql -u root webdb < /mnt/cdrom/web/dump.sql

cp /mnt/cdrom/web/index.php /var/www/html/
cp /mnt/cdrom/web/logo.png /var/www/html/
chown -R apache2:apache2 /var/www/html
chmod -R 777 /var/www/html

vim /var/www/html/index.php
# указываем:
$servername = "localhost";
$username = "web";
$password = "P@ssw0rd";
$dbname = "webdb";
systemctl restart httpd2
```

### 🐧 HQ-CLI

```
http://192.168.1.10  #открываем сайт через браузер с клиента
```


![zadanie-5](../pictures-m2/5-hq-srv-install-apache.png)

![zadanie-5](../pictures-m2/5-hq-srv-install-mariadb.png)

![zadanie-5](../pictures-m2/5-hq-srv-create-database.png)

![zadanie-5](../pictures-m2/5-hq-srv-import-db-copy-site-files.png)

![zadanie-5](../pictures-m2/5-hq-srv-config-connection-to-db.png)

![zadanie-5](../pictures-m2/5-hq-srv-restart-httpd2.png)

![zadanie-5](../pictures-m2/5-hq-cli-site-open-work-test.png)

## 6. Настройка Ansible на BR-SRV

> [!NOTE]
> Ansible — система управления конфигурациями, которая помогает автоматизировать настройку удалённых серверов в сети. Позволяет выполнять действия на множестве серверов одновременно.
> 
> Ansible ping — это модуль, который проверяет возможность подключения к управляемому узлу и наличие на нём работоспособного интерпретатора Python. По умолчанию при успешном выполнении модуль возвращает строку «pong». Использутся ssh для проверки подключения. Поэтому для начала подготавливаем его, чтобы ping сработал. Плэйбук для Ansible ping необязателен.

Перед началом настройки проверяем работу ssh: systemctl status sshd на HQ-SRV, HQ-CLI, BR-SRV. 

Запустить службу (systemctl enable --now sshd) пришлось только на HQ-CLI.

![zadanie-6](../pictures-m26-hq-cli-enable-sshd.png)

Поскольку стенд 2025 года, имеем созданного пользователя sshuser с uid 1010 и без сконфигурированного порта из модуля 1. Файл /etc/openssh/sshd_config не изменён. Исправим это и приведём к тому виду, в котором оно должно быть на HQ-SRV и BR-SRV. 

![zadanie-6](../pictures-m2/no-configured-port-but-needed-to-ssh.png)

### 🐧 HQ-SRV и BR-SRV

```
vim /etc/openssh/sshd_config
# раскомментируем и меняем эту строчку:
Port 2026

systemctl restart sshd
```

![zadanie-6](../pictures-m2/6-srv-sshd-port.png)

![zadanie-6](../pictures-m2/6-srv-sshd-restart-status.png)

У EcoRouter'ов SSH специально заблокирован по умолчанию, проверить можно с помощью show management. Исправляем:

### 🍃 HQ-RTR и BR-RTR

```
(config)#security-profile 0
rule 0 permit tcp any any eq 22
exit
security 0
exit

#write memory

# проверить:
show security-profile
```

![zadanie-6](../pictures-m2/6-ecorouter-ssh-allow.png)

После проверяем подключение по ssh. Для HQ-SRV и HQ-CLI проверка обязательна - на вопрос "Are you sure you want to continue connecting?" пишем: yes. Это важно — убирает "failed!" у ansible. Логиниться не обязательно

### 🐧 BR-SRV

```
ssh sshuser@192.168.1.10 -p 2026
ssh user@192.168.2.10
ssh admin@192.168.1.1
ssh admin@192.168.3.1
```

Устанавливаем ansible и настравиваем инвентарь

```
apt-get update
apt-get install ansible sshpass -y

vim /etc/ansible/hosts
# пишем на каждый хост в одну строчку, переносы не делаем
[hq]
HQ-SRV ansible_port=2026 ansible_host=192.168.1.10 ansible_user=sshuser ansible_ssh_pass=P@ssw0rd ansible_python_interpreter=/usr/bin/python3
HQ-CLI ansible_host=192.168.2.10 ansible_user=user ansible_ssh_pass=resu ansible_python_interpreter=/usr/bin/python3

[routers]
HQ-RTR ansible_host=192.168.1.1 ansible_user=admin ansible_password=admin ansible_connection=network_cli ansible_network_os=ios ansible_python_interpreter=/usr/bin/python3
BR-RTR ansible_host=192.168.3.1 ansible_user=admin ansible_password=admin ansible_connection=network_cli ansible_network_os=ios ansible_python_interpreter=/usr/bin/python3

ansible all -m ping
```

![zadanie-6](../pictures-m2/6-br-srv-ansible-inventory.png)

![zadanie-6](../pictures-m2/6-br-srv-ansiblle-successful-pong-answer.png)

## 7. Развёртывание веб-приложения в Docker на BR-SRV

> [!NOTE]
> Docker используется для быстрого и изолированного развёртывания приложений.
>
> В данном случае с его помощью поднимается стек из двух контейнеров: веб-приложение (testapp) и база данных (db)
>
> Это позволяет:
>
> - не устанавливать сервисы (apache и mariadb) напрямую в систему
>
> - обеспечить одинаковую работу приложения в любой среде
>
> - быстро развернуть связку web + database
>
> - обеспечить доступ к приложению извне через указанный порт
>
> Docker упрощает настройку и делает развёртывание одинаково воспроизводимым
>
> Этапы выполнения:
>
> 1) Установка Docker
>
> 2) Запуск и автозапуск Docker
>
> 3) Монтируем ISO
>
> 4) Импортируем образы в Docker
>
> 5) Создаём docker-compose.yml
>
> 6) Запускаем стек
>
> 7) Проверяем доступ
>
> В названии контейнера в задании скорее всего опечатка, поэтому оставлю testapp вместо tespapp

### 🐧 BR-SRV

```
apt-get update
apt-get install -y docker-engine docker-compose
systemctl enable --now docker

lsblk
mkdir /mnt/cdrom
mount /dev/sr0 /mnt/cdrom
ls /mnt/cdrom/docker

cd /mnt/cdrom/docker
docker load -i site_latest.tar
docker load -i mariadb_latest.tar

cd
mkdir /usr/docker
vim /usr/docker/docker-compose.yml
# прописываем
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
# проверить контейнеры
docker ps -a
```

![zadanie-7](../pictures-m2/7-br-srv-docker-load-images.png)

![zadanie-7](../pictures-m2/7-br-srv-docker-compose-file.png)

![zadanie-7](../pictures-m2/7-br-srv-docker-up-containers.png)

### 🐧 HQ-CLI

```
http://192.168.3.10:8080/
```

![zadanie-7](../pictures-m2/7-hq-cli-docker-check-web-app.png)

## 8. Настройка статической трансляции портов на маршрутизаторах

> [!NOTE]
> Доступ к веб-приложению и SSH осуществляется через статическую трансляцию портов на маршрутизаторах
> 
> Пробрасываем порт 80 (веб-приложение HQ-SRV), порт 8080 (веб-приложение на BR-SRV) на порт 8080 маршрутизаторов HQ-RTR и BR-RTR, чтобы ISP мог "открыть" эти веб-приложения
>
> Пробрасываем порт 2026 (подключение ssh к HQ-SRV), порт 2026 (подключение ssh к BR-SRV) на порт 2026 маршрутизаторов HQ-RTR и BR-RTR, чтобы ISP мог подключиться
>
> Статическая трансляция портов (DNAT) решает ряд задач:
> 
> - обеспечивает единую точку входа через маршрутизатор
> 
> - скрывает внутреннюю адресацию
> 
> - позволяет контролировать доступ к сервисам, открывая только необходимые порты
>
> Для доступа из внешних сетей в реальных условиях требуется использование публичного IP-адреса или дополнительных технологий (например, VPN).

### 🍃 HQ-RTR

```
(config)#ip nat source static tcp 192.168.1.10 80 172.16.4.4 8080
(config)#ip nat source static tcp 192.168.1.10 2026 172.16.4.4 2026

exit
#write memory
```

![zadanie-8](../pictures-m2/8-hq-rtr-dnat.png)

### 🍃 BR-RTR

```
(config)#ip nat source static tcp 192.168.3.10 8080 172.16.5.5 8080
(config)#ip nat source static tcp 192.168.3.10 2026 172.16.5.5 2026

exit
#write memory
```

![zadanie-8](../pictures-m2/8-br-rtr-dnat.png)

### 🐧 ISP

```
apt-get update
apt-get install curl -y

# проверяем подключение к сервисам
curl http://172.16.5.5:8080
curl http://172.16.4.4:8080

ssh sshuser@172.16.4.4 -p 2026
ssh sshuser@172.16.5.5 -p 2026
```

![zadanie-8](../pictures-m2/8-isp-dnat-check-work.png)

## 9. Настройка nginx как обратного прокси на ISP

> [!NOTE]
> Как это работает: пользователь открывает нужное доменное имя, а nginx на сервере ISP сам пересылает запрос дальше на внутренний сервер/контейнер.
>
> Это позволяет публиковать внутренние сервисы через один внешний узел без прямого доступа клиентов к внутренней сети.
>
> nginx — это высокопроизводительный веб-сервер и программный комплекс для обработки сетевых запросов.

Поскольку стенд 2025 года перед началом необходимо изменить конфигурацию DNS-сервера dnsmasq на HQ-SRV.
Открываем /etc/dnsmasq.conf, убираем cname записи, они нам не нужны и добавлем прямые записи.

![zadanie-9](../pictures-m2/9-hq-srv-change-dnsmasq.png)

Перезагружаем dnsmasq: systemctl restart dnsmasq

Также убеждаемся что на HQ-SRV работает сайт и Docker testapp запущен

### 🐧 ISP

```
apt-get update
apt-get install nginx -y
systemctl enable --now nginx

vim /etc/nginx/sites-enabled.d/reverse.conf
# создаём конфиг
server {
    listen 80;
    server_name web.au-team.irpo;

    location / {
        proxy_pass http://172.16.4.4:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

server {
    listen 80;
    server_name docker.au-team.irpo;

    location / {
        proxy_pass http://172.16.5.5:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

# проверка конфига
nginx -t

systemctl restart nginx
```

![zadanie-9](../pictures-m2/9-isp-nginx-config.png)

![zadanie-9](../pictures-m2/9-isp-nginx-check-syntax-and-restart.png)

### 🐧 HQ-CLI

```
# в строке браузера вводим:
http://web.au-team.irpo
http://docker.au-team.irpo
```

![zadanie-9](../pictures-m2/9-hq-cli-reverse-proxy-check-work-1.png)

![zadanie-9](../pictures-m2/9-hq-cli-reverse-proxy-check-work-2.png)

## 10. Настройка web-based аутентификации на ISP

> [!NOTE]
> Для web-based аутентификации в nginx используется механизм HTTP Basic Authentication
>
> Это встроенный способ защиты веб-ресурса логином и паролем без установки дополнительных приложений.
>
> При обращении к ресурсу nginx запрашивает логин и пароль, сверяет их с файлом /etc/nginx/.htpasswd и только после успешной проверки перенаправляет пользователя на внутренний веб-сервер.

### 🐧 ISP

```
apt-get update
apt-get install apache2-htpasswd -y

# создаём пользователя утилитой htpasswd
htpasswd -c /etc/nginx/.htpasswd WEB
# задаём пароль P@ssw0rd

vim /etc/nginx/sites-enabled.d/reverse.conf
# изменяем блок с web.au-team.irpo, после строчки server_name добавим:
auth_basic "Restricted Access";
auth_basic_user_file /etc/nginx/.htpasswd;

nginx -t
systemctl restart nginx
```

![zadanie-10](../pictures-m2/10-isp-htpasswd-install-create-user.png)

![zadanie-10](../pictures-m2/10-isp-change-nginx-config.png)

![zadanie-10](../pictures-m2/10-isp-test-restart-nginx.png)

### 🐧 HQ-CLI

```
http://web.au-team.irpo

# теперь получаем окно с авторизацией
# введём неправильные данные - окно с авторизацией перезагрузиться
# введём правильные данные - откроется сайт
# после закрывания вкладки и повторного ввода снова потребуется авторизация
# то есть авторизация срабатывает при каждой новой сессии
```

![zadanie-10](../pictures-m2/10-hq-cli-web-auth.png)

![zadanie-10](../pictures-m2/10-hq-cli-web-auth-login.png)

![zadanie-10](../pictures-m2/10-hq-cli-web-auth-login-success.png)

![zadanie-10](../pictures-m2/10-hq-cli-web-auth-login-cancel.png)

![zadanie-10](../pictures-m2/10-hq-cli-web-auth-login-in-terminal.png)

## 11. Настройка Samba DC на BR-SRV

> [!NOTE]
> Самое объёмное и чувствительное к ошибкам задание всего модуля.
>
> Samba DC это сразу несколько служб работающих вместе, сервер становиться:
> - контроллером домена
> - DNS сервером
> - Kerberos KDC
> - LDAP каталогом пользователей
>
> Что будем делать на BR-SRV:
> - Поднимем Samba AD DC
> - Домен au-team.irpo
> - Создадим пользователей
> - Создадим группу hq
>
> Что будем делать на HQ-CLI:
> - Введём в домен
> - Разрешим вход пользователям группы hq
> - Настроим sudo только cat grep id
> 
> У нас уже есть HQ-SRV как основной DNS (dnsmasq). Мы поступим так - Samba DC на BR-SRV будет DNS-сервером для домена au-team.irpo, а HQ-SRV оставим как внешний резолвер. Так правильнее и домен будет работать стабильнее. Поскольку Active Directory использует специальные SRV-записи.

Начнём с подготовки

В /etc/resolv.conf на HQ-CLI у нас "nameserver 192.168.1.10". Укажем 2 DNS, первым будет именно Samba DC (BR-SRV)

```
vim /etc/resolv.conf
# приводим к такому ввиду:
search au-team.irpo
nameserver 192.168.3.10
nameserver 192.168.1.10
```

![zadanie-11](../pictures-m2/11-hq-cli-change-resolv-conf.png)

На HQ-SRV добавим forwarding запросов к BR-SRV для домена в /etc/dnamasq.conf:

```
vim /etc/dnamasq.conf
# добавляем строчку:
server=/au-team.irpo/192.168.3.10

systemctl restart dnsmasq
```

![zadanie-11](../pictures-m2/11-hq-srv-add-forwarding-dnsmasq.png)

![zadanie-11](../pictures-m2/11-hq-srv-dnsmasq-restart.png)

### 🐧 BR-SRV

Часть 1

```
# 1. Правка hosts

vim /etc/hosts
# добавим:
127.0.0.1 localhost
192.168.3.10 BR-SRV.au-team.irpo BR-SRV

# 2. DNS на самом BR-SRV

vim /etc/resolv.conf
# приводим к этому ввиду:
search au-team.irpo
nameserver 192.168.3.10
nameserver 192.168.1.10

# 3. Установка пакетов

apt-get update
apt-get install task-samba-dc task-auth-ad-sssd -y

# 4. Очистка старой Samba

systemctl stop samba
rm -f /etc/samba/smb.conf
rm -rf /var/lib/samba
mkdir -p /var/lib/samba/sysvol

# 5. Создание домена

samba-tool domain provision
# в интерактивном режиме вводим/задаём параметры:
# (в первых 4-ёх можно просто нажимать Enter)
realm - AU-TEAM.IRPO
domain - AU-TEAM
server-role - dc
dns-backend - SAMBA_INTERNAL
dns-forwarder - 192.168.1.10
adminpass - P@ssw0rd!

# 6. Kerberos

cp /var/lib/samba/private/krb5.conf /etc/krb5.conf

# 7. Запуск

systemctl enable --now samba

# 8. Проверка

samba-tool domain level show
host -t SRV _ldap._tcp.au-team.irpo
kinit administrator
klist
host br-srv.au-team.irpo
```

![zadanie-11](../pictures-m2/11-br-srv-change-hosts.png)

![zadanie-11](../pictures-m2/11-br-srv-change-resolv.png)

![zadanie-11](../pictures-m2/11-br-srv-packages-install.png)

![zadanie-11](../pictures-m2/11-br-srv-clear-samba.png)

![zadanie-11](../pictures-m2/11-br-srv-create-domain.png)

![zadanie-11](../pictures-m2/11-br-srv-create-domain-result.png)

![zadanie-11](../pictures-m2/11-br-srv-kerberos-enable-samba.png)

![zadanie-11](../pictures-m2/11-br-srv-samba-check.png)

host br-srv.au-team.irpo выдаёт 172.17.0.1, это ip интерфейса docker0

При создании домена Samba автоматически выбрала один из интерфейсов и внесла его в DNS как A-запись хоста.

Произошёл выбор не того интерфейса, исправляем

```
samba-tool dns delete 127.0.0.1 au-team.irpo br-srv A 172.17.0.1 -U administrator
samba-tool dns add 127.0.0.1 au-team.irpo br-srv A 192.168.3.10 -U administrator

# проверка, должно быть 192.168.3.10
host br-srv.au-team.irpo
```

![zadanie-11](../pictures-m2/11-br-srv-solve-problem-wrong-ip-record-samba.png)

На будущее - остановить Docker перед тем как создавать домен

```
systemctl stop docker
```

Часть 2

```

```

![zadanie-11](../pictures-m2/)

![zadanie-11](../pictures-m2/)

![zadanie-11](../pictures-m2/)
