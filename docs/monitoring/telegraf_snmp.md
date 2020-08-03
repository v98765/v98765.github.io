Информацию по протоколу snmp можно получать разную: счетчики интерфейсов, статус sla, описание интерфейсов,
состав оборудования из entity-mib, счетчики политик qos. Все это можно опрашивать с разными интервалами,
например, данные по sla нужны раз в минуту, счетчики интерфейсов и политик раз в пять минут, инвентаризацию раз в сутки.
Разные интервалы подразумевают разные конфигурационные файлы для плагина snmp, т.к. интервал указывается для плагина,
а не для job'а, как в prometheus/vmagent. Опрашивать все с одним интервалом не получится, т.к. нет необходимости хранить одни и те же данные,
например, строковые значения инвертацизации, описаний интерфейсов, ip-адреса, arp'ы и т.п.
Еще опрос такого количества таблиц занимает приличное время и в минутный интервал просто не успеть.
Это легко проверить, запустив в тестовом режиме telegraf.
```shell
$ telegraf -config test.cfg -test
```
Отсюда появляется ограничение, что интервал опроса для конкретного конфигурационного файла не может быть меньше,
чем время опроса всех агентов в т.ч. с учетом таймаутов из-за недоступности.
Если все агенты не будут опрошены до наступления следующего интервала опроса, то данные,
полученные от всех агентов будут потеряны с фиксацией события в логах "input "inputs.snmp" did not complete within its interval".
Т.е. приходится как-то оптимизировать запрашиваемую информацию, опрашивать какие-то таблицы менее часто, но с бОльшим таймаутом по времени.
Ниже будут примеры, но это больше для информации, истории.
Теперь пользуюсь vmagent'ом и snmp_exporter'ом для сбора каунтеров и конфигурация для него
немного отличается в сторону упрощения и уменьшения количества лейблов.
Telegraf оставил для записи строк в clickhouse. Дальше в примерах будут некоторые штуки, которые пришли с опытом:

* Если какой-то field будет использоваться как tag, то имя  в конфиге будет отличаться. Это связано с использованием ранних версий influxdb, где с наименованием в самой базе случалась каная-то каша. Так появлялись ifName и ifName_1 и я уже не помню что из этого что. Опять же в запросах меньше путаницы будет, когда ifName - это field, а Name - это tag. Позже стал использовать fieldpass , в котором не указывал все field, используемые в качестве tag.
* Вместо имени oid'а используется сам oid, а его имя для удобства указано в комментарии выше. Старая версия telegraf'а при старте запускала в shell'е snmptranslate и чем больше было конфигурационных файлов, тем выше была загрузка cpu. Рестарт был очень неудобен и позже алгоритм немного поменяли на однократный запуск snmptranslate. Все таблицы все равно проверяются в мибах и опросить таблицу совсем без миба не получится.

Пример для cisco ip sla, где не опрашивается вся таблица rttMonLatestRttOperTable (закоменнирован oid), а только два значения из нее.
Тэг для замены индекса берется из другой таблицы rttMonCtrlAdmintable. На оборудовании cisco настроен следующий sla:
```text
ip sla 1
 icmp-echo 10.4.3.1 source-ip 10.4.3.2
 tos 96
 threshold 200
 owner IPSName-vlanid@AddrA-AddrB
ip sla schedule 1 life forever start-time now
```
Конфигурация плагина с фильтрацией по тегу, чтобы не записывать все тестовые, временные пробы:
```text
[[inputs.snmp]]
 interval = "1m"
 agents = [ "cisco" ]
 version = 2
 community = "public"

#CISCO-RTTMON-MIB::rttMonCtrlAdminOwner.5 = STRING: "IPSName-vlanid@AddrA-AddrB"

  [[inputs.snmp.table]]
    name = "rttsla"
    #oid = "CISCO-RTTMON-MIB::rttMonLatestRttOperTable"

    [[inputs.snmp.table.field]]
      name = "Descr"
      #oid = "CISCO-RTTMON-MIB::rttMonCtrlAdminOwner"
      oid = "1.3.6.1.4.1.9.9.42.1.2.1.1.2"
      is_tag = true

    [[inputs.snmp.table.field]]
      name = "icmpResults"
      #oid = "CISCO-RTTMON-MIB::rttMonLatestRttOperSense"
      oid = "1.3.6.1.4.1.9.9.42.1.2.10.1.2"

    [[inputs.snmp.table.field]]
      name = "rttAvg"
      #oid = "CISCO-RTTMON-MIB::rttMonLatestRttOperCompletionTime"
      oid = "1.3.6.1.4.1.9.9.42.1.2.10.1.1"

 [inputs.snmp.tagpass]
   Descr = [ "*@*" ]
```
Более сложным оказался опрос аналогичных проберов на коммутаторах huawei s5720ei,
где в качестве одного из индексов используется растущий счетчик для истории проверок.
Чтобы его удалить, понадобилось ограничить oid. Для удобства, используются такие же имена метрик:
```text
[[inputs.snmp]]
 interval = "1m"
 agents = [ "huawei" ]
 version = 2
 community = "public"

# for  huawei s5720-ei
#NQA-MIB::nqaAdminCtrlTag."1"."1" = STRING: IPSName-vlanid@AddrA-AddrB
#.1.3.6.1.4.1.2011.5.25.111.2.1.1.3.1.49.1.49 = STRING: IPSName-vlanid@AddrA-AddrB
#index is 1.49.1.49. oid_index_length = 4
#NQA-MIB::nqaResultsCompletions."1"."1".186177.1 = INTEGER: success(1)
#s5720-ei has a history index

  [[inputs.snmp.table]]
    name = "rttsla"
    #oid = "NQA-MIB::nqaAdminCtrlTable"

    [[inputs.snmp.table.field]]
      name = "Descr"
      #oid = "NQA-MIB::nqaAdminCtrlTag"
      oid = "1.3.6.1.4.1.2011.5.25.111.2.1.1.3"
      is_tag = true

    [[inputs.snmp.table.field]]
      name = "icmpResults"
      #NQA-MIB::nqaResultsCompletions
      oid = "1.3.6.1.4.1.2011.5.25.111.4.1.1.3"
      oid_index_length = 4


    [[inputs.snmp.table.field]]
      name = "rttAvg"
      #NQA-MIB::nqaResultsRttAvg
      oid = "1.3.6.1.4.1.2011.5.25.111.4.1.1.26"
      oid_index_length = 4

 [inputs.snmp.tagpass]
   Descr = [ "*@*" ]
```
Вот закончились небольшие таблицы, которые можно опросить за короткое время с интервалом опроса одна минута.
Далее конфигурация для маршрутизатора cisco, где есть ряд ограничений,
чтобы случайно в базу influxdb не записать лишнего и не было такого, когда одно устройство пишет больше полей в базу, чем другое.
Эта конфигурация не запишет в базу значения mtu, ifalias и прочее. Только то, что определено в fieldpass.
Метрики с тегами логических интерфейсов будут удалены. Сейчас я удаляю по лейблу ifType, а не по имени интерфейса.
Вопреки примерам, не следует опрашивать таблицы ifTable и ifTableX целиком и записывать в influxdb все подряд: дескрипшены, таймстампы и прочую ненужную информацию.
Необходимо сосредоточится на скорости выполения запроса, хотя тут тоже много лишнего для маршрутизатора.
Жаль было расставаться с функционалом, который позволил использовать именно такую замену индексов в таблице cbqos,
потому что этот миб самый невменяемый, а chain lookup в snmp_exporter'е сделали относительно недавно.
```text
[[inputs.snmp]]
 interval = "5m"
 agents = [ "cisco_router" ]
 version = 2
 community = "public"
 fieldpass = [ "cbQos*", "bgpPeerState", "bgpPeerFsmEstablishedTransitions", "Duplex" , "Speed" , "ifAdminStatus" , "ifOperStatus" , "ifInDiscards" , "ifInErrors" , "ifOutDiscards" , "ifOutErrors" , "ifHCInOctets" , "ifHCInUcastPkts" , "ifHCInMulticastPkts" , "ifHCInBroadcastPkts" , "ifHCOutOctets" , "ifHCOutUcastPkts" , "ifHCOutMulticastPkts" , "ifHCOutBroadcastPkts" ]
 tagexclude = [ "ifIndex" ]

 [[inputs.snmp.table]]
  name = "interface32"
  #oid = "IF-MIB::ifTable"

   [[inputs.snmp.table.field]]
    name = "Name"
    #IF-MIB::ifName
    oid = "1.3.6.1.2.1.31.1.1.1.1"
    is_tag = true

   [[inputs.snmp.table.field]]
    name = "Speed"
    #IF-MIB::ifHighSpeed
    oid = "1.3.6.1.2.1.31.1.1.1.15"

   [[inputs.snmp.table.field]]
    name = "Duplex"
    #EtherLike-MIB::dot3StatsDuplexStatus
    oid = "1.3.6.1.2.1.10.7.2.1.19"

   [[inputs.snmp.table.field]]
    name = "ifAdminStatus"
    #IF-MIB::ifAdminStatus
    oid = "1.3.6.1.2.1.2.2.1.7"

   [[inputs.snmp.table.field]]
    name = "ifOperStatus"
    #IF-MIB::ifOperStatus
    oid = "1.3.6.1.2.1.2.2.1.8"

   [[inputs.snmp.table.field]]
    name = "ifInDiscards"
    #IF-MIB::ifInDiscards
    oid = "1.3.6.1.2.1.2.2.1.13"

   [[inputs.snmp.table.field]]
    name = "ifInErrors"
    #IF-MIB::ifInErrors
    oid = "1.3.6.1.2.1.2.2.1.14"

   [[inputs.snmp.table.field]]
    name = "ifOutDiscards"
    #IF-MIB::ifOutDiscards
    oid = "1.3.6.1.2.1.2.2.1.19"

   [[inputs.snmp.table.field]]
    name = "ifOutErrors"
    #IF-MIB::ifOutErrors
    oid = "1.3.6.1.2.1.2.2.1.20"

 [[inputs.snmp.table]]
  name = "interface64"
  #oid = "IF-MIB::ifXTable"

   [[inputs.snmp.table.field]]
    name = "Name"
    #IF-MIB::ifName
    oid = "1.3.6.1.2.1.31.1.1.1.1"
    is_tag = true

   [[inputs.snmp.table.field]]
     name = "Speed"
     #IF-MIB::ifHighSpeed
     oid = "1.3.6.1.2.1.31.1.1.1.15"

   [[inputs.snmp.table.field]]
     name = "ifHCInOctets"
     #IF-MIB::ifHCInOctets
     oid = "1.3.6.1.2.1.31.1.1.1.6"

   [[inputs.snmp.table.field]]
     name = "ifHCInUcastPkts"
     #IF-MIB::ifHCInUcastPkts
     oid = "1.3.6.1.2.1.31.1.1.1.7"

   [[inputs.snmp.table.field]]
     name = "ifHCInMulticastPkts"
     #IF-MIB::ifHCInMulticastPkts
     oid = "1.3.6.1.2.1.31.1.1.1.8"

   [[inputs.snmp.table.field]]
     name = "ifHCInBroadcastPkts"
     #IF-MIB::ifHCInBroadcastPkts
     oid = "1.3.6.1.2.1.31.1.1.1.9"

   [[inputs.snmp.table.field]]
     name = "ifHCOutOctets"
     #IF-MIB::ifHCOutOctets
     oid = "1.3.6.1.2.1.31.1.1.1.10"

   [[inputs.snmp.table.field]]
     name = "ifHCOutUcastPkts"
     #IF-MIB::ifHCOutUcastPkts
     oid = "1.3.6.1.2.1.31.1.1.1.11"

   [[inputs.snmp.table.field]]
     name = "ifHCOutMulticastPkts"
     #IF-MIB::ifHCOutMulticastPkts
     oid = "1.3.6.1.2.1.31.1.1.1.12"

   [[inputs.snmp.table.field]]
     name = "ifHCOutBroadcastPkts"
     #IF-MIB::ifHCOutBroadcastPkts
     oid = "1.3.6.1.2.1.31.1.1.1.13"

  [[inputs.snmp.table]]
    name = "bgpPeer"
    #oid = "1.3.6.1.2.1.15.3"

    [[inputs.snmp.table.field]]
      name = "bgpPeerIdentifier"
      #BGP4-MIB::bgpPeerIdentifier"
      oid = "1.3.6.1.2.1.15.3.1.1"
      is_tag = true

    [[inputs.snmp.table.field]]
      name = "bgpPeerState"
      #BGP4-MIB::bgpPeerState"
      oid = "1.3.6.1.2.1.15.3.1.2"

    [[inputs.snmp.table.field]]
      name = "bgpPeerFsmEstablishedTransitions"
      #BGP4-MIB::bgpPeerFsmEstablishedTransitions
      oid = "1.3.6.1.2.1.15.3.1.15"

 [[inputs.snmp.table]]
   name ="cbQos"
   oid = "CISCO-CLASS-BASED-QOS-MIB::cbQosCMStatsTable"

   [[inputs.snmp.table.field]]
     name = "ConfigIndex"
     #oid = "CISCO-CLASS-BASED-QOS-MIB::cbQosConfigIndex"
     oid = "1.3.6.1.4.1.9.9.166.1.5.1.1.2"
     is_tag = true

   [[inputs.snmp.table.field]]
     name = "ParentObjectsIndex"
     #oid = "CISCO-CLASS-BASED-QOS-MIB::cbQosParentObjectsIndex"
     oid = "1.3.6.1.4.1.9.9.166.1.5.1.1.4"
     is_tag = true

 [inputs.snmp.tagdrop]
   Name = [ "SONET*", "Vo*", "Nu0", "E1*", "Lo*", "Vl*", "CH*", "*:*", "St*", "VL*" ]
```
Раз в 4 часа неспеша c getnext, а не getbulk собрать текст из ifalias. Сейчас подобное пишу в clickhouse, чтобы вывести описание интерфейсов в графане.
```text
[[inputs.snmp]]
 interval = "4h"
 agents = [ "device" ]
 version = 2
 max_repetitions = 1
 retries = 1
 community = "public"
 fieldpass = [ "Alias", "Name" ]

 [[inputs.snmp.table]]
  name = "ifalias"

   [[inputs.snmp.table.field]]
    name = "Ifindex"
    #IF-MIB::ifIndex
    oid = "1.3.6.1.2.1.2.2.1.1"
    is_tag = true

   [[inputs.snmp.table.field]]
    name = "Name"
    #IF-MIB::ifName
    oid = "1.3.6.1.2.1.31.1.1.1.1"

   [[inputs.snmp.table.field]]
    name = "Alias"
    #IF-MIB::ifAlias
    oid = "1.3.6.1.2.1.31.1.1.1.18"
```
Этот конфигурационный файл для сбора информации о модели, серийниках, когда устройство поддерживает entity-mib.
Juniper не поддерживает. Тут обратить внимание на index_as_tag, чтобы не потерять информацию о устройствах в стеке, точках доступа,
зарегистрированных на контроллере. Т.к. в строке вендора может быть все что угодно (dlink,d-link и тп), то для каждого вендора решил использовать тег MfgName.
Дополнительная проверка с ModelName в inputs.snmp.tagpass для фильтрации избыточных данных о портах.
```text
[[inputs.snmp]]
 interval = "24h"
 agents = [ "dlink" ]
 version = 2
 timeout = "2m"
 community = "public"
 fieldpass = [ "entPhysicalDescr" , "entPhysicalName" , "entPhysicalFirmwareRev" , "entPhysicalHardwareRev" , "entPhysicalSerialNum" , "entPhysicalSoftwareRev" ]

 [[inputs.snmp.field]]
  name = "location"
  #SNMPv2-MIB::sysLocation.0
  oid = "1.3.6.1.2.1.1.6.0"
  is_tag = true

 [[inputs.snmp.field]]
  name = "contact"
  #SNMPv2-MIB::sysContact.0
  oid = "1.3.6.1.2.1.1.4.0"
  is_tag = true

 [[inputs.snmp.table]]
  name = "inventory_entity"
  inherit_tags = [ "location", "contact" ]
  #oid = "ENTITY-MIB::entPhysicalTable"
  index_as_tag = true # stack, access points
   [[inputs.snmp.table.field]]
    name = "ModelName"
    #ENTITY-MIB::entPhysicalModelName
    oid = "1.3.6.1.2.1.47.1.1.1.1.13"
    is_tag = true

   [[inputs.snmp.table.field]]
    name = "Class"
    #ENTITY-MIB::entPhysicalClass
    oid = "1.3.6.1.2.1.47.1.1.1.1.5"
    is_tag = true

   [[inputs.snmp.table.field]]
    name = "entPhysicalName"
    #ENTITY-MIB::entPhysicalName
    oid = "1.3.6.1.2.1.47.1.1.1.1.7"

   [[inputs.snmp.table.field]]
    name = "entPhysicalDescr"
    #ENTITY-MIB::entPhysicalDescr
    oid = "1.3.6.1.2.1.47.1.1.1.1.2"

   [[inputs.snmp.table.field]]
    name = "entPhysicalHardwareRev"
    #ENTITY-MIB::entPhysicalHardwareRev
    oid = "1.3.6.1.2.1.47.1.1.1.1.8"

   [[inputs.snmp.table.field]]
    name = "entPhysicalFirmwareRev"
    #ENTITY-MIB::entPhysicalFirmwareRev
    oid = "1.3.6.1.2.1.47.1.1.1.1.9"

   [[inputs.snmp.table.field]]
    name = "entPhysicalSoftwareRev"
    #ENTITY-MIB::entPhysicalSoftwareRev
    oid = "1.3.6.1.2.1.47.1.1.1.1.10"

   [[inputs.snmp.table.field]]
    name = "entPhysicalSerialNum"
    #ENTITY-MIB::entPhysicalSerialNum
    oid = "1.3.6.1.2.1.47.1.1.1.1.11"

 [inputs.snmp.tags]
  MfgName = "dlink"
  obj = "object_id"

 [inputs.snmp.tagpass]
  ModelName = [ "*" ]
```
Информация в полученных строках бывает кривая, с пробелами в конце, символами, чем грешат sfp и wic модули cisco.
Поэтому в telegraf.d дополнительный конфигурационный файл proc_trim.conf. К сожалению, процессоры в телеграфе никак нельзя тестировать.
```text
[[processors.strings]]
 namepass = ["inventory_entity"]
 [[processors.strings.trim]]
  tag = "ModelName"
 [[processors.strings.trim]]
  field = "entPhysicalDescr"
 [[processors.strings.trim]]
  field = "entPhysicalName"
 [[processors.strings.trim]]
  field = "entPhysicalFirmwareRev"
 [[processors.strings.trim]]
  field = "entPhysicalHardwareRev"
 [[processors.strings.trim]]
  field = "entPhysicalSerialNum"
 [[processors.strings.trim]]
  field = "entPhysicalSoftwareRev"
 [[processors.strings.lowercase]]
  tag = "contact"
 [[processors.strings.lowercase]]
  tag = "location"
```
Вроде бы можно было и сделать обработку, но нет. Запускаешь тест и получаешь:
```text
inventory_entity,Class=9,MfgName=cisco,ModelName=WIC-2T\ \ \ \ \ \ \ \ \ \ \ \ ������
```
Чем больше устройств опрашивается, тем больше LimitNOFILE требуется указать в /lib/systemd/system/telegraf.service.
Необходимо увеличивать metric_buffer_limit в настройках агента в /etc/telegraf/telegraf.conf, когда в логах появляются сообщения о дропах.
Размер этого кэша влияет на время хранения данных в момент недоступности базы.
Беда, когда нужен мониторинг не метрик, которые в каждый интервал возвращают значение, а событий, когда временной ряд метрики прерывается.
Тут показателен мониторинг видеокодеков polycom, которые по snmp отдают данные только в момент сеанса видеоконференцсвязи.
Странным образом все перечисленные oid'ы ниже - это строки. Тестовый запуск без преобразования строки в число приводит к тому,
что field отсутствует, если строка пустая.
Но преобразование делает из пустой строки 0i и все значения пишутся в базу, временной ряд метрики не прерывается.
С генератором для snmp_exporter'а так не получится.
```text
[[inputs.snmp]]
  interval = "1m"
  agents = [ "polycom" ]
  version = 2
  community = "public"

  name = "polycom"
  [[inputs.snmp.field]]
   name = "PercentPacketLoss"
   oid = "1.3.6.1.4.1.2684.1.1.21.0"
   conversion = "int"

  [[inputs.snmp.field]]
   name = "Jitter"
   oid = "1.3.6.1.4.1.2684.1.1.22.0"
   conversion = "int"

  [[inputs.snmp.field]]
   name = "Latency"
   oid = "1.3.6.1.4.1.2684.1.1.23.0"
   conversion = "int"
```
