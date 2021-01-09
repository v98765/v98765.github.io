Далее по cisco nexus 3000 модели

## Обновление ПО

Ссылки:

* [Рекомендованная версия ПО](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/nexus3000/sw/recommended_release/b_Minimum_and_Recommended_Cisco_NX-OS_Releases_for_Cisco_Nexus_3000_Series_Switches.html). Это 7.0(3)I7(7) сейчас.
* [Cisco Nexus 3000 Series NX-OS Software Upgrade and Downgrade Guide, Release 7.x](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/nexus3000/sw/upgrade/7_x/b_Cisco_Nexus_3000_Series_NX_OS_Software_Upgrade_and_Downgrade_Release_7_x/b_Cisco_Nexus_3000_Series_NX_OS_Software_Upgrade_and_Downgrade_Release_7_x_newGuide_chapter_01.html)

> If you have a Cisco Nexus 3000 release prior to Release 7.0(3)I2(1), upgrade to Cisco Nexus 3000 Release 6.0.2.U6(3) first. 

Скопировать, установить 6.0.2.U6(3), когда софт на коммутаторе слишком старый
```text
witch# show file n3000-uk9-kickstart.6.0.2.U6.3a.bin md5
f0b67d113c1d9adafc16349e7ae36cd4
switch# show file n3000-uk9.6.0.2.U6.3a.bin md5
cde4522663172733c15a15855d90d303
switch# install all system n3000-uk9.6.0.2.U6.3.bin kickstart n3000-uk9-kickstart.6.0.2.U6.3.bin
```
Проверка
```text
switch# show file bootflash:nxos.7.0.3.I7.7.bin md5sum
a9d40fbfaf43c214c3d97cb290788d06
switch# show install all impact nxos bootflash:nxos.7.0.3.I7.7.bin
```
Перезагрузка, установка рекомендуемой версии в варианте Guidelines for Upgrading in Non-Fast Reload Scenarios
```text
switch# install all nxos bootflash:nxos.7.0.3.I7.7.bin
```

Когда места нет, удалить текущий image 7 версии нельзя
```text
Switch is booted with 'nxos.7.0.3.I4.7.bin'. Overwriting/deleting this image is not allowed
``` 
Есть параметр [compact](https://www.cisco.com/c/en/us/support/docs/switches/nexus-3000-series-switches/215781-nexus-3000-3100-and-3500-nx-os-compact.html) для такого случая.
```text
switch# install all nxos bootflash:nxos.7.0.3.I4.7.bin compact
Installer will perform compatibility check first. Please wait.
Compacting currently loaded image bootflash:/nxos.7.0.3.I4.7.bin
..................................................
Compact bootflash:/nxos.7.0.3.I4.7.bin done
```
До
```text
Usage for bootflash://
  798937088 bytes used
  849686528 bytes free
 1648623616 bytes total
```
После
```text
Usage for bootflash://
  514617344 bytes used
 1134006272 bytes free
 1648623616 bytes total

```

Иногда проще вставить usb и сделать компактный имадж там
```text
install all nxos usb1:nxos.7.0.3.I4.7.bin compact
```
Полученный в итоге софт будет примерно 400гб и спокойно скопируется на bootflash
```text
copy usb1:nxos.7.0.3.I4.7.bin bootflash:compactnxos.7.0.3.I4.7.bin
```
Как загрузиться с флешки при включении питания
```text
Press  ctrl L to go to loader prompt in 2 secs

User break into bootloader

loader> boot usb1:nxos.7.0.3.I7.7.bin
```
7.0(3)I4(7)
install all nxos usb1:nxos.7.0.3.I7.7.bin compact
copy usb1:nxos.7.0.3.I7.7.bin bootflash:
install all nxos nxos.7.0.3.I7.7.bin

## работа с файлами nx-os на флешке

Взято за основу [эта заметка](https://community.cisco.com/t5/data-center-documents/getting-winscp-on-n9k-to-work-settings-config-required/ta-p/3818490)
где используется winscp для копирования файлов на nexus. В настройках winscp Enviroment\Directories указать Remote directory `/bootflash`,
в Enviroment\SCP/Shell указать Shell `run bash`. Эти пункты доступны при подключении по SCP протоколу к удаленному устройству.

На коммутаторе в режиме конфигурирования включить сервисы
```text
feature bash-shell
feature sсp-server
```

Скачать файл , посмотреть MD5 на сайте cisco (клик на имя файла)
```text
a9d40fbfaf43c214c3d97cb290788d06 nxos.7.0.3.I7.7.bin
```

В powershell (windows) посмотреть md5
```text
Get-FileHash C:\Soft\nx-os\nxos.7.0.3.I7.7.bin -Algorithm MD5 | Format-List
Algorithm : MD5
Hash      : A9D40FBFAF43C214C3D97CB290788D06
Path      : C:\Soft\nx-os\nxos.7.0.3.I7.7.bin
```
Посмотреть наличие свободного места на флешке командой `dir` и при необходимости удалить старые версии, оставив текущую.
После копирования nx-os на коммутатор проверить md5
```text
switch# show file bootflash:nxos.7.0.3.I7.7.bin md5sum
a9d40fbfaf43c214c3d97cb290788d06
```

## snmp

acl с адресом nms. В конфигурации отобразится иначе.
```
ip access-list snmp-acl
  permit udp host [snmp-ip] any eq snmp
```
community public
```
snmp-server community public group network-operator
snmp-server community public use-ipv4acl snmp-acl
```

## password recovery

При включении питания жать crtl ] или ctrl+c
```text
switch(boot)# config terminal
switch(boot)(config)# admin-password [newpass]
switch(boot)# load-nxos
```

## logging

По умолчанию включено логирование авторизаций но не хватает привилегий
```text
logging level authpriv 6
login on-success log
```


## ecmp

[Configuring ECMP for Host Routes](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/nexus3000/sw/unicast/7x/b_Cisco_Nexus_3000_Series_NX-OS_Unicast_Routing_Configuration_Guide_Release_7_x/b_Cisco_Nexus_3000_Series_NX-OS_Unicast_Routing_Configuration_Guide_Release_7_x_chapter_01101.html)

Требуется перезагрузка, о чем будет написано после каждой команды с напоминанием сохранить конфигурацию.
```text
system urpf disable
hardware profile unicast enable-host-ecmp
```
Было в `show hardware profile status`
```text
Total LPM Entries = 8191.
Used Host Entries in LPM (Total) = 0.
Used Host Entries in Host (Total) = 139.
```
Стало
```text
Total LPM Entries = 16383.
Used Host Entries in LPM (Total) = 116.
Used Host Entries in Host (Total) = 0.
```

## mtu

```text
policy-map type network-qos jumbo-nq
  class type network-qos class-default
    mtu 9216
system qos
  service-policy type network-qos jumbo-nq
```

Проверить
```text
switch# sh queuing interface eth1/2 | in MTU
HW MTU of Ethernet1/2 : 9216 bytes
```
По sh int будет все равно 1500 и по cdp будут видеть 1500

## сброс в дефолт

```text
write erase
```

## bfd

```text
feature bfd
interface Ethernet1/1
  no switchport
  bfd interval 250 min_rx 250 multiplier 3
  no ip redirects
  ip address 10.0.0.1/30
```
bfd сессия устанавливается когда будет соотв. настройка в протоколе маршрутизации

## vlan

Для создания интерфейсов необходима фича `feature interface-vlan`

## bgp

```text
router bgp 65000
 router-id 10.10.10.10
  log-neighbor-changes
  address-family ipv4 unicast
    network 10.0.0.0/8
  neighbor 10.0.0.2 remote-as 65001
    bfd
    address-family ipv4 unicast
      route-map test-map out
	  next-hop-self
      soft-reconfiguration inbound
```
фильтрация применяется сразу по факту применения route-map
