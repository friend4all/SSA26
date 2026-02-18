# 7. Настройка коммутации между sw1-a и sw2-a

### Вариант реализации:

#### 

#### sw1-a (alt-server):

##### Назначение имени на устройство:

* Для назначения имени устройства согласно топологии используем следующую команду:

```bash
hostnamectl set-hostname sw1-a.office.ssa2026.region; exec bash
```

* Так же рекомендуется указать имя в файле **/etc/sysconfig/network**:

```bash
vim /etc/sysconfig/network
```

* + указать имя в параметре **HOSTNAME**:

![](images/07_image__1__2ce239c084.png)

* Проверить можно с помощью команды **hostname** с ключём **-f**:

![](images/07_image__2__2ff1bdb370.png)

##### Установка пакета Open vSwitch:

* Для установки пакета ****openvswitch****необходим доступ в сеть Интернет, для этого необходимо на виртуальной машине **sw1-a**:
  + назначив средствами **iproute2** временно на интерфейс,смотрящий в сторону **rtr-a (ens19)**,
  + тегированный подинтерфейс с IP-адресом из подсети для **vlan300**, а также шлюзом по умолчанию и DNS

```bash
ip link add link ens19 name ens19.300 type vlan id 300
ip link set dev ens19.300 up
ip addr add 172.20.30.1/24 dev ens19.300
ip route add 0.0.0.0/0 via 172.20.30.254
echo "nameserver 77.88.8.8" > /etc/resolv.conf
```

* Почле чего обновляем список пакетов и устанавливаем **openvswitch**:

```bash
apt-get update && apt-get install -y openvswitch
```

* Включаем и добавляем в автозагрузку **openvswitch**:

```bash
systemctl enable --now openvswitch
```

* Правим основной файл **options** в котором по умолчанию сказано:
  + удалять настройки заданые через **ovs-vsctl,**
  + т.к. через **etcnet** будет выполнено только создание интерфейса типа **internal**
  + с назначением необходимого IP-адреса, а настройка коммутации будет выполнена средствами **openvswitch**

```bash
sed -i "s/OVS_REMOVE=yes/OVS_REMOVE=no/g" /etc/net/ifaces/default/options
```

* Перезагрузить сервер - будет быстрее чем удалять параметры заданые в ручную через пакет **iproute2**:

```bash
reboot
```

* Проверяем интерфейсы и определяемся какой к кому направлен:
  + таким образом, имеем:
    - **ens19** - интерфейс в сторону **rtr-a**;
    - **ens20** - интерфейс в строну **dc-a**;
    - **ens21** - интерфейс в сторону **sw2-a**;
    - **ens22** - интерфейс в сторону **sw2-a**;

![](images/07_image_45d7c89d9f.png)

* Поднимаем физические интерфейсы, создавая директорию для каждого интерфейса в **/etc/net/ifaces** и описывая файл **options**:
  + если в файле **options** для порты **ens19** есть параметры **TYPE=eth** и **BOOTPROTO=static,**
  + то можно рекурсивно выполнить копирование **ens19**во все необходимые каталоги

```bash
cp -r /etc/net/ifaces/ens19 /etc/net/ifaces/ens20
```

```bash
cp -r /etc/net/ifaces/ens19 /etc/net/ifaces/ens21
```

```bash
cp -r /etc/net/ifaces/ens19 /etc/net/ifaces/ens22
```

* Перезагружаем службу **network**:

```bash
systemctl restart network
```

* Проверить, что интерфейсы перешли из статуса **DOWN** в статус **UP**:

![](images/07_image__3__59f64b4be0.png)

* Чтобы на **sw2-a** была возможность установить пакет **openvswitch**:
  + временно сосдадим простой коммутатор с именем, например **br0**
  + и добавим в него интерфейсы **ens19** в сторону **rtr-a** и **ens21** в сторону **sw2-a**

```bash
ovs-vsctl add-br br0
```

```bash
ovs-vsctl add-port br0 ens19
```

```bash
ovs-vsctl add-port br0 ens21
```

#### sw2-a (alt-server):

##### Назначение имени на устройство:

* Реализация аналогично **sw1-a**:

![](images/07_image__5__fd7789cfeb.png)

* Для установки пакета ****openvswitch****необходим доступ в сеть Интернет, для этого необходимо на виртуальной машине **sw2-a**:
  + назначив средствами **iproute2** временно на интерфейс,смотрящий в сторону **sw1-a (ens19)**,
  + тегированный подинтерфейс с IP-адресом из подсети для **vlan300**, а также шлюзом по умолчанию и DNS

```bash
ip link add link ens19 name ens19.300 type vlan id 300
ip link set dev ens19.300 up
ip addr add 172.20.30.2/24 dev ens19.300
ip route add 0.0.0.0/0 via 172.20.30.254
echo "nameserver 77.88.8.8" > /etc/resolv.conf
```

* Почле чего обновляем список пакетов и устанавливаем **openvswitch**:

```bash
apt-get update && apt-get install -y openvswitch
```

* Включаем и добавляем в автозагрузку **openvswitch**:

```bash
systemctl enable --now openvswitch
```

* Правим основной файл **options** в котором по умолчанию сказано:
  + удалять настройки заданые через **ovs-vsctl,**
  + т.к. через **etcnet** будет выполнено только создание интерфейса типа **internal**
  + с назначением необходимого IP-адреса, а настройка коммутации будет выполнена средствами **openvswitch**

```bash
sed -i "s/OVS_REMOVE=yes/OVS_REMOVE=no/g" /etc/net/ifaces/default/options
```

* Перезагрузить сервер - будет быстрее чем удалять параметры заданые в ручную через пакет **iproute2**:

```bash
reboot
```

#### sw1-a (alt-server):

##### Настройка коммутации:

* Удаляем ранее созанный временный коммутатор с именем **br0**:

```bash
ovs-vsctl del-br br0
```

* Создадим коммутатор с именем **sw1-a**:

```bash
ovs-vsctl add-br sw1-a
```

* Проверить создание коммутатора можно с помощью команды **ovs-vsctl show**:

![](images/07_image__6__f1094ef293.png)

* Добавим интерфейс, направленный в сторону **dc-a** (ens20) в созданный коммутатор и назначим его в качестве порта доступа (access), указав принадлежность к **VLAN 100**:

```bash
ovs-vsctl add-port sw1-a ens20 tag=100
```

* Интерфейс в сторону **rtr-a** (ens19) добавляем в созданный коммутатор, но настраиваем как магистральный (trunk) порт:
  + также разрешаем пропуск только требуемых VLAN (100,200 и 300)

```bash
ovs-vsctl add-port sw1-a ens19 trunk=100,200,300
```

* Интерфейсы в стороны **sw2-a** (ens21, ens22) добавляем в созданный коммутатор, но настраиваем как магистральный (trunk) порт:
  + также разрешаем пропуск только требуемых VLAN (100,200 и 300)

```bash
ovs-vsctl add-port sw1-a ens21 trunk=100,200,300
```

```bash
ovs-vsctl add-port sw1-a ens22 trunk=100,200,300
```

* Проверить добавление портов в коммутатор можно с помощью команды **ovs-vsctl show**:

![](images/07_image__7__694f51d630.png)

* Включаем модуль ядра отвечающий за тегированный трафик (**802.1Q**):

```bash
modprobe 8021q
```

```bash
echo "8021q" | tee -a /etc/modules
```

* Запускаем процесс STP на коммутаторе:
  + в режиме **802.1w** (rstp)

```bash
ovs-vsctl set bridge sw1-a rstp_enable=true
```

```bash
ovs-vsctl set bridge sw1-a other_config:stp-protocol=rstp
```

* Задаём данному коммутатору наименьший приоритет:

```bash
ovs-vsctl set bridge sw1-a other_config:rstp-priority=0
```

* Проверить заданный приоритет можно с помощью команды **ovs-appctl rstp/show:**

![](images/07_image__9__235b168399.png)

* Проверить заданный протокол и приоритет можно с помощью команды **ovs-vsctl list bridge:**

![](images/07_image__10__725d88fd14.png)

* Сетевая подсистема **etcnet** будет взаимодействовать с **openvswitch**, для того чтобы корректно можно было назначить  IP-адрес на  интерфейс управления  
  + создаём каталог для management интерфейса с именем **mgmt**:

```bash
mkdir /etc/net/ifaces/mgmt
```

* Описываем файл **options** для создания management интерфейса с именем **mgmt**:

```bash
vim /etc/net/ifaces/mgmt/options
```

* + где:
    - **TYPE** - тип интерфейса (**internal**);
    - **BOOTPROTO** - определяет как будут назначаться сетевые параметры (статически);
    - **CONFIG\_IPV4** - определяет использовать конфигурацию протокола IPv4 или нет;
    - **BRIDGE** - определяет к какому мосту необходимо добавить данный интерфейс;
    - **VID** - определяет принадлежность интерфейса к VLAN;

![](images/07_image__11__afc2a33103.png)

* Назначаем IP-адрес и шлюз на созданный интерфейс **mgmt**:

```bash
echo "172.20.30.1/24" > /etc/net/ifaces/mgmt/ipv4address
```

```bash
echo "default via 172.20.30.254" > /etc/net/ifaces/mgmt/ipv4route
```

* Для применения настроек, необходимо перезагрузить службу **network**:

```bash
systemctl restart network
```

* Проверить назначенный IP-адрес можно командой **ip a**:

![](images/07_image__14__9f95c72e48.png)

* Проверить назначенный IP-адрес шлюза по умолчанию можно команжой **ip r**:

![](images/07_image__13__a44990b3c5.png)

* Также стоит с помощью команды **ovs-vsctl show** проверить, что данный интерфейс добавился в коммутатор **sw1-a**:

![](images/07_image__15__f8f28d7670.png)

* Помимо того, что интерфейс **mgmt** является портом доступа (access) необходимо использовать NativeVLAN:

```bash
ovs-vsctl set port mgmt vlan_mode=native-untagged
```

* Проверить можно с помощью команды **ovs-vsctl list port mgmt:**

![](images/07_image__16__4c8bd67636.png)

#### sw2-a (alt-server):

* Проверяем интерфейсы и определяемся какой к кому направлен:
  + таким образом, имеем:
    - **ens19** - интерфейс в сторону **sw-1**;
    - **ens20** - интерфейс в строну **sw-1**;
    - **ens21** - интерфейс в сторону **cli2-a**;
    - **ens22** - интерфейс в сторону **cli1-a**;

![](images/07_image__17__cf3d12ae87.png)

* Поднимаем физические интерфейсы, создавая директорию для каждого интерфейса в **/etc/net/ifaces** и описывая файл **options**:
  + если в файле **options** для порты **ens19** есть параметры **TYPE=eth** и **BOOTPROTO=static,**
  + то можно рекурсивно выполнить копирование **ens19**во все необходимые каталоги

```bash
cp -r /etc/net/ifaces/ens19 /etc/net/ifaces/ens20
```

```bash
cp -r /etc/net/ifaces/ens19 /etc/net/ifaces/ens21
```

```bash
cp -r /etc/net/ifaces/ens19 /etc/net/ifaces/ens22
```

* Перезагружаем службу **network**:

```bash
systemctl restart network
```

* Проверить, что интерфейсы перешли из статуса **DOWN** в статус **UP**:

![](images/07_image__18__5ae8b4df64.png)

##### Настройка коммутации:

* Создадим коммутатор с именем **sw2-a**:

```bash
ovs-vsctl add-br sw2-a
```

* Проверить создание коммутатора можно с помощью команды **ovs-vsctl show**:

![](images/07_image__19__0186c2c594.png)

* Добавим интерфейс, направленный в сторону **cli1-a** (ens22) в созданный коммутатор и назначим его в качестве порта доступа (access), указав принадлежность к **VLAN 200**:

```bash
ovs-vsctl add-port sw2-a ens22 tag=200
```

* Добавим интерфейс, направленный в сторону **cli2-a** (ens21) в созданный коммутатор и назначим его в качестве порта доступа (access), указав принадлежность к **VLAN 200**:

```bash
ovs-vsctl add-port sw2-a ens21 tag=200
```

* Интерфейсы в стороны **sw1-a** (ens19, ens20) добавляем в созданный коммутатор, но настраиваем как магистральный (trunk) порт:
  + также разрешаем пропуск только требуемых VLAN (100,200 и 300)

```bash
ovs-vsctl add-port sw1-a ens19 trunk=100,200,300
```

```bash
ovs-vsctl add-port sw1-a ens20 trunk=100,200,300
```

* Проверить добавление портов в коммутатор можно с помощью команды **ovs-vsctl show**:

![](images/07_image__20__dec42492b7.png)

* Включаем модуль ядра отвечающий за тегированный трафик (**802.1Q**):

```bash
modprobe 8021q
```

```bash
echo "8021q" | tee -a /etc/modules
```

* Запускаем процесс STP на коммутаторе:
  + в режиме **802.1w** (rstp)

```bash
ovs-vsctl set bridge sw2-a rstp_enable=true
```

```bash
ovs-vsctl set bridge sw2-a other_config:stp-protocol=rstp
```

* Проверить можно с помощью команды **ovs-appctl rstp/show:**

![](images/07_image__21__8607ce37cc.png)

* Проверить заданный протокол и приоритет можно с помощью команды **ovs-vsctl list bridge:**

![](images/07_image__22__8cf99a7ec1.png)

* Сетевая подсистема **etcnet** будет взаимодействовать с **openvswitch**, для того чтобы корректно можно было назначить  IP-адрес на  интерфейс управления  
  + создаём каталог для management интерфейса с именем **mgmt**:

```bash
mkdir /etc/net/ifaces/mgmt
```

* Описываем файл **options** для создания management интерфейса с именем **mgmt**:

```bash
vim /etc/net/ifaces/mgmt/options
```

* + где:
    - **TYPE** - тип интерфейса (**internal**);
    - **BOOTPROTO** - определяет как будут назначаться сетевые параметры (статически);
    - **CONFIG\_IPV4** - определяет использовать конфигурацию протокола IPv4 или нет;
    - **BRIDGE** - определяет к какому мосту необходимо добавить данный интерфейс;
    - **VID** - определяет принадлежность интерфейса к VLAN;

![](images/07_image__23__8ffab52771.png)

* Назначаем IP-адрес и шлюз на созданный интерфейс **mgmt**:

```bash
echo "172.20.30.2/24" > /etc/net/ifaces/mgmt/ipv4address
```

```bash
echo "default via 172.20.30.254" > /etc/net/ifaces/mgmt/ipv4route
```

* Для применения настроек, необходимо перезагрузить службу **network**:

```bash
systemctl restart network
```

* Проверить назначенный IP-адрес можно командой **ip a**:

![](images/07_image__25__2d9eb6c1fa.png)

* Проверить назначенный IP-адрес шлюза по умолчанию можно команжой **ip r**:

![](images/07_image__26__6c132879a7.png)

* Также стоит с помощью команды **ovs-vsctl show** проверить, что данный интерфейс добавился в коммутатор **sw2-a**:

![](images/07_image__27__502cc87ca3.png)

* Помимо того, что интерфейс **mgmt** является портом доступа (access) необходимо использовать NativeVLAN:

```bash
ovs-vsctl set port mgmt vlan_mode=native-untagged
```

* Проверить можно с помощью команды **ovs-vsctl list port mgmt:**

![](images/07_image__29__09e9035c90.png)

* Проверить доступность коммутаторов **sw1-a** и **sw2-a** можно как с маршрутизатора **rtr-a**:

![](images/07_image__30__2300e2bd3b.png)

* + так и с маршрутизатора **rtr-cod**:

![](images/07_image__33__c87b10933e.png)

* Также с коммутаторов **sw1-a** и **sw2-a** должна быть доступна сеть Интернет:

![](images/07_image__31__18a96515ec.png)

![](images/07_image__32__d714a481b7.png)

Последнее изменение: пятница, 14 ноября 2025, 11:25