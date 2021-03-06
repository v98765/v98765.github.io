[Документация на 6.2х Cisco MDS 9000 Family NX-OS Fabric Configuration Guide](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/mds9000/sw/6_2/configuration/guides/fabric/nx-os/nx_os_fabric.html)

## обновление nxos

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
Поэтому необходимо при подключении нового коммутатора вывести `show zoneset` или `show zoneset brief` т.к порядок тоже важен и добиться идентичного вывода вплоть до порядка строк pwwn на вновь подключаемом коммутаторе.
В этом поможет `vimdiff`, например. При подключении коммутатора с различными zoneset, когда merge зон невозможен,
происходит блокировка линка со статусом `isolated` и соотвествующими сообщениями в логах.

## netapp

> Physical WWPNs (beginning with 50:0a:09:8x) do not present a SCSI target
> service and should not be included in any zone configurations on the FC fabric, though they show as logged in to the fabric.
> Instead, use only virtual WWPNs (WWPNs  starting  with 20:)

## single initiator and single targets

Цитата с [cisco](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/mds9000/sw/7_3/configuration/config_limits/cisco_mds9000_config_limits_7x.html):

> The preferred number of members per zone is 2, and the maximum recommended limit is 50.

Т.к. зон будет много, то настаивать необходимо с применением средств автоматизации.
Есть модуль [zone_zoneset](https://github.com/ansible-collections/cisco.nxos/blob/main/docs/cisco.nxos.nxos_zone_zoneset_module.rst) в коллекции cisco.nxos. В примере описан task, который является частью playbook'а.
Скрипт для геренации playbook'а [cisco-1i-1t-zoning](https://github.com/v98765/cisco-1i-1t-zoning) для этого модуля.
Модуль не идемпотентный, т.е. изменения будут применяться каждый раз и каждый раз активироваться zoneset.
В частности поэтому ниже в примере не будет handler'а, записывающего конфигурацию на коммутаторе фабрики.
Как установить ansible и коллекцию описано в разделе [work/venv](https://v98765.github.io/work/venv/).

В рабочем каталоге создать файл `ansible.cfg`.

```text
[defaults]
host_key_checking = False
inventory=./inventory
[ssh_connection]
pipelining = true
```
Создать файл для описания коммутаторов фабрик `inventory`.

```text
[all:vars]

ansible_network_os=nxos
ansible_become=no
ansible_connection=network_cli
ansible_python_interpreter=~/base/bin/python

[fabric_a]
mdsA ansible_host=10.0.0.1
[fabric_b]
mdsB ansible_host=10.0.0.2
```
Проверить доступность устройств по ssh с хоста. `-k` для ввода пароля. `-vv` для отладки. `all` - все хосты в inventory.
```sh
ansible all -m ping -u [username] -k -vv
```
Создать файл `fabric-A.yml`. В `[]` указано что нужно прописать в этом месте. Посмотреть еще раз пример task'а в описании модуля в ссылке выше.
```text
---
- name: configure zoneset on A
  hosts: fabric_a
  gather_facts: false

  tasks:
  - name: task configure zoneset on A
    cisco.nxos.nxos_zone_zoneset:
      zone_zoneset_details:
      - mode: enhanced
        vsan: 1
        zone:
        - members:
          - pwwn: [netapp pwwn]
          - pwwn: [esxi pwwn]
          name: [zone name]
        zoneset:
        - action: activate
          members:
          - name: [zone name]
          name: [zoneset name]
```
Проверить синтаксис
```sh
ansible-playbook fabric-A.yml --syntax-check
```
Проверить применение на коммутаторе фабрики
```sh
ansible-playbook fabric-A.yml -u [username] -k --check
```
Запустить без `--check`

Для сохранения кофигурации отдельный playbook `mds-save.yml`
```text
---
- name: save configuration
  hosts: all
  gather_facts: false

  tasks:
    - name: copy running-config startup-config
      cli_command:
        command: copy running-config startup-config
```
Запуск
```sh
ansible-playbook mds-save.yml -u [username] -k
```

## single initiator and multiple targets

Для случая, когда зоны настраиваются по принципу single initiator and multiple targets.

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
Когда номер vsan отличается, то помимо указания его номера в зонах, необходимо включить соотв порт в нужный vsan.
```text
vsan database
 vsan 2 interface fc1/33
```


## Полезные команды

Что подключено
```text
show flogi database
```
Список инициаторов и таргетов. Если нужны только локальные то с `local`. Обязательно к просмотру, если номера vsan отличные от 1.
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

## smart zoning

Решешие от cisco [smart zoning](https://www.cisco.com/c/en/us/support/docs/storage-networking/zoning/116390-technote-smartzoning-00.html) подразумевает одну зону, но с функионалом Single Initiator and Single Targets.

Миграция возможна при отключении [interoperability](https://www.cisco.com/en/US/docs/storage/san_switches/mds9000/interoperability/guide/ICG_test.html) на коммутаторах фабрики.
Коммутаторы Brocade должны быть отключены, либо переведены в режим access gateway.
```text
vsan database
  no vsan 1 interop
```
Если отключено, то будет по умолчанию
```
MDS# show vsan 1
vsan 1 information
         name:VSAN0001  state:active
         interoperability mode:default
         loadbalancing:src-id/dst-id/oxid
         operational state:up
```

Порядок внесения pwwn имеет значение, поэтому новые таргеты и инициаторы необходимо добавлять в конец. В таске используется модуль
[zone_zoneset](https://github.com/ansible-collections/cisco.nxos/blob/main/docs/cisco.nxos.nxos_zone_zoneset_module.rst)
```yaml
---
- name: configure smart-zoning
  hosts:
    - mdsA
  gather_facts: false

  tasks:

  - name: task configure zoneset on fabric_a
    cisco.nxos.nxos_zone_zoneset:
      zone_zoneset_details:
        - mode: basic
          smart_zoning: true
          vsan: 1
          zone:
          - members:
            - devtype: initiator
              pwwn: 21:00:00:24:ff:50:82:79
            - devtype: initiator
              pwwn: 21:00:00:24:ff:50:81:b1
            - devtype: initiator
              pwwn: 21:00:00:24:ff:32:cf:e8
            - devtype: initiator
              pwwn: 21:00:00:24:ff:49:b9:6e
            - devtype: target
              pwwn: 29:00:74:3a:65:ea:50:8f
            - devtype: target
              pwwn: 21:00:74:3a:65:ea:50:8f
            name: smartzoneA
          zoneset:
          - action: activate
            members:
            - name: smartzoneA
            name: smartsetA
```
Проверка статуса. Обратить внимание на Interop в т.ч.
```
MDS# show zone status
VSAN: 1 default-zone: permit distribute: active only Interop: default
    mode: basic merge-control: allow
    session: none
    hard-zoning: enabled broadcast: unsupported
    smart-zoning: enabled
    rscn-format: fabric-address
    activation overwrite control: disabled
Default zone:
    qos: none broadcast: unsupported ronly: unsupported
Full Zoning Database :
    DB size: 106104 bytes
    Zonesets:2  Zones:1066 Aliases: 78
Active Zoning Database :
    DB size: 1076 bytes
    Name: smartsetA  Zonesets:1  Zones:2
Current Total Zone DB Usage: 107180 / 2097152 bytes (5 % used)
Pending (Session) DB size:
    Full DB Copy size: n/a
    Active DB Copy size: n/a
SFC size: 107180 / 2097152 bytes (5 % used)
Status: Activation completed at 23:36:57 MSK May 24 2021
```
Просто и понятно стало
```
MDS# show zoneset active
zoneset name smartsetA vsan 1
  zone name smartzoneA vsan 1
  * fcid 0x101b00 [pwwn 21:00:00:24:ff:50:82:79]  init
  * fcid 0x100e00 [pwwn 21:00:00:24:ff:50:81:b1]  init
  * fcid 0x100c00 [pwwn 21:00:00:24:ff:32:cf:e8]  init
  * fcid 0x100d00 [pwwn 21:00:00:24:ff:49:b9:6e]  init
  * fcid 0x691500 [pwwn 29:00:74:3a:65:ea:50:8f]  target
  * fcid 0x691800 [pwwn 21:00:74:3a:65:ea:50:8f]  target
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

## snmp monitoring

[Appendix D: Cisco MDS 9000 Family Slow-Drain–Specific SNMP MIBs](https://www.cisco.com/c/dam/en/us/products/collateral/storage-networking/mds-9700-series-multilayer-directors/whitepaper-c11-737315.pdf)

SNMP Object | Description
---|---
fcIfTxWaitCount (1.3.6.1.4.1.9.9.289.1.2.1.1.15) | TxWait counter
fcHCIfBBCreditTransistionFromZero (1.3.6.1.4.1.9.9.289.1.2.1.1.40) | Tx B2B credit transition to zero. Tx B2B credit transition to zero indicates that the port on the other end of the link is not returning R_RDY.
fcIfBBCreditTransistionToZero (1.3.6.1.4.1.9.9.289.1.2.1.1.41) | Rx B2B credit transition to zero. Rx B2B credit transition to zero indicates that the port is not able to return R_RDY to the device on the other end of the link. This may happen if frames cannot be switched to another port on the same switch fast enough.
fcIfTxWtAvgBBCreditTransitionToZero (1.3.6.1.4.1.9.9.289.1.2.1.1.38) | Credit unavailability at 100 ms
fcIfCreditLoss (1.3.6.1.4.1.9.9.289.1.2.1.1.37) | Credit Loss (recovery)
fcIfTimeOutDiscards (1.3.6.1.4.1.9.9.289.1.2.1.1.35) | Timeout discards
fcIfOutDiscard (1.3.6.1.4.1.9.9.289.1.2.1.1.36) | Total number of frames discarded in egress direction, which includes timeout discards
fcIfLinkResetIns (1.3.6.1.4.1.9.9.289.1.2.1.1.9) | Number of link reset protocol errors received by the FC port from the attached FC port
fcIfLinkResetOuts (1.3.6.1.4.1.9.9.289.1.2.1.1.10) | Number of link reset protocol errors issued by the FC port to the attached FC port.
fcIfSlowportCount (1.3.6.1.4.1.9.9.289.1.2.1.1.44) | Duration for which Tx B2B credits were unavailable on a port
fcIfSlowportOperDelay (1.3.6.1.4.1.9.9.289.1.2.1.1.45) | Number of times for which Tx B2B credits were unavailable on a port for a duration longer than the configured admin-delay value in slowport-monitor


fcHCIfBBCreditTransistionFromZero и fcIfBBCreditTransistionToZero это что-то вроде flow-control
[Check Transitions to Zero counters лист 164](https://www.ciscolive.com/c/dam/r/ciscolive/emea/docs/2017/pdf/BRKSAN-3446.pdf)

В данный порт включена СХД и тут рост `Rx B2B credit transition to zero`. Описание в таблице.
```text
MDS# show interface fc1/1 counters | in transitions
     852 Transmit B2B credit transitions to zero
     308015715 Receive B2B credit transitions to zero
```
Это аплинк. Тут рост `Tx B2B credit transition to zero`. Описание в таблице.
```text
MDS# show interface fc1/48 counters | in transitions
     50532726 Transmit B2B credit transitions to zero
     845047928 Receive B2B credit transitions to zero
```

## port channel

[Configuring Port Channels](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/mds9000/sw/8_x/config/interfaces/cisco_mds9000_interfaces_config_guide_8x/configuring_portchannels.html)

В каждой модели конечное число forwarding engines (Table 4. Ports to Forwarding Engines Mapping) и у каждого свой диапазон портов.
Выбор портов целиком зависит от конкретной модели. Для 9148 выбрал порты 23,35

```text
mds(config)# int fc1/23
mds(config-if)# channel-group 1 force
fc1/23 added to port-channel 1 and disabled
please do the same operation on the switch at the other end of the port-channel,
then do "no shutdown" at both ends to bring it up
mds(config)# int fc1/35
mds(config-if)#channel-group 1 force

mds# sh port-channel database
port-channel1
    Administrative channel mode is on
    Operational channel mode is on
    Last membership update succeeded
    2 ports in total, 0 ports up
    Ports:   fc1/35   [down]
             fc1/23   [down]

mds# sh port-channel summary
------------------------------------------------------------------------------
Interface                 Total Ports        Oper Ports        First Oper Port
------------------------------------------------------------------------------
port-channel 1                 2                 0                  --
```

Практики от циски.
Consequently, most of the best practices followed for the F and TF port channels should be followed to ensure that TCAM
is utilized as efficiently as possible for the following purposes:

* Distribute port-channel member interfaces into different forwarding engines, especially on fabric switches.
* Distribute member interfaces into different linecards on director-class switches.
* Distribute member interfaces into forwarding engines with lower TCAM zoning region usage.
* Use single-initiator zones, single-target zones, or Smart Zoning.

Это про brocade. To avoid congestion ISL Trunking can be used. With ISL Trunking several ISLs are grouped into a logical ISL.
When using Brocade switches, a license has to be installed on all switches, that use ISL Trunking.
An ISL Trunk is automatically formed, when two or more (up to 8) adjacent ports are used to connect two switches.
The adjacent ports must belong to the same port group. That’s the cause why you can’t add more than 8 ports to an ISL Trunk.
Instead of using an exchange-based distribution, a frame-based distribution is used for ISL Trunks.
This method is more finer and allows a better distribution. The Ports, that belongs to an ISL Trunk, are known as trunking members.
One port of the trunking members is the trunking master. This port has a special function, because it assigns traffic to the other trunking members.
Even if an ISL Trunk is an logical ISL, it preserves “in-order delivery” of frames.

## FSPF

FSPF: “Fabric shortest path first” is a routing protocol used in Fibre Channel fabrics.
It’s used to establish routes accross the fabric and to re-calculate this routes, if a topology change occurs (e.g. link failures).

ISL: An ISL is an inter-switch link. It’s a connection between two Fibre Channel switches.

Как минимум можно посмотреть номер удаленного порта. Ниже cost is 125 для 8Gb порта, а cost is 62 для port-channel из двух портов 8Gb

```text
mds# show fspf interface
FSPF interface fc1/48 in VSAN 1
FSPF routing administrative state is active
Interface cost is 125
Timer intervals configured, Hello 20 s, Dead 80 s, Retransmit 5 s
FSPF State is FULL
Neighbor Domain Id is 0x01(1)
Neighbor Interface is Port 23  (0x00000017 )

Statistics counters :
   Number of packets received : LSU  10385  LSA  3464  Hello 311486  Error packets 0
   Number of packets transmitted : LSU  3464  LSA  10385  Hello 311492  Retransmitted LSU  0
  Number of times inactivity timer expired for the interface = 0

FSPF interface port-channel1 in VSAN 1
FSPF routing administrative state is active
Interface cost is 62
Timer intervals configured, Hello 20 s, Dead 80 s, Retransmit 5 s
FSPF State is FULL
Neighbor Domain Id is 0x69(105)
Neighbor Interface is port-channel1 (0x00040000 )

Statistics counters :
   Number of packets received : LSU  3  LSA  3  Hello 3  Error packets 0
   Number of packets transmitted : LSU  3  LSA  3  Hello 3  Retransmitted LSU  0
  Number of times inactivity timer expired for the interface = 0

```
