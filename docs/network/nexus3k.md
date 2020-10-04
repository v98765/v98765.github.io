Далее по cisco nexus 3000 модели

## Обновление ПО

Ссылки:

* [Рекомендованная версия ПО](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/nexus3000/sw/recommended_release/b_Minimum_and_Recommended_Cisco_NX-OS_Releases_for_Cisco_Nexus_3000_Series_Switches.html). Это 7.0(3)I7(7) сейчас.
* [Cisco Nexus 3000 Series NX-OS Software Upgrade and Downgrade Guide, Release 7.x](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/nexus3000/sw/upgrade/7_x/b_Cisco_Nexus_3000_Series_NX_OS_Software_Upgrade_and_Downgrade_Release_7_x/b_Cisco_Nexus_3000_Series_NX_OS_Software_Upgrade_and_Downgrade_Release_7_x_newGuide_chapter_01.html)

> If you have a Cisco Nexus 3000 release prior to Release 7.0(3)I2(1), upgrade to Cisco Nexus 3000 Release 6.0.2.U6(3) first. 

Скопировать, установить 6.0.2.U6(3), когда софт на коммутаторе слишком старый
```text
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

