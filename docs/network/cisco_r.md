## совет

Вот ссылка на [Guide to Harden Cisco IOS Devices](https://www.cisco.com/c/en/us/support/docs/ip/access-lists/13608-21.html).
Там и vstack уже есть для коммутаторов.

## aux

Наверное, в каждом маршрутизаторе cisco такой порт есть и он позволяет настраивать удаленно через консоль другое оборудование.
Для этого нужен пачкодр обжатый с одной стороны в обратном порядке. Ну, в общем, rj45 коннектор перевернуть и обжать.
Скорее всего можно было проще настроить, но вот столько команд в линии сейчас:
```text
line aux 0
 session-timeout 5 
 exec-timeout 5 0
 absolute-timeout 60
 modem InOut
 no exec
 transport preferred telnet
 transport input all
```
Узнать порт линии командой `sh line`. Напротив AUX будет номер и чаще это 1. Реже 65. Т.о. надо выполнить telnet на порт 2001 маршрутизатора.
Сброс линии `clear line aux 0`

## aaa для tacacs

Для версии IOS 15.7
```text
aaa new-model
!
!
aaa authentication login default group tacacs+ local
aaa authentication enable default group tacacs+ enable none
aaa authorization console
aaa authorization exec default group tacacs+ local 
aaa accounting commands 15 default
 action-type start-stop
 group tacacs+
!
aaa accounting network default
 action-type start-stop
 group tacacs+
!
aaa accounting connection default
 action-type start-stop
 group tacacs+
!
aaa accounting system default
 action-type start-stop
 group tacacs+
```
Сам tacacs тоже надо прописать. Есть легаси `tacacs-server host ...`, есть рекомендованный вариант с группами.

Для IOS XE все более привычно как было в 12 ветке ios.
```text
aaa new-model
!
!
aaa authentication login default group tacacs+ local
aaa authentication enable default group tacacs+ enable none
aaa authorization console
aaa authorization exec default group tacacs+ local 
aaa accounting commands 15 default start-stop group tacacs+
aaa accounting network default start-stop group tacacs+
aaa accounting connection default start-stop group tacacs+
aaa accounting system default start-stop group tacacs+
```

## logging

Про aux писал выше. Если остался кабель, то в устройстве надо выключить логирование в консоль.
В противном случае в логах такакса будет много чего почитать.
```text
no logging console
```
В консоли и vty прописать
```text
line con 0
 logging synchronous
line vty 0 4
 logging synchronous
```
Фильтрация сообщений. Например, трансивер левый показывает некорректную температуру, чем создает сообщения от шасси в логах
```text
%SFF8472-5-THRESHOLD_VIOLATION: Te1/2: Temperature low alarm; Operating value: -127.2 C, Threshold value:   -4.0 C.
%SFF8472-5-THRESHOLD_VIOLATION: Te1/2: Temperature high alarm; Operating value:  118.0 C, Threshold value:   74.0 C.
```
```text
logging discriminator FIX mnemonics drops THRESHOLD_VIOLATION 
logging buffered discriminator FIX
logging host 10.9.8.1 discriminator FIX
```
По `sh logging` можно увидеть активные дискриминаторы и счетчики дропов `message lines dropped-by-MD`.

Время в логах:
```text
service timestamps debug datetime msec localtime
service timestamps log datetime msec localtime
```

##cisco 4321 throughput
После пробного периода использования throughput лицензии в 4321 (16.9.4), она становится Life time и RightToUse.
```text
license accept end user agreement
platform hardware throughput level 100000
```
Это на прием и передачу. Как мало.
```text
c4321#sh ver | in throughput
The current throughput level is 100000 kbps
c4321#sh lic | beg throughput
Index 9 Feature: throughput
        Period left: Life time
        License Type: RightToUse
        License State: Active, In Use
        License Count: Non-Counted
```

## output service-policy

Приоритезировать трафик на сабинтерфейсы в рамках какой-то полосы
```text
policy-map myservice
 class Voip
  priority 128
 class class-default
  fair-queue
```
Попытка применить.
```text
router(config-subif)#service-policy output myservice
CBWFQ : Not supported on subinterfaces
```
Надо переделать и прописать политику в шейпер дефолтового класса. И уже эту политику применить.
```text
policy-map 512k
 class class-default
  shape average 512000
  service-policy myservice

router(config-subif)#service-policy output 512k
```

## eem

Периодические проблемы с потоком Е1 на голосовом шлюзе Cisco. Cкрипт, выключающий ночью D-канал.
```text
event manager session cli username "operator"
event manager applet restartd
 event timer cron cron-entry "26 4 * * *" maxrun 120
 action 1.0 syslog msg "restart d-channel"
 action 2.0 cli command "enable"
 action 3.0 cli command "conf t"
 action 4.0 cli command "voice-port 0/0/0:15"
 action 5.0 cli command "shutdown"
 action 6.0 wait 30
 action 7.0 cli command "no shutdown"
 action 8.0 cli command "end"
```

## kron

Периодическая перезагрузка по расписанию
```text
kron occurrence reboot at 4:00 13 recurring
 policy-list reboot
!
kron policy-list reboot
 cli reload
```

## tcp-mss

Описано в [/network/tcp_mss/](/network/tcp_mss/)
```text
ip tcp adjust-mss <значение>
```

## frame-relay, rtp compression

В целом, протокол полезный:

* Frame-relay позволяет создавать виртуальные каналы в пределах одного физического интерфейса. Инкапсуляции HDLC, PPP такого не позволяют;
* Для каждого логического интерфейса можно выделить полосу, настроить QoS;
* Один из логических интерфейсов можно включить в bridge-группу с ethernet'ом, к примеру;
* Можно настроить frame-relay switching и так передавать данные между виртуальными каналами;
* У VoFR самое эффективное использование полосы при передаче голоса;
* MFR работает лучше Multilink PPP. Меньше rtt, субъективно стабильнее.

Раньше передача данных осуществлялась по очень дорогим спутниковым каналам или дорогим цифровым междугородним/международным каналам связи.
Качество "голоса" и рациональное использование полосы за счет мультиплексирования виртуальных каналов позволяли операторам оказывать мультисервисные услуги связи.
Компрессия заголовков работает только на WAN интерфейсах с PPP,HDLC,FR-инкапсуляцией.
Пример:
```text
interface Serial0/2/0:0.100 point-to-point
ip unnumbered Loopback0
frame-relay interface-dlci 100
frame-relay ip rtp header-compression

interface Serial0/1/0
ip unnumbered Loopback0
ip rtp header-compression
encapsulation ppp
```
Из калькулятора:
```text
Codec Bit Rate 8 kbps = (Codec Sample Size * 8) / (Codec Sample Interval)
Codec Sample Size 10 bytes size of each individual codec sample
Codec Sample Interval 10 msec the time it takes for a single sample

Codec: g729_All_Variants
Voice Payload Size: 20 bytes
Voice Protocol: VoIP
Compression: Not Applicable
Media Access: Ethernet
Tunnel/Security/Misc: None
Number of Calls: 1

Total Bandwidth (including Overhead) 32.76 kbps

Codec: g729_All_Variants
Voice Payload Size: 20 bytes
Voice Protocol: VoIP
Compression: off
Media Access: Frame-Relay
Tunnel/Security/Misc: None
Number of Calls: 1

Total Bandwidth (including Overhead) 28.14 kbps

Codec: g729_All_Variants
Voice Payload Size: 20 bytes
Voice Protocol: VoIP
Compression: on
Media Access: Frame-Relay
Tunnel/Security/Misc: None
Number of Calls: 1

Total Bandwidth (including Overhead) 12.18 kbps


Codec: g729_All_Variants
Voice Payload Size: 30 bytes
Voice Protocol: VoFR
Media Access: Not Applicable
Tunnel/Security/Misc: Not Applicable
Number of Calls: 1

Cisco IOS Total Bandwidth Needed for 1.0 Calls 10 kbps
```

## acl

Было сохранено в качестве памятки откуда-то. Как раз для случаев, когда "permit ip any any" почему-то "не работает".

Here’s a handy list of ACL entries to allow your devices to speak routing protocols, availability protocols, and some other stuff.
We’ll assume you have ACL 101 applied to your Ethernet inbound; your Ethernet has an IP of 192.168.0.1.

BGP : Runs on TCP/179 between the neighbors
```text
access-list 101 permit tcp any host 192.168.0.1 eq 179
```
EIGRP : Runs on its own protocol number from the source interface IP to the multicast address of 224.0.0.10
```text
access-list 101 permit eigrp any host 224.0.0.10
```
OSPF : Runs on its own protocol number from the source interface IP to the multicast address of 224.0.0.5; also talks to 224.0.0.6 for DR/BDR routers
```text
access-list 101 permit ospf any host 224.0.0.5
access-list 101 permit ospf any host 224.0.0.6
```
HSRP : Runs on UDP from the source interface IP to the multicast address of 224.0.0.2. I’ve seen in the past that it runs on UDP/1985, but I didn’t find any evidence of that in a quick Google for it. Can someone verify?
```text
access-list 101 permit udp any host 224.0.0.2
```
RIP : Runs on UDP/520 from the source interface IP to the multicast address of 224.0.0.9
```text
access-list 101 permit udp any host 224.0.0.9 eq 520
```
VRRP : Runs on its own protocol number from the source interface IP to the multicast address of 224.0.0.18
```text
access-list 101 permit 112 any host 224.0.0.18
```
GLBP : Runs on UDP from the source interface IP to the multicast address of 224.0.0.102
```text
access-list 101 permit udp any host 224.0.0.102
```
DHCPD (or bootps) : Runs on UDP/67 from 0.0.0.0 (since the client doesn’t have an address yet) to 255.255.255.255 (the broadcast).
```text
access-list 101 permit udp any host 255.255.255.255 eq 67
```

## session mib

Это вообще времен диалапа и прочего доступа, когда использовались cisco в качестве AS и BRAS.
Так с сервера доступа Cisco AS5350 можно получить данные о состоянии цифровых модемов и тем самым судить о загрузке модемного пула:
```text
cmSystemInstalledModem      1.3.6.1.4.1.9.9.47.1.1.1.0
cmSystemModemsInUse         1.3.6.1.4.1.9.9.47.1.1.6.0 
cmSystemModemsAvailable     1.3.6.1.4.1.9.9.47.1.1.7.0
cmSystemModemsUnavailable   1.3.6.1.4.1.9.9.47.1.1.8.0 
cmSystemModemsOffline       1.3.6.1.4.1.9.9.47.1.1.9.0
cmSystemModemsDead          1.3.6.1.4.1.9.9.47.1.1.10.0
```
Можно запрашивать информацию, чтобы обработать далее по номеру доступа.
Нужно будет подсчитать каким-нибудь скриптом количество строк на каждый номер, если номеров доступа несколько
```text
cpmActiveLocalPhoneNumber 1.3.6.1.4.1.9.10.19.1.3.1.1.13
```
Количество PPPoE-сессий
```text
cPppoeSystemCurrSessions 1.3.6.1.4.1.9.9.194.1.1.1
```
Количество PPtP-сессий
```text
cvpdnSystemSessionTotal  1.3.6.1.4.1.9.10.24.1.1.4.1.3
```
Чтобы завершать сессии по протоколу snmp необходимо включить на серверах доступа
```text
aaa session-mib disconnect
```
И использовать MIB: CISCO-AAA-SESSION-MIB. Вот шаблон, на основе которого можно написать нужный shell-скрипт:
```text
export MIBS=+CISCO-AAA-SESSION-MIB
#список активных пользователей
snmpwalk -v2c -c public $hostname casnUserId 
snmpwalk -v2c -c public $hostname casnUserId.$session_id
# вывод .. = "login"
# сброс сессии. используется community на запись
snmpset -v2c -c private $hostname casnDisconnect.$session_id i 1
# вывод .. = INTEGER: true(1)
```

## backup interface

Фича [backup interface](https://www.cisco.com/c/en/us/td/docs/routers/access/1900/software/configuration/guide/Software_Configuration/backup.html) может пригодится
в случае, когда ассиметрия недопустима, как например, при включении в маршрутизатор межсетевых экранов asa, где flow должен передаваться через один и тот же интерфейс в обе стороны.
Писал об этом [тут](http://prosto-seti.blogspot.com/2018/11/blog-post.html). Это был единственный раз когда данный функционал пригодился.
```text
interface GigabitEthernet0/0
 description asa1-eth1
 ip address 10.1.1.97 255.255.255.252
!
interface GigabitEthernet0/1
 description asa2-eth1
 ip address 10.1.1.109 255.255.255.252
!
interface GigabitEthernet0/2
 description r2-gi0/2
 backup interface GigabitEthernet0/1
 ip address 10.1.1.104 255.255.255.254
 ip ospf network point-to-point

r1#sh int desc  
Interface                      Status         Protocol Description
Em0/0                          admin down     down     
Gi0/0                          up             up       asa1-eth1
Gi0/1                          standby mode   down     asa2-eth1
Gi0/2                          up             up       r2-gi0/2
```

## object group

Можно использовать в acl, которые используются потом в PBR.

## acl for default route

Стандартный лист
```text
access-list 10 permit 0.0.0.0
```
Расширенный лист
```text
access-list 100 permit ip host 0.0.0.0 host 0.0.0.0
```
Префикс лист
```text
ip prefix-list 1 permit 0.0.0.0/0
```

## copp

Открытые порты, сессии:
```text
show control-plane host open-ports
```

Была уязвимость на маршрутизаторах, рекомендовали настраивать copp.
Политика с дропами, поэтому все указанное в acl как permit будет дропнуто.
В deny указываются ipsec пиры, если такие имеются. Разрешается обращения из приватной сети 10/8 практически без ограничений.
```text
access-list 198 deny   tcp 10.0.0.0 0.255.255.255 any
access-list 198 permit tcp any any eq telnet
access-list 198 permit tcp any any eq bgp
access-list 198 permit tcp any any eq 161

access-list 199 permit udp any any eq 18999
access-list 199 deny   udp 10.0.0.0 0.255.255.255 any
access-list 199 deny   udp host ipsec-peer-ip any eq non500-isakmp
access-list 199 permit udp any any eq non500-isakmp
access-list 199 deny   udp host host ipsec-peer-ip any eq isakmp
access-list 199 permit udp any any eq isakmp
access-list 199 permit udp any any eq ntp
access-list 199 permit udp any any eq snmp

class-map match-all undesirable-tcp
 match access-group 198
class-map match-all undesirable-udp
 match access-group 199

policy-map copp
 class undesirable-udp
  drop
 class undesirable-tcp
  drop

control-plane
 service-policy input copp
```
copp описан с примерами в rfc 2011 года [rfc6192](https://tools.ietf.org/html/rfc6192).


## high cpu usage

Для софтовых маршрутизаторов надо мониторить суммарный pps и сравнивать с графиком загрузки cpu.
При высокой загрузке надо смотреть `sh proc cpu history`, где фиксирутся на псевдографиках максимальные значения загрузки cpu.
При появлении override и ignored ошибок на интерфейсах менять маршрутизатор на более производительный, если дальнейшая оптимизация конфигурации невозможна.
Если загрузка cpu выросла внезапно, то проверить включен ли cef. Он может автоматически выключиться, если закончилась память, например.

## netflow для ios

```text
ip flow-cache timeout active 5
ip flow-export source Loopback0
ip flow-export version 5
ip flow-export destination <ip> <port>
ip flow-top-talkers
 top 50
 sort-by bytes
```
На всех интерфейсах с ip-адресами включить `ip flow ingress`. Проверить сбор данных:
```
show ip flow top-talkers
show ip cache flow
```
Есть разные коллекторы для netflow 5. Если ресурсы позвляют, то [elastiflow](https://github.com/robcowart/elastiflow).
Сложить netflow в tsdb (ifluxdb) это очень плохая идея и трата времени.

## neflow для ios xe

```
flow exporter DOH
 destination <ip>
 source Loopback0
 transport udp <port>
 export-protocol netflow-v5
!
!
flow monitor DOH
 exporter DOH
 cache timeout active 60
 record netflow-original
```
На всех интерфейсах с ip-адресами включить `ip flow monitor DOH input`. Проверить сбор данных:
```text
show flow monitor DOH statistics
show flow monitor DOH cache sort ipv4 tos format table
```
Без коллектора никуда.
