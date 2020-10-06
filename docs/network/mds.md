MDS 9000 и 9148 в частности

[Cisco MDS 9000 Series Release Notes for Cisco MDS NX-OS Release 6.2(29)](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/mds9000/sw/6_2/release/notes/nx-os/mds_nxos_rn_6_2_29.html)
Рекомендуемый релиз из того что есть 

file | md5
---|---
[m9100-s3ek9-mz.6.2.29.bin](https://software.cisco.com/download/home/282867260/type/282088129/release/6.2(29)) | `md5 60891b50b773222ecfef778f263f0af3` 
[m9100-s3ek9-kickstart-mz.6.2.29.bin](https://software.cisco.com/download/home/282867260/type/282088130/release/6.2(29)) | `md5 227276a329ffca6cbabd5264df40762d`

[Cisco MDS 9000 NX-OS Software Upgrade and Downgrade Guide](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/mds9000/sw/6_2/upgrade/guides/nx-os/upgrade.html)

If your switch is running software that is earlier than Cisco NX-OS Release 5.2(x), you must upgrade to Release 6.2(x). Follow this upgrade path:

* Release 5.0(x): upgrade to 5.2(x), and then upgrade to 6.2(x).
* Release 4.1(x) or release 4.2(x): upgrade to Release 5.0(x), upgrade to Release 5.2(x) and then upgrade to Release 6.2(x).
* Release 3.3(2), Release 3.3(3), Release 3.3(4x), and Release 3.3(5x), upgrade to release 4.1(x) or Release 4.2(x), then upgrade to Release 5.0(x), and then upgrade to Release 5.2(x), and then upgrade to 6.2(x).
* Release 3.3(1c), all Release 3.2(x), all Release 3.1(x), and all Release 3.0(x), upgrade to release 3.3(5b), then upgrade to release 4.1(x) or release 4.2(x), then upgrade to Release 5.0(x), and then upgrade to Release 5.2(x), and then upgrade to 6.2(x).

[Документация на 6.2х Cisco MDS 9000 Family NX-OS Fabric Configuration Guide]
(https://www.cisco.com/c/en/us/td/docs/switches/datacenter/mds9000/sw/6_2/configuration/guides/fabric/nx-os/nx_os_fabric.html)

Включить scp-server на mds
```text
feature scp-server
```

Скопировать файлы по scp
```text
C:\>pscp.exe -scp m9100-s3ek9-kickstart-mz.6.2.29.bin user@mds_ip:/
Keyboard-interactive authentication prompts from server:
| Password:
End of keyboard-interactive prompts from server
m9100-s3ek9-kickstart-mz. | 21361 kB | 5340.3 kB/s | ETA: 00:00:00 | 100%

C:\>pscp.exe -scp m9100-s3ek9-mz.6.2.29.bin user@mds_ip:/
Keyboard-interactive authentication prompts from server:
| Password:
End of keyboard-interactive prompts from server
m9100-s3ek9-mz.6.2.29.bin | 73713 kB | 3204.9 kB/s | ETA: 00:00:00 | 100%
```

Проверить md5 и должно совпасть.
```text
mds# show file m9100-s3ek9-kickstart-mz.6.2.29.bin md5sum
227276a329ffca6cbabd5264df40762d
mds# show file m9100-s3ek9-mz.6.2.29.bin md5sum
60891b50b773222ecfef778f263f0af3
```

Предварительная проверка перед установкой
```text
mds# show install all impact kickstart m9100-s3ek9-kickstart-mz.6.2.29.bin system 9100-s3ek9-mz.6.2.29.bin
```

Установка
```text
mds# install all kickstart  m9100-s3ek9-kickstart-mz.6.2.29.bin system m9100-s3ek9-mz.6.2.29.bin
```

Полезные команды:

flogi - fabric login
```text
MDS# show flogi database interface fc1/22
--------------------------------------------------------------------------------
INTERFACE        VSAN    FCID           PORT NAME               NODE NAME
--------------------------------------------------------------------------------
fc1/22           1     0x693c00  21:00:00:24:ff:33:55:ee 20:00:00:24:ff:33:55:ee

Total number of flogi = 1.
```
FCNS (Fibre Channel Name Server). Смотреть на любом коммутаторе сети. Покажет свич и порт с нужным fcid.
На другом коммутаторе если смотреть, то будет type/feature `scsi-fcp`
```text
MDS# show fcns database fcid 0x693c00 detail vsan 1
------------------------
VSAN:1     FCID:0x693c00
------------------------
port-wwn (vendor)           :21:00:00:24:ff:33:55:ee
node-wwn                    :20:00:00:24:ff:33:55:ee
class                       :3
node-ip-addr                :0.0.0.0
ipa                         :ff ff ff ff ff ff ff ff
fc4-types:fc4_features      :scsi-fcp:init
symbolic-port-name          :
symbolic-node-name          :QLE2562 FW:v5 DVR:v1
port-type                   :N
port-ip-addr                :0.0.0.0
fabric-port-wwn             :20:16:00:05:73:ee:11:55
hard-addr                   :0x000000
permanent-port-wwn (vendor) :21:00:00:24:ff:33:55:ee
connected interface         :fc1/22
switch name (IP address)    :MDS (10.9.8.7)

Total number of entries = 1
```
В каких зонах есть или нет. Если пусто, то нет настроек.
```text
MDS# show zone member pwwn 21:00:00:24:ff:33:55:ee
pwwn 21:00:00:24:ff:33:55:ee vsan 1
  fcalias aliasid
      zone zonename
```
Алиас
```text
MDS# sh fcalias name [aliasid]
fcalias name aliasid vsan 1
  pwwn 21:00:00:24:ff:33:55:ee
```
Зона
```text
MDS# show zone name [zonename] vsan 1
* fcid 0x693c00 [pwwn 21:00:00:24:ff:33:55:ee]
```
Применить изменения в зонах
```text
MDS(config)# zoneset activate name [zonesetname] vsan 1
```
В домене
```text
MDS# show fcdomain fcid persistent | in 0x693c00
   1 21:00:00:24:ff:33:55:ee 0x693c00 SINGLE  YES  DYNAMIC    --
```
