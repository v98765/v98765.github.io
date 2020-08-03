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
