## E1 gateway

Написано в 2009. В зависимости от телефонной станции или имеющихся лицензий на нее, можно использовать два протокола сигнализации: QSIG и Euro ISDN (net5 в cisco).
Первый используется в корпоративных сетях и известен своими функциями, в частности, идентификацией.
Очень удобно, когда на дисплее вашего телефона помимо номера отображается ФИО звонящего сотрудника.
Второй используется для подключения к сетям общего пользования, а так же в корпоративных сетях, если по каким-то причинам вы не можете использовать QSIG.
Если один порт, например станция, настраивается для работы в режиме network, то маршрутизатор - user. И наоборот.

В рассматриваемом примере имеются две телефонных станции, каждая из которых подключается интерфейсом PRI к маршрутизаторам Cisco,
соединенных между собой некой сетью передачи данных.

Требования к маршрутизаторам следующие:

1. Модель из серии 2800, например, 2811. Модель из серии 3800, например, 3845. Используйте конфигуратор Cisco, чтобы составить спецификацию оборудования.
2. Программное обеспечение маршрутизатора. Если вы воспользовались конфигуратором, то наверняка включили программное обеспечение IOS IP VOICE/SP SERVICES, которое входит в состав т.н. bundle Voice (V_K9)
3. Если подключение производится только одним потоком, то достаточно карты VWIC2-1MFT-T1/E1 (1-Port 2nd Gen Multiflex Trunk Voice/WAN Int. Card - T1/E1)
4. Для преобразования голоса в ip и обратно необходимы карты PVDM2 с определенным количеством цифровых процессоров DSP, выполняющих кодирование/декодирование голоса. Есть карты на 8,16,32,48,64 процессора. Если прогнозируется, что не будет больше, чем 16 одновременных звонков, то можно обойтись PVDM2-16, входящий в состав СISCO2811-V_K9. Если необходимо использовать все таймслоты на карте станции, то - PVDM2-32 минимум. Если маршрутизатор комплектуется картами с FXS/FXO портами, то каждый порт будет использовать 1 DSP из имеющихся. Неоходимо это предусматривать.

Пусть имеем маршрутизатор CISCO3845-V_K9 в следующей комплектации (серийные номера удалены):
```text
router1#show inventory
NAME: "3845 chassis", DESCR: "3845 chassis"
PID: CISCO3845         , VID: V01 , SN: x

NAME: "c3845 Motherboard with Gigabit Ethernet on Slot 0", DESCR: "c3845 Motherboard with Gigabit Ethernet"
PID: CISCO3845-MB      , VID: V06 , SN: x

NAME: "VWIC2-1MFT-T1/E1 - 1-Port RJ-48 Multiflex Trunk - T1/E1 on Slot 0 SubSlot 0", DESCR: "VWIC2-1MFT-T1/E1 - 1-Port RJ-48 Multiflex Trunk - T1/E1"
PID: VWIC2-1MFT-T1/E1  , VID: V01 , SN: x

NAME: "One-Port Fast Ethernet High Speed WAN Interface Card on Slot 0 SubSlot 1", DESCR: "One-Port Fast Ethernet High Speed WAN Interface Card"
PID: HWIC-1FE          , VID: V01 , SN: x

NAME: "VWIC2-1MFT-T1/E1 - 1-Port RJ-48 Multiflex Trunk - T1/E1 on Slot 0 SubSlot 3", DESCR: "VWIC2-1MFT-T1/E1 - 1-Port RJ-48 Multiflex Trunk - T1/E1"
PID: VWIC2-1MFT-T1/E1  , VID: V01 , SN: x

NAME: "PVDMII DSP SIMM with four DSPs on Slot 0 SubSlot 4", DESCR: "PVDMII DSP SIMM with four DSPs"
PID: PVDM2-64          , VID: V01 , SN: x
```
Телефонная станция подключается в порт Slot 0 Subslot 3. Она не имеет QSIG лиценции, поэтому подключение производится по Euro ISDN.
К тому же станция может быть только как user. Для начала задается режим работы карты VWIC2-1MFT-T1/E1:
```text
card type e1 0 3
```
Разрешается использовать синхронизацию от этой карты:
```text
network-clock-participate wic 3
```
Можно использовать ее в качестве основной, если источников несколько:
```text
network-clock-select 1 E1 0/3/0
```
Определяется тип сигнализации Euro ISDN:
```text
isdn switch-type primary-net5
```
После этого появится контроллер E1. Договариваемся, что в E1 не будет использоваться CRC4 и будут использоваться все имеющиеся таймслоты:
```text
controller E1 0/3/0
 framing NO-CRC4
 pri-group timeslots 1-31
```
Автоматически создается интерфейс D-канала, в который надо внести изменения с учетом того, что маршрутизатор выполняет функции network:
```text
interface Serial0/3/0:15
 no ip address
 encapsulation hdlc
 isdn switch-type primary-net5
 isdn protocol-emulate network
 isdn incoming-voice voice
 isdn send-alerting
 isdn sending-complete
 no cdp enable
```
Производится проверка подключения, где MULTIPLE_FRAME_ESTABLISHED - все работает.
```text
router1#show controllers e1
E1 0/3/0 is up.
  Applique type is Channelized E1 - balanced
  Cablelength is Unknown
  No alarms detected.
  alarm-trigger is not set
  Version info Firmware: 20071011, FPGA: 13, spm_count = 0
  Framing is NO-CRC4, Line Code is HDB3, Clock Source is Line.
  Data in current interval (581 seconds elapsed):
     0 Line Code Violations, 0 Path Code Violations
     0 Slip Secs, 0 Fr Loss Secs, 0 Line Err Secs, 0 Degraded Mins
     0 Errored Secs, 0 Bursty Err Secs, 0 Severely Err Secs, 0 Unavail Secs
  Total Data (last 24 hours)
     0 Line Code Violations, 0 Path Code Violations,
     0 Slip Secs, 0 Fr Loss Secs, 0 Line Err Secs, 0 Degraded Mins,
     0 Errored Secs, 0 Bursty Err Secs, 0 Severely Err Secs, 0 Unavail Secs
router1#show isdn status
Global ISDN Switchtype = primary-net5
ISDN Serial0/3/0:15 interface
 ******* Network side configuration ******* 
 dsl 0, interface ISDN Switchtype = primary-net5
    Layer 1 Status:
 ACTIVE
    Layer 2 Status:
 TEI = 0, Ces = 1, SAPI = 0, State = MULTIPLE_FRAME_ESTABLISHED
    Layer 3 Status:
 2 Active Layer 3 Call(s)
 CCB:callid=C78C, sapi=0, ces=0, B-chan=2, calltype=VOICE
 CCB:callid=3335, sapi=0, ces=0, B-chan=9, calltype=VOICE
    Active dsl 0 CCBs = 2
    The Free Channel Mask:  0xFFFF7EFD
    Number of L2 Discards = 0, L2 Session ID = 1
    Total Allocated ISDN CCBs = 2
```
Теперь настраивается voip часть. Старая заметка без sip.
```text
voice rtp send-recv
!
voice service voip
 allow-connections h323 to h323
 h323
```
Пора приступить к номерному плану. Автоматически был создан порт `voice-port 0/3/0:15`.
Именно он будет использоваться в т.н. dial-peer - маршрутах звонков. Исходные данные:

* 86101XXX - номера на АТС, подключенной к маршрутизатору router1;
* 86102XXXX - номера на АТС, подключенной к маршрутизатору router2;
* 10.0.0.1 - ip-адрес на интерфейсе маршрутизатора router1, через который доступен router2;
* 10.0.0.2 - ip-адрес на интерфейсе маршрутизатора router2, через который доступен router1.

Номера dial-peer выбираются произвольно. Главное, чтобы были уникальными.
```text
dial-peer voice 86101 pots
 destination-pattern 86101...
 progress_ind alert enable 8
 direct-inward-dial
 port 0/3/0:15
 forward-digits all
!
dial-peer voice 86102 voip
 destination-pattern 86102....
 session target ipv4:10.0.0.2
 dtmf-relay rtp-nte
 no vad
 можно добавить настройки для факсов
 fax-relay ecm disable
 fax rate 14400
 fax nsf 000000
 fax protocol t38 nse ls-redundancy 0 hs-redundancy 0 fallback cisco
```
Следует учитывать, что в dial-peer voip, где используется session target ipv4 нельзя указать адрес источника.
Источник можно указать:

* для session target ras, когда вы настраиваете h323-шлюз и вводите соответствующие команды на интерфейсе для регистрации на гейткипере;
* для session target sip-server, когда вы настраиваете подключение маршрутизатора к SIP-серверу.

Расписывать конфигурацию router2 подробно не стану. Отмечу, что для связи со станцией используется сигнализация QSIG,
станция является network для маршрутизатора.
```text
card type e1 0 0
!
network-clock-participate wic 0
network-clock-select 1 E1 0/0/0
!
isdn switch-type primary-qsig
!
voice rtp send-recv
!
voice service voip
 qsig decode
 allow-connections h323 to h323
 h323
!
controller E1 0/0/0
 framing NO-CRC4
 pri-group timeslots 1-31
!
interface Serial0/0/0:15
 no ip address
 encapsulation hdlc
 isdn switch-type primary-qsig
 isdn incoming-voice voice
 isdn send-alerting
 isdn sending-complete
 no cdp enable
!
dial-peer voice 86102 pots
 destination-pattern 86102....
 progress_ind alert enable 8
 direct-inward-dial
 port 0/0/0:15
 forward-digits all
!
dial-peer voice 86101 voip
 destination-pattern 86101...
 session target ipv4:10.0.0.1
 dtmf-relay rtp-nte
 no vad
 fax-relay ecm disable
 fax rate 14400
 fax nsf 000000
 fax protocol t38 nse ls-redundancy 0 hs-redundancy 0 fallback cisco
```
Чтобы слушать привычные тоны КПВ, необходимо в voice-port настроить cptone:
```text
voice-port 0/0/0:15
 cptone RU
```
Когда что-то не работает, необходимо включать отладку и смотреть:
```text
router#debug isdn q931

router#debug voice dcapi inout
```
Для корректной работы сервисов желательно использовать один тип сигнализации.
Передача имен в данном примере была бы возможна, если бы router1 и ATC использовали QSIG, как и вторая станция.

## resource unavailable, unspecified

Симптомы проблемы:

* при звонках на телефонную станцию отрабатывает только сигнализация. Слышен звонок и сразу сброс.
* звонящим на станцию часто выдается "занято"
* в логах отладки (debug isdn q931) Resource unavailable, unspecified

Под ресурсами подразумевается DSP. Смотрим используемые каналы:
```text
voice-gw#show voice dsp detailed | in 0/0/0:15
C5510 001 01 g729r8        23.8.3 busy  idle      0  0 0/0/0:15  28    0    9300/9539
C5510 001 02 g729r8        23.8.3 busy  idle      0  0 0/0/0:15  03    0 122789/12573
C5510 001 03 g729r8        23.8.3 busy  idle      0  0 0/0/0:15  01    0    3411/5237
C5510 001 04 g711ulaw      23.8.3 busy  idle      0  0 0/0/0:15  27    0         0/13
C5510 001 06 g729r8        23.8.3 busy  idle      0  0 0/0/0:15  04    0 123864/12559
C5510 001 07 g729r8        23.8.3 busy  idle      0  0 0/0/0:15  24    0    2435/2516
C5510 002 01 g729r8        23.8.3 busy  idle      0  0 0/0/0:15  02    0    7396/7584
C5510 002 02 g729r8        23.8.3 busy  idle      0  0 0/0/0:15  05    0    7116/7308
C5510 002 03 g729r8        23.8.3 busy  idle      0  0 0/0/0:15  06    0 121376/12432
C5510 002 04 g729r8        23.8.3 busy  idle      0  0 0/0/0:15  07    0 120839/12376
C5510 002 05 g729r8        23.8.3 busy  idle      0  0 0/0/0:15  08    0 119916/12284
```
Смотрим текущие звонки. Количество звонков и "занятых" на порту 0/0/0:15 каналов существенно не совпадает.
```text
voice-gw#sh isdn active
--------------------------------------------------------------------------------
                                ISDN ACTIVE CALLS
--------------------------------------------------------------------------------
Call    Calling      Called       Remote  Seconds Seconds Seconds Charges
Type    Number       Number       Name    Used    Left    Idle    Units/Currency
--------------------------------------------------------------------------------
In         1234        2345                   187       0       0
Out        5678        3456                   147       0       0      0
Out   123456789       87654                   140       0       0      0
Out   987654321       45632                   100       0       0      0
In   8765432123        6432                    50       0       0
--------------------------------------------------------------------------------
```
В таком случае следует произвести выключение/включение голосового порта, чтобы освободить все каналы.
```text
voice-gw(config)#voice-port 0/0/0:15
voice-gw(config-voiceport)#shutdown
voice-gw(config-voiceport)#no shutdown
```

## E1 синхра

Для 2800 и 3800 серии, в частности:
```text
network-clock-participate wic 0
network-clock-select 1 e1 0/0/0
```
Для 7200 серии:
```text
frame-clock-select 1 E1 1/0
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
