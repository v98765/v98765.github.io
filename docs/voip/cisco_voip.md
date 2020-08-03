## cisco phone reset

Сброс Cisco IP Phone. Отключить питание телефона.Зажав  # на телефоне, включить питание и отпустить после индикации на трубе.
Нажать все кнопки поочередно 123456789*0#

Перезагрузка Cisco IP Phone/ Перейти в меню "Настройка", нажать ##*##


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

## fxs sip password

Есть некоторые особенности при использовании маршрутизатора Cisco в качестве FXS-шлюза. Например, в пароле не может быть символа "?".
```text
dial-peer voice 220 pots
 destination-pattern 220
 authentication username 220 password 7 02110C4218091C245E47060C16
 port 0/0/0
 forward-digits 0
```

## connection-reuse

При регистрации учетных данных со стороны маршрутизатора будет по умолчанию использоваться порт, отличный от 5060.
Несмотря на успешную регистрацию, использовать отладку debug ccsip не получится, т.к. данная команда,
как оказалось, может показывать сообщения принятые только на порт 5060. Эдакий tcpdump port 5060.
Предлагается решить эту задачу через connection-reuse
```text
sip-ua 
 registrar dns:pbx.a.b expires 3600
 sip-server dns:pbx.a.b
 connection-reuse
```

## white list, source-interface

Обязательно требуется указать список доверенных хостов. Так как INVITE придет от хоста pbx.a.b, необходимо учитывать возможную смену адреса ip-атс.
```text
voice service voip
 ip address trusted list
  ipv4 100.99.999.1
 fax protocol t38 version 0 ls-redundancy 0 hs-redundancy 0 fallback none
 sip
  bind control source-interface Loopback0
  bind media source-interface Loopback0
```

## cdr

Для выгрузки CDR на сервер биллинга не всегда подходит использование стандартного syslog-а.
В IOS предусмотрено использование внешнего или съемного хранилища:
```text
router(config)#gw-accounting file
router(config-gw-accounting-file)#primary ?
  ftp  ftp mode of file transfer
  ifs  Local file system, i.e.,flash/ mem slot device name such as flash: slot0:
```
Такой метод называется File Accounting. Метод File Accounting обеспечивает сбор CDR записей в файлы CSV формата
(поля разделяются запятыми) и запись этих файлов на внутреннюю флэш память или на внешний FTP сервер.
CDR записи формируются для каждой ветви вызова, выполняемого через голосовые шлюзы Cisco. Для CDR записей в CSV формате применяются следующие правила:

* Каждая CDR запись имеет свой номер, который находится в начале записи. Поля, не содержащие данных, включаются как пустые поля.
* Двенадцать полей являются общими и используются для сбора функционально-зависимой информации. Для основного вызова CDR запись генерируется с базовой информацией о вызове в функциональной части полей. Поля являются статическими относительно своих позиций, однако, назначение VSA* полей определяется типом функции.
* CDR записи генерируются для каждой используемой функции. Например, если при выполнении вызова был выполнена передача вызова, то будут сгенерированы 2-е CDR записи: - CDR запись для основного этапа вызова; - CDR запись для этапа передачи вызова.

При настройке этого метода получения CDR информации определяются основное и дополнительное устройства для хранения информации.
В случае если передача на основное устройство по какой-то причине прерывается, шлюз пытается восстановить связь с основным устройством заданное число раз,
и если это не удается, то переключается на дополнительное устройство.
Пользователь может затем вручную переключить вывод CDR на основное устройство при восстановления его работы.
В случае, когда и дополнительное устройство прерывают свою работу, процесс учета вызовов останавливается и система фиксирует ошибку.
При этом новые CDR записи будут потеряны до тех пор, пока одно из устройств записи не восстановит работоспособности и вы вручную не сделаете перезагрузку.
Шлюз временно хранит информацию о выполненных вызовах в буфере памяти перед тем, как она будет записана в заданный файл.
Информация добавляется в CDR файл после истечения заданного временного периода или когда буфер памяти переполняется.
Шлюз закрывает CDR файл и создает новый после истечения заданного временного интервала или этот процесс может быть инициирован вручную.
Другие опции позволяют выбрать специфические параметры, которые будут фиксироваться в CDR записях.
CDR записи могут генерироваться в одном из следующих форматов:

* Подробный формат;
* Компактный формат;
* Определяемый пользователем формат.

В минимальной конфигурации это выглядит примерно так:
```text
gw-accounting syslog
gw-accounting file
 primary ftp 1.2.3.4/voip-cdr.txt username cdr password cdr
 secondary ftp 4.3.2.1/voip-cdr.txt username cdr password cdr
 maximum cdrflush-timer 15
 cdr-format compact
```
Хотя и выбран компактный формат для файлов, но не обольщайтесь.
По документации в этом формате выгружаются всего 23 поля (0-22), а их там больше сотни!
Имеется также возможность подстроить выдачу CDR полей под собственные потребности (создать пользовательский формат).
Для этого необходимо создать шаблон, который представляет собой текстовый файл, с перечисленными в нем именами требуемых полей.
Только эти CDR поля будут записываться в CDR файл.
Данные из маршрутизатора можно выгрузить и принудительно:
```text
router#file-acct flush ?
  with-close     File accounting flush pending accounting to file,and close file
  without-close  File accounting flush pending accounting to file
```
`file-acct flush with-close` - Сбрасывает данные из буфера в файл с закрытием файла.

Настройки маршрутизатора с присоединением к оператору связи по Е1 следующие:
```text
gw-accounting file
 primary ftp 999.99.9.1/ username cdr password cdr
 maximum cdrflush-timer 15
```
На ftp-сервере, где разрешена запись пользователю cdr, по-умолчанию создается "скрытый" (ls -la) файл вида .имя_хоста_дата_время.
Файл периодически дописывается, если есть о чем писать. Новый создается раз в сутки относительно времени настройки аккаунтинга.
```text
VOIP-C2901(config-gw-accounting-file)#maximum fileclose-timer ?
  <60-1440>  in minutes, default is 1440min
```
Нет желания использовать компактный режим, так как в детализированном виде есть время соединения в секундах.
Все поля разделены "," и идут ровно в том порядке, в котором они указаны в документации. 
ужны поля, выделенные ниже в примере: unix_time, leg-type, username, clid, dnis, h323-setup-time, override-session-time.
Поле leg-type имеет несколько значений и это удобно использовать, так как на каждую "ногу" создается отдельная запись.
Интересует только стык с оператором - "1", а именно вторая запись о звонке с id "80CD1398 50739150 854F5801 AC117AEF".
```text
1426682736,788957,0,2,"80CD1398 50739150 854F5801 AC117AEF","","","19:45:04.169 KRSK Wed Mar 18 2015","","19:45:11.829 KRSK Wed Mar 18 2015","19:45:36.849 KRSK Wed Mar 18 2015","","","answer",0,"",594,93528,1413,226080,"662","662","089830000000","","","","","","","","","","","","","","","","","","","","","","","","","","","","","","","","","","","","","","","","","","","","",25,"","","0","999.99.9.100","","","","","","","","","","","","","","","","","","","","","","ton:0,npi:0,#:089830000000","ton:0,npi:0,pi:0,si:1,#:662","","","","","","","","","","","","Unknown","","","cisco","","","TWC","03/18/2015 19:45:04.168","662","089830000000",0,700847,80CD1398 50739150 854F5801 AC117AEF,C09DD,"","","","","","","",""
1426682737,788958,0,1,"80CD1398 50739150 854F5801 AC117AEF","","","19:45:04.525 KRSK Wed Mar 18 2015","19:45:08.415 KRSK Wed Mar 18 2015","19:45:11.815 KRSK Wed Mar 18 2015","19:45:37.185 KRSK Wed Mar 18 2015","","","originate",0,"",1413,237384,598,94168,"662","3912000000","89830000000","","","","","","","","","","","","","","","","","","","","","","","","","","","","","","","","","","","","","","","","","","","","",25,"Tariff:Unknown","","0","","","","","","","","","","","","","0/1:1","","","","","","","","","","ton:0,npi:0,#:089830000000","ton:0,npi:0,pi:0,si:1,#:662","","","","ton:0,npi:0,#:89830000000","ton:0,npi:0,pi:0,si:1,#:3912000000","","","","","","","Unknown","","","","","","TWC","03/18/2015 19:45:04.172","3912000000","89830000000",0,700848,80CD1398 50739150 854F5801 AC117AEF,C09DE,"","","","","","","",""
```
Мне нужны данные только об исходящих в телефонную сеть оператора звонках, которые будут складываться в базу sqlite:
```shell
bash$ awk -F"," '{if ($4=='1' && $14~'originate' && $68>0 && $22~'3912000') {print "insert into cisco values ("$1","$21","$22","$23","$8","$68");"} }' имя_файла_cdr > OUT.sql
bash$ sqlite3 ciscocdr.db < OUT.sql
```
Пример работы с базой:
```shell
bash$ sqlite3 ciscocdr.db 
SQLite version 3.7.9 2011-11-01 00:52:41
Enter ".help" for instructions
Enter SQL statements terminated with a ";"
sqlite> .schema cisco
CREATE TABLE cisco ( unix_time integer, username , clid , dnis , setuptime , sessiontime integer );
sqlite> select * from cisco where sessiontime>300 and dnis like '8%' and setuptime like '%Mar 18%' order by sessiontime desc;
```

## CME

Это вообще 2010 год.

Для начала необходимо в 2800 или 3800 серию "залить" соответствующий IOS. Это может быть 12.4Т voice или advipservices.
PVDM требуется только при подключении к телефонной станции потоком E1, либо при организации на cisco ip-телефонах hardware конференций,
т.е. конференций, где участников больше чем 3.
Если маршрутизатор не подключается к телефонной станции, к нему планируются подключаться еще и телефонные SIP ip-шлюзы с FXS-портами,
то необходимо разрешить устанавливать соединения между ними.
Предполагается, что для локального использования будет достаточно трехзначной нумерации и со всеми другими маршрутизаторами
передача голоса будет осуществляться по H323-протоколу
```text
voice rtp send-recv
!
voice service voip 
 allow-connections h323 to h323
 allow-connections h323 to sip
 allow-connections sip to h323
 allow-connections sip to sip
 supplementary-service h450.12
 fax protocol t38 nse ls-redundancy 0 hs-redundancy 0 fallback cisco
 h323
 sip
```
Отсутствие строк allow-connections ** to ** в конфигурационном файле означает, что взаимодействие voip-устройств возможно только через dial-peer pots.

Пусть имеем некий интерфейс:
```text
interface FastEthernet0/1.100
 description LAN
 encapsulation dot1Q 100
 ip address 10.10.9.254 255.255.255.0

interface FastEthernet0/1.110
 description Voice LAN
 encapsulation dot1Q 110
 ip address 10.10.10.254 255.255.255.0
 h323-gateway voip bind srcaddr 10.10.10.254
```
Отсутствие лубкека и наличие желания фиксировать адрес, с которого будет устанавливаться rtp-сессия с другими маршрутизаторами,
подразумевает наличие строки h323-gateway voip bind srcaddr в интерфейсе.

Настраивается сервис телефонии для ip-телефонов Сisco.
```text
telephony-service
 ip source-address 10.10.10.254 port 2000
 calling-number initiator
 timeouts interdigit 3
 system message MyCompany
 cnf-file location flash:
 time-zone 42
 time-format 24
 date-format dd-mm-yy
 dialplan-pattern 1 ... extension-length 3
 max-conferences 6 gain -6
 call-forward pattern .T
 call-forward system redirecting-expanded
 transfer-system full-consult
 transfer-pattern .T
```
В настройках указан часовой пояс, который никак не связан с настройками часового пояса в маршрутизаторе.
Текущее время телефон получает только при загрузке конфигурации и самостоятельно никак не синхронизируется.
Поэтому важно синхронизировать маршрутизатор и не забывать осуществлять руками в сервисе телефонии переход на летнее/зимнее время.
Понятно, что смена часового пояса должна в итоге сопровождаться командой:
```text
router(config)#telephony-service 
router(config-telephony)#reset all
```
Разные модели маршрутизаторов поддерживают разное количество телефонов.
Таким образом, в зависимости от модели, оперативной памяти и еще чего-нибудь, задается количество телефонов и DN (что-то вроде профиля для номера).
```text
telephony-service
 max-ephones 30
 max-dn 100
```
Далее создаются телефонные номера. Например, хочу создать 10 номеров и чтобы все они раздались автоматически при подключении телефона.
```text
ephone-dn  10  dual-line
 number 210
!
!
ephone-dn  11  dual-line
 number 211
!
!
ephone-dn  12  dual-line
 number 212
!
!
ephone-dn  13  dual-line
 number 213
!
!
ephone-dn  14  dual-line
 number 214
!
!
ephone-dn  15  dual-line
 number 215
!
!
ephone-dn  16  dual-line
 number 216
!
!
ephone-dn  17  dual-line
 number 217
!
!
ephone-dn  18  dual-line
 number 218
!
!
ephone-dn  19  dual-line
 number 219

telephony-service
 auto assign 10 to 19
```
Настраиваются DHCP пулы, который будет раздавать адреса рабочим станциям и телефонам. Наличие опции 150 укажет им с какого адреса они будут скачивать конфигурационный файл в формате xml по протоколу tftp. В качестве tftp-сервера выступает сам маршрутизатор.
Часть адресов резервируется для голосовых шлюзов, которые должны иметь статические адреса.
```text
ip dhcp excluded-address 10.10.10.1 10.10.10.50

ip dhcp pool LAN
   network 10.10.9.0 255.255.255.0
   default-router 10.10.9.254 
   dns-server 10.0.0.1 10.0.1.1
   domain-name mycompany.com
   netbios-node-type h-node

ip dhcp pool VoiceLAN
   network 10.10.10.0 255.255.255.0
   default-router 10.10.10.254 
   dns-server 10.0.0.1 10.0.1.1 
   option 150 ip 10.10.10.254
```
Предполагается, что использоваться будет только кодек G729, который поддерживается всеми телефонами Cisco, в том числе и софтфоном Cisco Communicator. Поэтому после регистрации телефонов в каждом ephone прописывается:
```text
ephone  1
..
 codec g729r8
..
```
Если будут использоваться различные сервисы на телефонах, например конференции, то необходимо включить sccp.
```text
sccp local FastEthernet0/1.110
sccp
```

Существуют два типа конференций:

1. Ad-Hoc, когда конференция создается с телефона cisco путем нажатия кнопки Confrn. Без настройки конференций (conference hardware) на маршрутизаторе максимальное число участников такой конференции равно трем.
2. Meetme, когда конференция создается с телефона cisco путем дозвона на предварительно созданный номер на маршрутизаторе. После чего все остальные участники с любого телефона могут звонить на тот же номер.

Сейчас будет рассматриваться только Ad-Hoc конференция. 
Считается, что один DSP поддерживает 8 конференций по 8 участников при использовании кодека G711 и 2 конференции по 8
участников при использовании кодека G729. Так же допускается настроить конференцию на 16 участников.
Мне пока неясно, как использовать больше чем один DSP. Пока все настраивается так, что больше чем один не может использоваться.
Соответственно, получаю лимит на 2 по 8.

В маршрутизаторе вставлен PVDM2, который необходимо настроить, включив dspfarm:
```text
router(config)#voice-card 0
router(config-voicecard)#?
Voice-card configuration commands:
  codec         Manage codec configuration parameters for voice card
  default       Set a command to its defaults
  dsp           Manage DSP configuration for the voice card
  dspfarm       Enable dspFarm feature for this voice card (command will be deprecated in future releases, use the new command 'dsp tdm pooling')
  exit          Exit from voice card configuration mode
  local-bypass  Enable TDM hairpinning
  no            Negate a command or set its defaults

router(config-voicecard)#dspfarm ?
  

router(config-voicecard)#dsp ?   
  allocation  dsp allocation scheme
  services    Manage DSP services configuration for the voice card
  tdm         Manage TDM configuration for the DSP on voice card

router(config-voicecard)#dsp services ?
  dspfarm  Enable dspfarm services on the Voice Card


router(config-voicecard)#dsp services dspfarm ?
  

router(config-voicecard)#dsp services dspfarm 
router(config-voicecard)#
```
В профиле dspfarm будут указываться тоны, которые проигрываются при входе и выходе из конференции:
```text
voice class custom-cptone jointone
 dualtone conference
  frequency 1200 1200
  cadence 150 50 150 50
!
voice class custom-cptone leavetone
 dualtone conference
  frequency 900 900
  cadence 150 50 150 50
```
Далее настраивается сам профиль. Обратите внимание, что в конфигурационном файле профиль по-умолчанию выключен:
```text
!
dspfarm profile 1 conference  
 codec g711ulaw
 codec g711alaw
 codec g729ar8
 codec g729abr8
 codec g729r8
 codec g729br8
 maximum sessions 2
 conference-join custom-cptone jointone
 conference-leave custom-cptone leavetone
 associate application SCCP
 shutdown
!
```
Чтобы задать профиль для sccp необходим идентификатор.
```text
sccp ccm 10.10.10.254 identifier 1 version 7.0 

sccp ccm group 1
 bind interface FastEthernet0/1.110
 associate ccm 1 priority 1
 associate profile 1 register mtp002414de5631
 keepalive retries 16
 keepalive timeout 10
```
Здесь mtp002414de5631 случайный набор понятных букв для указания их в сервисе телефонии. Напомню, что ранее sccp был уже включен:
```text
sccp local FastEthernet0/1.110
sccp
```
Правим сервис телефонии, чтобы указать там используемый профиль sccp и включить конференции:
```text
telephony-service
 sdspfarm units 1
 sdspfarm tag 1 mtp002414de5631
 conference hardware
```
Включаем профиль dspfarm
```text
router(config)#dspfarm profile 1 conference  
router(config-dspfarm-profile)#no sh
router(config-dspfarm-profile)#
```
Смотрим, что получилось:
```text
router#sh sccp all
SCCP Admin State: UP
Gateway Local Interface: FastEthernet0/1.110
        IPv4 Address: 10.10.10.254
        Port Number: 2000
IP Precedence: 5
User Masked Codec list: None
Call Manager: 10.10.10.254, Port Number: 2000
  Priority: N/A, Version: 7.0, Identifier: 1
  Trustpoint: N/A

SCCP application services(dspfarm profiles) are not enabled 
or haven't associated to SCCP CCM group

CCM Group Identifier: 1
 Description: None
 Binded Interface: FastEthernet0/1.110
        IPv4 Address: 10.10.10.254
 Associated CCM Id: 1, Priority in this CCM Group: 1
 Associated Profile: 1, Registration Name: mtp002414de5631
 Registration Retries: 3, Registration Timeout: 10 sec
 Keepalive Retries: 16, Keepalive Timeout: 10 sec
 CCM Connect Retries: 3, CCM Connect Interval: 10 sec
 Switchover Method: GRACEFUL, Switchback Method: GRACEFUL_GUARD
 Switchback Interval: 10 sec, Switchback Timeout: 7200 sec
 Signaling DSCP value: cs3, Audio DSCP value: ef
[skip]

router#sh dspfarm all
Dspfarm Profile Configuration

 Profile ID = 1, Service = CONFERENCING, Resource ID = 1  
 Profile Description :  
 Profile Service Mode : Non Secure 
 Profile Admin State : UP 
 Profile Operation State : RESOURCE ALLOCATED 
 Application : SCCP   Status : NOT ASSOCIATED 
 Resource Provider : FLEX_DSPRM   Status : UP 
[skip]
```
Есть ошибки, при наличии которых создать конференции нельзя.
Документация описывает причину, по которым такой статус возможен - не включен созданный профиль dspfarm.
Но тут необходимо обратить внимание на sccp, где, как оказывается, тоже важна последовательность команд.
```text
sccp local FastEthernet0/1.110
sccp ccm 10.10.10.254 identifier 1 version 7.0 
sccp
```
Любое изменение (определили идентификатор в данном случае) sccp должно сопровождаться выключением/включением sccp:
```text
router(config)#no sccp
router(config)#sccp
router(config)#^Z
router#sh dspfarm all
Dspfarm Profile Configuration

 Profile ID = 1, Service = CONFERENCING, Resource ID = 1  
 Profile Description :  
 Profile Service Mode : Non Secure 
 Profile Admin State : UP 
 Profile Operation State : ACTIVE 
 Application : SCCP   Status : ASSOCIATED 
 Resource Provider : FLEX_DSPRM   Status : UP
[skip] 
```
Для Ad-Hoc конференций необходимо создать DN-ки. Количество штук определяется максимальным количеством участников.
Если DN-ки dual-line , то необходимо вполовину меньше.
Пусть будет на 32 участника 16 DN dual-line:
```text
ephone-dn  100  dual-line
 number A0001
 name conference
 conference ad-hoc
 preference 1
 no huntstop
!
!
ephone-dn  101  dual-line
 number A0001
 name conference
 conference ad-hoc
 preference 2
 no huntstop
!
!
ephone-dn  102  dual-line
 number A0001
 name conference
 conference ad-hoc
 preference 3
 no huntstop
!
!
ephone-dn  103  dual-line
 number A0001
 name conference
 conference ad-hoc
 preference 4
 no huntstop
!
!
ephone-dn  104  dual-line
 number A0001
 name conference
 conference ad-hoc
 preference 5
 no huntstop
!
!
ephone-dn  105  dual-line
 number A0001
 name conference
 conference ad-hoc
 preference 6
 no huntstop
!
!
ephone-dn  106  dual-line
 number A0001
 name conference
 conference ad-hoc
 preference 7
 no huntstop
!
!
ephone-dn  107  dual-line
 number A0001
 name conference
 conference ad-hoc
 preference 8
 no huntstop
!
!
ephone-dn  108  dual-line
 number A0001
 name conference
 conference ad-hoc
 preference 9
 no huntstop
!
!
ephone-dn  109  dual-line
 number A0001
 name conference
 conference ad-hoc
 preference 9
 no huntstop
!
!
ephone-dn  110  dual-line
 number A0001
 name conference
 conference ad-hoc
 preference 9
 no huntstop
!
!
ephone-dn  111  dual-line
 number A0001
 name conference
 conference ad-hoc
 preference 9
 no huntstop
!
!
ephone-dn  112  dual-line
 number A0001
 name conference
 conference ad-hoc
 preference 9
 no huntstop
!
ephone-dn  113  dual-line
 number A0001
 name conference
 conference ad-hoc
 preference 9
 no huntstop
!
!
ephone-dn  114  dual-line
 number A0001
 name conference
 conference ad-hoc
 preference 9
 no huntstop
!
!
ephone-dn  115  dual-line
 number A0001
 name conference
 conference ad-hoc
 preference 9
 huntstop
!
```
Интерестно, что указывается number A0001. Непривычно. Для meetme необходимо указывать нормальный рабочий номер.
По настройкам - все.
Проверить работу конференции можно с зарегистрированного на CME телефона. Например, с Cisco IP Communicator'а, который поддерживает G711 и G729 кодеки.

Смотрим тестовую конференцию из трех участников:
```text
router#sh sccp all
SCCP Admin State: UP
Gateway Local Interface: FastEthernet0/1.110
        IPv4 Address: 10.10.10.254
        Port Number: 2000
IP Precedence: 5
User Masked Codec list: None
Call Manager: 10.10.10.254, Port Number: 2000
  Priority: N/A, Version: 7.0, Identifier: 100
  Trustpoint: N/A

Conferencing Oper State: ACTIVE - Cause Code: NONE
Active Call Manager: 10.10.10.254, Port Number: 2000
TCP Link Status: CONNECTED, Profile Identifier: 1
Reported Max Streams: 16, Reported Max OOS Streams: 0
Supported Codec: g711ulaw, Maximum Packetization Period: 30
Supported Codec: g711alaw, Maximum Packetization Period: 30
Supported Codec: g729ar8, Maximum Packetization Period: 60
Supported Codec: g729abr8, Maximum Packetization Period: 60
Supported Codec: g729r8, Maximum Packetization Period: 60
Supported Codec: g729br8, Maximum Packetization Period: 60
Supported Codec: rfc2833 dtmf, Maximum Packetization Period: 30
Supported Codec: rfc2833 pass-thru, Maximum Packetization Period: 30
Supported Codec: inband-dtmf to rfc2833 conversion, Maximum Packetization Period: 30


SCCP Application Service(s) Statistics:

Profile Identifier: 1, Service Type: Conferencing
TCP packets rx 1835, tx 1819
Unsupported pkts rx 0, Unrecognized pkts rx 0
Register tx 1, successful 1, rejected 0, failed 0
Unregister tx 0, successful 0
KeepAlive tx 1808, successful 1808, failed 0
OpenReceiveChannel rx 6, successful 6, failed 0
CloseReceiveChannel rx 3, successful 0, failed 0
StartMediaTransmission rx 6, successful 3, failed 0
StopMediaTransmission rx 3, successful 0, failed 0
PortReq rx 0
PortRes tx 0, successful 0, failed 0
PortClose rx 0
QosListen rx 0
QosPath rx 0
QosTeardown rx 0, send 0, recv 0, sendrecv 0
QosResvNotify tx 0, send 0, recv 0, sendrecv 0
QosErrorNotify tx 0, send 0, recv 0, sendrecv 0
   err0 0, err1 0, err2 0, err3 0, err4 0, err5 0,
   err6 0, err7 0, err8 0, err9 0, err10 0, err11 0,
   err12 0
QosModify rx 0, send 0, recv 0, sendrecv 0
UpdateDscp rx 0
Reset rx 0, successful 0, failed 0
MediaStreamingFailure rx 6
MediaStreamingFailure tx 0
Switchover 0, Switchback 0

CCM Group Identifier: 1
 Description: None
 Binded Interface: FastEthernet0/1.110
        IPv4 Address: 10.10.10.254
 Associated CCM Id: 1, Priority in this CCM Group: 1
 Associated Profile: 1, Registration Name: mtp002414de5631
 Registration Retries: 3, Registration Timeout: 10 sec
 Keepalive Retries: 16, Keepalive Timeout: 10 sec
 CCM Connect Retries: 3, CCM Connect Interval: 10 sec
 Switchover Method: GRACEFUL, Switchback Method: GRACEFUL_GUARD
 Switchback Interval: 10 sec, Switchback Timeout: 7200 sec
 Signaling DSCP value: cs3, Audio DSCP value: ef

sess_id    conn_id      stype mode     codec   sport rport ripaddr

3221356546 65542        conf  sendrecv g729    17742 2000  10.10.10.254
3221356546 65541        conf  sendrecv g729    17264 2000  10.10.10.254
3221356546 65540        conf  sendrecv g729    17148 2000  10.10.10.254

Total number of active session(s) 1, and connection(s) 3

sess_id    conn_id    st ev orc_ts     orca_ts    crc_ts     stmt_ts    spmt_ts    dis_ind_ts preq_ts    pc_ts      qosl_ts    qost_r_ts  qosp_ts    qost_s_ts 

3221356546 65542      -  -  61980160   61980200   0          61980212   0          0          0          0          0          0          0          0         
3221356546 65541      -  -  61978476   61978492   0          61978504   0          0          0          0          0          0          0          0         
3221356546 65540      -  -  61978388   61978440   0          61978452   0          0          0          0          0          0          0          0         

Total number of active session(s) 1, and connection(s) 3

          
bridge-info(bid, cid) - Normal bridge information(Bridge id, Calleg id)
mmbridge-info(bid, cid) - Mixed mode bridge information(Bridge id, Calleg id)

sess_id    conn_id    call-id    codec   pkt-period dtmf_method    type        bridge-info(bid, cid)   mmbridge-info(bid, cid) srtp_cryptosuite          dscp      

3221356546 -          1249       N/A     N/A        none              confmsp   All RTPSPI Callegs      All MM-MSP Callegs      N/A                       N/A       

3221356546 65542      1255       g729    20         none              rtpspi    (390,1249)               N/A                     N/A                       0         

3221356546 65541      1252       g729    20         rfc2833_pthru     rtpspi    (388,1249)               N/A                     N/A                       0         

3221356546 65540      1248       g729    20         rfc2833_pthru     rtpspi    (386,1249)               N/A                     N/A                       0         


Total number of active session(s) 1, connection(s) 3, and callegs 4

SCCP Application Service(s) Statistics Summary:
Total Conferencing Sessions: 1, Connections: 3
Total Transcoding Sessions: 0, Connections: 0
Total MTP Sessions: 0, Connections: 0
Total ALG-Phone Sessions: 0, Connections: 0
Total BRI-Phone Sessions: 0, Connections: 0
Total SCCP Sessions: 1, Connections: 3


sess_id    conn_id    rsvp_id    dir  local ip       :port  remote ip      :port 


 Total active sessions 1, connections 3, rsvp sessions 0
Statistic                  Count
-------------------------  -----------
Send queue enqueue error   0
Socket send error          1
Msgs discarded upon error  0
```

Есть комментарии в выводе
```text
router#sh dspfarm all
Dspfarm Profile Configuration

 Profile ID = 1, Service = CONFERENCING, Resource ID = 1  
 Profile Description :  
 Profile Service Mode : Non Secure 
 Profile Admin State : UP 
 Profile Operation State : ACTIVE 
 Application : SCCP   Status : ASSOCIATED 
 Resource Provider : FLEX_DSPRM   Status : UP 
 Number of Resource Configured : 2 
 Number of Resource Available : 2         # как увеличить это число?
 Codec Configuration 
 Codec : g711ulaw, Maximum Packetization Period : 30 , Transcoder: Not Required 
 Codec : g711alaw, Maximum Packetization Period : 30 , Transcoder: Not Required 
 Codec : g729ar8, Maximum Packetization Period : 60 , Transcoder: Not Required 
 Codec : g729abr8, Maximum Packetization Period : 60 , Transcoder: Not Required 
 Codec : g729r8, Maximum Packetization Period : 60 , Transcoder: Not Required 
 Codec : g729br8, Maximum Packetization Period : 60 , Transcoder: Not Required


SLOT DSP VERSION  STATUS CHNL USE   TYPE    RSC_ID BRIDGE_ID PKTS_TXED PKTS_RXED

0    1   24.3.2   UP     1    USED  conf    1      0x182       6165      6174     # используется 01 канал (см. ниже)
0    1   24.3.2   UP     1    USED  conf    1      0x184       6165      6172     
0    1   24.3.2   UP     1    USED  conf    1      0x186       6086      6007     
0    1   24.3.2   UP     N/A  FREE  conf   1      -         -         -        

Total number of DSPFARM DSP channel(s) 2





router#sh voice dsp voice 
 
DSP  DSP                DSPWARE CURR  BOOT                         PAK     TX/RX
TYPE NUM CH CODEC       VERSION STATE STATE   RST AI VOICEPORT TS ABORT  PACK COUNT
==== === == ======== ========== ===== ======= === == ========= == ===== ============
edsp 0001 01 g729r8    0.1 IDLE  50/0/1.1     
edsp 0002 02 g729r8 p  0.1 IDLE  50/0/1.2     
[skip]  
edsp 0010 01 g729r8    0.1 busy  50/0/40.1 # телефон с которого организуется конференция   
[skip]
edsp 0030 01 g729r8    0.1 busy  50/0/100.1  # 
edsp 0031 02 g729r8    0.1 busy  50/0/100.2  # используются 3 линии Ad-Hoc
edsp 0032 01 g729r8    0.1 busy  50/0/101.1  # 
edsp 0033 02 g729r8 p  0.1 IDLE  50/0/101.2   
edsp 0034 01 g729r8 p  0.1 IDLE  50/0/102.1   
edsp 0035 02 g729r8 p  0.1 IDLE  50/0/102.2   
[skip]

 
----------------------------FLEX VOICE CARD 0 ------------------------------
                           *DSP VOICE CHANNELS*

CURR STATE : (busy)inuse (b-out)busy out (bpend)busyout pending 
LEGEND     : (bad)bad    (shut)shutdown  (dpend)download pending

DSP   DSP                 DSPWARE CURR  BOOT                         PAK   TX/RX
TYPE  NUM CH CODEC        VERSION STATE STATE   RST AI VOICEPORT TS ABRT PACK COUNT
===== === == ========= ========== ===== ======= === == ========= == ==== ============
# в списке отсутствует канал 01
C5510 001 02 None          24.3.2 idle  idle      0  0                 0          0/0 
C5510 001 03 None          24.3.2 idle  idle      0  0                 0          0/0
C5510 001 04 None          24.3.2 idle  idle      0  0                 0          0/0
C5510 001 05 None          24.3.2 idle  idle      0  0                 0          0/0
C5510 001 06 None          24.3.2 idle  idle      0  0                 0          0/0
C5510 001 07 None          24.3.2 idle  idle      0  0                 0          0/0
C5510 001 08 None          24.3.2 idle  idle      0  0                 0          0/0
C5510 001 09 None          24.3.2 idle  idle      0  0                 0          0/0
C5510 001 10 None          24.3.2 idle  idle      0  0                 0          0/0
C5510 001 11 None          24.3.2 idle  idle      0  0                 0          0/0
C5510 001 12 None          24.3.2 idle  idle      0  0                 0          0/0
C5510 001 13 None          24.3.2 idle  idle      0  0                 0          0/0
C5510 001 14 None          24.3.2 idle  idle      0  0                 0          0/0
C5510 001 15 None          24.3.2 idle  idle      0  0                 0          0/0
C5510 001 16 None          24.3.2 idle  idle      0  0                 0          0/0
------------------------END OF FLEX VOICE CARD 0 ----------------------------
```

CUCME позволяет реализовать что-то наподобие сервера конференций.
Функционал называется Meetme, когда можно позвонить на определенный номер и попасть в конференцию, в отличии от Ad-hoc конференций,
когда до каждого участника вручную необходимо дозвониться.

Настройки маршрутизатора остаются прежними. Добавляется только ephone-dn:
```text
ephone-dn  111  dual-line
 number 2222
 conference meetme
 preference 2
 no huntstop
!
!
ephone-dn  112  dual-line
 number 2222
 conference meetme
 preference 3
 no huntstop
!
!
ephone-dn  113  dual-line
 number 2222
 conference meetme
 preference 4
 no huntstop
!
!
ephone-dn  114  dual-line
 number 2222
 conference meetme
 preference 5
 no huntstop
!
!
ephone-dn  115  dual-line
 number 2222
 conference meetme
 preference 6
 no huntstop
!
!
ephone-dn  116  dual-line
 number 2222
 conference meetme
 preference 7
 no huntstop
!
!
ephone-dn  117  dual-line
 number 2222
 conference meetme
 preference 8
 no huntstop
!
!
ephone-dn  118  dual-line
 number 2222
 conference meetme
 preference 9
 no huntstop
!
!
ephone-dn  119  dual-line
 number 2222
 conference meetme
 preference 10
 no huntstop
!
!
ephone-dn  120  dual-line
 number 2222
 conference meetme
 preference 10
 no huntstop
!
!
ephone-dn  121  dual-line
 number 2222
 conference meetme
 preference 10
 no huntstop
!
!
ephone-dn  122  dual-line
 number 2222
 conference meetme
 preference 10
 no huntstop
!
!
ephone-dn  123  dual-line
 number 2222
 conference meetme
 preference 10
 no huntstop
!
!
ephone-dn  124  dual-line
 number 2222
 conference meetme
 preference 10
```
Количество dn определяет число участников, как и в Ad-hoc конференциях. Понятно, используются ресурсы DSP.
Конференция создается только при наличии строки conference meetme хотя бы в одной DN.
И только с телефона или ip-коммуникатора Cisco имеющего программные клавиши.
Их необходимо создать и применить к телефону, который будет создавать конференцию. В примере ниже такое проделано для ip-коммуникатора:
```text
ephone-template  1
 softkeys hold  Newcall Resume Select Join
 softkeys idle  Cfwdall ConfList Dnd Gpickup HLog Join Login Newcall Pickup Redial RmLstC
 softkeys seized  Redial Pickup Gpickup HLog Meetme Endcall
 softkeys connected  Acct ConfList Confrn Endcall Flash HLog Hold Join Park RmLstC Select Trnsfer
ephone  89
 device-security-mode none
 mac-address 0025.B311.E0D0
 ephone-template 1
 type CIPC
 button  1:10
```
Последовательность создания конференции:

* снимаем "трубку" на телефоне.
* выбираем программную клавишу Meetme (для русифицированных это Конф№). Она активна, если есть DN c conference meetme. Услышите тоновый сигнал по нажатию.
* набираем номер конференции 2222.

Все остальные могут звонить на 2222 без ограничений как на обычный телефонный номер.
Подключение и отключение участников будет сопровождаться join и leave тоновыми сигналами.
К недостатку можно отнести невозможность управления участниками конференции из-за ошибок русификации. Все меню с участниками в кракозябрах.
К ограничениям можно отнести возможность создания конференции только с аппаратов, использующих sccp сигнализацию.
Cisco IP-Phone с прошивками SIP, а так же телефонные шлюзы ATA не могут создавать конференции Meetme.

## COR list

Это наиболее востребованная функция, когда применяют COR-листы на голосовых шлюзах cisco.
Необходимо четко понимать, когда они работают и в каком случае не работают в принципе.
По-умолчанию все звонки с неопределенным номером, не попавшие ни в какой входящий dial-peer, используют системный dial-peer 0 , который нельзя изменить.
Такой вызов будет игнорировать все corlist'ы на любом исходящем dial-peer'е. Потому что:

COR List on Incoming dial-peer | COR List on Outgoing dial-peer	| Result | Reason
---|---|---|---
No COR. | No COR. | Call succeeds. | COR is not in the picture.
No COR.	| COR list applied for outgoing calls. | Call succeeds.	| The incoming dial-peer, by default, has the highest COR priority when no COR is applied. Therefore, if you apply no COR for an incoming call leg to a dial-peer, then this dial-peer can make calls out of any other dial-peer, irrespective of the COR configuration on the outgoing dial-peer.
The COR list applied for incoming calls. | No COR. | Call succeeds.	| The outgoing dial-peer, by default, has the lowest priority. Since there are some COR configurations for incoming calls on the incoming/originating dial-peer, it is a super set of the outgoing call COR configurations on the outgoing/terminating dial-peer.
The COR list applied for incoming calls (super set of COR lists applied for outgoing calls on the outgoing dial-peer). | The COR list applied for outgoing calls (subset of COR lists applied for incoming calls on the incoming dial-peer.)	| Call succeeds. | The COR list for incoming calls on the incoming dial-peer is a super set of COR lists for outgoing calls on the outgoing dial-peer
The COR list applied for incoming calls (subset of COR lists applied for outgoing calls on the outgoing dial-peer).	| The COR list applied for outgoing calls (super set of COR lists applied for incoming calls on the incoming dial-peer).	| Call cannot be completed using this outgoing dial-peer. | COR lists for incoming calls on the incoming dial-peer are not a super set of COR lists for outgoing calls on the outgoing dial-peer.

Пусть имеется голосовой шлюз Сisco. Для corlist'а непринципиально применение на voip или pots dial-peer'ах.
Для начала необходимо определиться с ограничениями:

1. Ограничить "межгород", набор которого производится через 98
2. Разрешить выход в город, набор которого производится через 9
3. Разрешить выход на корпоративные номера, начинающиеся на 2 и 3.

Имеем ip-телефоны Cisco в нумерации 211x, одному из которых (2111) надо закрыть "межгород" и удаленный шлюз с номерами 311х, где только номеру 3111 необходимо открыть "межгород".
Понадобится всего два corlist'а. Будем проще и назовем их Limit и NoLimit, так как название не имеет никакого значения вообще.
```text
dial-peer cor custom
 name Limit
 name NoLimit
!
!
dial-peer cor list Limit
 member Limit
!
dial-peer cor list NoLimit
 member Limit
 member NoLimit
```
Исходя из правила, что при отсутствии входящих corlist'ов на dial-peer'ах все вызовы будут беспрепятственно осуществляться, "вешаем" исходящие corlist'ы
```text
dial-peer voice 98 voip
 corlist outgoing NoLimit
 description MejGorod
 destination-pattern 98T
 session target ipv4:10.0.0.100
 no vad
dial-peer voice 9 voip
 corlist outgoing Limit
 description Gorod
 destination-pattern 9[012345679]T
 session target ipv4:10.0.0.100
 no vad
dial-peer voice 2000 voip
 corlist outgoing Limit
 description Corporate
 destination-pattern [23]...
 session target ipv4:10.0.0.100
 no vad
```
Теперь необходимо применить входящие corlist'ы. Помним, что отсутствие листа равноценно максимальному приоритету, а несовпадение листов хотя бы одним признаком приведет к отказу в выполнении вызова.
```text
ephone-dn 10
 number 2110
!
ephone-dn 11
 number 2111
 corlist incoming Limit
!
ephone-dn 12
 number 2112
!
ephone-dn 13
 number 2113
!
ephone-dn 14
 number 2114
!
ephone-dn 15
 number 2115
!
ephone-dn 16
 number 2116
!
ephone-dn 17
 number 2117
!
ephone-dn 18
 number 2118
!
ephone-dn 19
 number 2119
!
dial-peer voice 3110 voip
 corlist incoming Limit
 description Remote Gateway
 huntstop
 destination-pattern 311.
 session target ipv4:10.100.0.100
 no vad
!
dial-peer voice 3111 voip
 description Happy User on Remote Gateway
 huntstop
 destination-pattern 3111
 session target ipv4:10.100.0.100
 no vad
```
На ephone-dn и dial-peer'ах выше отсутствуют любые "corlist outgoing", а это значит, что 2111 и 311. 
могут беспрепятственно звонить на все номера (211х и 311х) на данном шлюзе. Потому что так задумано и описано в таблице.
