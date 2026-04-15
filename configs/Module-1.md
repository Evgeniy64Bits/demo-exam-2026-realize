# 📘 Модуль 1 — Пояснения к настройке сети 

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

> [!NOTE]
> Имя устройства нужно для удобной идентификации и корректной работы DNS.
> Доменное имя устройства используется для генерации FQDN (полного имени) и работы DNS и SSH

### 🐧 ALT Linux (HQ-SRV, BR-SRV, HQ-CLI, ISP)

Устанавливаем hostname в системе Linux

```
hostnamectl set-hostname hq-srv.au-team.irpo
hostnamectl set-hostname br-srv.au-team.irpo
hostnamectl set-hostname hq-cli.au-team.irpo
hostnamectl set-hostname ISP
```

Настройка IP-адресов






