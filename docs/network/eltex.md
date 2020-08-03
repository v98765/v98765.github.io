# Настройка eltex 23xx

в 2018-2020 было дело

## firmware update
Для моделей 23хх и 33хх сейчас выпускается одна версия ПО, которая одинаково просто устанавливается на коммутаторы. Для стека всё то же.
```text
boot system tftp://10.9.8.7/mes3300-4xx.ros
```
Посмотреть текущую версию `sh bootvar`. Новая версия загрузится только после reload. Отключение питания приведет к загрузке старой версии, а не новой.
Не нужно рассчитывать на отключение питания, как на часть процедуры обновления.
При подключении коммутатора на удаленном объекте через спутниковый канал не представляется возможным обновить его через tftp или веб-интерфейс.
Наиболее надежным способом оказалось копирование прошивки на коммутатор 2300-серии по ssh
```text
boot system scp://user:password@10.9.8.7/mes3300-4013-1R2.ros
```
Копирование занимает около 4 часов для rtt ~ 600ms. Коммутатор поддерживает dual boot и вновь загруженный софт будет активным только если перезагрузка выполнена командой.
В этом отношении можно не опасаться случайного отключения питания на объекте и обрыва сессии при загрузке ПО.
Для сравнения ниже размер ПО в байтах для eltex и для cisco 3850
```text
25168175 mes3300-4013-1R2.ros
404622964 cat3k_caa-universalk9.16.06.05.SPA.bin
```

## negotiation
История про duplex mismatch на оптике. Например, ошибки на линке, т.к. полудуплекс на коммутаторе cisco 3560 с установленным 100мбит/с трансивером
```text
cisco3560#sh int GigabitEthernet0/1
GigabitEthernet0/1 is up, line protocol is up (connected) 
  Hardware is Gigabit Ethernet, address is 
  Half-duplex, 100Mb/s, link type is auto, media type is 100BaseBX-10D SFP
С другой стороны стоит Eltex, у которого тоже ошибки на порту.
974395770 packets input, 234551040866 bytes received
      1248389 broadcasts, 0 multicasts
      6734584 input errors, 4459185 FCS, 0 alignment
```
Статус интерфейса eltex'а
```text
gigabitethernet 2/0/20 is up (connected)
  Interface index is 176
  Hardware is gigabitethernet, MAC address is
  Interface MTU is 10218
  Full-duplex, 100Mbps, link type is 100Mbps none-duplex, media type is 100M-Fiber
```
История про автосогласование, когда SFP включен в SFP+ порт в обоих коммутаторах Eltex. Уровень по приему диагностика показывает, линка нет, порт флапает.
Ситуация может произойти после перезагрузки. Говорят поправили. Как исправлял:
```text
interface tengigabitethernet1/0/1
 speed 1000
 no negotiation bypass
```
Что-то похожее было на линке с WDM SFP Dlink'а в des-3452 в SFP+ порту и с WDM SFP в des-3600 в SFP+ порту.
После перезагрузки 3452 линк пропал, настроек нет никаких на порту.

Выставить на гигабитном порту автосогласование скорости для 10мбит максимум.
```text
interface gigabitethernet1/0/1
 negotiation 10f
```
Смотреть
```text
eltex#sh int gi1/0/1 | in link
  Full-duplex, 10Mbps, link type is auto, media type is 1G-Copper
  Advertised link modes: 10baseT/Full  
```


## trunk, access, qinq

trunk, все запрещено
```text
interface tengigabitethernet1/0/2
 switchport mode trunk
 switchport forbidden default-vlan
```
trunk, разрешены указанные vlan и нативный
```text
interface tengigabitethernet1/0/2
 switchport mode trunk
 switchport trunk allowed vlan add 10-20,110-120,200
```
general, где 120 - без тега, 110 с тегом. Для voice vlan.
```text
interface gigabitethernet1/0/1
 switchport mode general
 switchport general allowed vlan add 110 tagged
 switchport general allowed vlan add 120 untagged
 switchport general pvid 120
```
access
```text
interface gigabitethernet1/0/1
 switchport access vlan 10
```
qinq нужен mtu, т.к. доп тег это 4 байта.
```text
port jumbo-frame
!
interface gigabitethernet1/0/1
 switchport mode customer
 switchport customer vlan 20
```

## vlan
Создание
`vlan 9 name nine`
Описание окажется в виде дескрипшена
```text
interface vlan 9
 name nine
exit
```
Смотреть
`sh vlan tag 9`, `sh vlan`

## spanning-tree

mstp
```text
spanning-tree mode mst
spanning-tree bpdu filtering
spanning-tree mst configuration
 instance 1 vlan 1-4094
 name abc
 revision 1
exit
spanning-tree bpdu filtering
```
выключить совсем
```text
no spanning-tree
spanning-tree bpdu filtering
```
Если получит pvst bpdu, то переведет порт в режим совместимости с pvst. pvst поддерживает 63 инстанса,
согласно документации и вообще там много достаточно ограничений. Кол-во инстансов влияет на cpu.
Если нет резервных линков с коммутатором cisco, то stp на интерфейсе выключаю.

bpdu filtering не фильтрует pvst bpdu, фильтрует входящие bpdu.

bpdu guard не использую. Вместо этого есть loopback-detection, а на портах доступа stp отключен. portfast не нужен по этой же причине.

guard loop на линках между коммутаторами, на которых настроен mstp.

## qos
В 4-ой ветке ПО и в 2324P, например, по умолчанию коммутатор "доверяет" меткам dscp.
```text
eltex-2324#sh qos 
Qos: Basic mode
Basic trust: dscp
CoS to DSCP mapping: disabled
DSCP to CoS mapping: disabled
```
Довольно редкая ситуация для недорогих коммутаторов, когда помимо рабочего qos можно посмотреть еще и статистику по каждому интерфейсу.
"Старая" реализация статистики очередей тоже работает.
Для ее корректной настройки необходимо посмотреть map'ы, т.к. EF попадает в разные очереди и статистику настраивать надо соответственно.
```text
eltex-3324#sh qos map dscp-queue 
Dscp-queue map:
     d1 : d2 0  1  2  3  4  5  6  7  8  9
     -------------------------------------
      0 :   01 01 01 01 01 01 01 01 01 02
      1 :   02 02 02 02 02 02 06 03 03 03
      2 :   03 03 03 03 06 04 04 04 04 04
      3 :   04 04 07 05 05 05 05 05 05 05
      4 :   06 07 07 07 07 07 07 07 06 06
      5 :   06 06 06 06 06 06 06 06 06 06
      6 :   06 06 06 06
eltex-2324#sh qos map dscp-queue 
Dscp-queue map:
     d1 : d2 0  1  2  3  4  5  6  7  8  9
     -------------------------------------
      0 :   02 01 01 01 01 01 01 01 01 03
      1 :   03 03 03 03 03 03 07 04 04 04
      2 :   04 04 04 04 07 05 05 05 05 05
      3 :   05 05 07 06 06 06 06 06 06 06
      4 :   07 08 08 08 08 08 08 08 07 07
      5 :   07 07 07 07 07 07 07 07 07 07
      6 :   07 07 07 07
```
Настройка strict priority для крайней очереди, куда попадает dscp 46
```text
priority-queue out num-of-queues 1
```
Настройка статистики, счетчики которой, кстати, можно опрашивать по snmp. Для 3324 это будет 7, а для 2324 будет 8.
```text
qos statistics queues 1 8 all all
qos statistics queues 2 1 all all
```
Чтобы отображалась статистика по каждому интерфейсу, включить ее.
```text
qos statistics interface
```
Посмотреть первое и второе
```text
eltex-2324#sh qos statistics 
 
Policers 
-------- 

 Interface   Policy map  Class Map In-profile bytes  Out-of-profile bytes 
----------- ------------ --------- ----------------- -------------------- 

 
Aggregate Policers
------------------ 

       Name         In-profile bytes  Out-of-profile bytes 
------------------- ----------------- -------------------- 

 
Output Queues
------------- 

  Interface      Queue      Dp    Total packets   TD packets   
------------- ------------ ----- --------------- ------------- 
     All           8       Low      56729052           0       
     All           1       Low      387446539          0       

eltex-2324#sh int te1/0/4
tengigabitethernet1/0/4 is up (connected)
  Interface index is 108
  Hardware is tengigabitethernet, MAC address is e0:d9:e3:e7:59:5c
  Interface MTU is 1500
  Full-duplex, 1000Mbps, link type is 10000Mbps full-duplex, media type is 10G-Fiber
  Link is up for 20 days, 23 hours, 29 minutes and 15 seconds
  Flow control is off, MDIX mode is off
  15 second input rate is 5561 Kbit/s
  15 second output rate is 999 Kbit/s
      753544120 packets input, 838942350541 bytes received
      13645818 broadcasts, 16798613 multicasts
      0 input errors, 0 FCS, 0 alignment
      0 oversize, 0 internal MAC
      0 pause frames received
      384858748 packets output, 127715629000 bytes sent
      1226637 broadcasts, 2273150 multicasts
      0 output errors, 0 collisions
      0 excessive collisions, 0 late collisions
      0 pause frames transmitted
      0 symbol errors, 0 carrier, 0 SQE test error
  Output queues: (queue #: packets passed/packets dropped)
      1: 418380/0
      2: 1500378/0                                    
      3: 0/0
      4: 0/0
      5: 0/0
      6: 0/0
      7: 24251/0
      8: 0/0
```
## advanced qos
На коммутаторах есть два режима настройки qos: basic и advanced.
О том, как настраивается qos на eltex mes в режиме basic писал ранее.
Если стоит задача раскрасить трафик политикой, то необходимо настроить коммутатор в режиме qos advanced и при этом необходимо учесть пару вещей:

* В политиках нет class-default, куда попадает весь остальной трафик, поэтому его надо создать самому.
* Переключение из режима advanced в basic приведет к удалению классов и политик из конфигурации.
* Команды qos advanced ports-trusted и qos advanced являются взаимоисключающими, поэтому необходимо проверять вывод show qos.
```text
qos advanced ports-trusted
qos advanced-mode trust dscp
eltex2300#show qos
Qos: Advanced mode
Advanced mode trust type: dscp
Advanced mode ports state: Trusted
CoS to DSCP mapping: disabled
DSCP to CoS mapping: disabled
```
Acl, классы и политика. Все как у cisco, кроме class-default. Ниже для примера политикой будет красится трафик до терминального сервера.
Имя класса all заключено в кавычки автоматом, т.к. скорее всего название я выбрал неудачное.
```text
ip access-list extended mark
 permit tcp any any any 3389 ace-priority 20
exit
!
ip access-list extended "all"
 permit ip any any any any ace-priority 20
exit
!
class-map mark
 match access-group mark
exit
!
class-map all
 match access-group all
exit
!
policy-map mark
 class mark
  set dscp 16
 exit
 class all
 exit
exit
```
Применить к интерфейсу
```text
interface gigabitethernet1/0/1
 service-policy input mark
```
Чтобы отображалась статистика по каждому интерфейсу, включить ее.
```text
qos statistics interface
```
Пакет с dscp 16 попадет в 7 очередь
```text
eltex2300#sh qos map dscp-queue
Dscp-queue map:
     d1 : d2 0  1  2  3  4  5  6  7  8  9
     -------------------------------------
      0 :   02 01 01 01 01 01 01 01 01 03
      1 :   03 03 03 03 03 03 07 04 04 04
      2 :   04 04 04 04 07 05 05 05 05 05
      3 :   05 05 07 06 06 06 06 06 06 06
      4 :   07 08 08 08 08 08 08 08 07 07
      5 :   07 07 07 07 07 07 07 07 07 07
      6 :   07 07 07 07

eltex2300#sh int gi1/0/24
gigabitethernet1/0/24 is up (connected)
[skip]
  Output queues: (queue #: packets passed/packets dropped)
      1: 13949119/0
      2: 18046/0
      3: 0/0
      4: 0/0
      5: 0/0
      6: 9/0
      7: 109083/0
      8: 2442/0
```


## voice vlan

В 4-ой ветке ПО для 2324P мало что поменялось в сравнении с настройками порта в MES2124P.
В 4.0.10.1 все работает аналогично, но почему-то в документации до сих пор предлагается только oui.
Ниже 110 - это голосовой vlan, который отдается телефону, поддерживающему lldp-med.
```text
lldp med network-policy 1 voice vlan 110 vlan-type tagged
lldp med network-policy 2 voice-signaling vlan 110 vlan-type tagged
lldp med network-policy 3 video-conferencing vlan 110 vlan-type tagged

interface gigabitethernet1/0/1
 switchport mode general
 switchport general allowed vlan add 110 tagged
 switchport general allowed vlan add 100 untagged
 switchport general pvid 100                          
 lldp optional-tlv sys-cap
 lldp optional-tlv 802.1 pvid enable
 lldp optional-tlv 802.1 vlan-name add 110
 lldp med enable network-policy poe-pse
 lldp med network-policy add 1
 lldp med network-policy add 2
 lldp med network-policy add 3
```
Если по какой-то причине не включается телефон по poe, а кабель в порядке, то посмотреть счетчики и обратиться в техподдержку.
```text
eltex-2324#sh power inline gi1/0/10

Interface    Admin       Oper         Power (W)     Class     Device     Priority 
---------- ---------- ----------- ----------------- ----- -------------- -------- 
gi1/0/10   Auto       On          1.900             2                    low      

 
Port Status:               Port is on. Valid resistor detected
Port standard:             802.3AT
Admin power limit (for port power-limit mode): 30.0   watts
Time range:
Link partner standard:     802.3AF
Operational power limit:   7.0   watts
Spare pair:                Disabled
Negotiated power:          0 watts (None)
Current (mA):              35
Voltage(V):                54.285
Overload Counter:          0 
Short Counter:             0 
Denied Counter:            0 
Absent Counter:            0 
Invalid Signature Counter: 511 
```
## copp
Ограничение доступа к коммутатору хорошо реализовано.
```text
management access-list mgmt
 deny service http
 deny service https
 permit ip-source 10.9.8.0 mask 255.255.255.0 service snmp
 permit ip-source 10.9.8.0 mask 255.255.255.0 service telnet
 permit ip-source 10.9.8.0 mask 255.255.255.0 service ssh
exit
!
management access-class mgmt
```
## stack
Для стека необходимо использовать два порта на каждом коммутаторе, dac-кабели для соединения портов.
```text
console(config)#stack configuration links te 3-4
```
Тот коммутатор, который предполагается оставить мастером, должен проработать не менее 10 минут.
После чего включить питание на втором коммутаторе стека и подождать пока автоматически обновится софт с мастера.
Удобно, когда не требуется обновлять версию ПО до включения свича в стек.
```text
eltex-3324#sh stack 

Topology is Chain

Unit Id     MAC Address       Role   
------- ------------------- -------- 
   1     e0:d9:e3:d9:55:40   master  
   2     e0:d9:e3:d9:6c:40   backup  
eltex-3324#sh stack links 

Topology is Chain

Unit Id     Active Links        Neighbor Links    Operational     Down/Standby     
                                                  Link Speed      Links            
------- -------------------- -------------------- ----------- -------------------- 
1       te1/0/3-4            te2/0/3-4            10G                              
2       te2/0/3-4            te1/0/3-4            10G                              
```

## span, qinq

Для версии 4.х. В документации есть понятия контролируемого и контролирующего порта. Контролирующий, куда сливать данные (MONITOR), может быть только один.
Надо больше чем один. Для этого есть qinq. Порты 1 и 2 соединены физически кабелем. Т.к. qinq, то увеличить mtu
```text
port jumbo-frame
!
no mac address-table learning vlan 666
!
interface gigabitethernet1/0/1
 description LOOP
 switchport mode customer
 switchport customer vlan 666
exit
!
interface gigabitethernet1/0/2
 description MONITOR
 port monitor vlan 2
 port monitor vlan 100
exit
!
interface gigabitethernet1/0/3
 description DST1
 switchport mode customer
 switchport customer vlan 666
exit
!
interface gigabitethernet1/0/4
 description DST2
 switchport mode customer
 switchport customer vlan 666
exit

eltex#sh ports monitor 

Port monitor mode: monitor-only
    RSPAN configuration
RX: not configured
TX: not configured

Source Port Destination Port  Type     Status    RSPAN   
----------- ---------------- ------- ---------- -------- 
  VLAN 2        gi1/0/2        N/A     active   Disabled 
 VLAN 100       gi1/0/2        N/A     active   Disabled 
```

## jumbo frame

Глобальная настройка, необходима перезагрузка
```text
port jumbo-frame
```

## aaa

Только tacacs, а если недоступен, то локальная учетка
```text
aaa authentication mode break
```
Остальное
```text
aaa accounting commands stop-only group tacacs+
aaa authentication login authorization default tacacs local
aaa authentication enable default tacacs enable
```
Команда в конфиге преобразуется из `tacacs-server host 10.9.9.9 key xx` в
```text
encrypted tacacs-server host 10.9.9.9 key <hash>
```
## ntp

```text
clock timezone KRSK +7
clock source sntp
!
sntp client poll timer 60
sntp unicast client enable
sntp unicast client poll
sntp server 10.9.9.9 poll
```
Слабо помогает. Возможна ситуация с с несвоевременной синхронизацией по времени при старте.
База dchp-snooping'а почистится, DAI и source guard всех забанят. Из-за этого в security фичах, кроме снупинга, нет смысла.

## loopback-detection

`loopback-detection enable` включается глобально и в портах. В портах по умолчанию выключена.
```text
eltex#sh loopback-detection 

Loopback detection: Enabled
Mode: broadcast-mac-addr
Loopback detection interval: 30
VLAN based mode: Disabled
VLAN auto-recovery : Disabled

Interface Loopback Detection Loopback Detection 
          Admin State        Operational State  
--------- ------------------ ------------------ 
gi1/0/1   disabled           inactive           
gi1/0/2   disabled           inactive
```
vlan mode, если в каждом влане порта слать. Если в нативном, то хватит дефолта port mode.
```text
interface gigabitethernet1/0/1
 loopback-detection enable
 spanning-tree disable
```
Теперь
```text
Interface Loopback Detection Loopback Detection 
          Admin State        Operational State  
--------- ------------------ ------------------ 
gi1/0/1   enabled            active
```

## port security

По умолчанию lock mode.
```text
eltex(config-if)#port security mode 
  secure               Delete the current dynamic MAC addresses associated
                       with the port. Learn up to the maximum addresses
                       allowed on the port. Relearning and aging are disabled.
  lock                 Keep the current dynamic MAC addresses associated with
                       the port. Learning, relearning and aging are disabled.
  max-addresses        Delete the current dynamic MAC addresses associated
                       allowed on the port. Relearning and aging are enabled.
```
Ограничение для 3 адресов без ограничений с learning
```text
 port security max 3
 port security mode max-addresses
 port security discard
```
Смотреть
```text
eltex#sh ports security 


Port        status      Learning     Action     Maximum   Trap     Frequency 
---------  ---------  ------------ ----------- --------- --------  ----------
gi1/0/1     Enabled Max-Addresses        Discard        3   Disabled          -
```

## dhcp snooping, relay

Перечисляются vlan'ы, где есть dhcp-клиенты, включается хранение базы на флешке. На tftp сохранить нельзя.
```text
ip dhcp snooping
ip dhcp snooping database
ip dhcp snooping vlan 100
ip dhcp snooping vlan 200
```
Настраиваются доверенные порты, т.е. те, за которыми располагается dhcp-сервер, либо те, данные с которых собирать в локальную базу не нужно.
```text
interface gigabitethernet 1/0/24
 ip dhcp snooping trust
```
Для relay запросов на сервер 10.9.8.7 
```text
ip dhcp relay address 10.9.8.7
ip dhcp relay enable

interface vlan 100
 name user
 ip address 10.0.0.254 255.255.252.0
 ip dhcp relay enable
```
Смотреть
```text
eltex#sh ip dhcp relay 
DHCP relay is Enabled
Option 82 is Disabled
Maximum number of supported VLANs without IP Address is 256
Number of DHCP Relays enabled on VLANs without IP Address is 0
DHCP relay is not configured on any port.
DHCP relay is enabled on Vlans: 100
Active: 100
Inactive: 
Servers: 10.9.8.7

```

## lacp

Можно несколько интерфейсов по sh run вывести. Видно что lacp на стеке настроен.
```text
eltex#sh run int gigabitethernet1/0/5,gigabitethernet2/0/5,port2
interface gigabitethernet1/0/5
 description port1
 channel-group 2 mode auto
!
interface gigabitethernet2/0/5
 description port2
 channel-group 2 mode auto
!
interface Port-channel2
 description lacp
 spanning-tree disable
 switchport access vlan 123
```

## combo ports

```text
interface gigabitethernet1/0/23
 media-type force-fiber
```

## ospf

В базе есть ospf. Настраивается непривычно. Здесь 10.0.1.x - p2p, 10.0.0.0/24 необходимо анонсировать.
```text
interface vlan 10
 name p2p_1
 ip address 10.0.1.238 255.255.255.252
exit
!
interface vlan 20
 name p2p_2
 ip address 10.0.1.234 255.255.255.252
exit
!
interface vlan 100
 name user
 ip address 10.0.0.254 255.255.252.0
 ip dhcp relay enable
! 
router ospf 1
 network 10.0.1.234 area 0.0.0.0
 network 10.0.1.238 area 0.0.0.0
 network 10.0.0.254 area 0.0.0.0
 router-id 10.0.1.238
exit
!
interface ip 10.0.1.234
 ip ospf cost 15
 ip ospf network point-to-point
exit
!
interface ip 10.0.1.238
 ip ospf cost 20
 ip ospf network point-to-point
exit
!
interface ip 10.0.0.254
 ip ospf passive-interface
exit
```


## 21xx подключение к консоли коммутатора

Консольный порт расположен с фронтальной стороны коммутатора. Для подключения необходим обычный консольный кабель и USB2COM переходник,
если на ноутбуке или ПК отсутствует COM-порт. В настройках терминальной программы, например, PuTTY,
необходимо выбрать соответсвующий serial-порт и выставить скорость 115200.
Конфигурация по-умолчанию не содержит никаких учетных записей и сразу можно приступать к настройке,
перейдя в привилегированный режим командой enable. Переходите в режим конфигурации командой conf и задайте hostname.
```text
console>
console>enable
console#conf
console(config)#hostname mes2124p
mes2124p(config)#
```

## 21xx Обновление прошивки

На сайте eltex.nsk.ru на странице коммутатора есть ссылка на архив с крайней версией прошивки, которую рекомендуется установить.
Для начала необходимо установить на ноутбуке tftp-сервер, например, tftpd32.
Распаковать архив прошивки в любой каталог, который будет указан в качестве корневого для tftp-сервера.
Далее нужно прописать ip-адрес на любом интерфейсе коммутатора.
```text
mes2124p(config)#int gi1/0/23
mes2124p(config-if)#ip address 1.0.0.2 255.0.0.0
mes2124p(config-if)#exit
mes2124p(config)#exit
```
Прописать на ноутбуке адрес из той же подсети 1.0.0.1 255.0.0.0, подключиться в настроенный порт и проверить доступность.
```text
mes2124p#ping 1.0.0.1
Pinging 1.0.0.1 with 18 bytes of data:

18 bytes from 1.0.0.1: icmp_seq=1. time=7 ms
18 bytes from 1.0.0.1: icmp_seq=2. time=1 ms
18 bytes from 1.0.0.1: icmp_seq=3. time=8 ms
```
Скопировать прошивку на коммутатор.
```text
copy tftp://1.0.0.1/mes2000-1145.ros image
```
Выбрать новую прошивку для загрузки.
```text
mes2124p#sh bootvar
Image  Filename   Version               Date                    Status
-----  ---------  -------------------   ---------------------   -----------
1      image-1    1.1.36                17-Feb-2015  16:46:12   Active*
2      image-2    1.1.45[5bb753fb]      30-May-2016  12:24:53   Not active

"*" designates that the image was selected for the next boot

mes2124p#boot system image-2
mes2124p#sh bootvar
Image  Filename   Version               Date                    Status
-----  ---------  -------------------   ---------------------   -----------
1      image-1    1.1.36                17-Feb-2015  16:46:12   Active
2      image-2    1.1.45[5bb753fb]      30-May-2016  12:24:53   Not active*

"*" designates that the image was selected for the next boot
```
Сохранить конфигурацию командой write и перезагрузить командой reload.
!! Тоже самое в моделях 23хх и версиях 4.х новая прошивка будет загружена только после релоада. отключения питания недостаточно

## 21xx Особенности новой версии ПО

зачем я это копирую в 2020?

Начиная с версии 1.1.40 для MES2124P реализовано автоматическое отключение/включение вентиляторов в зависимости от температуры встроенного датчика. Реализованы следующие пороги температуры:
<=50: вентиляторы выключены;
45-60: включен правый вентилятор;
>=55: включены оба вентилятора.
Для MES2124P revB и revC автоматическая регулировка скорости вращения вентиляторов в зависимости от температуры.

## 21xx Организация удаленного доступа к коммутатору и подключение в существующую сеть

Предполагается, что все коммутаторы в существующей сети уже настроены, в сети настроен MSTP, порты, которыми соединены коммутаторы ЛВС настроены для работы в режиме trunk.
Выполняется начальная конфигурация MSTP, где количество инстансов и название региона идентичны настройкам коммутатора, к которому будет подключаться eltex mes.
```text
spanning-tree mode mst
spanning-tree bpdu filtering
spanning-tree mst configuration
 instance 1 vlan 1-4094
 name abc
 revision 1
exit
```
Создается vlan управления коммутатора и прописываются сетевые настройки. В случае небольшой сети vlan для управления может быть один для всего оборудования.
```text
mes2124p(config)#vlan database
mes2124p(config-vlan)#vlan 111 name management
mes2124p(config-vlan)#exit
mes2124p(config)#int vlan 111
mes2124p(config-if)#ip address 192.168.10.100 255.255.255.0
mes2124p(config-if)#exit
mes2124p(config)#ip default-gateway 192.168.10.1
```
Настраивается транковый порт, которым eltex mes будет подключаться к вышестоящему коммутатору. В конкретном примере это 24 медный порт, но можно использовать любые, в том числе и combo-порты.
```text
mes2124p(config)#int gi1/0/24
mes2124p(config-if)#switchport mode trunk
mes2124p(config-if)#switchport trunk allowed vlan add all
mes2124p(config-if)#description uplink
```
Принципиальное отличие от коммутаторов cisco в том, то в IOS можно указать только "switchport mode trunk" и сразу в порту будут разрешены все vlan без ограничения. Здесь же в отсутствии строки "switchport trunk allowed vlan add" разрешен только первый (1) vlan, через который передаются нетегированные фреймы. "all" в конфигурации порта заменяется на список всех созданных на коммутаторе vlan'ов.
Теперь надо проверить работу mstp и убедиться что роль "Root" есть на аплинке.
```text
mes2124p#sh spanning-tree active

*********************************** Process 0 ***********************************

Spanning tree enabled mode MSTP
Default port cost method:  long
Loopback guard:   Disabled


Gathering information ..........
###### MST 0 Vlans Mapped:

CST Root ID    Priority    28672
               Address     00:15:63:00:5d:80
               The IST ROOT is the CST ROOT
               Root Port   gi1/0/24
               Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
IST Master ID  Priority    28672
               Address     00:15:63:00:5d:80
               Path Cost   220000
               Rem hops    18
Bridge ID      Priority    32768
               Address     a8:f9:4b:a6:4a:c0
               Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
               Max hops    20
  Name     State   Prio.Nbr   Cost    Sts   Role PortFast       Type
--------- -------- -------- -------- ------ ---- -------- -----------------
gi1/0/23  enabled  128.71   200000   Frw    Desg No       P2P Intr
gi1/0/24  enabled  128.72   200000   Frw    Root No       P2P Intr

###### MST 1 Vlans Mapped: 1,111

Root ID        Priority    28672
               Address     00:15:63:00:5d:80
               Path Cost   220000
               Root Port   gi1/0/24
               Rem hops    18


Bridge ID      Priority    32768
               Address     a8:f9:4b:a6:4a:c0
Interfaces
Name       State     Prio.Nbr   Cost      Sts  Role  PortFast  Type
--------   --------  --------   --------- ---- ----  --------  ----------
gi1/0/23   enabled   128.71     200000    Frw  Desg  No        P2P Inter
gi1/0/24   enabled   128.72     200000    Frw  Root  No        P2P Inter
```
Если настройки MSTP выполнены некорректно, например, имя региона не совпадает с именем региона на соседнем
коммутаторе или оно написано в другом регистре (abc не идентично ABC), то статус будет как в примере ниже.
В общем случае, необходимо перепроверить настройки на соседних коммутаторах.
```text
mes2124p#sh spanning-tree active

*********************************** Process 0 ***********************************

Spanning tree enabled mode MSTP
Default port cost method:  long
Loopback guard:   Disabled


Gathering information ..........
###### MST 0 Vlans Mapped:

CST Root ID    Priority    28672
               Address     00:15:63:00:5d:80
               Path Cost   200000
               Root Port   gi1/0/24
               This switch is the IST master
               Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
Bridge ID      Priority    32768
               Address     a8:f9:4b:a6:4a:c0
               Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
               Max hops    20
  Name     State   Prio.Nbr   Cost    Sts   Role PortFast       Type
--------- -------- -------- -------- ------ ---- -------- -----------------
gi1/0/23  enabled  128.71   200000   Frw    Desg No       P2P Intr
gi1/0/24  enabled  128.72   200000   Frw    Root No       P2P Bound (RSTP)

###### MST 1 Vlans Mapped: 1,111

Root ID        Priority    32768
               Address     a8:f9:4b:a6:4a:c0
               This switch is the regional Root
Interfaces
Name       State     Prio.Nbr   Cost      Sts  Role  PortFast  Type
--------   --------  --------   --------- ---- ----  --------  ----------
gi1/0/23   enabled   128.71     200000    Frw  Desg  No        P2P Inter
gi1/0/24   enabled   128.72     200000    Frw  Mstr  No        P2P Bound (RSTP)
```
Остается создать учетную запись администратора с максимальным уровнем привилегий.
До работы с учетными записями рекомендую выполнить сохранение конфигурации.
Если забудется пароль или еще что-нибудь, то всегда можно перезагрузить и попробовать еще раз.
```text
mes2124p(config)#username oper privilege 15 password gfhjkm
mes2124p(config)#aaa authentication login default local
mes2124p(config)#exit
mes2124p#exit
```
Если все сделано корректно, то появится приглашение `User Name:` и у вас получится авторизоваться в качестве локального администратора.
После этого можно еще раз сохранить конфигурацию.

## 21xx bpdufilter и bpduguard

Конструкция ниже заблокировала порт, так как bpduguard отрабатывает раньше, чем фильтр.
Да и сам фильтр фильтрует только входящие bpdu.
```text
mes2124p#sh run int gi1/0/23
interface gigabitethernet 1/0/23
 switchport mode trunk
 spanning-tree bpdu filtering
 spanning-tree bpduguard enable
```

## 21xx dhcp-snooping

Перечисляются vlan'ы, где есть dhcp-клиенты, включается хранение базы на флешке. На tftp сохранить нельзя.
```text
ip dhcp snooping
ip dhcp snooping database
ip dhcp snooping vlan 100
ip dhcp snooping vlan 200
```
Настраиваются доверенные порты, т.е. те, за которыми располагается dhcp-сервер, либо те, данные с которых собирать в базу не нужно.
```text
interface gigabitethernet 1/0/24
 ip dhcp snooping trust
```
Например, есть последовательная цепочка из двух коммутаторов.
Если настроить в качестве доверенного порта только аплинки обоих коммутаторов, то на первом коммутаторе будет собираться база клиентов обоих коммутаторов.
Это не смертельно в общем случае, но при использовании всего свободного места на флешке коммутатор перезагрузится из-за возникшей ошибки,
как у меня было с mes3124.
```text
mes2124p#dir
Directory of flash:

     File Name      Permission Flash Size Data Size        Modified
------------------- ---------- ---------- --------- -----------------------
copyhist                rw       65520       12      17-Feb-2015 16:46:38
dhcpsn.prv              --       131040      --      05-Mar-2015 13:32:16
directry.prv            --       65520       --      09-Aug-2011 10:35:05
image-1                 rw      6815744    6815744   25-Feb-2015 17:26:00
image-2                 rw      6815744    6815744   17-Feb-2015 16:54:01
mirror-config           rw       131040     3596     01-Jun-2016 06:19:39
sshkeys.prv             --       131040      --      11-Aug-2011 12:35:14
startup-config          rw       458640     86045    30-May-2016 18:19:59
syslog1.sys             r-       131072      --      31-May-2016 17:34:10
syslog2.sys             r-       131072      --      31-May-2016 17:34:10

Total size of flash: 16252928 bytes
Free size of flash: 1376496 bytes
```
## 21xx voice vlan
Сейчас техническая документация на коммутатор с поддержкой PoE Eltex MES2124P описывает два способа подключения ip-телефонов,
настройки voice vlan и в обоих случаях на коммутаторе необходимо указывать часть mac-адреса - oui. Цитата из документации:

Voice VLAN используется для выделения VoIP-оборудования в отдельную VLAN. Для VoIP-фреймов могут быть назначены QoS-атрибуты для приоритезации трафика.
Классификация фреймов, относящихся к фреймам VoIP-оборудования, базируется на OUI (Organizationally Unique Identifier – первые 24 бита MAC-адреса) отправителя.
Назначение Voice VLAN для порта происходит автоматически - когда на порт поступает фрейм с OUI из таблицы Voice VLAN.
Когда порт определяется, как принадлежащий к Voice VLAN – данный порт добавляется во VLAN как tagged. Voice VLAN применим для следующих схем:

* VoIP-оборудование настраивается, чтобы рассылать тегированные пакеты с Voice VLAN ID, настроенным на MES2124.
* VoIP-оборудование рассылает нетегированные DHCP-запросы. В ответе от DHCP-сервера присутствует опция 132, содержащая VLAN ID, который VoIP-устройство автоматически назначает себе в качестве VLAN для маркировки VoIP-трафика (Voice VLAN ID).

В общем, получилось настроить как надо, т.е. с применением протоколов lldp, lldp-med и без указания oui.
Сразу хочу предупредить, что фраза "для VoIP-фреймов могут быть назначены QoS-атрибуты для приоритезации трафика" скорее всего "up 5",
а не dscp. Телефонам, как оказалось, до одного места настройки dscp, полученные через lldp-med.
Как в настройках (provisioning) указано, так и будет работать. Телефоны возвращают в своем выводе по lldp вообще погоду какую-то,
а не собственные настройки qos и т.д, но лучше лишний раз перестраховаться и посмотреть трафик wireshark'ом.
```text
lldp med network-policy 1 voice vlan 203 vlan-type tagged up 5 dscp 46
```
Начать нужно с отключения автоматического формирования network-policy, которое используется при настройке voice vlan по документации.
```text
no lldp med network-policy voice auto 
```
Настраивается network-policy вручную. В примере ниже 203 - это номер voice vlan'а. Всего разрешается настроить 32 политики.
В примере ниже я использую три штуки для одного телефона, но, полагаю, с одной только первой будет работать.
Таким образом, можно настроить несколько voice vlan на одном коммутаторе и обойти ограничение документированного способа,
который подразумевает только один voice vlan на коммутатор. Команды вводятся в режиме конфигурирования, но в самом конфиге увидеть их нельзя.
```text
lldp med network-policy 1 voice vlan 203 vlan-type tagged
lldp med network-policy 2 voice-signaling vlan 203 vlan-type tagged
lldp med network-policy 3 video-conferencing vlan 203 vlan-type tagged
```
Для теста использовал три телефона разных производителей, настроил три порта как в примере ниже.
Тут 203 - voice vlan (тегированный), 555 - data (нетегированный).
```text
interface gigabitethernet 1/0/1
 switchport mode general
 switchport general allowed vlan add 203 tagged
 switchport general allowed vlan add 555 untagged
 description Users_and_IpPhones_with_LLDP-MED
 spanning-tree portfast auto
 spanning-tree bpduguard enable
 lldp optional-tlv sys-cap
 lldp optional-tlv 802.1 pvid enable
 lldp optional-tlv 802.1 vlan-name add 203
 lldp med enable network-policy poe-pse
 lldp med network-policy add 1
 lldp med network-policy add 2
 lldp med network-policy add 3
 switchport general pvid 555
```
Вот так можно посмотреть настроенные network-policy и что вообще настроено на порту.
```text
mes2124p#sh lldp med configuration

Fast Start Repeat Count: 3.
LLDP MED network-policy voice: manual

Network policy 1
-------------------
Application type: voice
VLAN ID: 203 tagged
Layer 2 priority: 0
DSCP: 0

Network policy 2
-------------------
Application type: voiceSignaling
VLAN ID: 203 tagged
Layer 2 priority: 0
DSCP: 0

Network policy 3
-------------------
Application type: videoconferencing
VLAN ID: 203 tagged
Layer 2 priority: 0
DSCP: 0

  Port     Capabilities   Network policy   Location  POE  Notifications  Inventory
--------- -------------- ---------------- ---------- ---- -------------- ----------
gi1/0/1        Yes             Yes            No     Yes     Disabled        No
gi1/0/2        Yes             Yes            No     Yes     Disabled        No
gi1/0/3        Yes             Yes            No     Yes     Disabled        No
gi1/0/4         No              No            No      No     Disabled        No
[пропущено]
```
Далее все просто. Включается телефон и его видно по lldp. Это, как минимум, говорит о том, что разными настройками на телефоне lldp включен и в текущей прошивке этот протокол и lldp-med поддерживается.
```text
mes2124p#sh lldp neighbors

System capability legend:
B - Bridge; R - Router; W - Wlan Access Point; T - telephone;
D - DOCSIS Cable Device; H - Host; r - Repeater;
TP - Two Ports MAC Relay; S - S-VLAN; C - C-VLAN; O - Other

  Port        Device ID          Port ID         System Name    Capabilities  TTL
--------- ----------------- ----------------- ----------------- ------------ -----
gi1/0/1    01 0a 2e 21 42    3CCE73582D2B:P1  SEP3CCE73582D2B       B, T      157
gi1/0/2    01 0a 2e 21 fa   00:15:65:9e:44:21    SIP-T21P_E2         T        151
gi1/0/3    01 0a 2e 21 07   00:24:b5:64:39:81                       B, T      166
mes2124p#sh lldp neighbors gi1/0/1

Device ID: 01:0a:2e:21:42:00
Port ID: 3CCE73582D2B:P1
Capabilities: Bridge, Telephone
System Name: SEP3CCE73582D2B
System description: Cisco IP Phone 7911G,V9, SIP11.8-5-3S
Port description: SW PORT
Time To Live: 159
[пропущено]

mes2124p#sh lldp neighbors gi1/0/2

Device ID: 01:0a:2e:21:fa:00
Port ID: 00:15:65:9e:44:21
Capabilities: Telephone
System Name: SIP-T21P_E2
System description: 52.80.14.1
Port description: WAN PORT
Time To Live: 142
[пропущено]

mes2124p#sh lldp neighbors gi1/0/3

Device ID: 01:0a:2e:21:07:00
Port ID: 00:24:b5:64:39:81
Capabilities: Bridge, Telephone
System Name:
System description: Nortel IP Telephone 1120E, Firmware:0624C7F
Port description: Nortel IP Phone
Time To Live: 150
Проверка mac-адресов в голосовом влане.
mes2124p#sh mac address-table vlan 203 interface gi1/0/1

  Vlan        Mac Address         Port       Type
-------- --------------------- ---------- ----------
  203      3c:ce:73:58:2d:2b    gi1/0/1    dynamic

mes2124p#sh mac address-table vlan 203 interface gi1/0/2

  Vlan        Mac Address         Port       Type
-------- --------------------- ---------- ----------
  203      00:15:65:9e:44:21    gi1/0/2    dynamic

mes2124p#sh mac address-table vlan 203 interface gi1/0/3

  Vlan        Mac Address         Port       Type
-------- --------------------- ---------- ----------
  203      00:24:b5:64:39:81    gi1/0/3    dynamic
```
Просмотрим что с PoE на коммутаторе.
```text
mes2124p#sh power inline

Class based power-limit mode

 Power  Nominal Power   Consumed Power   Usage Threshold   Traps
------- ------------- ------------------ --------------- ---------
  On      350 Watts      7 Watts (2%)          95         Disable


  Port      Powered Device         State          Status    Priority   Class
-------- -------------------- ---------------- ------------ -------- ---------
gi1/0/1                             Auto            On        low     class2
gi1/0/2                             Auto            On        low     class2
gi1/0/3                             Auto            On        low     class3

mes2124p#sh power inline consumption

   Port      Power[W]     Power Limit[W]  Current[mA]   Voltage[V]
---------- ------------- ---------------- ------------ ------------
 gi1/0/1       2.806          30.000           53         52.006
 gi1/0/2       1.499          30.000           28         52.248
 gi1/0/3       3.487          30.000           67         51.898
```

## 21xx Настройка QoS
Speed mismatch для коммутаторов - это серьезная потенциальная проблема, которая может привести к потерям при передаче данных.
Если денег нет, то лучше лишний раз протестировать "коммутатор, который дешевле на xxx%", особенно когда:

* закладывается решение с аплинком 1гбит/с и портами доступа 100мбит/с или с аплинком 10гбит/с и портами доступа 1гбит/с;
* предполагается включать в гигабитный коммутатор доступа ip-телефоны со 100мбит/с интерфейсом;
* включили сервера по 10ге. Замечательно. Готовьтесь купить дорогостоящие коммутаторы ядра/агрегации с 10ге, чтобы все возможные микробёрсты буферизировать до коммутаторов доступа без потерь.

Одним из способов экономии при построении ЛВС будет выбор таких решений, где скорость всех интерфейсов одинакова, а сеть минимально нагружена.

Коммутаторы 1124 и 2124 имеют по 4 очереди. 3124 - 8 очередей. Настройки всех моделей одинаковы, но ориентироваться лучше на крайние версии прошивок, так как qos tail-drop появился относительно недавно.
Включается qos одной командой.
```text
mes2124p(config)#qos
```
Проверка работы qos с настройками по умолчанию.
```text
mes2124p#sh qos
Qos: Basic mode
Basic trust: cos

mes2124p#sh qos interface gi1/0/1
Ethernet gi1/0/1
Default CoS: 0
Trust mode: enable
```
Вот, к слову, eltex mes - одни из немногих недорогих коммутаторов, которые позволяют смотреть и собирать по snmp статистику по очередям.
Хотя можно мониторить всего два параметра, но это уже достижение, поверьте, если сравнивать с глючным DCN (qtech,snr).
Сейчас коммутатор стоит на столе, к нему подключено мое рабочее место через ip-телефон. Настрою статистику для этого порта.
1ый счетчик - это приоритетная очередь, куда попадет телефония. 2 - первая очередь, куда попадет весь остальной трафик.
```text
qos statistics queues 1 4 all gigabitethernet1/0/1
qos statistics queues 2 1 all gigabitethernet1/0/1
```
Цитата из документации по snmp-мониторингу коммутаторов Eltex MES:
```text
1.3.6.1.4.1.89.88.35.4.1.10.1 для 1 счетчика
1.3.6.1.4.1.89.88.35.4.1.10.2 для 2 счетчика
snmpwalk -v2c -c <community> <IP address> 1.3.6.1.4.1.89.88.35.4.1.10.{номер tail drop счетчика}
```
Т.е. если настроить счетчик очереди для всех интерфейсов, то можно опросить всего один oid
и получить информацию о дропах в приоритетной очереди на всех портах.
Правда, придется искать потом на каком порту конкретно, но это мелочи ;) 
Просмотр статистики не показывает попадания в приоритетную очередь, так как на телефонах настроена маркировка Diffserv.
```text
mes2124p#sh qos statistics
Output Queues
-------------

  Interface      Queue      Dp    Total packets   TD packets
------------- ------------ ----- --------------- -------------
   gi1/0/1         4        All         1              0
   gi1/0/1         1        All       6424             0
Включаю trust dscp одной командой. На портах прописывать не нужно.
mes2124p(config)#qos trust dscp

mes2124p#sh qos
Qos: Basic mode
Basic trust: dscp
```
Теперь все корректно, голосовой трафик попадает в свою очередь. Потерь в первой очереди нет, но это еще из-за совпадения скоростей на порту доступа и аплинке.
```text
mes2124p#sh qos statistics

Output Queues
-------------

  Interface      Queue      Dp    Total packets   TD packets
------------- ------------ ----- --------------- -------------
   gi1/0/1         4        All       1445             0
   gi1/0/1         1        All       10909            0
```
В свое время обратил внимание при мониторинге очередей QoS на то, что есть потери только на тех коммутаторах Eltex MES 3124,
где скорости на портах отличаются. Например, 10ге-1ге и 1ге-100мбит/с.
А на  коммутаторах агрегации с только гигабитными подключениями потерь не было вообще.
Проблема была частично решена обновлением прошивки и настройкой очередей.

### Проверка буферов и настроек очередей

Для тестирования коммутатора необходимо порт доступа включить на скорость существенно меньшую, чем аплинк.
Есть два способа изменения скоростей подключения на всех коммутаторах и оба приведены в примере ниже.
Принципиальное отличие в том, что первая команда выставить фиксированную скорость подключения, но отключит автосогласование.
Отключенное автосогласование на скоростях 10 и 100мбит/с приведет к тому, что на удаленной стороне включится half-duplex.
Поэтому корректно это делать используя команду "negotiation". Если рассматривать циску,
то там это делается одной командой, но добавляется параметр auto, т.е. "speed auto 10 100".
```text
mes2124p(config)#int gi1/0/1
mes2124p(config-if)#speed
  10                   Force operation at 10 Mbps.
  100                  Force operation at 100 Mbps.
  1000                 Force operation at 1000 Mbps.
mes2124p(config-if)#negotiation
  10h                  Advertise 10 Half-Duplex.
  10f                  Advertise 10 Full-Duplex.
  100h                 Advertise 100 Half-Duplex.
  100f                 Advertise 100 Full-Duplex.
  1000f                Advertise 1000 Full-Duplex.
   
mes2124p(config-if)#negotiation 10f
mes2124p(config-if)#04-Jun-2016 13:02:53 %LINK-W-Down:  gi1/0/1
04-Jun-2016 13:02:55 %LINK-I-Up:  gi1/0/1, 10Mbps FULL duplex
```
Теперь необходимо очистить счетчики, позвонить, качнуть чего-нибудь.
В выводе ниже появились потери в 1 очереди, 10мбит/с интерфейс загружен практически максимально.
```text
mes2124p#clear qos statistics
mes2124p#sh qos statistics

Output Queues
-------------

  Interface      Queue      Dp    Total packets   TD packets
------------- ------------ ----- --------------- -------------
   gi1/0/1         4        All        566             0
   gi1/0/1         1        All       20893           51

mes2124p#sh int gi1/0/1
gigabitethernet 1/0/1 is up (connected)
  Interface index is 49
  Hardware is gigabitethernet, MAC address is a8:f9:4b:a6:4a:c1
  Description: Users_and_IpPhones_with_LLDP-MED
  Interface MTU is 1500
  Full-duplex, 10Mbps, link type is auto, media type is 1G-Copper
  Link is up for 0 days, 0 hours, 2 minutes and 54 seconds
  Advertised link modes: 10baseT/Full
  Flow control is off, MDIX mode is on
  15 second input rate is 283 Kbit/s
  15 second output rate is 9849 Kbit/s
```
Что еще можно поделать? В документации в разделе настройки qos сообщается:
По умолчанию, все очереди обрабатываются по алгоритму «strict priority».
Я этого понять не смог, простите. Настрою strict priority только для голоса.
Выглядит это в конфигурации как указание количества таких очередей.
Поначалу путался, но потом подумал, что если рассматривать не только 2124, где всего 4 очереди, а еще и 3124, где их 8 штук,
то настройка в таком виде имеет смысл.
```text
mes2124p(config)#priority-queue out num-of-queues
  <0-4>                Assign the number of queues to be expedite queues. The
                       expedite queues would be the queues with higher indexes
  0                    all queues are assured forwarding (WRR)
  4                    all queues are expedite
mes2124p(config)#priority-queue out num-of-queues 1
mes2124p(config)#
```
Теперь надо настроить буферы очередей, чтобы таких дропов, как в примере выше, избежать по возможности.
Ниже состояние буферов и очереди интерфейса по умолчанию. Меня не покидает ощущение недопонимания процесса оптимальной настройки,
но делать надо, учиться тоже надо, даже на ошибках (лучше на стенде).
```text
mes2124p#sh qos tail-drop
  Physical buffers enqueued:   87        (limit 4032)
  Total descriptors enqueued:  0         (limit 3854)
  Total buffers enqueued:      0         (limit 262143)
  MC descriptors enqueued:     0         (limit 896)
  Shared descriptors enqueued: 0         (limit 1056)
  Shared buffers enqueued:     0         (limit 1056)

mes2124p#sh qos tail-drop interface gi1/0/1
Port: gi1/0/1 is bound to default tail-drop profile
  Port descriptors enqueued:           0         (limit 44)
  Port buffers enqueued:               0         (limit 264)
    Queue 1 descriptors enqueued:      0         (limit 12)
            buffers enqueued:          0
    Queue 2 descriptors enqueued:      0         (limit 12)
            buffers enqueued:          0
    Queue 3 descriptors enqueued:      0         (limit 12)
            buffers enqueued:          0
    Queue 4 descriptors enqueued:      0         (limit 12)
            buffers enqueued:          0
По аналогии с цисковким queue-set тут есть tail-drop profile, но если у циски всего 2 профиля, то тут можно сделать 4.
mes2124p(config)#qos tail-drop profile 1
mes2124p(config-tdprofile)#queue 1 limit
  <0-400>              Queue limit in packets. Minimal recommended value is
                       64 in case when WRTD is enabled. Check user manual for
                       details.
mes2124p(config-tdprofile)#queue 1 limit 400
mes2124p(config-tdprofile)#queue 2 limit 150
mes2124p(config-tdprofile)#queue 3 limit 150
mes2124p(config-tdprofile)#queue 4 limit 150
```
Профиль необходимо применить к интерфейсу.
```text
mes2124p(config)#int gi1/0/1
mes2124p(config-if)#qos tail-drop profile 1
mes2124p(config-if)#exit
mes2124p(config)#
mes2124p#sh qos tail-drop interface gi1/0/1
Port: gi1/0/1 is bound to tail-drop profile 1
  Port descriptors enqueued:           0         (limit 44)
  Port buffers enqueued:               0         (limit 264)
    Queue 1 descriptors enqueued:      0         (limit 400)
            buffers enqueued:          0
    Queue 2 descriptors enqueued:      0         (limit 150)
            buffers enqueued:          0
    Queue 3 descriptors enqueued:      0         (limit 150)
            buffers enqueued:          0
    Queue 4 descriptors enqueued:      0         (limit 150)
            buffers enqueued:          0
```
Очередной тест показывает отсутствие потерь в очередях.
```text
mes2124p#sh qos statistics

Output Queues
-------------

  Interface      Queue      Dp    Total packets   TD packets
------------- ------------ ----- --------------- -------------
   gi1/0/1         4        All         0              0
   gi1/0/1         1        All      115571            0

mes2124p#sh qos tail-drop
  Physical buffers enqueued:   764       (limit 4032)
  Total descriptors enqueued:  111       (limit 3854)
  Total buffers enqueued:      665       (limit 262143)
  MC descriptors enqueued:     0         (limit 896)
  Shared descriptors enqueued: 0         (limit 1056)
  Shared buffers enqueued:     0         (limit 1056)
```
