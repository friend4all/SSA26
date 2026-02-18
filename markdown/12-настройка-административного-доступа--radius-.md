# 12. Настройка административного доступа (RADIUS)

### Вариант реализации:

#### 

#### srv1-cod (alt-server):

* Временно для возможности установки необходимых пакетов зададим публичный DNS-сервер:

```bash
echo "nameserver 77.88.8.8" > /etc/resolv.conf
```

* Устанавливаем пакеты **freeradius** и **freeradius-utils**:

```bash
apt-get update && apt-get install -y freeradius freeradius-utils
```

* Включаем службу **radiusd** в автозагрузку и запускаем ее:

```bash
systemctl enable --now radiusd
```

* Создаем клиентов добавляя в файле **/etc/raddb/clients.conf** следующее содержимое:

  + пример конфигурации файла **/etc/raddb/clients.conf** для любого клиента

![](../images/12_image_a4451a6066.png)

* Редактируем файл с пользователями **/etc/raddb/users** добавляя в самый конец следующее содержимое:

![](../images/12_image__2__110471a752.png)

* Перезапускаем службу **radiusd**:

```bash
systemctl restart radiusd
```

#### rtr-cod (ecorouter):

* Выполняем подключение к RADIUS-серверу:

```bash
rtr-cod(config)#security none 
rtr-cod(config)#aaa radius-server 192.168.10.1 port 1812 secret P@ssw0rd auth
rtr-cod(config)#
```

* Задаём приоритет:

```bash
rtr-cod(config)#aaa precedence local radius
rtr-cod(config)#write memory
Building configuration...

rtr-cod(config)#
```

* Проверить возможность входа из-под пользователя **netuser** с паролем **P@ssw0rd**:

![](../images/12_image__3__f5ad2a3fd7.png)

* Проверить вход под локальной учетной записью даже при доступности RADIUS-сервера:

![](../images/12_image__4__7ff589c451.png)

* Проверить доступ по **SSH** из-под пользователя **netuser** с паролем **P@ssw0rd**:
  + например с **srv1-cod**

![](../images/12_image__5__d0666d7fbf.png)

#### sw1-cod и sw2-cod (alt-server):

* Временно для возможности установки необходимых пакетов зададим публичный DNS-сервер:

```bash
echo "nameserver 77.88.8.8" > /etc/resolv.conf
```

* Устанавливаем пакет **pam\_radius**:

```bash
apt-get update && apt-get install -y pam_radius
```

* Редактируем конфигурационный файл **/etc/pam\_radius\_auth.conf**:

![](../images/12_image__6__967e9504e7.png)

* Редактируем конфигурационный файл **/etc/pam.d/sshd**:

![](../images/12_image__8__61bc6848b6.png)

* Редактируем конфигурационный файл **/etc/pam.d/system-auth-local**:

![](../images/12_image__9__db727aa8fe.png)

* Также в случае с **Linux**, данного пользователя необходимо создать локально:

```bash
useradd netuser
```

* Проверить возможность входа из-под пользователя **netuser** с паролем **P@ssw0rd**:

![](../images/12_image__10__7ed233963b.png)

* Проверить доступ по **SSH** из-под пользователя **netuser** с паролем **P@ssw0rd**:
  + например с **srv1-cod**

![](../images/12_image__11__a69025f907.png)

Последнее изменение: понедельник, 17 ноября 2025, 13:01