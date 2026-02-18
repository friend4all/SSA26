# Настройка маршрутизаторов rtr-cod и rtr-a на ALT Linux (etcnet + FRR)

> **Альтернативный вариант реализации.** Данная инструкция заменяет пункты 1, 3, 4, 5, 6, 12 основного руководства (EcoRouter) на конфигурацию для ALT Linux с использованием **etcnet** для сетевых настроек, **FRR** для маршрутизации и **pam_radius** для авторизации.

---

## Содержание

1. [rtr-cod: имя хоста и сетевые интерфейсы](#1-rtr-cod-имя-хоста-и-сетевые-интерфейсы)
2. [rtr-a: имя хоста и сетевые интерфейсы](#2-rtr-a-имя-хоста-и-сетевые-интерфейсы)
3. [Настройка GRE-туннеля](#3-настройка-gre-туннеля)
4. [Установка пакетов](#4-установка-пакетов)
5. [Настройка FRR](#5-настройка-frr)
6. [Настройка сетевого экрана](#6-настройка-сетевого-экрана)
7. [Настройка авторизации по RADIUS](#7-настройка-авторизации-по-radius)

---

## Справочная информация

### Структура etcnet

Конфигурация сетевых интерфейсов в ALT Linux хранится в `/etc/net/ifaces/`. Каждому интерфейсу соответствует отдельная директория с файлами:

| Файл | Назначение |
|---|---|
| `options` | Тип интерфейса и параметры |
| `ipv4address` | IPv4-адреса (по одному на строку) |
| `ipv4route` | Статические маршруты |
| `resolv.conf` | DNS-серверы для интерфейса |

### Имена физических интерфейсов

В зависимости от гипервизора интерфейсы могут называться `eth0`/`eth1` или `ens192`/`ens224`. Определите их командой:

```bash
ip link show
```

В данной инструкции используются:
- **ens192** — порт в сторону ISP (аналог te0 в EcoRouter)
- **ens224** — порт в сторону fw-cod / коммутаторов (аналог te1 в EcoRouter)

> Замените имена интерфейсов на актуальные для вашей конфигурации.

---

## 1. rtr-cod: имя хоста и сетевые интерфейсы

### 1.1. Назначение имени хоста

```bash
hostnamectl set-hostname rtr-cod.cod.ssa2026.region
echo '127.0.0.1 rtr-cod.cod.ssa2026.region rtr-cod' >> /etc/hosts
```

### 1.2. Включение маршрутизации (forwarding)

В ALT Linux forwarding по умолчанию отключён в `/etc/net/sysctl.conf`. Перезаписываем файл:

```bash
echo 'net.ipv4.ip_forward = 1' > /etc/net/sysctl.conf
sysctl -p
```

> **Важно:** файл `/etc/net/sysctl.conf` применяется при перезапуске сети и перезаписывает системный `sysctl.conf`. Поэтому его нужно перезаписать целиком через `echo >`.

### 1.3. Настройка интерфейса ens192

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

Временный маршрут по умолчанию (будет удалён после настройки BGP) и DNS:

`/etc/net/ifaces/ens192/ipv4route`:

```
default via 178.207.179.1
```

`/etc/net/ifaces/ens192/resolv.conf`:

```
nameserver 100.100.100.100
```

### 1.4. Настройка интерфейса ens224

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

### 1.5. Применение и проверка

```bash
systemctl restart network
```

```bash
ip addr show ens192
ip addr show ens224
ping -c 3 178.207.179.1
ping -c 3 ya.ru
```

---

## 2. rtr-a: имя хоста и сетевые интерфейсы

### 2.1. Назначение имени хоста

```bash
hostnamectl set-hostname rtr-a.office.ssa2026.region
echo '127.0.0.1 rtr-a.office.ssa2026.region rtr-a' >> /etc/hosts
```

### 2.2. Включение маршрутизации (forwarding)

```bash
echo 'net.ipv4.ip_forward = 1' > /etc/net/sysctl.conf
sysctl -p
```

### 2.3. Настройка интерфейса ens192

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

### 2.4. Маршрут по умолчанию и DNS

`/etc/net/ifaces/ens192/ipv4route`:

```
default via 178.207.179.25
```

`/etc/net/ifaces/ens192/resolv.conf`:

```
nameserver 100.100.100.100
```

### 2.5. Настройка VLAN-интерфейсов на ens224

> **ens224** — внутренний интерфейс, обращён в сторону коммутаторов **sw1-a** и **sw2-a**

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

### 2.6. Применение и проверка

```bash
systemctl restart network
```

```bash
ip addr show
ip -d link show ens224.100
ip -d link show ens224.200
ip -d link show ens224.300
ping -c 3 178.207.179.25
ping -c 3 ya.ru
```

---

## 3. Настройка GRE-туннеля

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

## 4. Установка пакетов

> Сеть уже настроена и доступ в Интернет есть — можно устанавливать пакеты.

### На rtr-cod (FRR + firewalld + RADIUS)

```bash
apt-get update
apt-get install -y frr firewalld pam_radius
```

### На rtr-a (FRR + firewalld + RADIUS)

```bash
apt-get update
apt-get install -y frr firewalld pam_radius
```

---

## 5. Настройка FRR

### 5.1. Включение демонов

Отредактируйте `/etc/frr/daemons`:

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

### 5.2. Запуск FRR (на обоих роутерах)

```bash
systemctl enable --now frr
```

### 5.3. Настройка BGP на rtr-cod

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

Проверка BGP:

```bash
vtysh -c "show ip bgp summary"
vtysh -c "show ip route"
```

Убедитесь, что маршрут по умолчанию получен по BGP. После этого **удалите временный статический маршрут**:

```bash
echo -n > /etc/net/ifaces/ens192/ipv4route
```

### 5.4. Настройка OSPF на rtr-cod

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

### 5.5. Настройка OSPF на rtr-a

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

### 5.6. Проверка OSPF (на обоих роутерах)

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

### 5.7. Замена DNS на постоянный

После того как OSPF заработает и появится связность с srv1-cod, замените временный DNS:

```bash
echo 'nameserver <IP_srv1-cod>' > /etc/net/ifaces/ens192/resolv.conf
```

---

## 6. Настройка сетевого экрана

> Настройка выполняется через **firewalld**. Зона **external** включает masquerade по умолчанию.

### 6.1. Включение firewalld (на обоих роутерах)

```bash
systemctl enable --now firewalld
```

### 6.2. rtr-cod: зоны

> **ens192** (ISP) → зона **external** (masquerade), разрешить GRE
> **ens224** (fw-cod) и **tun0** (туннель) → зона **trusted**

```bash
firewall-cmd --zone=external --add-interface=ens192 --permanent
firewall-cmd --zone=external --add-protocol=gre --permanent

firewall-cmd --zone=trusted --add-interface=ens224 --permanent
firewall-cmd --zone=trusted --add-interface=tun0 --permanent

firewall-cmd --reload
```

### 6.3. rtr-a: зоны

> **ens192** (ISP) → зона **external** (masquerade), разрешить GRE
> **ens224**, VLAN-интерфейсы и **tun0** → зона **trusted**

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

### 6.4. Проверка

```bash
firewall-cmd --get-active-zones
firewall-cmd --zone=external --list-all
firewall-cmd --zone=trusted --list-all
```

Проверка NAT — с fw-cod (после назначения временного маршрута):

```bash
ip route add 0.0.0.0/0 via 172.16.1.1
ping -c 3 8.8.8.8
```

На rtr-cod проверка трансляции:

```bash
conntrack -L
```

---

## 7. Настройка авторизации по RADIUS

> RADIUS-сервер настраивается на **srv1-cod** (см. инструкцию 12). Здесь описана только клиентская часть на роутерах.

### 7.1. Настройка pam_radius (на обоих роутерах)

Пакет `pam_radius` уже установлен (шаг 4).

Создайте локального пользователя:

```bash
useradd netuser
```

Отредактируйте `/etc/pam_radius_auth.conf` — укажите RADIUS-сервер и общий секрет:

```
192.168.10.1    P@ssw0rd    3
```

> `192.168.10.1` — IP-адрес srv1-cod (RADIUS-сервер), `P@ssw0rd` — общий секрет, `3` — таймаут в секундах.

### 7.2. Настройка PAM для SSH

Отредактируйте `/etc/pam.d/sshd` — добавьте в начало файла (перед остальными строками `auth`):

```
auth sufficient pam_radius_auth.so
```

Отредактируйте `/etc/pam.d/system-auth-local` — добавьте:

```
auth sufficient pam_radius_auth.so
```

### 7.3. Проверка

Проверьте вход по SSH под пользователем **netuser** с паролем **P@ssw0rd** (например, с srv1-cod):

```bash
ssh netuser@<IP_роутера>
```

---

## Итоговая структура файлов etcnet

### rtr-cod

```
/etc/net/ifaces/
├── ens192/
│   ├── options          # TYPE=eth
│   ├── ipv4address      # 178.207.179.4/29
│   ├── ipv4route        # default убирается после настройки BGP
│   └── resolv.conf      # DNS (временно 100.100.100.100, потом srv1-cod)
├── ens224/
│   ├── options          # TYPE=eth
│   └── ipv4address      # 172.16.1.1/30
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
│   ├── ipv4route        # default via 178.207.179.25
│   └── resolv.conf      # DNS (временно 100.100.100.100, потом srv1-cod)
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
