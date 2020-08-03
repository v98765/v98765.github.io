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
