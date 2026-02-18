# 19. Настройка устройства хранения данных (iSCSI)

### Вариант реализации:

#### 

#### srv2-cod (alt-server):

* Установить пакет **scsitarget-utils**:

```bash
apt-get update && apt-get install -y scsitarget-utils
```

* Включить и добавить в автозагрузку службу **tgt**:

```bash
systemctl enable --now tgt
```

* С помощью утилибы **lsblk** посмотреть список блочных устройств и определиться с диском, который будет использоваться:
  + в данном примере это **sda**

![](images/19_image_3a8700a6a9.png)

* Настроить отдачу нашего диска по iSCSI отредактировав конфигурационный файл **/etc/tgt/targets.conf**:  
  + добавив в конец файла следующее содержимое

![](images/19_image__2__207bd47bc7.png)

* Перезапустить службу **tgt**:

```bash
systemctl restart tgt
```

* Проверить можно с помощью команды **tgtadm --lld iscsi --op show --mode target**:

![](images/19_image__3__8edee21de7.png)

* Для корректной работы необходимо указать, чтобы LVM не сканировал наши iSCSI-диски (sd\*) в конфигурационном файле **/etc/lvm/lvm.conf**:  
  + в блоке **devices**

![](images/19_image__4__997c162c2d.png)

#### srv1-cod (alt-server):

* Установим пакет **open-iscsi**:

```bash
apt-get update && apt-get install -y open-iscsi
```

* Включаем и добавляем в автозагрузку службу **iscsid**:

```bash
systemctl enable --now iscsid
```

* Посмотреть доступные для подключения target-ы можно с помощью команды:

```bash
iscsiadm -m discovery -t sendtargets -p 192.168.20.2
```

* Подключить target-ы:

```bash
iscsiadm -m node --login
```

* В файле **/etc/iscsi/iscsid.conf** внести изменения:
  + закомментировать **node.startup = manual**
  + раскомментировать **node.startup = automatic**

![](images/19_image__7__6ad33f2b7c.png)

* В файле **/var/lib/iscsi/send\_targets/<TargetServer>,<Port>/st\_config** внести изменения:
  + параметр **discovery.sendtargets.use\_discoveryd = No** поменять на **discovery.sendtargets.use\_discoveryd = Yes**

![](images/19_image__8__c657ce5022.png)

* Выполнить перезагрузку устройства:

```bash
reboot
```

* Проверить командой **lsblk** наличие блочного устройства подключённого по сети с **srv2-cod**:
  + у **srv1-cod** всего **1 диск** на **20 ГБ**

![](images/19_image__9__459354361e.png)

* + значит диск в **5 гб** тот самый по iSCSI:

![](images/19_image__10__9a6eb0157c.png)

Последнее изменение: пятница, 21 ноября 2025, 08:19