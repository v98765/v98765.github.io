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

## обновление до 7.0(3)I7(9)

[Cisco Nexus 3000 Series NX-OS Release Notes, Release 7.0(3)I7(9)](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/nexus3000/sw/release/703i79/70379_3000_3500_rn.html)


MD5
```text
d31d5b556cc4d92f2ff2d83b5df7b943  nxos.7.0.3.I7.9.bin
```

В RelNotes сразу предлагается использовать compact image. Его можно сделать на нексусе
```text
switch# install all nxos usb1:nxos.7.0.3.I7.9.bin compact
Installer will perform compatibility check first. Please wait.
Compacting usb1:/nxos.7.0.3.I7.9.bin
...................................................
Compact usb1:/nxos.7.0.3.I7.9.bin done
switch# move usb1:nxos.7.0.3.I7.9.bin usb1:cnxos.7.0.3.I7.9.bin
switch# copy usb1:cnxos.7.0.3.I7.9.bin bootflash:
Copy progress 100% 473665KB                                                     
Copy complete, now saving to disk (please wait)...                              
Copy complete.
```

Резервная копия и сброс
```text
copy running-config bootflash:bck.cfg
Copy complete, now saving to disk (please wait)...                              
Copy complete.                                                                  
S0017_fabA# write erase                                                         
Warning: This command will erase the startup-configuration.                     
Do you wish to proceed anyway? (y/n)  [n] y                                     
switch# reload

write erase
```

Change the switching-mode from cut-through to store-and-forward and then reload the switch:
```text
switch#show switching-mode
Configured switching mode: Cut through

Module Number                   Operational Mode
     1                          Cut-Through
switch# configure terminal
switch(config)# switching-mode store-forward
```

Для изменения режима работы портов тоже понадобится сброс и перезагрузка. Вот кусок из документации.

```text
switch# configure terminal
switch(config)# copy running-config bootflash:my-config.cfg
switch(config)# write erase
switch(config)# reload
WARNING: This command will reboot the system
Do you want to continue? (y/n) [n] y
switch(config)# hardware profile portmode 48x10g+4x40g
Warning: This command will take effect only after saving the configuration and reload!
Port configurations could get lost when port mode is changed!
switch(config)# copy running-config startup-config
switch(config)# reload
WARNING: This command will reboot the system
Do you want to continue? (y/n) [n] y
```

The following limitations are applicable when you upgrade from Releases 7.0(3)I7(2) or later to the NX-OS Release 7.0(3)I7(9):
You must run the setup script after you upgrade to Cisco NX-OS Release 7.0(3)I7(9).

Как видно из строк выше, после удаления конфигурации все равно надо запустить setup скрипт.
Он создаст какую-то конфигурацию и в итоге запишет ее в startup.
Если этого не сделать и выбрать skip, то из-за "фичи" записать конфигурацию после
`hardware profile portmode 48x10g+4x40g` не получится

```text
Abort Power On Auto Provisioning [yes - continue with normal setup, skip - bypad
yes
Disabling POAP.......Disabling POAP



         ---- System Admin Account Setup ----


Do you want to enforce secure password standard (yes/no): no

  Enter the password for "admin":
  Confirm the password for "admin":

         ---- Basic System Configuration Dialog ----

This setup utility will guide you through the basic configuration of
the system. Setup configures only enough connectivity for management
of the system.

Please register Cisco Nexus 3000 Family devices promptly with your
supplier. Failure to register may affect response times for initial
service calls. Nexus devices must be registered to receive entitled
support services.

Press Enter at anytime to skip a dialog. Use ctrl-c at anytime
to skip the remaining dialogs.

Would you like to enter the basic configuration dialog (yes/no):

....

  Enable the telnet service? (yes/no) [n]:

  Enable the ssh service? (yes/no) [y]:

    Type of ssh key you would like to generate (dsa/rsa) :

  Configure the ntp server? (yes/no) [n]:

  Configure default interface layer (L3/L2) [L2]:

  Configure default switchport interface state (shut/noshut) [noshut]: shut

  Configure CoPP System Policy Profile ( default / l2 / l3 ) [default]:

The following configuration will be applied:
  switchname switch
  no telnet server enable
  system default switchport
  system default switchport shutdown
  policy-map type control-plane copp-system-policy ( default )

Would you like to edit the configuration? (yes/no) [n]:
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
Либо сбросить в дефолт в этом же режиме, если софт старый
```text
write erase
reload
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
