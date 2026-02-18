# Настройка маршрутизаторов rtr-cod и rtr-a на ALT Linux (etcnet + FRR)

> **Альтернативный вариант реализации.** Данная инструкция заменяет пункты 1, 3, 4, 5, 6 основного руководства (EcoRouter) на конфигурацию для ALT Linux с использованием **etcnet** для сетевых настроек и **FRR** для маршрутизации.

---

## Содержание

1. [Подготовка системы](#1-подготовка-системы)
2. [rtr-cod: имя хоста и IP-адресация](#2-rtr-cod-имя-хоста-и-ip-адресация)
3. [rtr-a: имя хоста и IP-адресация](#3-rtr-a-имя-хоста-и-ip-адресация)
4. [Настройка GRE-туннеля](#4-настройка-gre-туннеля)
5. [Настройка сетевого экрана (firewalld)](#5-настройка-сетевого-экрана-firewalld)
6. [Установка и настройка FRR](#6-установка-и-настройка-frr)
7. [Настройка BGP на rtr-cod](#7-настройка-bgp-на-rtr-cod)
8. [Настройка OSPF между rtr-cod и rtr-a](#8-настройка-ospf-между-rtr-cod-и-rtr-a)

---

## Справочная информация

### Структура etcnet

Конфигурация сетевых интерфейсов в ALT Linux хранится в `/etc/net/ifaces/`. Каждому интерфейсу соответствует отдельная директория с файлами:

| Файл | Назначение |
|---|---|
| `options` | Тип интерфейса и параметры (TYPE, BOOTPROTO и т.д.) |
| `ipv4address` | IPv4-адреса (по одному на строку) |
| `ipv4route` | Статические маршруты |
| `iplink` | Параметры уровня L2 |

### Имена физических интерфейсов

В зависимости от гипервизора и версии ALT Linux интерфейсы могут называться `eth0`/`eth1` или `ens192`/`ens224`. Определите их командой:

```bash
ip link show
```

В данной инструкции используются:
- **ens192** — порт в сторону ISP (аналог te0 в EcoRouter)
- **ens224** — порт в сторону fw-cod / коммутаторов (аналог te1 в EcoRouter)

> Замените имена интерфейсов на актуальные для вашей конфигурации.

---

## 1. Подготовка системы

### Установка необходимых пакетов

```bash
apt-get update
apt-get install frr etcnet firewalld
```

### Включение маршрутизации (forwarding)

В ALT Linux forwarding контролируется двумя файлами. Необходимо включить в обоих:

```bash
echo 'net.ipv4.ip_forward = 1' > /etc/net/sysctl.conf
sysctl -p
```

> **Важно:** файл `/etc/net/sysctl.conf` по умолчанию содержит `net.ipv4.ip_forward=0` и применяется при перезапуске сети, перезаписывая системный `sysctl.conf`. Поэтому его нужно перезаписать целиком через `echo >`.

---

## 2. rtr-cod: имя хоста и IP-адресация

### Назначение имени хоста

```bash
hostnamectl set-hostname rtr-cod.cod.ssa2026.region
```

Добавьте запись в `/etc/hosts`:

```bash
echo '127.0.0.1 rtr-cod.cod.ssa2026.region rtr-cod' >> /etc/hosts
```

### Настройка интерфейса ens192

> **ens192** — внешний интерфейс, обращён в сторону машины **ISP**

```bash
mkdir -p /etc/net/ifaces/ens192
```

`/etc/net/ifaces/ens192/options`:

```
TYPE=eth
```

`/etc/net/ifaces/ens192/ipv4address`:

```
178.207.179.4/29
```

### Настройка интерфейса ens224

> **ens224** — внутренний интерфейс, обращён в сторону машины **fw-cod**

```bash
mkdir -p /etc/net/ifaces/ens224
```

`/etc/net/ifaces/ens224/options`:

```
TYPE=eth
```

`/etc/net/ifaces/ens224/ipv4address`:

```
172.16.1.1/30
```

### Применение настроек

```bash
systemctl restart network
```

### Проверка

```bash
ip addr show ens192
ip addr show ens224
ping -c 3 178.207.179.1
```

> Маршрут по умолчанию на rtr-cod **не задаётся** вручную — он будет получен по BGP.

---

## 3. rtr-a: имя хоста и IP-адресация

### Назначение имени хоста

```bash
hostnamectl set-hostname rtr-a.office.ssa2026.region
echo '127.0.0.1 rtr-a.office.ssa2026.region rtr-a' >> /etc/hosts
```

### Настройка интерфейса ens192

> **ens192** — внешний интерфейс, обращён в сторону машины **ISP**

```bash
mkdir -p /etc/net/ifaces/ens192
```

`/etc/net/ifaces/ens192/options`:

```
TYPE=eth
```

`/etc/net/ifaces/ens192/ipv4address`:

```
178.207.179.28/29
```

### Маршрут по умолчанию

`/etc/net/ifaces/ens192/ipv4route`:

```
default via 178.207.179.25
```

### Настройка VLAN-интерфейсов на ens224

> **ens224** — внутренний интерфейс, обращён в сторону коммутаторов **sw1-a** и **sw2-a**

На rtr-a вместо одного интерфейса создаются VLAN-подинтерфейсы для маршрутизации между VLAN.

Создаём базовый интерфейс:

```bash
mkdir -p /etc/net/ifaces/ens224
```

`/etc/net/ifaces/ens224/options`:

```
TYPE=eth
```

#### VLAN 100 (SRV)

```bash
mkdir -p /etc/net/ifaces/ens224.100
```

`/etc/net/ifaces/ens224.100/options`:

```
TYPE=vlan
BOOTPROTO=static
ONBOOT=yes
CONFIG_IPV4=yes
HOST=ens224
VID=100
```

`/etc/net/ifaces/ens224.100/ipv4address`:

```
172.20.10.254/24
```

#### VLAN 200 (CLI)

```bash
mkdir -p /etc/net/ifaces/ens224.200
```

`/etc/net/ifaces/ens224.200/options`:

```
TYPE=vlan
BOOTPROTO=static
ONBOOT=yes
CONFIG_IPV4=yes
HOST=ens224
VID=200
```

`/etc/net/ifaces/ens224.200/ipv4address`:

```
172.20.20.254/24
```

#### VLAN 300 (MGMT)

```bash
mkdir -p /etc/net/ifaces/ens224.300
```

`/etc/net/ifaces/ens224.300/options`:

```
TYPE=vlan
BOOTPROTO=static
ONBOOT=yes
CONFIG_IPV4=yes
HOST=ens224
VID=300
```

`/etc/net/ifaces/ens224.300/ipv4address`:

```
172.20.30.254/24
```

### Применение настроек

```bash
systemctl restart network
```

### Проверка

```bash
ip addr show
ip -d link show ens224.100
ip -d link show ens224.200
ip -d link show ens224.300
ping -c 3 178.207.179.25
```

---

## 4. Настройка GRE-туннеля

GRE-туннель между rtr-cod и rtr-a настраивается через etcnet.

### rtr-cod: туннель tun0

> **tun0** — GRE-туннель до rtr-a, используется для OSPF-соседства между офисами

```bash
mkdir -p /etc/net/ifaces/tun0
```

`/etc/net/ifaces/tun0/options`:

```
TYPE=iptun
TUNTYPE=gre
TUNLOCAL=178.207.179.4
TUNREMOTE=178.207.179.28
TUNTTL=64
TUNOPTIONS='ttl 64'
ONBOOT=yes
CONFIG_IPV4=yes
```

`/etc/net/ifaces/tun0/ipv4address`:

```
10.10.10.1/30
```

### rtr-a: туннель tun0

> **tun0** — GRE-туннель до rtr-cod, используется для OSPF-соседства между офисами

```bash
mkdir -p /etc/net/ifaces/tun0
```

`/etc/net/ifaces/tun0/options`:

```
TYPE=iptun
TUNTYPE=gre
TUNLOCAL=178.207.179.28
TUNREMOTE=178.207.179.4
TUNTTL=64
TUNOPTIONS='ttl 64'
ONBOOT=yes
CONFIG_IPV4=yes
```

`/etc/net/ifaces/tun0/ipv4address`:

```
10.10.10.2/30
```

### Применение и проверка (на обоих роутерах)

```bash
systemctl restart network
```

Проверка на rtr-cod:

```bash
ip tunnel show
ip addr show tun0
ping -c 3 10.10.10.2
```

Проверка на rtr-a:

```bash
ip tunnel show
ip addr show tun0
ping -c 3 10.10.10.1
```

---

## 5. Настройка сетевого экрана (firewalld)

### Включение firewalld (на обоих роутерах)

```bash
systemctl enable --now firewalld
```

### rtr-cod: зоны и NAT

Внешний интерфейс (ISP) добавляем в зону **external** (masquerade включён по умолчанию), разрешаем GRE:

```bash
firewall-cmd --zone=external --add-interface=ens192 --permanent
firewall-cmd --zone=external --add-protocol=gre --permanent
```

Все остальные интерфейсы добавляем в зону **trusted**:

```bash
firewall-cmd --zone=trusted --add-interface=ens224 --permanent
firewall-cmd --zone=trusted --add-interface=tun0 --permanent
```

Применяем:

```bash
firewall-cmd --reload
```

Статические маршруты к внутренним сетям COD (через fw-cod).

Добавьте в `/etc/net/ifaces/ens224/ipv4route`:

```
192.168.10.0/24 via 172.16.1.2
192.168.30.0/24 via 172.16.1.2
192.168.40.0/24 via 172.16.1.2
192.168.50.0/24 via 172.16.1.2
```

> Сеть 192.168.20.0/24 (VLAN DATA) **не маршрутизируется** по условиям задания.

### rtr-a: зоны и NAT

```bash
firewall-cmd --zone=external --add-interface=ens192 --permanent
firewall-cmd --zone=external --add-protocol=gre --permanent

firewall-cmd --zone=trusted --add-interface=ens224 --permanent
firewall-cmd --zone=trusted --add-interface=ens224.100 --permanent
firewall-cmd --zone=trusted --add-interface=ens224.200 --permanent
firewall-cmd --zone=trusted --add-interface=ens224.300 --permanent
firewall-cmd --zone=trusted --add-interface=tun0 --permanent

firewall-cmd --reload
```

### Проверка firewalld

```bash
firewall-cmd --get-active-zones
firewall-cmd --zone=external --list-all
firewall-cmd --zone=trusted --list-all
```

### Проверка NAT

С fw-cod (после назначения временного маршрута):

```bash
ip route add 0.0.0.0/0 via 172.16.1.1
ping -c 3 8.8.8.8
```

На rtr-cod проверка таблицы трансляции:

```bash
conntrack -L
```

---

## 6. Установка и настройка FRR

### Установка (на обоих роутерах)

```bash
apt-get install frr
```

### Включение демонов

Отредактируйте `/etc/frr/daemons` — включите нужные демоны:

На **rtr-cod** (BGP + OSPF):

```
zebra=yes
bgpd=yes
ospfd=yes
```

На **rtr-a** (только OSPF):

```
zebra=yes
ospfd=yes
```

### Запуск FRR

```bash
systemctl enable frr
systemctl start frr
```

### Вход в консоль FRR

```bash
vtysh
```

---

## 7. Настройка BGP на rtr-cod

Через `vtysh` на rtr-cod:

```bash
vtysh
```

```
configure terminal

router bgp 64500
 bgp router-id 178.207.179.4
 neighbor 178.207.179.1 remote-as 31133
 !
 address-family ipv4 unicast
  network 178.207.179.0/29
 exit-address-family
exit

write memory
exit
```

### Проверка BGP

```bash
vtysh -c "show ip bgp summary"
```

Убедитесь, что состояние соседа `178.207.179.1` — `Established`.

Проверка маршрута по умолчанию (должен быть получен по BGP):

```bash
vtysh -c "show ip route"
```

Проверка доступа в Интернет:

```bash
ping -c 3 8.8.8.8
```

---

## 8. Настройка OSPF между rtr-cod и rtr-a

### rtr-cod: OSPF

Через `vtysh`:

```bash
vtysh
```

```
configure terminal

router ospf
 ospf router-id 10.10.10.1
 passive-interface default
 redistribute static
exit

interface tun0
 ip ospf area 0
 no ip ospf passive
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 P@ssw0rd
exit

interface ens224
 ip ospf area 0
exit

write memory
exit
```

### rtr-a: OSPF

Через `vtysh`:

```bash
vtysh
```

```
configure terminal

router ospf
 ospf router-id 10.10.10.2
 passive-interface default
exit

interface tun0
 ip ospf area 0
 no ip ospf passive
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 P@ssw0rd
exit

interface ens224.100
 ip ospf area 0
exit

interface ens224.200
 ip ospf area 0
exit

interface ens224.300
 ip ospf area 0
exit

write memory
exit
```

### Проверка OSPF (на обоих роутерах)

```bash
vtysh -c "show ip ospf neighbor"
vtysh -c "show ip ospf interface"
vtysh -c "show ip route ospf"
```

Убедитесь, что:
- Соседство установлено (State: Full)
- Маршруты до удалённых сетей получены по OSPF

Проверка связности между офисами:

```bash
# С rtr-cod
ping -c 3 172.20.10.254

# С rtr-a
ping -c 3 172.16.1.1
```

---

## Итоговая структура файлов etcnet

### rtr-cod

```
/etc/net/ifaces/
├── ens192/
│   ├── options          # TYPE=eth
│   └── ipv4address      # 178.207.179.4/29
├── ens224/
│   ├── options          # TYPE=eth
│   ├── ipv4address      # 172.16.1.1/30
│   └── ipv4route        # статические маршруты к сетям COD
└── tun0/
    ├── options          # TYPE=iptun, TUNTYPE=gre
    └── ipv4address      # 10.10.10.1/30
```

### rtr-a

```
/etc/net/ifaces/
├── ens192/
│   ├── options          # TYPE=eth
│   ├── ipv4address      # 178.207.179.28/29
│   └── ipv4route        # default via 178.207.179.25
├── ens224/
│   └── options          # TYPE=eth
├── ens224.100/
│   ├── options          # TYPE=vlan, HOST=ens224, VID=100
│   └── ipv4address      # 172.20.10.254/24
├── ens224.200/
│   ├── options          # TYPE=vlan, HOST=ens224, VID=200
│   └── ipv4address      # 172.20.20.254/24
├── ens224.300/
│   ├── options          # TYPE=vlan, HOST=ens224, VID=300
│   └── ipv4address      # 172.20.30.254/24
└── tun0/
    ├── options          # TYPE=iptun, TUNTYPE=gre
    └── ipv4address      # 10.10.10.2/30
```
