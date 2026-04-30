# ⏱ Модуль 1 — Быстрый готовый конфиг с командами без лишнего 🔥

## Заточен под реальный стенд на Proxmox у меня в колледже

### 🐧 ISP

```
hostnamectl set-hostname ISP;exec bash
echo HTTP_PROXY=http://10.0.21.52:3128 >> /etc/sysconfig/network
mkdir /etc/net/ifaces/ens19
mkdir /etc/net/ifaces/ens20
vim /etc/net/ifaces/ens19/options


```
