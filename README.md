# 🖧 Настройка сетевой инфраструктуры на Debian 12

![Debian](https://img.shields.io/badge/Debian-12-red?style=for-the-badge&logo=debian)
![Linux](https://img.shields.io/badge/Linux-Networking-yellow?style=for-the-badge&logo=linux)
![License](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)

> **Методичка** по выполнению задания ГИА ДЭ (Демонстрационный экзамен)  
> Специальность: 09.02.06 — Сетевое и системное администрирование  
> КОД 09.02.06-1-2026

---

## 📋 Оглавление

- [Описание проекта](#описание-проекта)
- [Топология сети](#топология-сети)
- [Модуль 1. Настройка сетевой инфраструктуры](#модуль-1-настройка-сетевой-инфраструктуры)
  - [0. Подготовка и планирование](#0-подготовка-и-планирование)
  - [1. Настройка имён устройств](#1-настройка-имён-устройств)
  - [2. Настройка ISP](#2-настройка-isp)
  - [3. Базовая настройка маршрутизаторов](#3-базовая-настройка-маршрутизаторов)
  - [4. Настройка VLAN-коммутации](#4-настройка-vlan-коммутации)
  - [5. NAT на маршрутизаторах](#5-nat-на-маршрутизаторах)
  - [6. Создание учётных записей](#6-создание-учётных-записей)
  - [7. Настройка SSH](#7-настройка-ssh)
  - [8. GRE-туннель](#8-gre-туннель)
  - [9. Динамическая маршрутизация OSPF](#9-динамическая-маршрутизация-ospf)
  - [10. DHCP-сервер](#10-dhcp-сервер)
  - [11. DNS-сервер](#11-dns-сервер)
  - [12. Часовой пояс](#12-часовой-пояс)
- [Модуль 2. Организация сетевого администрирования](#модуль-2-организация-сетевого-администрирования)
  - [1. Samba DC (контроллер домена)](#1-samba-dc-контроллер-домена)
  - [2. RAID-массив](#2-raid-массив)
  - [3. NFS-сервер](#3-nfs-сервер)
  - [4. Chrony (NTP)](#4-chrony-ntp)
  - [5. Ansible](#5-ansible)
  - [6. Docker](#6-docker)
  - [7. Веб-приложение Apache + MariaDB](#7-веб-приложение-apache--mariadb)
  - [8. Проброс портов](#8-проброс-портов)
  - [9. Nginx reverse proxy](#9-nginx-reverse-proxy)
  - [10. Web-аутентификация](#10-web-аутентификация)
  - [11. Яндекс Браузер](#11-яндекс-браузер)
- [Полезные команды](#полезные-команды)

---

## Описание проекта

Данная методичка описывает **пошаговую** настройку сетевой инфраструктуры из 6 виртуальных машин на базе **Debian 12**, включая:

- ✅ Расчёт IP-адресации и подсетей
- ✅ Настройка VLAN-коммутации
- ✅ NAT и маршрутизация
- ✅ GRE-туннель между офисами
- ✅ Динамическая маршрутизация (OSPF)
- ✅ DHCP и DNS серверы
- ✅ SSH с ограничениями доступа
- ✅ Samba DC, RAID, NFS, Ansible, Docker, Nginx

---

## Топология сети
<img width="417" height="473" alt="image" src="https://github.com/user-attachments/assets/481124c6-1a34-402b-8612-3854c30da323" />

### Виртуальные машины

| ВМ | RAM | CPU | Диск | ОС |
|:---|:---:|:---:|:----:|:---|
| ISP | 1 ГБ | 1 | 5 ГБ | Debian 12 |
| HQ-RTR | 1 ГБ | 1 | 10 ГБ | Debian 12 |
| BR-RTR | 1 ГБ | 1 | 10 ГБ | Debian 12 |
| HQ-SRV | 2 ГБ | 1 | 10 ГБ | Debian 12 |
| BR-SRV | 2 ГБ | 1 | 10 ГБ | Debian 12 |
| HQ-CLI | 2 ГБ | 2 | 15 ГБ | Debian 12 |

---

## Модуль 1. Настройка сетевой инфраструктуры

---

### 0. Подготовка и планирование

#### 0.1. Таблица IP-адресации

| Сеть | Подсеть | Маска | CIDR | Назначение |
|:-----|:--------|:------|:----:|:-----------|
| ISP ↔ HQ-RTR | 172.16.1.0 | 255.255.255.240 | /28 | WAN HQ |
| ISP ↔ BR-RTR | 172.16.2.0 | 255.255.255.240 | /28 | WAN BR |
| VLAN 100 (HQ-SRV) | 192.168.100.0 | 255.255.255.224 | /27 | ≤32 адреса |
| VLAN 200 (HQ-CLI) | 192.168.200.0 | 255.255.255.224 | /27 | ≥16 адресов |
| VLAN 999 (управление) | 192.168.99.0 | 255.255.255.248 | /29 | ≤8 адресов |
| BR-SRV | 192.168.10.0 | 255.255.255.240 | /28 | ≤16 адресов |
| GRE-туннель | 10.0.0.0 | 255.255.255.252 | /30 | Точка-точка |

#### 0.2. Назначение IP-адресов

| Устройство | Интерфейс | IP-адрес | Шлюз |
|:-----------|:----------|:---------|:-----|
| **ISP** | ens18 (Интернет) | DHCP | — |
| **ISP** | ens19 (→HQ-RTR) | 172.16.1.1/28 | — |
| **ISP** | ens20 (→BR-RTR) | 172.16.2.1/28 | — |
| **HQ-RTR** | ens18 (→ISP) | 172.16.1.2/28 | 172.16.1.1 |
| **HQ-RTR** | ens19.100 | 192.168.100.1/27 | — |
| **HQ-RTR** | ens19.200 | 192.168.200.1/27 | — |
| **HQ-RTR** | ens19.999 | 192.168.99.1/29 | — |
| **HQ-RTR** | gre1 | 10.0.0.1/30 | — |
| **BR-RTR** | ens18 (→ISP) | 172.16.2.2/28 | 172.16.2.1 |
| **BR-RTR** | ens19 (→BR-SRV) | 192.168.10.1/28 | — |
| **BR-RTR** | gre1 | 10.0.0.2/30 | — |
| **HQ-SRV** | ens18.100 | 192.168.100.2/27 | 192.168.100.1 |
| **HQ-CLI** | ens18.200 | DHCP | 192.168.200.1 |
| **BR-SRV** | ens18 | 192.168.10.2/28 | 192.168.10.1 |

> ⚠️ **Важно:** Имена интерфейсов (`ens18`, `ens19`) зависят от гипервизора.
> - **Proxmox/KVM:** `ens18`, `ens19`, `ens20`
> - **VirtualBox:** `enp0s3`, `enp0s8`, `enp0s9`
> - **VMware:** `ens33`, `ens34`, `ens35`
> 
> Узнайте свои интерфейсы командой: `ip link show`

#### 0.3. Сетевые подключения ВМ в гипервизоре

| ВМ | NIC1 | NIC2 | NIC3 |
|:---|:-----|:-----|:-----|
| ISP | Bridge/NAT (Интернет) | Внутр. сеть `ISP-HQ` | Внутр. сеть `ISP-BR` |
| HQ-RTR | Внутр. сеть `ISP-HQ` | Внутр. сеть `HQ-LAN` | — |
| BR-RTR | Внутр. сеть `ISP-BR` | Внутр. сеть `BR-LAN` | — |
| HQ-SRV | Внутр. сеть `HQ-LAN` | — | — |
| HQ-CLI | Внутр. сеть `HQ-LAN` | — | — |
| BR-SRV | Внутр. сеть `BR-LAN` | — | — |

#### 0.4. Предварительные действия на каждой ВМ

После установки Debian 12 выполните на каждой машине:

# Узнайте имена интерфейсов
ip link show

# Временный DNS
echo "nameserver 77.88.8.7" > /etc/resolv.conf

---

1. Настройка имён устройств
На каждой машине выполните соответствующую команду:

Bash

# ISP
hostnamectl set-hostname isp.au-team.irpo

# HQ-RTR
hostnamectl set-hostname hq-rtr.au-team.irpo

# BR-RTR
hostnamectl set-hostname br-rtr.au-team.irpo

# HQ-SRV
hostnamectl set-hostname hq-srv.au-team.irpo

# BR-SRV
hostnamectl set-hostname br-srv.au-team.irpo

# HQ-CLI
hostnamectl set-hostname hq-cli.au-team.irpo
Проверка:

Bash

hostnamectl
2. Настройка ISP
2.1. Сетевые интерфейсы ISP
Bash

nano /etc/network/interfaces
ini

source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

# Интерфейс к Интернету (DHCP)
auto ens18
iface ens18 inet dhcp

# Интерфейс в сторону HQ-RTR (172.16.1.0/28)
auto ens19
iface ens19 inet static
    address 172.16.1.1
    netmask 255.255.255.240

# Интерфейс в сторону BR-RTR (172.16.2.0/28)
auto ens20
iface ens20 inet static
    address 172.16.2.1
    netmask 255.255.255.240
2.2. Включение маршрутизации
Bash

echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
2.3. NAT (MASQUERADE)
Bash

apt update && apt install -y iptables iptables-persistent

# Создаём правило NAT
iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE

# Сохраняем
netfilter-persistent save
2.4. Перезапуск и проверка
Bash

systemctl restart networking

# Проверка
ping -c 3 77.88.8.7
ip addr show
3. Базовая настройка маршрутизаторов
3.1. HQ-RTR — интерфейс к ISP
Bash

nano /etc/network/interfaces
ini

source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

# Интерфейс к ISP
auto ens18
iface ens18 inet static
    address 172.16.1.2
    netmask 255.255.255.240
    gateway 172.16.1.1
    dns-nameservers 77.88.8.7
Bash

# Включаем маршрутизацию
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p

# Перезапускаем сеть
systemctl restart networking

# Проверяем
ping -c 3 77.88.8.7

# Устанавливаем необходимые пакеты
apt update
apt install -y vlan iptables iptables-persistent isc-dhcp-server frr

# Включаем модуль VLAN
modprobe 8021q
echo "8021q" >> /etc/modules
3.2. BR-RTR — интерфейс к ISP
Bash

nano /etc/network/interfaces
ini

source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

# Интерфейс к ISP
auto ens18
iface ens18 inet static
    address 172.16.2.2
    netmask 255.255.255.240
    gateway 172.16.2.1
    dns-nameservers 77.88.8.7

# Интерфейс к BR-SRV
auto ens19
iface ens19 inet static
    address 192.168.10.1
    netmask 255.255.255.240
Bash

echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
systemctl restart networking

apt update
apt install -y iptables iptables-persistent frr
4. Настройка VLAN-коммутации
4.1. VLAN-субинтерфейсы на HQ-RTR
Bash

nano /etc/network/interfaces
ini

source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

# Интерфейс к ISP
auto ens18
iface ens18 inet static
    address 172.16.1.2
    netmask 255.255.255.240
    gateway 172.16.1.1
    dns-nameservers 77.88.8.7

# Транковый интерфейс (без IP)
auto ens19
iface ens19 inet manual
    up ip link set $IFACE up
    down ip link set $IFACE down

# VLAN 100 — сеть HQ-SRV
auto ens19.100
iface ens19.100 inet static
    address 192.168.100.1
    netmask 255.255.255.224
    vlan-raw-device ens19

# VLAN 200 — сеть HQ-CLI
auto ens19.200
iface ens19.200 inet static
    address 192.168.200.1
    netmask 255.255.255.224
    vlan-raw-device ens19

# VLAN 999 — управление
auto ens19.999
iface ens19.999 inet static
    address 192.168.99.1
    netmask 255.255.255.248
    vlan-raw-device ens19
Bash

systemctl restart networking

# Проверка: должны появиться ens19.100, ens19.200, ens19.999
ip addr show
4.2. VLAN на HQ-SRV
Bash

apt update && apt install -y vlan
modprobe 8021q
echo "8021q" >> /etc/modules

nano /etc/network/interfaces
ini

source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

# Физический интерфейс (без IP)
auto ens18
iface ens18 inet manual
    up ip link set $IFACE up
    down ip link set $IFACE down

# VLAN 100
auto ens18.100
iface ens18.100 inet static
    address 192.168.100.2
    netmask 255.255.255.224
    gateway 192.168.100.1
    vlan-raw-device ens18
Bash

systemctl restart networking
echo "nameserver 77.88.8.7" > /etc/resolv.conf
4.3. VLAN на HQ-CLI
Bash

apt install -y vlan
modprobe 8021q
echo "8021q" >> /etc/modules

nano /etc/network/interfaces
Временно статический IP (позже переключим на DHCP):

ini

source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

auto ens18
iface ens18 inet manual
    up ip link set $IFACE up
    down ip link set $IFACE down

auto ens18.200
iface ens18.200 inet static
    address 192.168.200.2
    netmask 255.255.255.224
    gateway 192.168.200.1
    vlan-raw-device ens18
Bash

systemctl restart networking
4.4. BR-SRV (без VLAN)
Bash

nano /etc/network/interfaces
ini

source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

auto ens18
iface ens18 inet static
    address 192.168.10.2
    netmask 255.255.255.240
    gateway 192.168.10.1
Bash

systemctl restart networking
echo "nameserver 77.88.8.7" > /etc/resolv.conf
5. NAT на маршрутизаторах
5.1. HQ-RTR
Bash

iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE
netfilter-persistent save
5.2. BR-RTR
Bash

iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE
netfilter-persistent save
5.3. Проверка Интернета со всех машин
Bash

# С HQ-SRV
ping -c 3 192.168.100.1    # шлюз
ping -c 3 172.16.1.1        # ISP
ping -c 3 77.88.8.7         # Интернет

# С BR-SRV
ping -c 3 192.168.10.1      # шлюз
ping -c 3 77.88.8.7         # Интернет
6. Создание учётных записей
6.1. sshuser на HQ-SRV и BR-SRV
Выполните одинаковые команды на обоих серверах

Bash

# Создаём пользователя с UID 2026
useradd -m -u 2026 -s /bin/bash sshuser

# Задаём пароль
echo "sshuser:P@ssw0rd" | chpasswd

# sudo без пароля
apt install -y sudo
echo "sshuser ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/sshuser
chmod 440 /etc/sudoers.d/sshuser

# Проверка
id sshuser
# uid=2026(sshuser) ...
6.2. net_admin на HQ-RTR и BR-RTR
Выполните одинаковые команды на обоих маршрутизаторах

Bash

useradd -m -s /bin/bash net_admin
echo "net_admin:P@ssw0rd" | chpasswd

apt install -y sudo
echo "net_admin ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/net_admin
chmod 440 /etc/sudoers.d/net_admin
7. Настройка SSH
Выполните на HQ-SRV и BR-SRV (одинаково)

Bash

apt install -y openssh-server

# Создаём баннер
echo "Authorized access only" > /etc/ssh/banner

# Редактируем конфигурацию
nano /etc/ssh/sshd_config
Найдите и измените/добавьте строки:

ini

Port 2026
AllowUsers sshuser
MaxAuthTries 2
Banner /etc/ssh/banner
Bash

systemctl restart sshd

# Проверка
ss -tlnp | grep 2026
Параметр	Значение	Описание
Port	2026	Нестандартный порт SSH
AllowUsers	sshuser	Только этот пользователь может подключаться
MaxAuthTries	2	Максимум 2 попытки ввода пароля
Banner	/etc/ssh/banner	Баннер при подключении
8. GRE-туннель
8.1. На HQ-RTR
Добавьте в конец /etc/network/interfaces:

Bash

nano /etc/network/interfaces
ini

# GRE-туннель к BR-RTR
auto gre1
iface gre1 inet static
    address 10.0.0.1
    netmask 255.255.255.252
    pre-up ip tunnel add gre1 mode gre remote 172.16.2.2 local 172.16.1.2 ttl 255
    post-down ip tunnel del gre1
Bash

ifup gre1
8.2. На BR-RTR
Добавьте в конец /etc/network/interfaces:

Bash

nano /etc/network/interfaces
ini

# GRE-туннель к HQ-RTR
auto gre1
iface gre1 inet static
    address 10.0.0.2
    netmask 255.255.255.252
    pre-up ip tunnel add gre1 mode gre remote 172.16.1.2 local 172.16.2.2 ttl 255
    post-down ip tunnel del gre1
Bash

ifup gre1
8.3. Проверка
Bash

# С HQ-RTR
ping -c 3 10.0.0.2

# С BR-RTR
ping -c 3 10.0.0.1
✅ Если пинг проходит — туннель работает!

9. Динамическая маршрутизация OSPF
Используем FRRouting (FRR).

9.1. HQ-RTR
Bash

# Включаем OSPF-демон
nano /etc/frr/daemons
Измените строку:

text

ospfd=yes
Bash

systemctl restart frr
systemctl enable frr

# Входим в консоль маршрутизации
vtysh
Введите конфигурацию:

text

configure terminal

router ospf
 network 10.0.0.0/30 area 0
 network 192.168.100.0/27 area 0
 network 192.168.200.0/27 area 0
 network 192.168.99.0/29 area 0
 passive-interface default
 no passive-interface gre1
exit

interface gre1
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 P@ssw0rd
exit

end
write memory
exit
9.2. BR-RTR
Bash

nano /etc/frr/daemons
# Измените: ospfd=yes

systemctl restart frr
systemctl enable frr
vtysh
text

configure terminal

router ospf
 network 10.0.0.0/30 area 0
 network 192.168.10.0/28 area 0
 passive-interface default
 no passive-interface gre1
exit

interface gre1
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 P@ssw0rd
exit

end
write memory
exit
9.3. Проверка OSPF
Bash

# Соседи OSPF
vtysh -c "show ip ospf neighbor"

# Маршруты OSPF
vtysh -c "show ip route ospf"

# Проверка связности между офисами (с HQ-SRV → BR-SRV)
ping -c 3 192.168.10.2
10. DHCP-сервер
10.1. Настройка на HQ-RTR
Bash

# Указываем интерфейс для DHCP
nano /etc/default/isc-dhcp-server
ini

INTERFACESv4="ens19.200"
Bash

nano /etc/dhcp/dhcpd.conf
Добавьте в конец:

ini

# DHCP для VLAN 200 (HQ-CLI)
subnet 192.168.200.0 netmask 255.255.255.224 {
    range 192.168.200.2 192.168.200.30;
    option routers 192.168.200.1;
    option domain-name-servers 192.168.100.2;
    option domain-name "au-team.irpo";
}
Bash

systemctl restart isc-dhcp-server
systemctl enable isc-dhcp-server
systemctl status isc-dhcp-server
10.2. DHCP-клиент на HQ-CLI
Bash

nano /etc/network/interfaces
ini

source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

auto ens18
iface ens18 inet manual
    up ip link set $IFACE up
    down ip link set $IFACE down

auto ens18.200
iface ens18.200 inet dhcp
    vlan-raw-device ens18
Bash

systemctl restart networking

# Проверка
ip addr show ens18.200
11. DNS-сервер
11.1. Установка BIND9 на HQ-SRV
Bash

apt install -y bind9 bind9utils
11.2. Основная конфигурация
Bash

nano /etc/bind/named.conf.options
text

options {
    directory "/var/cache/bind";

    forwarders {
        77.88.8.7;
        77.88.8.3;
    };

    listen-on { any; };
    allow-query { any; };
    dnssec-validation no;
    recursion yes;
};
11.3. Объявление зон
Bash

nano /etc/bind/named.conf.local
text

// Прямая зона
zone "au-team.irpo" {
    type master;
    file "/etc/bind/db.au-team.irpo";
};

// Обратная зона 192.168.100.x
zone "100.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.100";
};

// Обратная зона 192.168.200.x
zone "200.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.200";
};
11.4. Прямая зона
Bash

nano /etc/bind/db.au-team.irpo
text

$TTL    86400
@       IN      SOA     hq-srv.au-team.irpo. admin.au-team.irpo. (
                        2024010101      ; Serial
                        3600            ; Refresh
                        1800            ; Retry
                        604800          ; Expire
                        86400 )         ; Minimum TTL

@       IN      NS      hq-srv.au-team.irpo.

hq-rtr  IN      A       192.168.100.1
br-rtr  IN      A       192.168.10.1
hq-srv  IN      A       192.168.100.2
hq-cli  IN      A       192.168.200.2
br-srv  IN      A       192.168.10.2
docker  IN      A       172.16.1.1
web     IN      A       172.16.2.1
11.5. Обратная зона 192.168.100.x
Bash

nano /etc/bind/db.192.168.100
text

$TTL    86400
@       IN      SOA     hq-srv.au-team.irpo. admin.au-team.irpo. (
                        2024010101      ; Serial
                        3600            ; Refresh
                        1800            ; Retry
                        604800          ; Expire
                        86400 )         ; Minimum TTL

@       IN      NS      hq-srv.au-team.irpo.

1       IN      PTR     hq-rtr.au-team.irpo.
2       IN      PTR     hq-srv.au-team.irpo.
11.6. Обратная зона 192.168.200.x
Bash

nano /etc/bind/db.192.168.200
text

$TTL    86400
@       IN      SOA     hq-srv.au-team.irpo. admin.au-team.irpo. (
                        2024010101      ; Serial
                        3600            ; Refresh
                        1800            ; Retry
                        604800          ; Expire
                        86400 )         ; Minimum TTL

@       IN      NS      hq-srv.au-team.irpo.

2       IN      PTR     hq-cli.au-team.irpo.
11.7. Проверка и запуск
Bash

# Проверка конфигурации
named-checkconf
named-checkzone au-team.irpo /etc/bind/db.au-team.irpo
named-checkzone 100.168.192.in-addr.arpa /etc/bind/db.192.168.100
named-checkzone 200.168.192.in-addr.arpa /etc/bind/db.192.168.200

# Запуск
systemctl restart named
systemctl enable named
11.8. Настройка DNS-клиентов
На всех машинах (кроме HQ-CLI):

Bash

echo "nameserver 192.168.100.2" > /etc/resolv.conf
echo "search au-team.irpo" >> /etc/resolv.conf
11.9. Проверка DNS
Bash

apt install -y dnsutils

# Прямое разрешение
nslookup hq-srv.au-team.irpo 192.168.100.2
nslookup docker.au-team.irpo 192.168.100.2

# Обратное разрешение
nslookup 192.168.100.2 192.168.100.2
12. Часовой пояс
На каждой ВМ:

Bash

# Установить часовой пояс (пример: Москва)
timedatectl set-timezone Europe/Moscow

# Проверить
timedatectl

# Список доступных зон
timedatectl list-timezones | grep Europe
Модуль 2. Организация сетевого администрирования
⚠️ Модуль 2 использует отдельный стенд с преднастроенными IP, NAT, туннелем, DHCP, DNS.

1. Samba DC (контроллер домена)
На BR-SRV
Bash

apt install -y samba smbclient winbind krb5-user

# Остановка сервисов
systemctl stop smbd nmbd winbind
systemctl disable smbd nmbd winbind

# Бэкап оригинальной конфигурации
mv /etc/samba/smb.conf /etc/samba/smb.conf.bak

# Провижининг домена
samba-tool domain provision \
  --realm=AU-TEAM.IRPO \
  --domain=AU-TEAM \
  --server-role=dc \
  --dns-backend=SAMBA_INTERNAL \
  --adminpass='P@ssw0rd'

# Копируем Kerberos конфигурацию
cp /var/lib/samba/private/krb5.conf /etc/krb5.conf

# Запускаем Samba DC
systemctl unmask samba-ad-dc
systemctl enable samba-ad-dc
systemctl start samba-ad-dc
Создание пользователей и группы
Bash

# Создаём 5 пользователей
for i in 1 2 3 4 5; do
  samba-tool user create hquser$i P@ssw0rd
done

# Создаём группу hq
samba-tool group add hq

# Добавляем пользователей в группу
for i in 1 2 3 4 5; do
  samba-tool group addmembers hq hquser$i
done

# Проверка
samba-tool user list
samba-tool group listmembers hq
Ввод HQ-CLI в домен
На HQ-CLI:

Bash

apt install -y realmd sssd sssd-tools adcli packagekit samba-common-bin

# Настройка DNS на контроллер домена
echo "nameserver 192.168.10.2" > /etc/resolv.conf
echo "search au-team.irpo" >> /etc/resolv.conf

# Обнаружение и ввод в домен
realm discover au-team.irpo
realm join au-team.irpo -U Administrator
# Введите пароль: P@ssw0rd

# Разрешаем вход доменным пользователям
realm permit --all
Ограниченный sudo для группы hq
На HQ-CLI:

Bash

nano /etc/sudoers.d/hq_group
text

%hq ALL=(ALL) /usr/bin/cat, /usr/bin/grep, /usr/bin/id
Bash

chmod 440 /etc/sudoers.d/hq_group
2. RAID-массив
На HQ-SRV
Bash

apt install -y mdadm

# Посмотрите дополнительные диски
lsblk
# Предположим, это /dev/sdb и /dev/sdc

# Создаём RAID 0 из двух дисков
mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sdb /dev/sdc

# Подтвердите: y

# Сохраняем конфигурацию
mdadm --detail --scan >> /etc/mdadm/mdadm.conf
update-initramfs -u

# Создаём файловую систему
mkfs.ext4 /dev/md0

# Создаём точку монтирования
mkdir -p /raid

# Автоматическое монтирование через fstab
echo "/dev/md0  /raid  ext4  defaults  0  0" >> /etc/fstab

# Монтируем
mount -a

# Проверка
df -h /raid
cat /proc/mdstat
3. NFS-сервер
На HQ-SRV (сервер)
Bash

apt install -y nfs-kernel-server

# Создаём папку для шары
mkdir -p /raid/nfs
chmod 777 /raid/nfs

# Настраиваем экспорт (только для сети HQ-CLI)
nano /etc/exports
text

/raid/nfs  192.168.200.0/27(rw,sync,no_subtree_check)
Bash

exportfs -ra
systemctl restart nfs-kernel-server
systemctl enable nfs-kernel-server
На HQ-CLI (клиент)
Bash

apt install -y nfs-common

mkdir -p /mnt/nfs

# Автомонтирование через fstab
echo "192.168.100.2:/raid/nfs  /mnt/nfs  nfs  defaults  0  0" >> /etc/fstab

mount -a

# Проверка
df -h /mnt/nfs
4. Chrony (NTP)
На ISP (NTP-сервер)
Bash

apt install -y chrony

nano /etc/chrony/chrony.conf
Добавьте/измените:

ini

# Вышестоящий NTP-сервер
server pool.ntp.org iburst

# Стратум 5
local stratum 5

# Разрешаем клиентам
allow 172.16.0.0/12
allow 192.168.0.0/16
Bash

systemctl restart chronyd
systemctl enable chronyd
На клиентах (HQ-SRV, HQ-CLI, BR-RTR, BR-SRV)
Bash

apt install -y chrony

nano /etc/chrony/chrony.conf
Замените серверы на ISP:

ini

server 172.16.1.1 iburst   # для HQ-стороны
# или
server 172.16.2.1 iburst   # для BR-стороны
Bash

systemctl restart chronyd
systemctl enable chronyd

# Проверка
chronyc sources
5. Ansible
На BR-SRV
Bash

apt install -y ansible sshpass

# Рабочий каталог
mkdir -p /etc/ansible
Файл инвентаря
Bash

nano /etc/ansible/hosts
ini

[servers]
hq-srv ansible_host=192.168.100.2 ansible_user=sshuser ansible_ssh_port=2026
hq-cli ansible_host=192.168.200.2 ansible_user=sshuser

[routers]
hq-rtr ansible_host=192.168.100.1 ansible_user=net_admin
br-rtr ansible_host=192.168.10.1 ansible_user=net_admin

[all:vars]
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
ansible_password=P@ssw0rd
Конфигурация ansible
Bash

nano /etc/ansible/ansible.cfg
ini

[defaults]
inventory = /etc/ansible/hosts
host_key_checking = False
Проверка
Bash

ansible all -m ping
Все хосты должны ответить pong.

6. Docker
На BR-SRV
Bash

apt install -y docker.io docker-compose

# Подключаем образ Additional.iso (через гипервизор)
mkdir -p /mnt/iso
mount /dev/cdrom /mnt/iso

# Импортируем Docker-образы
docker load -i /mnt/iso/docker/site_latest.tar
docker load -i /mnt/iso/docker/mariadb_latest.tar

# Проверяем загруженные образы
docker images
Docker Compose
Bash

mkdir -p /opt/testapp
nano /opt/testapp/docker-compose.yml
YAML

version: '3'

services:
  db:
    image: mariadb:latest
    container_name: db
    environment:
      MYSQL_ROOT_PASSWORD: P@ssw0rd
      MYSQL_DATABASE: testdb
      MYSQL_USER: test
      MYSQL_PASSWORD: P@ssw0rd
    restart: always

  testapp:
    image: site:latest
    container_name: testapp
    ports:
      - "8080:8080"
    environment:
      DB_HOST: db
      DB_NAME: testdb
      DB_USER: test
      DB_PASSWORD: P@ssw0rd
    depends_on:
      - db
    restart: always
Bash

cd /opt/testapp
docker-compose up -d

# Проверка
docker ps
curl http://localhost:8080
7. Веб-приложение Apache + MariaDB
На HQ-SRV
Bash

apt install -y apache2 mariadb-server php php-mysql libapache2-mod-php

# Подключаем Additional.iso
mount /dev/cdrom /mnt/iso

# Запускаем MariaDB
systemctl start mariadb
systemctl enable mariadb

# Импортируем дамп БД
mysql -u root <<EOF
CREATE DATABASE webdb;
CREATE USER 'web'@'localhost' IDENTIFIED BY 'P@ssw0rd';
GRANT ALL PRIVILEGES ON webdb.* TO 'web'@'localhost';
FLUSH PRIVILEGES;
EOF

mysql -u root webdb < /mnt/iso/web/dump.sql

# Копируем файлы веб-приложения
cp /mnt/iso/web/index.php /var/www/html/
cp -r /mnt/iso/web/images /var/www/html/

# Редактируем подключение к БД
nano /var/www/html/index.php
В файле index.php найдите и измените параметры подключения:

PHP

$db_host = 'localhost';
$db_name = 'webdb';
$db_user = 'web';
$db_pass = 'P@ssw0rd';
Bash

# Перезапускаем Apache
systemctl restart apache2
systemctl enable apache2

# Проверка
curl http://localhost
8. Проброс портов
На BR-RTR
Bash

# Проброс 8080 → BR-SRV:8080 (testapp)
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 192.168.10.2:8080

# Проброс 2026 → BR-SRV:2026 (SSH)
iptables -t nat -A PREROUTING -p tcp --dport 2026 -j DNAT --to-destination 192.168.10.2:2026

# Разрешаем пересылку
iptables -A FORWARD -p tcp -d 192.168.10.2 --dport 8080 -j ACCEPT
iptables -A FORWARD -p tcp -d 192.168.10.2 --dport 2026 -j ACCEPT

netfilter-persistent save
На HQ-RTR
Bash

# Проброс 8080 → HQ-SRV:80 (Apache)
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 192.168.100.2:80

# Проброс 2026 → HQ-SRV:2026 (SSH)
iptables -t nat -A PREROUTING -p tcp --dport 2026 -j DNAT --to-destination 192.168.100.2:2026

# Разрешаем пересылку
iptables -A FORWARD -p tcp -d 192.168.100.2 --dport 80 -j ACCEPT
iptables -A FORWARD -p tcp -d 192.168.100.2 --dport 2026 -j ACCEPT

netfilter-persistent save
9. Nginx reverse proxy
На ISP
Bash

apt install -y nginx

nano /etc/nginx/sites-available/reverse-proxy
nginx

# Прокси для web.au-team.irpo → HQ-SRV
server {
    listen 80;
    server_name web.au-team.irpo;

    location / {
        proxy_pass http://172.16.1.2:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

# Прокси для docker.au-team.irpo → BR-SRV testapp
server {
    listen 80;
    server_name docker.au-team.irpo;

    location / {
        proxy_pass http://172.16.2.2:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
Bash

ln -s /etc/nginx/sites-available/reverse-proxy /etc/nginx/sites-enabled/
rm /etc/nginx/sites-enabled/default

nginx -t
systemctl restart nginx
systemctl enable nginx
10. Web-аутентификация
На ISP
Bash

apt install -y apache2-utils

# Создаём файл паролей
htpasswd -cb /etc/nginx/.htpasswd WEB P@ssw0rd

# Добавляем аутентификацию в конфигурацию web.au-team.irpo
nano /etc/nginx/sites-available/reverse-proxy
Измените блок web.au-team.irpo:

nginx

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
Bash

nginx -t
systemctl restart nginx
11. Яндекс Браузер
На HQ-CLI
Bash

# Скачиваем и устанавливаем
wget https://repo.yandex.ru/yandex-browser/deb/pool/main/y/yandex-browser-stable/yandex-browser-stable_latest_amd64.deb

apt install -y ./yandex-browser-stable_latest_amd64.deb

# Или через репозиторий
wget -qO - https://repo.yandex.ru/yandex-browser/YANDEX-BROWSER-KEY.GPG | apt-key add -
echo "deb https://repo.yandex.ru/yandex-browser/deb stable main" > /etc/apt/sources.list.d/yandex-browser.list
apt update
apt install -y yandex-browser-stable
Полезные команды
Bash

# === Общие ===
ip addr show                          # IP-адреса
ip route show                         # Маршруты
ip link show                          # Интерфейсы
systemctl status <service>            # Статус сервиса

# === Сеть ===
ping -c 3 <ip>                        # Проверка связности
traceroute <ip>                       # Трассировка
ss -tlnp                              # Открытые порты

# === NAT/Firewall ===
iptables -t nat -L -v -n              # Правила NAT
iptables -L -v -n                     # Правила фильтрации

# === OSPF (FRR) ===
vtysh -c "show ip ospf neighbor"      # Соседи OSPF
vtysh -c "show ip route ospf"         # Маршруты OSPF
vtysh -c "show ip route"              # Все маршруты

# === DNS ===
nslookup <domain> <dns-server>        # Разрешение имени
dig <domain> @<dns-server>            # Подробный DNS-запрос
dig -x <ip> @<dns-server>             # Обратный DNS

# === DHCP ===
cat /var/lib/dhcp/dhcpd.leases        # Выданные аренды

# === GRE ===
ip tunnel show                        # Туннели

# === Docker ===
docker ps                             # Запущенные контейнеры
docker-compose logs                   # Логи
docker images                         # Образы

# === RAID ===
cat /proc/mdstat                      # Статус RAID
mdadm --detail /dev/md0               # Подробности

# === NFS ===
showmount -e localhost                 # Экспортированные шары

# === NTP ===
chronyc sources                       # Источники времени
chronyc tracking                      # Статус синхронизации

# === Samba ===
samba-tool user list                   # Пользователи домена
samba-tool group listmembers <group>   # Члены группы

# === Ansible ===
ansible all -m ping                   # Проверка подключения
📄 Лицензия
Данная методичка создана в образовательных целях.
Свободна для использования и распространения.

✍️ Автор
Создано для подготовки к ГИА ДЭ по специальности 09.02.06
«Сетевое и системное администрирование»
