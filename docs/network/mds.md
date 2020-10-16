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
В 5 версии нет scp-server но можно скопировать с любого хоста, подключившись к ноуту через mgmt интерфейс
```text
copy scp://username@192.168.1.181/home/v98765/soft/m9100-s3ek9-kickstart-mz.6.2.29.bin bootflash:/
copy scp://username@192.168.1.181/home/v98765/soft/m9100-s3ek9-mz.6.2.29.bin bootflash:/
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

## настройка портов

В общем случае не нужно, но порты с fc картами настраиваются как `F`, а линк до коммутатора san - `E`.
Если подключается СХД с несколькими pwwn, но на коммутаторе необходимо включить `feature npiv`.
```text
interface fc1/1
  switchport mode F
  switchport description "[hostname]"
  switchport trunk mode off

interface fc1/48
  switchport description "[another fc switch]"
  switchport mode E
```

## zone, zoneset

Зоны на коммутаторах могут быть различными, но включенные в zoneset зоны должны быть идентичны на всей сети.
Поэтому необходимо при подклчючении нового коммутатора вывести `show zoneset` и добиться идентичного вывода вплоть до порядка строк pwwn на вновь подключаемом коммутаторе.
В этом поможет `vimdiff`, например. При подключении коммутатора с различными zoneset, когда merge зон невозможен,
происходит блокировка линка со статусом `isolated` и соотвествующими сообщениями в логах.

## подключение нового устройства

По разным советам необходимо для каждого (одного) "инициатора" создавать отдельную зону, где будет только он и доступные для него схд "таргеты".

Посмотреть данные по устройству на портах
```text
show flogi database
```
Создать fcalias для инициатора.
```text
fcalias name [init] vsan 1
 member pwwn [pwwn]
```
Если это дополнительное хранилище, то добавить все его pwwn в существующий алиас таргетов.
```text
fcalias name [targets] vsan 1
 member pwwn [pwwn]
```
Создать новую зону
```text
zone name [init_and_targets] vsan 1
 member fcalias [init]
 member fcalias [targets]
```
Добавить зону в zoneset
```text
zoneset name [zonesetname] vsan 1
 member [init_and_targets]
```
Активировать изменения в режиме конфигурирования. Кстати, `show` тоже работает и необязательно переключаться из режима конфигурирования.
```text
MDS(config)# zoneset activate name [zonesetname] vsan 1
```

## Полезные команды

Что подключено
```text
show flogi database
```
Список инициаторов и таргетов. Если нужны только локальные то с `local`
```text
show fcns database
```
Прописан ли pwwn. Пусто, если не прописан.
```text
show zone member pwwn [pwwn]
```
Где подключено устройство
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

## проблемы

Смотреть интерфейсы кратко
```text
show int bri
```
Смотреть состояние порта на предмет ошибок (errors), когда скорее всего надо менять трансивер. Из-за битовых ошибок порт может отключиться и в статусе будет `errdisabled`
```text
show int fc1/1
```
Смотреть какой трансивер установлен. Т.к. они перешиваются, то не факт что это полезно.
```text
show port internal info interface fc1/1
```
Смотреть сообщения о нехватке rxbbcredits и других ошибках. Эта и другие команды в [MDS 9148 Slow Drain Counters and Commands](https://www.cisco.com/c/en/us/support/docs/storage-networking/mds-9100-series-multilayer-fabric-switches/116401-trouble-mds9148-00.html)
```text
show logging onboard error-stats
```
В частности нехватка буферов это FCP_CNTR_RX_WT_AVG_B2B_ZERO

> This counter increments when the remaining Rx B2B value is at zero for 100ms or more. This typically indicates the switch is withholding R_RDYs (B2B credits) from the attached device due to upstream congestion (congestion away from this port).
