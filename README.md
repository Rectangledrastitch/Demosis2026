ЧАСТЬ 0. ПОДГОТОВКА
0.1. Планирование IP-адресации
Прежде чем приступить к настройке, нужно рассчитать IP-адресацию. Ниже — готовый план:

Сеть	Подсеть	Маска	Назначение
ISP ↔ HQ-RTR	172.16.1.0/28	255.255.255.240	Дана в задании
ISP ↔ BR-RTR	172.16.2.0/28	255.255.255.240	Дана в задании
VLAN 100 (HQ-SRV)	192.168.100.0/27	255.255.255.224	≤32 адреса (30 хостовых)
VLAN 200 (HQ-CLI)	192.168.200.0/27	255.255.255.224	≥16 адресов (30 хостовых)
VLAN 999 (управление)	192.168.99.0/29	255.255.255.248	≤8 адресов (6 хостовых)
BR-SRV	192.168.10.0/28	255.255.255.240	≤16 адресов (14 хостовых)
GRE-туннель	10.0.0.0/30	255.255.255.252	Между HQ-RTR и BR-RTR
Таблица IP-адресов (Таблица 2 для отчёта):

Устройство	Интерфейс	IP-адрес	Шлюз
ISP	ens18 (Интернет)	DHCP	—
ISP	ens19 (к HQ-RTR)	172.16.1.1/28	—
ISP	ens20 (к BR-RTR)	172.16.2.1/28	—
HQ-RTR	ens18 (к ISP)	172.16.1.2/28	172.16.1.1
HQ-RTR	ens19.100	192.168.100.1/27	—
HQ-RTR	ens19.200	192.168.200.1/27	—
HQ-RTR	ens19.999	192.168.99.1/29	—
HQ-RTR	gre1	10.0.0.1/30	—
BR-RTR	ens18 (к ISP)	172.16.2.2/28	172.16.2.1
BR-RTR	ens19 (к BR-SRV)	192.168.10.1/28	—
BR-RTR	gre1	10.0.0.2/30	—
HQ-SRV	ens18.100	192.168.100.2/27	192.168.100.1
HQ-CLI	ens18.200	DHCP	192.168.200.1
BR-SRV	ens18	192.168.10.2/28	192.168.10.1
Важно: Имена интерфейсов (ens18, ens19, ens20) зависят от гипервизора. В VirtualBox это enp0s3, enp0s8, enp0s9. Узнайте свои имена командой: ip link show

0.2. Создание ВМ в гипервизоре
Создайте виртуальные машины с Debian 12 и следующими сетевыми подключениями:

ВМ	NIC1 (ens18)	NIC2 (ens19)	NIC3 (ens20)
ISP	Bridge/NAT (Интернет)	Внутр. сеть «ISP-HQ»	Внутр. сеть «ISP-BR»
HQ-RTR	Внутр. сеть «ISP-HQ»	Внутр. сеть «HQ-LAN»	—
BR-RTR	Внутр. сеть «ISP-BR»	Внутр. сеть «BR-LAN»	—
HQ-SRV	Внутр. сеть «HQ-LAN»	—	—
HQ-CLI	Внутр. сеть «HQ-LAN»	—	—
BR-SRV	Внутр. сеть «BR-LAN»	—	—
0.3. Общие предварительные действия на каждой ВМ
После установки Debian 12, на каждой машине выполните вход как root и проверьте имена интерфейсов:

Bash

ip link show
Настройте временный DNS для доступа к репозиториям:

Bash

echo "nameserver 77.88.8.7" > /etc/resolv.conf
ЧАСТЬ 1. НАСТРОЙКА ИМЁН УСТРОЙСТВ (Задание 1)
На каждой ВМ выполните команду (подставьте нужное имя):

ISP:

Bash

hostnamectl set-hostname isp.au-team.irpo
HQ-RTR:

Bash

hostnamectl set-hostname hq-rtr.au-team.irpo
BR-RTR:

Bash

hostnamectl set-hostname br-rtr.au-team.irpo
HQ-SRV:

Bash

hostnamectl set-hostname hq-srv.au-team.irpo
BR-SRV:

Bash

hostnamectl set-hostname br-srv.au-team.irpo
HQ-CLI:

Bash

hostnamectl set-hostname hq-cli.au-team.irpo
Проверьте результат:

Bash

hostnamectl
ЧАСТЬ 2. НАСТРОЙКА ISP (Задание 2)
2.1. Настройка сетевых интерфейсов ISP
Откройте файл конфигурации сети:

Bash

nano /etc/network/interfaces
Приведите содержимое к следующему виду (замените имена интерфейсов на свои):

text

source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

# Интерфейс к Интернету (получает адрес по DHCP)
auto ens18
iface ens18 inet dhcp

# Интерфейс в сторону HQ-RTR
auto ens19
iface ens19 inet static
    address 172.16.1.1
    netmask 255.255.255.240

# Интерфейс в сторону BR-RTR
auto ens20
iface ens20 inet static
    address 172.16.2.1
    netmask 255.255.255.240
Сохраните файл (Ctrl+O, Enter, Ctrl+X) и перезапустите сеть:

Bash

systemctl restart networking
2.2. Включение маршрутизации (IP forwarding)
Bash

echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
2.3. Настройка NAT (MASQUERADE) на ISP
Установите необходимые пакеты:

Bash

apt update
apt install -y iptables iptables-persistent
Создайте правило NAT (замените ens18 на имя вашего интернет-интерфейса):

Bash

iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE
Сохраните правила:

Bash

netfilter-persistent save
2.4. Проверка ISP
Bash

# Проверьте наличие интернета
ping -c 3 77.88.8.7

# Проверьте IP-адреса
ip addr show ens19
ip addr show ens20
ЧАСТЬ 3. БАЗОВАЯ НАСТРОЙКА МАРШРУТИЗАТОРОВ
3.1. HQ-RTR — интерфейс к ISP
На машине HQ-RTR откройте файл:

Bash

nano /etc/network/interfaces
Пока настройте только интерфейс к ISP (VLANы добавим позже):

text

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
Перезапустите сеть:

Bash

systemctl restart networking
Включите маршрутизацию:

Bash

echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
Проверьте доступ к Интернету:

Bash

ping -c 3 77.88.8.7
Установите необходимые пакеты (понадобятся далее):

Bash

apt update
apt install -y vlan iptables iptables-persistent isc-dhcp-server frr
Включите модуль VLAN:

Bash

modprobe 8021q
echo "8021q" >> /etc/modules
3.2. BR-RTR — интерфейс к ISP
На машине BR-RTR откройте файл:

Bash

nano /etc/network/interfaces
text

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
Перезапустите сеть:

Bash

systemctl restart networking
Включите маршрутизацию:

Bash

echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
Установите пакеты:

Bash

apt update
apt install -y iptables iptables-persistent frr
ЧАСТЬ 4. НАСТРОЙКА VLAN-КОММУТАЦИИ (Задание 4)
4.1. Настройка VLAN-субинтерфейсов на HQ-RTR
На HQ-RTR отредактируйте /etc/network/interfaces:

Bash

nano /etc/network/interfaces
Добавьте к уже существующей конфигурации блоки для VLAN:

text

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

# Транковый интерфейс к HQ-LAN (без IP-адреса)
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
Перезапустите сеть:

Bash

systemctl restart networking
Проверьте, что субинтерфейсы создались:

Bash

ip addr show
Вы должны увидеть интерфейсы ens19.100, ens19.200, ens19.999 с назначенными IP.

4.2. Настройка VLAN на HQ-SRV
На HQ-SRV установите пакет для VLAN:

Bash

apt update
apt install -y vlan
modprobe 8021q
echo "8021q" >> /etc/modules
Отредактируйте /etc/network/interfaces:

Bash

nano /etc/network/interfaces
text

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
Перезапустите сеть:

Bash

systemctl restart networking
Настройте DNS:

Bash

echo "nameserver 77.88.8.7" > /etc/resolv.conf
4.3. Настройка VLAN на HQ-CLI
На HQ-CLI установите пакет VLAN:

Bash

apt update  # может не работать до полной настройки сети
apt install -y vlan
modprobe 8021q
echo "8021q" >> /etc/modules
Если apt не работает (нет Интернета), установку можно сделать позже, после настройки NAT. Пока настройте интерфейс вручную.

Отредактируйте /etc/network/interfaces:

Bash

nano /etc/network/interfaces
Временно настроим статический IP (позже переключим на DHCP):

text

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
Перезапустите сеть:

Bash

systemctl restart networking
4.4. Настройка BR-SRV
На BR-SRV отредактируйте /etc/network/interfaces:

Bash

nano /etc/network/interfaces
text

source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

auto ens18
iface ens18 inet static
    address 192.168.10.2
    netmask 255.255.255.240
    gateway 192.168.10.1
Перезапустите сеть:

Bash

systemctl restart networking
echo "nameserver 77.88.8.7" > /etc/resolv.conf
ЧАСТЬ 5. NAT НА МАРШРУТИЗАТОРАХ (Задание 8)
5.1. NAT на HQ-RTR
На HQ-RTR:

Bash

# NAT для всего трафика из внутренних сетей в сторону ISP
iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE

# Сохраняем правила
netfilter-persistent save
5.2. NAT на BR-RTR
На BR-RTR:

Bash

iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE
netfilter-persistent save
5.3. Проверка доступа к Интернету
Со всех машин проверьте связность:

С HQ-SRV:

Bash

ping -c 3 192.168.100.1    # шлюз (HQ-RTR)
ping -c 3 172.16.1.1        # ISP
ping -c 3 77.88.8.7         # Интернет
С BR-SRV:

Bash

ping -c 3 192.168.10.1      # шлюз (BR-RTR)
ping -c 3 172.16.2.1        # ISP
ping -c 3 77.88.8.7         # Интернет
Если пинг не проходит — проверьте правильность шлюзов, включение ip_forward и правила iptables.

Теперь на всех машинах можно устанавливать пакеты через apt.

ЧАСТЬ 6. СОЗДАНИЕ УЧЁТНЫХ ЗАПИСЕЙ (Задание 3)
6.1. Пользователь sshuser на HQ-SRV и BR-SRV
Выполните на HQ-SRV и BR-SRV (одинаковые команды):

Bash

# Создаём пользователя с UID 2026
useradd -m -u 2026 -s /bin/bash sshuser

# Задаём пароль P@ssw0rd
echo "sshuser:P@ssw0rd" | chpasswd

# Настраиваем sudo без пароля
apt install -y sudo
echo "sshuser ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/sshuser
chmod 440 /etc/sudoers.d/sshuser
Проверьте:

Bash

id sshuser
# Вывод: uid=2026(sshuser) ...
6.2. Пользователь net_admin на HQ-RTR и BR-RTR
Выполните на HQ-RTR и BR-RTR:

Bash

useradd -m -s /bin/bash net_admin

echo "net_admin:P@ssw0rd" | chpasswd

apt install -y sudo
echo "net_admin ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/net_admin
chmod 440 /etc/sudoers.d/net_admin
ЧАСТЬ 7. НАСТРОЙКА SSH (Задание 5)
На HQ-SRV и BR-SRV выполните одинаковые действия:
Установите SSH-сервер (если не установлен):

Bash

apt install -y openssh-server
Создайте файл баннера:

Bash

echo "Authorized access only" > /etc/ssh/banner
Отредактируйте конфигурацию SSH:

Bash

nano /etc/ssh/sshd_config
Найдите и измените (или добавьте) следующие строки:

text

Port 2026
AllowUsers sshuser
MaxAuthTries 2
Banner /etc/ssh/banner
Пояснение:

Port 2026 — меняем порт SSH с 22 на 2026
AllowUsers sshuser — разрешаем вход только пользователю sshuser
MaxAuthTries 2 — максимум 2 попытки ввода пароля
Banner /etc/ssh/banner — показывает баннер при подключении
Перезапустите SSH:

Bash

systemctl restart sshd
Проверьте, что SSH слушает на порту 2026:

Bash

ss -tlnp | grep 2026
ЧАСТЬ 8. GRE-ТУННЕЛЬ (Задание 6)
8.1. На HQ-RTR
Добавьте в /etc/network/interfaces:

Bash

nano /etc/network/interfaces
Допишите в конец:

text

# GRE-туннель к BR-RTR
auto gre1
iface gre1 inet static
    address 10.0.0.1
    netmask 255.255.255.252
    pre-up ip tunnel add gre1 mode gre remote 172.16.2.2 local 172.16.1.2 ttl 255
    post-down ip tunnel del gre1
Поднимите туннель:

Bash

ifup gre1
8.2. На BR-RTR
Добавьте в /etc/network/interfaces:

Bash

nano /etc/network/interfaces
Допишите в конец:

text

# GRE-туннель к HQ-RTR
auto gre1
iface gre1 inet static
    address 10.0.0.2
    netmask 255.255.255.252
    pre-up ip tunnel add gre1 mode gre remote 172.16.1.2 local 172.16.2.2 ttl 255
    post-down ip tunnel del gre1
Поднимите туннель:

Bash

ifup gre1
8.3. Проверка туннеля
С HQ-RTR:

Bash

ping -c 3 10.0.0.2
С BR-RTR:

Bash

ping -c 3 10.0.0.1
Если пинг проходит — туннель работает!

ЧАСТЬ 9. ДИНАМИЧЕСКАЯ МАРШРУТИЗАЦИЯ OSPF (Задание 7)
Будем использовать FRRouting (FRR) — пакет динамической маршрутизации.

9.1. Настройка FRR на HQ-RTR
Включите демон OSPF:

Bash

nano /etc/frr/daemons
Найдите строку ospfd=no и измените на:

text

ospfd=yes
Перезапустите FRR:

Bash

systemctl restart frr
systemctl enable frr
Войдите в консоль маршрутизации:

Bash

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
Пояснение:

passive-interface default — все интерфейсы пассивные (не отправляют OSPF-пакеты)
no passive-interface gre1 — только через GRE-туннель идёт обмен маршрутами
message-digest — парольная защита OSPF
9.2. Настройка FRR на BR-RTR
Включите OSPF:

Bash

nano /etc/frr/daemons
Измените:

text

ospfd=yes
Перезапустите:

Bash

systemctl restart frr
systemctl enable frr
Настройте:

Bash

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
На HQ-RTR:

Bash

vtysh -c "show ip ospf neighbor"
Вы должны увидеть соседа (BR-RTR).

Проверьте маршруты:

Bash

vtysh -c "show ip route ospf"
Должны появиться маршруты к сетям другого офиса.

Проверьте связность между офисами:

Bash

# С HQ-SRV пингуем BR-SRV
ping -c 3 192.168.10.2
ЧАСТЬ 10. DHCP-СЕРВЕР (Задание 9)
10.1. Настройка DHCP на HQ-RTR
Пакет isc-dhcp-server уже установлен (мы его установили ранее).

Укажите интерфейс для раздачи DHCP:

Bash

nano /etc/default/isc-dhcp-server
Найдите строку INTERFACESv4="" и измените:

text

INTERFACESv4="ens19.200"
Настройте DHCP-пул:

Bash

nano /etc/dhcp/dhcpd.conf
Добавьте в конец файла (или замените содержимое):

text

# DHCP для VLAN 200 (HQ-CLI)
subnet 192.168.200.0 netmask 255.255.255.224 {
    range 192.168.200.2 192.168.200.30;
    option routers 192.168.200.1;
    option domain-name-servers 192.168.100.2;
    option domain-name "au-team.irpo";
}
Пояснение:

range — диапазон выдаваемых адресов (адрес .1 маршрутизатора исключён)
option routers — шлюз по умолчанию (HQ-RTR)
option domain-name-servers — DNS-сервер (HQ-SRV)
option domain-name — DNS-суффикс
Перезапустите DHCP-сервер:

Bash

systemctl restart isc-dhcp-server
systemctl enable isc-dhcp-server
Проверьте статус:

Bash

systemctl status isc-dhcp-server
10.2. Настройка DHCP-клиента на HQ-CLI
На HQ-CLI измените /etc/network/interfaces:

Bash

nano /etc/network/interfaces
Замените статическую конфигурацию VLAN 200 на DHCP:

text

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
Перезапустите сеть:

Bash

systemctl restart networking
Проверьте получение IP:

Bash

ip addr show ens18.200
Вы должны увидеть IP из диапазона 192.168.200.2–192.168.200.30.

ЧАСТЬ 11. DNS-СЕРВЕР (Задание 10)
11.1. Установка BIND9 на HQ-SRV
Bash

apt install -y bind9 bind9utils
11.2. Настройка параметров BIND
Bash

nano /etc/bind/named.conf.options
Замените содержимое:

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
Добавьте:

text

// Прямая зона
zone "au-team.irpo" {
    type master;
    file "/etc/bind/db.au-team.irpo";
};

// Обратная зона для 192.168.100.0/27
zone "100.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.100";
};

// Обратная зона для 192.168.200.0/27
zone "200.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.200";
};
11.4. Создание файла прямой зоны
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

; NS-запись
@       IN      NS      hq-srv.au-team.irpo.

; A-записи
hq-rtr  IN      A       192.168.100.1
br-rtr  IN      A       192.168.10.1
hq-srv  IN      A       192.168.100.2
hq-cli  IN      A       192.168.200.2
br-srv  IN      A       192.168.10.2
docker  IN      A       172.16.1.1
web     IN      A       172.16.2.1
11.5. Создание обратной зоны для 192.168.100.x
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

; PTR-записи
1       IN      PTR     hq-rtr.au-team.irpo.
2       IN      PTR     hq-srv.au-team.irpo.
11.6. Создание обратной зоны для 192.168.200.x
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

; PTR-запись
2       IN      PTR     hq-cli.au-team.irpo.
11.7. Проверка и запуск
Проверьте конфигурацию:

Bash

named-checkconf
named-checkzone au-team.irpo /etc/bind/db.au-team.irpo
named-checkzone 100.168.192.in-addr.arpa /etc/bind/db.192.168.100
named-checkzone 200.168.192.in-addr.arpa /etc/bind/db.192.168.200
Все проверки должны вернуть OK.

Перезапустите BIND:

Bash

systemctl restart named
systemctl enable named
11.8. Настройка DNS-клиентов
На всех машинах (кроме HQ-CLI, который получает DNS по DHCP) укажите DNS-сервер:

Bash

echo "nameserver 192.168.100.2" > /etc/resolv.conf
echo "search au-team.irpo" >> /etc/resolv.conf
На HQ-CLI DNS-сервер уже задан через DHCP (192.168.100.2).

11.9. Проверка DNS
С любой машины:

Bash

# Прямое разрешение
nslookup hq-srv.au-team.irpo 192.168.100.2
nslookup br-srv.au-team.irpo 192.168.100.2
nslookup docker.au-team.irpo 192.168.100.2

# Обратное разрешение
nslookup 192.168.100.2 192.168.100.2
nslookup 192.168.100.1 192.168.100.2
Если nslookup не установлен:

Bash

apt install -y dnsutils
ЧАСТЬ 12. ЧАСОВОЙ ПОЯС (Задание 11)
На каждой ВМ выполните:

Bash

# Посмотреть текущий часовой пояс
timedatectl

# Установить часовой пояс (пример для Москвы)
timedatectl set-timezone Europe/Moscow
Замените Europe/Moscow на ваш часовой пояс. Список доступных:

Bash

timedatectl list-timezones | grep Europe
Проверьте:

Bash

timedatectl
ИТОГОВАЯ СВОДКА: ЧТО И ГДЕ НАСТРОЕНО
Задание	Где выполнялось	Что сделано
1. Имена и IP	Все ВМ	hostname, IP-адреса
2. ISP	ISP	Интерфейсы, маршруты, NAT
3. Учётные записи	HQ-SRV, BR-SRV, HQ-RTR, BR-RTR	sshuser, net_admin
4. VLAN	HQ-RTR, HQ-SRV, HQ-CLI	VLAN 100, 200, 999
5. SSH	HQ-SRV, BR-SRV	Порт 2026, ограничения
6. GRE-туннель	HQ-RTR, BR-RTR	Туннель 10.0.0.0/30
7. OSPF	HQ-RTR, BR-RTR	FRR, парольная защита
8. NAT	HQ-RTR, BR-RTR	MASQUERADE
9. DHCP	HQ-RTR (сервер), HQ-CLI (клиент)	Пул для VLAN 200
10. DNS	HQ-SRV	BIND9, зоны, PTR
11. Часовой пояс	Все ВМ	timedatectl
ПОЛЕЗНЫЕ КОМАНДЫ ДЛЯ ОТЛАДКИ
Bash

# Посмотреть IP-адреса
ip addr show

# Посмотреть маршруты
ip route show

# Проверить NAT-правила
iptables -t nat -L -v -n

# Проверить состояние сервисов
systemctl status isc-dhcp-server
systemctl status named
systemctl status frr
systemctl status sshd

# Проверить OSPF
vtysh -c "show ip ospf neighbor"
vtysh -c "show ip route"

# Проверить DNS
dig hq-srv.au-team.irpo @192.168.100.2
dig -x 192.168.100.2 @192.168.100.2

# Проверить DHCP-выдачу
cat /var/lib/dhcp/dhcpd.leases

# Проверить GRE-туннель
ip tunnel show
