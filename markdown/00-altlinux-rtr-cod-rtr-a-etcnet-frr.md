# Настройка маршрутизаторов rtr-cod и rtr-a на ALT Linux (etcnet + FRR)

> **Альтернативный вариант реализации.** Данная инструкция заменяет пункты 1, 3, 4, 5, 6 основного руководства (EcoRouter) на конфигурацию для ALT Linux с использованием **etcnet** для сетевых настроек и **FRR** для маршрутизации.

---

## Содержание

1. [Подготовка системы](#1-подготовка-системы)
2. [rtr-cod: имя хоста и IP-адресация](#2-rtr-cod-имя-хоста-и-ip-адресация)
3. [rtr-a: имя хоста и IP-адресация](#3-rtr-a-имя-хоста-и-ip-адресация)
4. [Настройка GRE-туннеля](#4-настройка-gre-туннеля)
5. [Настройка NAT (masquerade)](#5-настройка-nat-masquerade)
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
apt-get install frr etcnet iptables
```

### Включение маршрутизации (forwarding)

```bash
echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
sysctl -p
```

---

## 2. rtr-cod: имя хоста и IP-адресация

### Назначение имени хоста

```bash
hostnamectl set-hostname rtr-cod
```

Для назначения доменного имени добавьте запись в `/etc/hosts`:

```bash
echo '127.0.0.1 rtr-cod.cod.ssa2026.region rtr-cod' >> /etc/hosts
```

### Настройка интерфейса ens192 (ISP, аналог te0)

```bash
mkdir -p /etc/net/ifaces/ens192
```

`/etc/net/ifaces/ens192/options`:

```
TYPE=eth
BOOTPROTO=static
ONBOOT=yes
CONFIG_IPV4=yes
```

`/etc/net/ifaces/ens192/ipv4address`:

```
178.207.179.4/29
```

### Настройка интерфейса ens224 (fw-cod, аналог te1)

```bash
mkdir -p /etc/net/ifaces/ens224
```

`/etc/net/ifaces/ens224/options`:

```
TYPE=eth
BOOTPROTO=static
ONBOOT=yes
CONFIG_IPV4=yes
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
hostnamectl set-hostname rtr-a
echo '127.0.0.1 rtr-a.a.ssa2026.region rtr-a' >> /etc/hosts
```

### Настройка интерфейса ens192 (ISP, аналог te0)

```bash
mkdir -p /etc/net/ifaces/ens192
```

`/etc/net/ifaces/ens192/options`:

```
TYPE=eth
BOOTPROTO=static
ONBOOT=yes
CONFIG_IPV4=yes
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

### Настройка VLAN-интерфейсов на ens224 (аналог te1)

На rtr-a вместо одного интерфейса создаются VLAN-подинтерфейсы для маршрутизации между VLAN.

Создаём базовый интерфейс:

```bash
mkdir -p /etc/net/ifaces/ens224
```

`/etc/net/ifaces/ens224/options`:

```
TYPE=eth
BOOTPROTO=static
ONBOOT=yes
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

## 5. Настройка NAT (masquerade)

### rtr-cod: NAT для выхода внутренних сетей в Интернет

Статические маршруты к внутренним сетям COD (через fw-cod):

```bash
mkdir -p /etc/net/ifaces/ens224
```

Добавьте в `/etc/net/ifaces/ens224/ipv4route`:

```
192.168.10.0/24 via 172.16.1.2
192.168.30.0/24 via 172.16.1.2
192.168.40.0/24 via 172.16.1.2
192.168.50.0/24 via 172.16.1.2
```

> Сеть 192.168.20.0/24 (VLAN DATA) **не маршрутизируется** по условиям задания.

Правила NAT (iptables):

```bash
iptables -t nat -A POSTROUTING -o ens192 -s 172.16.1.0/30 -j MASQUERADE
iptables -t nat -A POSTROUTING -o ens192 -s 192.168.10.0/24 -j MASQUERADE
iptables -t nat -A POSTROUTING -o ens192 -s 192.168.30.0/24 -j MASQUERADE
iptables -t nat -A POSTROUTING -o ens192 -s 192.168.40.0/24 -j MASQUERADE
iptables -t nat -A POSTROUTING -o ens192 -s 192.168.50.0/24 -j MASQUERADE
```

Для сохранения правил после перезагрузки:

```bash
iptables-save > /etc/sysconfig/iptables
systemctl enable iptables
```

### rtr-a: NAT для выхода внутренних сетей в Интернет

```bash
iptables -t nat -A POSTROUTING -o ens192 -s 172.20.10.0/24 -j MASQUERADE
iptables -t nat -A POSTROUTING -o ens192 -s 172.20.20.0/24 -j MASQUERADE
iptables -t nat -A POSTROUTING -o ens192 -s 172.20.30.0/24 -j MASQUERADE
```

Сохранение:

```bash
iptables-save > /etc/sysconfig/iptables
systemctl enable iptables
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
 no passive-interface tun0
 network 10.10.10.0/30 area 0
 network 172.16.1.0/30 area 0
 redistribute static
exit

interface tun0
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 P@ssw0rd
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
 no passive-interface tun0
 network 10.10.10.0/30 area 0
 network 172.20.10.0/24 area 0
 network 172.20.20.0/24 area 0
 network 172.20.30.0/24 area 0
exit

interface tun0
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 P@ssw0rd
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
│   ├── options          # TYPE=eth, BOOTPROTO=static
│   └── ipv4address      # 178.207.179.4/29
├── ens224/
│   ├── options          # TYPE=eth, BOOTPROTO=static
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
│   ├── options          # TYPE=eth, BOOTPROTO=static
│   ├── ipv4address      # 178.207.179.28/29
│   └── ipv4route        # default via 178.207.179.25
├── ens224/
│   └── options          # TYPE=eth, BOOTPROTO=static
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
