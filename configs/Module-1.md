# 📘 Модуль 1 — Пояснения к настройке сети 

1. [Произведите базовую настройку устройств](#1-произведите-базовую-настройку-устройств)
2. [Настройка IP-адресов](#2-настройка-ip-адресов)
3. [Настройте коммутацию в сегменте HQ следующим образом](#3-настройте-коммутацию-в-сегменте-hq-следующим-образом)
4. [Настройте часовой пояс на всех устройствах](#4-настройте-часовой-пояс-на-всех-устройствах)
5. [Настройте доступ к сети Интернет, на маршрутизаторе ISP](#5-настройте-доступ-к-сети-интернет-на-маршрутизаторе-isp)
6. [Создайте локальные учетные записи на серверах HQ-SRV и BR-SRV (и net_admin на HQ-RTR и BR-RTR)](#6-создайте-локальные-учетные-записи-на-серверах-hq-srv-и-br-srv-и-net_admin-на-hq-rtr-и-br-rtr)
7. [Настройте протокол динамической конфигурации хостов](#7-настройте-протокол-динамической-конфигурации-хостов)
8. [Настройка динамической трансляции адресов маршрутизаторах](#8-настройка-динамической-трансляции-адресов-маршрутизаторах)
9. [Между офисами HQ и BR необходимо сконфигурировать ip туннель](#9-между-офисами-hq-и-br-необходимо-сконфигурировать-ip-туннель)
10. [Обеспечьте динамическую маршрутизацию на маршрутизаторах HQ-RTR и BR-RTR](#10-обеспечьте-динамическую-маршрутизацию-на-маршрутизаторах-hq-rtr-и-br-rtr)
11. [Настройте безопасный удаленный доступ на серверах HQ-SRV и BR-SRV](#11-настройте-безопасный-удаленный-доступ-на-серверах-hq-srv-и-br-srv)
12. [Настройте инфраструктуру разрешения доменных имён для офисов HQ и BR](#12-настройте-инфраструктуру-разрешения-доменных-имён-для-офисов-hq-и-br)

## 1. Произведите базовую настройку устройств

### 🍃 Маршрутизаторы (HQ-RTR, BR-RTR)

Основные режимы конфигурирования EcoRouter:

1. ecorouter> - пользовательский режим. Режим по умолчанию при подключении. Доступны базовые команды просмотра состояния (например, show)

2. ecorouter#, команда enable - привилегированный режим. Позволяет просматривать детальную конфигурацию и выполнять диагностику.

3. ecorouter(config)#, команда configure terminal - глобальная конфигурация. Используется для изменения глобальных настроек

Сохранение конфигурации маршрутизатора - write memory (иначе после перезагрузки всё пропадёт)

```
ecorouter>en
ecorouter#conf t
ecorouter(config)#hostname hq-rtr.au-team.irpo
hq-rtr.au-team.irpo(config)#ip domain-name au-team.irpo
hq-rtr.au-team.irpo(config)#exit
hq-rtr.au-team.irpo#write memory
```

```
ecorouter>en
ecorouter#conf t
ecorouter(config)#hostname br-rtr.au-team.irpo
br-rtr.au-team.irpo(config)#ip domain-name au-team.irpo
br-rtr.au-team.irpo(config)#exit
br-rtr.au-team.irpo#write memory
```

![zadanie-1](../pictures-m1/1-hostname-ecorouter.png)

> [!NOTE]
> Имя устройства нужно для удобной идентификации и корректной работы DNS.
> Доменное имя устройства используется для генерации FQDN (полного имени) и работы DNS и SSH

### 🐧 ALT Linux (HQ-SRV, BR-SRV, HQ-CLI, ISP)

Устанавливаем hostname в системе Linux

```
hostnamectl set-hostname hq-srv.au-team.irpo;exec bash
hostnamectl set-hostname br-srv.au-team.irpo;exec bash
hostnamectl set-hostname hq-cli.au-team.irpo;exec bash
hostnamectl set-hostname ISP;exec bash
```

![zadanie-1](../pictures-m1/1-hostname-alt.png)

## 2. Настройка IP-адресов

(Согласно таблице ниже)

![ip-set-word-table](../pictures-m1/ip-set.png)

> [!NOTE]
> Не забываем сопоставить MAC-адрес интерфейса в системе с MAC-адресом соответствующего адаптера ВМ в среде виртуализации
> 
> Краткая шпаргалка по текстовому редактору VIM:
> 
> Нажать I на клавиатуре - режим редактирования текста
> 
> Нажать ESC на клавиатуре - выйти из текущего режима в командный
> 
> :wq - Записать изменения и выйти
> 
> :qa! - Выйти без применения изменений

### 🐧 ISP

```
echo HTTP_PROXY=http://10.0.21.52:3128 >> /etc/sysconfig/network
# Добавляет прокси (для выхода в интернет через прокси-сервер). Необходимо в условиях колледжской сети
# Для каждого региона используется свой прокси-сервер. У себя я его прописывать не буду.
ip a
# Показывает все сетевые интерфейсы и IP
ls /etc/net/ifaces
# Список сетевых интерфейсов (ALT Linux использует эту директорию)
mkdir /etc/net/ifaces/ens33
mkdir /etc/net/ifaces/ens34
# Создаёт папки для интерфейсов ens33 и ens34. Либо - mkdir /etc/net/ifaces/ens3{3,4}
vim /etc/net/ifaces/ens33/options
# Создание и открытие файла options через тексторый редактор vim
BOOTPROTO=dhcp
TYPE=eth
DISABLED=no
CONFIG_IPV4=yes
# Содержание файла options. Последние 2 строки необязательны во всех случаях
vim /etc/net/ifaces/ens33/options
BOOTPROTO=static
TYPE=eth
DISABLED=no
CONFIG_IPV4=yes
echo 172.16.4.1/28 > /etc/net/ifaces/ens33/ipv4address 
# Назначает статический IP
vim /etc/net/ifaces/ens34/options
BOOTPROTO=static
TYPE=eth
DISABLED=no
CONFIG_IPV4=yes
echo 172.16.5.1/28 > /etc/net/ifaces/ens34/ipv4address
systemctl restart network
# Перезагружаем службу для обновления настроек сети
```

![zadanie-2](../pictures-m1/2-isp-ls-mkdir.png)

![zadanie-2](../pictures-m1/2-isp-options.png)

![zadanie-2](../pictures-m1/2-isp-ipv4address.png)

### 🐧 HQ-SRV

```
vim /etc/net/ifaces/ens19/options
# options уже корректно настроен. Перепроверить не помешает. Если что-то не так - изменяем.
echo 192.168.100.2/26 > /etc/net/ifaces/ens19/ipv4address
echo default via 192.168.100.1 > /etc/net/ifaces/ens19/ipv4route
systemctl restart network
```

> [!NOTE]
> На этом моменте идёт особенность конфигурирования в VMware Workstation. В Proxmox показано выше (нумерация интерфейсов там другая, посмотрите самостоятельно)
> 
> Различие в интерфейсах HQ-SRV и HQ-CLI, поскольку в VMware Workstation нет VSwitch (как в ESXI)
>
> и нет присвоения VLAN-тэга из под инструментария (как в Proxmox), с помощью которых добавится VLAN тег.
> 
> Поэтому VLAN приходится делать внутри Linux (ens32.X). Ниже показаны команды.
> 
> Выставляется MTU на значение 1400 (Влановые интерфейсы чувствительны к этому параметру, обеспечивает ping и открытие веб-приложения в дальнейшем на HQ-CLI)

```
vim /etc/net/ifaces/ens32/options
TYPE=eth
BOOTPROTO=static
ONBOOT=yes
# Изменяем содержание к этому ввиду
mkdir /etc/net/ifaces/ens32.100
# VLAN 100, поэтому интерфейс ens32.100
vim /etc/net/ifaces/ens32.100/options
TYPE=vlan
HOST=ens32
VID=100
BOOTPROTO=static
ONBOOT=yes
echo 192.168.100.2/26 > /etc/net/ifaces/ens32.100/ipv4address
echo default via 192.168.100.1 > /etc/net/ifaces/ens32.100/ipv4route
echo mtu 1400 > /etc/net/ifaces/ens32.100/iplink
# Такая директория на alt linux для выставления нового значения MTU
# (файл применяет параметры в формате команды ip link)
systemctl restart network
```

![zadanie-2](../pictures-m1/2-hq-srv-ens32-options.png)

![zadanie-2](../pictures-m1/2-hq-srv-ens32.100-options.png)

![zadanie-2](../pictures-m1/2-hq-srv-echo-ip-mtu.png)

### 🐧 HQ-CLI

Для Proxmox

```
vim /etc/net/ifaces/ens32/options
# Изменяем следующие строчки
NM_CONTROLLED=no
DISABLED=no
BOOTPROTO=dhcp
echo default via 192.168.100.65 > /etc/net/ifaces/ens32/ipv4route
systemctl restart network
```

Для VMware Workstation

```
vim /etc/net/ifaces/ens32/options
TYPE=eth
BOOTPROTO=static
ONBOOT=yes
# Изменяем содержание к этому ввиду
mkdir /etc/net/ifaces/ens32.200
# VLAN 200, поэтому интерфейс ens32.200
vim /etc/net/ifaces/ens32.200/options
TYPE=vlan
HOST=ens32
VID=200
BOOTPROTO=dhcp
ONBOOT=yes
echo default via 192.168.100.65 > /etc/net/ifaces/ens32.200/ipv4route
echo mtu 1400 > /etc/net/ifaces/ens32.200/iplink
systemctl restart network
```

![zadanie-2](../pictures-m1/2-hq-cli-ens32-options.png)

![zadanie-2](../pictures-m1/2-hq-cli-ens32.200-options.png)

![zadanie-2](../pictures-m1/2-hq-cli-gateway-mtu.png)

### 🐧 BR-SRV

```
# options уже корректно настроен. Перепроверьте на всякий - не помешает.
echo 192.168.50.2/27 > /etc/net/ifaces/ens32/ipv4address
echo default via 192.168.50.1 > /etc/net/ifaces/ens32/ipv4route
systemctl restart network
```

![zadanie-2](../pictures-m1/2-br-srv-ip-gateway.png)

## 3. Настройте коммутацию в сегменте HQ следующим образом

### 🍃 HQ-RTR

VMware Workstation / Proxmox 

(В Proxmox интерфейсы te0 и te1 - поймёте где-будут отличия, no service-instance прописывать в proxmox не надо, там ненужной "сущности" по умолчанию нет)

```
#show run port
conf t
(config)#port ge0
no service-instance ge0
service-instance ge0/isp-hq
encapsulation untagged
exit
exit

(config)#port te0
service-instance te0/srv-net
encapsulation dot1q 100
rewrite pop 1
exit
service-instance te0/cli-net
encapsulation dot1q 200
rewrite pop 1
exit
service-instance te0/management
encapsulation dot1q 999
rewrite pop 1
exit
exit

(config)#interface eth1
ip address 172.16.4.2/28
connect port ge0 service-instance ge0/isp-hq
ip nat outside
exit

(config)#interface eth2
ip address 192.168.100.1/26
connect port te0 service-instance te0/srv-net
ip nat inside
exit

(config)#interface eth3
ip address 192.168.100.65/28
connect port te0 service-instance te0/cli-net
ip nat inside
exit

(config)#interface eth4
ip address 192.168.100.81/29
connect port te0 service-instance te0/management
ip nat inside
exit
exit

#write мемоry
```

![zadanie-3](../pictures-m1/3-hq-rtr-show-run-port.png)

![zadanie-3](../pictures-m1/3-hq-rtr-port-service-instance.png)

![zadanie-3](../pictures-m1/3-hq-rtr-int-config.png)

### 🍃 BR-RTR

```
(config)#port ge0
no service-instance ge0
service-instance ge0/isp-br
encapsulation untagged
exit
exit

(config)#port te0
service-instance te0/br-net
encapsulation-untagged
exit
exit

(config)#interface eth1
ip address 172.16.5.2/28
connect port ge0 service-instance ge0/isp-br
ip nat outside
exit

(config)#interface eth2
ip address 192.168.50.1/27
connect port te0 service-instance te0/br-net
ip nat inside
exit
exit

#write memory
```

![zadanie-3](../pictures-m1/3-br-rtr-port-int-conf.png)

Можно проверить пинги на данном этапе - ISP↔HQ-RTR, ISP↔BR-RTR, HQ-RTR↔HQ-SRV, BR-RTR↔BR-SRV

> [!NOTE]
> Проверьте включён ли адаптер в сторону LAN у HQ-RTR и BR-RTR в WMware - галочка на "Connect at power on". Говорю, потому что у меня эта галочка не была проставлена после ping проверок. Поэтому часть пингов не проходила.

## 4. Настройте часовой пояс на всех устройствах

### 🐧 ALT Linux (HQ-SRV, BR-SRV, HQ-CLI)

```
timedatectl set-timezone Europe/Moscow
```

![zadanie-4](../pictures-m1/4-srv-cli-timezone.png)

### 🐧 ISP

```
apt-get update
apt-get install tzdata -y
exec bash
timedatectl set-timezone Europe/Moscow
```

![zadanie-4](../pictures-m1/4-isp-timezone.png)

### 🍃 EcoRouters (HQ-RTR, BR-RTR)

```
(config)#ntp timezone utc+3
exit
#write memory
```

![zadanie-4](../pictures-m1/4-ecorouter-timezone.png)

## 5. Настройте доступ к сети Интернет, на маршрутизаторе ISP

> [!NOTE]
> Настраиваем NAT с помощью nftables, обеспечивая подмену всех внутренних IP-адресов на IP-адрес внешнего интерфейса ISP для исходящего трафика. Используем конструкцию "cat <<'EOT' > " для передачи многострочного текста в команду. Команда cat в Linux может работать не только с файлами, но и с потоком ввода (stdin). Если ей не указывать файл, она начинает получать данные из терминала. EOT здесь выступает в роли маркера, который указывает на завершение ввода данных для команды cat.
>
> Соблюдаем отступы. Также можно просто vim'ом открыть файл и вписать содержимое. Но так круче)
>
> Включаем глобальную маршрутизацию в Linux. Лучше именно так, после перезагрузки значение на 0 сбрасываться не будет.
>
> Либо echo net.ipv4.ip_forward=1 >> /etc/sysctl.conf, sysctl -p и потом каждый раз после reboot'а прописывать sysctl -p.

### 🐧 ISP

```
apt-get install nftables -y
cat <<'EOT' > /etc/nftables/nftables.nft
#!/usr/sbin/nft -f
flush ruleset
table ip nat {
 chain postrouting {
  type nat hook postrouting priority srcnat
  oifname "ens32" masquerade
 }
}
EOT
systemctl enable --now nftables

touch /etc/rc.d/rc.local
echo -e '#!/bin/bash\necho 1 > /proc/sys/net/ipv4/ip_forward' > /etc/rc.d/rc.local
chmod +x /etc/rc.d/rc.local
/etc/rc.d/rc.local
cat /proc/sys/net/ipv4/ip_forward  # для проверки
```

![zadanie-5](../pictures-m1/5-isp-nat.png)

![zadanie-5](../pictures-m1/5-isp-nftables-enable.png)

![zadanie-5](../pictures-m1/5-isp-global-routing.png)

## 6. Создайте локальные учетные записи на серверах HQ-SRV и BR-SRV (и net_admin на HQ-RTR и BR-RTR)

> [!NOTE]
> Создаём пользователя с ID 2026, нужным паролем, добавляем в группу wheel - для доступа этому пользователю к
> привилегированным командам (sudo) и позволяем ему вводить привилегированные команды (sudo) без ввода пароля учетки.
>
> На EcoRouter'ах создаём пользователя, назначаем пароль и выдаём ему максимальные привилегии

### 🐧 HQ-SRV и BR-SRV

```
useradd -s /bin/bash -u 2026 sshuser
echo "sshuser:P@ssw0rd" | chpasswd
gpasswd -a sshuser wheel
echo 'sshuser ALL = (root) NOPASSWD: ALL' >> /etc/sudoers
```

![zadanie-6](../pictures-m1/6-srv-sshuser-create.png)

### 🍃 Маршрутизаторы (HQ-RTR, BR-RTR)

```
(config)username net_admin
password P@ssw0rd
role admin
exit
exit
#write memory
```

![zadanie-6](../pictures-m1/6-ecorouter-net-admin-create.png)

## 7. Настройте протокол динамической конфигурации хостов

> [!NOTE]
> Создаём пул адресов для dhcp 1. В dhcp-server 1 добавляем созданный пул и настраиваем параметры, которые подхватит HQ-CLI. Применям dhcp-server 1 на interface eth3 (во VLAN 200 - сети HQ-CLI)

### 🍃 HQ-RTR

```
(config)#ip pool dhcp 1
range 192.168.100.66-192.168.100.78
exit

(config)#dhcp-server 1
pool dhcp 64
mask 255.255.255.240
gateway 192.168.100.65
dns 192.168.100.2
domain-name au-team.irpo
exit
exit

(config)#interface eth3
dhcp-server 1
exit
exit
#write memory
```

![zadanie-7](../pictures-m1/7-ecorouter-dhcp.png)

## 8. Настройка динамической трансляции адресов маршрутизаторах

> [!NOTE]
> pool nat создаём. На HQ-RTR - диапазон адресов VLAN 100, VLAN 200, VLAN 999. На BR-RTR - диапазон BR-сети.
> Настраиваем динамическую трансляцию исходящих адресов (PAT/NAT Overload) для всех внутренних сетей при выходе через внешние интерфейсы (isp-hq и isp-br - eth1 у обоих EcoRouter'ов)

### 🍃 HQ-RTR

```
(config)#ip nat pool nat 192.168.100.1-192.168.100.78,192.168.100.81-192.168.100.86
(config)#ip nat source dynamic inside-to-outside pool nat overload interface eth1
exit
write memory
```

![zadanie-8](../pictures-m1/8-hq-rtr-pat.png)

### 🍃 BR-RTR

```
(config)#ip nat pool nat 192.168.50.1-192.168.50.30
(config)#ip nat source dynamic inside-to-outside pool nat overload interface eth1
exit
write memory
```

![zadanie-8](../pictures-m1/8-br-rtr-pat.png)

## 9. Между офисами HQ и BR необходимо сконфигурировать ip туннель

> [!NOTE]
> Тип туннеля - GRE. Это будет логическая связка между HQ и BR сетями. Сеть туннеля может быть любой, как и пароль, но он должен совпадать с паролем на другом RTR. Включаем защиту протокола динамической маршрутизации (OSPF) для того, чтобы другие "левые" роутеры не смогли установить ospf соседство без указания пароля. Создаём маршрут по умолчанию, чтобы роутеры HQ-RTR и BR-RTR отправляли весь "неизвестный" им трафик на маршрутизатор ISP, который уже дальше отправит его в интернет

### 🍃 HQ-RTR

```
(config)#ip route 0.0.0.0/0 172.16.4.1 description default

(config)#interface tunnel.1
ip add 192.168.10.1/30
ip tunnel 172.16.4.2 172.16.5.2 mode gre
ip ospf authentication
ip ospf authentication-key P@$$word
exit
exit

#write memory
```

![zadanie-9](../pictures-m1/9-hq-rtr-ip-route-gre.png)

### 🍃 BR-RTR

```
(config)#ip route 0.0.0.0/0 172.16.5.1 description default

(config)#interface tunnel.1
ip add 192.168.10.2/30
ip tunnel 172.16.5.2 172.16.4.2 mode gre
ip ospf authentication
ip ospf authentication-key P@$$word
exit
exit

#write memory
```

![zadanie-9](../pictures-m1/9-br-rtr-ip-route-gre.png)

## 10. Обеспечьте динамическую маршрутизацию на маршрутизаторах HQ-RTR и BR-RTR

> [!NOTE]
> Используем протокол OSPF. Запускаем его (процесс №1), объявляем сети, устанавливаем OSPF-соседство только через GRE-туннель. Убеждаемся в работе туннеля - ping от hq-srv или hq-cli до br-srv и обратно.

### 🍃 HQ-RTR

```
(config)#router ospf 1
network 192.168.10.0/30 area 0.0.0.0
network 192.168.100.0/26 area 0.0.0.0
network 192.168.100.64/28 area 0.0.0.0
network 192.168.100.80/29 area 0.0.0.0
passive-interface default
no passive-interface tunnel.1
exit
exit

#write memory
# Проверки:
show ip ospf neighbor
show ip route ospf
```

![zadanie-10](../pictures-m1/10-hq-rtr-ospf.png)

### 🍃 BR-RTR

```
(config)#router ospf 1
network 192.168.10.0/30 area 0.0.0.0
network 192.168.50.0/27 area 0.0.0.0
passive-interface default
no passive-interface tunnel.1
exit
exit

#write memory
# Проверки:
show ip ospf neighbor
show ip route ospf
```

![zadanie-10](../pictures-m1/10-br-rtr-ospf.png)

## 11. Настройте безопасный удаленный доступ на серверах HQ-SRV и BR-SRV

> [!NOTE]
> Выставлем в конфигурационном файле подключение по ssh только на 2026-ой порт, кол-во попыток аутентификации - 2, вписываем и создаём отображающийся при подключении баннер. Разрешаем подключаться только под пользователем sshuser.
>
> Проверяем. с HQ-SRV - ssh sshuser@192.168.50.2 -p 2026. с BR-SRV - ssh sshuser@192.168.100.2 -p 2026

### 🐧 HQ-SRV и BR-SRV

```
vim /etc/openssh/sshd_config
# Раскомментируем и меняем эти строчки:
Port 2026
MaxAuthTries 2
Banner /etc/openssh/banner
# В конце добавлем:
AllowUsers sshuser
echo Authorized access only! > /etc/openssh/banner

systemctl restart sshd
```

![zadanie-11](../pictures-m1/11-srv-ssh-port-auth-tries.png)

![zadanie-11](../pictures-m1/11-srv-ssh-banner-allow-users.png)

![zadanie-11](../pictures-m1/11-srv-ssh-restart-banner.png)

![zadanie-11](../pictures-m1/11-srv-ssh-test.png)

## 12. Настройте инфраструктуру разрешения доменных имён для офисов HQ и BR

> [!NOTE]
> dnsmasq — лёгкий DNS-сервер, no-resolv — игнорировать /etc/resolv.conf, no-poll — не отслеживать изменения
> 
> no-hosts — не читать /etc/hosts, server=... - прописывание публичных DNS серверов: Yandex, MSK-IX, Google
> 
> cache-size — хранит ответы DNS в памяти, all-servers — отправляет запрос всем DNS и берёт самый быстрый
> 
> no-negcache — не кэшировать ошибки, host-record=... - прямые A DNS-записи
> 
> ptr-record=... - обратные (Связывают IP → домен)
>
> Минимальные проверки:
>
> nslookup hq-srv.au-team.irpo
> 
> nslookup br-srv.au-team.irpo
> 
> nslookup docker.au-team.irpo
>
> nslookup web.au-team.irpo
> 
> nslookup 192.168.50.2
>
> ping hq-rtr.au-team.irpo
> 
> ping br-rtr.au-team.irpo

> [!WARNING]
> apt-get update работать не будет, так как ещё нет настроенного dns-сервера, который будет резолвить имена. Для того чтобы установить dnsmasq временно впишем google dns в /etc/resolv.conf на HQ-SRV:
> 
> vim /etc/resolv.conf
> 
> nameserver 8.8.8.8
>
> Установив dnsmasq - уберём добавленную строчку из /etc/resolv.conf. После поднятия dnsmasq HQ-SRV будет нашим DNS-сервером
>
> После настройки, чтобы BR-SRV похватил DNS-сервер в /etc/resolv.conf пропишем nameserver 192.168.100.2

### 🐧 HQ-SRV

```
apt-get update
apt-get install dnsmasq -y
echo 'OPTIONS=""' > /etc/sysconfig/dnsmasq
service network restart

cat <<'EOT' > /etc/dnsmasq.conf
no-resolv
no-poll
no-hosts

server=77.88.8.7
server=77.88.8.3
server=195.208.4.1
server=195.208.5.1
server=8.8.8.8

cache-size=1000
all-servers
no-negcache

host-record=hq-rtr.au-team.irpo,192.168.100.1
host-record=hq-srv.au-team.irpo,192.168.100.2
host-record=hq-cli.au-team.irpo,192.168.100.66
host-record=br-rtr.au-team.irpo,192.168.50.1
host-record=br-srv.au-team.irpo,192.168.50.2

host-record=docker.au-team.irpo,172.16.4.1
host-record=web.au-team.irpo,172.16.5.1

ptr-record=1.100.168.192.in-addr.arpa,hq-rtr.au-team.irpo
ptr-record=2.100.168.192.in-addr.arpa,hq-srv.au-team.irpo
ptr-record=66.100.168.192.in-addr.arpa,hq-cli.au-team.irpo
EOT

systemctl enable --now dnsmasq
```

![zadanie-12](../pictures-m1/12-hq-srv-dnsmasq-install.png)

![zadanie-12](../pictures-m1/12-hq-srv-dnsmasq-config.png)

![zadanie-12](../pictures-m1/12-hq-cli-dnsmasq-test.png)

![zadanie-12](../pictures-m1/

![zadanie-12](../pictures-m1/

![zadanie-12](../pictures-m1/
