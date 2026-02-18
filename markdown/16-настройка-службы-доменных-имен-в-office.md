# 16. Настройка службы доменных имен в OFFICE

### Вариант реализации:

#### 

#### dc-a (alt-server):

* Средствами **samba-tool** можно посмотреть список зон DNS:

![](images/16_image_91e7e24ff6.png)

* Средствами **samba-tool** можно посмотреть список DNS-записей в определённой зоне:

![](images/16_image__1__c3664db834.png)

* Средствами **samba-tool** создадим не достающие записи типа **А** в зоне прямого просмотра:

```bash
samba-tool dns add 127.0.0.1 office.ssa2026.region rtr-a A 172.20.10.254 -U administrator
samba-tool dns add 127.0.0.1 office.ssa2026.region rtr-a A 172.20.20.254 -U administrator
samba-tool dns add 127.0.0.1 office.ssa2026.region rtr-a A 172.20.30.254 -U administrator
samba-tool dns add 127.0.0.1 office.ssa2026.region sw1-a A 172.20.30.1 -U administrator
samba-tool dns add 127.0.0.1 office.ssa2026.region sw2-a A 172.20.30.2 -U administrator
```

* Проверить наличие записей:

![](images/16_image__2__e3228c4d98.png)

* Проверить функционально:

![](images/16_image__3__032b803e9b.png)

* Добавить в конфигурационный файл **/etc/bind/local.conf** информацию о файле зоны обратного просмотра и о зонах на **srv1-cod**:

![](images/16_image__4__346bd9ee80.png)

* Скопировать файл шаблона для зоны обратного просмотра:

```bash
cp /etc/bind/zone/localhost /etc/bind/zone/20.172.in-addr.arpa
```

* Выдать права на файл зоны обратного просмотра:

```bash
chown root:named /etc/bind/zone/20.172.in-addr.arpa
```

* Привести файл **/etc/bind/zone/20.172.in-addr.arpa** зоны обратного просмотра к следующему виду:

![](images/16_image__5__f29d867cf7.png)

* Перезапустить службу **bind**:

```bash
systemctl restart bind
```

* Проверить записи типа **PTR**:

![](images/16_image__6__e284c93782.png)

* Проверить несколько записей для forward зон:

![](images/16_image__7__64d9b16638.png)

#### sw1-a и sw2-a (alt-server):

* Задаём в качестве DNS-сервера **dc-a**:

```bash
cat <<EOF > /etc/net/ifaces/mgmt/resolv.conf
  search office.ssa2026.region
  nameserver 172.20.10.10
EOF
```

* Перезагружаем службу **network**:

```bash
systemctl restart network
```

* Проверить:

![](images/16_image__8__cff93f3795.png)

Последнее изменение: среда, 19 ноября 2025, 15:59