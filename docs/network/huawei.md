## vlan

Создать
```text
vlan 2
vlan batch 3 4
```

## port mode

access
```text
interface GigabitEthernet0/0/1
 port link-type access
 port default vlan 2
```

trunk
```text
interface GigabitEthernet0/0/2
 port link-type trunk
 port trunk allow-pass vlan 2 3 4
```
Добавлять такой же командой, а удалять через `undo port trunk allow-pass vlan [vid]`

routed
```text
interface GigabitEthernet0/0/23
 undo portswitch
 ip address 10.9.8.7 255.255.255.254
```

## selective qinq

Не нужен dot1q-tunnel, т.к. не надо все паковать. Влан 100 передается как есть, а во влан 99 пакуется 10 20 30. Паковать 99 в 99 нельзя.
```text
interface GigabitEthernet0/0/1
 port link-type hybrid
 qinq vlan-translation enable
 port hybrid tagged vlan 100
 port hybrid untagged vlan 99
 port vlan-stacking vlan 10 stack-vlan 99
 port vlan-stacking vlan 20 stack-vlan 99
 port vlan-stacking vlan 30 stack-vlan 99
```

## qos

Все в порядке с очередями, но на интефейсах надо `trust dscp`, а на routed дополнительно еще `undo qos phb marking enable`

## dhcp snooping, relay

```text
dhcp enable
dhcp snooping enable

vlan 110 configuration
 description LAN
 dhcp snooping enable

dhcp server group 0
 dhcp-server [dhcp-server] 0

interface Vlanif110
 ip address 10.0.0.254 255.255.255.0
 dhcp select relay
 dhcp relay server-select 0
```

По умолчанию интерфейсы недоверенные, поэтому `dhcp snooping trusted` там где нужно.

## Фильтр pvst на коммутаторах huawei

Фреймы BPDU pvst протокола прозрачно передаются коммутаторами, на которых не настроен pvst.
Поэтому, когда в сети настроен протокол mstp, необходимо pvst фильтровать на коммутаторах отличных от cisco.
Например, есть схема, где root msti 0 и mst 1 - это cisco.
К cisco подключен коммутатор huawei и на нем настроен mstp как и на cisco.
Подключается новый коммутатор cisco к коммутатору huawei, но его дефолтовые настройки stp не менялись.
Т.о. через вновь созданный транковый порт в нативном влане передается pvst bpdu через huawei далее на root'а.
Root (Cisco) переводит порт из режима P2p в режим совместимости с pvst, что выглядит как P2p Bound(PVST).
Root остается прежним для msti 0 (cist), но не для msti 1. В итоге huawei в msti 1 остался без root'а.
При кольцевой топологии такие изменения могут иметь печальные последствия, поэтому pvst надо фильтровать.
Взято с [https://forum.huawei.com/enterprise/en/how-to-filter-pvst-bpdus/thread/394511-861](https://forum.huawei.com/enterprise/en/how-to-filter-pvst-bpdus/thread/394511-861).
```text
acl number 4000
 rule 10 permit destination-mac 0100-0ccc-cccd

traffic classifier c1 operator or
 if-match acl 4000

traffic policy p1 match-order config
 classifier c1 behavior b1

traffic-policy p1 global inbound
```

## Подключение телефонов cisco 7940

Телефоны без прошивки с lldp и с poe несовместимы, не включаются. Для настройки voice vlan есть костыли для совместимости с cdp. Команды `voice-vlan legacy enable` и `poe legacy enable`
```text
interface GigabitEthernet0/0/11
 port link-type hybrid
 voice-vlan 110 enable
 voice-vlan legacy enable
 port hybrid pvid vlan 520
 port hybrid tagged vlan 110
 port hybrid untagged vlan 520
 poe legacy enable
 lldp compliance cdp txrx
 lldp compliance cdp receive

>display cdp neighbor brief
Local Intf    Neighbor Dev             Neighbor Intf             Exptime(s)
GE0/0/1      SEP081FF363712B          Port 1                    134
```

## sflow

Для проверки tos, topN и тп
```text
sflow collector 1 ip [collector]
sflow agent ip [switch]

interface GigabitEthernet0/0/1
 sflow counter-sampling collector 1
 sflow flow-sampling collector 1
```
Смотреть
```text
>disp sflow statistics
sFlow Version 5 statistic Information:
--------------------------------------------------------------------------
Collector 1 Current sample sequence:
--------------------------------------------------------------------------
Port on slot 0 statistic Information:

Interface: GE0/0/1
```

## nqa

Мониторить доступность с коммутатора, а результат в snmp. Можно указать кол-во пингов и % потерь
```text
nqa test-instance 1 1
 test-type icmp
 destination-address ipv4 [dst-ip]
 source-address ipv4 [src-ip]
 records result 1
 records history 0
 threshold rtd 100
 description [some text]
 tos 96
 fail-percent 20
 frequency 60
 probe-count 10
 start now
```

## snmp

По умолчанию нельзя задать public, т.к. слишком просто
```text
snmp-agent
snmp-agent acl 2222
snmp-agent community complexity-check disable
snmp-agent community read public
snmp-agent sys-info contact [contact]
snmp-agent sys-info location [location]
snmp-agent sys-info version v2c v3
```
acl для ограничения доступа
```text
acl number 2222
 rule 10 permit source [nms-ip] 0
```

## traffic policy

Для статистики в тч, а счетчики доступны по snmp. Нужен bps по конкретной подсети прием/передача. Сначала описать acl

```text
acl number 3100
 rule 10 permit ip destination 10.0.0.0 0.0.0.255
acl number 3500
 rule 10 permit ip source 10.0.0.0 0.0.0.255
```

Классы
```
traffic classifier 3100 operator or
 if-match acl 3100
traffic classifier 3500 operator or
 if-match acl 3500

traffic behavior tbstat
 statistic enable
 permit
```

Политики, где может быть более одного класса.
```
traffic policy 31 match-order config
 classifier 3100 behavior tbstat
traffic policy 35 match-order config
 classifier 3500 behavior tbstat
```

Применить к интерфейсу
```text
interface GigabitEthernet0/0/1
 undo portswitch
 description TTK_vlan2943_p1
 ip address 10.1.46.49 255.255.255.254
 traffic-policy 31 inbound
 traffic-policy 35 outbound
```

Смотреть что применил
```text
>disp traffic-policy applied-record
#
-------------------------------------------------
  Policy Name:   31
  Policy Index:  2
     Classifier:3100     Behavior:tbstat
-------------------------------------------------
 *interface GigabitEthernet0/0/1
    traffic-policy 31 inbound
      slot 0    :  success
-------------------------------------------------
  Policy total applied times: 1.
#
-------------------------------------------------
  Policy Name:   35
  Policy Index:  3
     Classifier:3500     Behavior:tbstat
-------------------------------------------------
 *interface GigabitEthernet0/0/1
    traffic-policy 35 outbound
      slot 0    :  success
-------------------------------------------------
  Policy total applied times: 1.
```

Смотреть счетчики в классах
```
>disp traffic policy statistics interface g0/0/1 inbound verbose classifier-base class 3100

 Interface: GigabitEthernet0/0/1
 Traffic policy inbound: 31
 Rule number: 6
 Current status: success
 Statistics interval: 150
---------------------------------------------------------------------
 Classifier: 3100 operator or
 Behavior: tbstat
 Board : 0
---------------------------------------------------------------------
 Matched          |      Packets:                 8,426,225,438
                  |      Bytes:               4,237,940,385,174
                  |      Rate(pps):                         853
                  |      Rate(bps):                   3,305,096
---------------------------------------------------------------------
   Passed         |      Packets:                 8,426,225,438
                  |      Bytes:               4,237,940,385,174
                  |      Rate(pps):                         853
                  |      Rate(bps):                   3,305,096
---------------------------------------------------------------------
   Dropped        |      Packets:                             0
                  |      Bytes:                               0
                  |      Rate(pps):                           0
                  |      Rate(bps):                           0
---------------------------------------------------------------------
     Filter       |      Packets:                             0
                  |      Bytes:                               0
---------------------------------------------------------------------
     Car          |      Packets:                             0
                  |      Bytes:                               0
---------------------------------------------------------------------
```

## rate limit

На гигабитном линке ограничить полосу 100мбит, а для дефолтовой очереди сделать меньше

```text
interface GigabitEthernet0/0/1
 qos lr outbound cir 100000 cbs 12500000
 qos queue 0 shaping cir 60000 pir 80000
```
