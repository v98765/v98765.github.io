[vmagent](https://victoriametrics.github.io/) периодически опрашивает через [snmp_exporter](https://github.com/prometheus/snmp_exporter)
устройства по протоколу snmp и записывает полученные метрики в базу VictoriaMetrics.
После 0.20.0 заявлены изменения и snmp_exporter возможно пеередет [github.com/prometheus-community/snmp](https://github.com/prometheus-community/snmp)

## Генератор конфигурации

В версии 0.20 пока еще нужен генератор для формирования snmp.yml (конфигурационный файл snmp_exporter) из yaml файла generator.yml
Из исходиков не получилось собрать в этот раз, поэтому вариант с докером [https://hub.docker.com/r/prom/snmp-generator](https://hub.docker.com/r/prom/snmp-generator).
В примере ниже podman в ubuntu 20.10
Старый генератор искал мибы в домашнем каталоге .snmp/mibs, поэтому архив со всеми необходимыми мибами там.
Сам файл generator.yml в .snmp/
```sh
cd .snmp/
docker run -it -v "${PWD}:/opt/" prom/snmp-generator generate
```
Если завершится без ошибок, то сгенериться конфигурационный файл snmp.yml
```sh
$ ls
generator.yml  mibs  snmp.yml
```

## generator.yml

Для простоты все устройства, описываемые одним модулем имеют одну и ту же версию snmp и коммутити public.
В противном случае необходимо писать множество модулей и далее в конфигурации vmagent'а для каждого такого модуля делать отдельный job

Для примера простой модуль для коммутатора в generator.yml:
```yaml
modules:

  devicename:
    retries: 1
    walk:
      - 1.3.6.1.2.1.1.3         #sysUpTime
      - 1.3.6.1.2.1.2.2.1.8     #IF-MIB::ifOperStatus
      - 1.3.6.1.2.1.2.2.1.14    #IF-MIB::ifInErrors
      - 1.3.6.1.2.1.2.2.1.19    #IF-MIB::ifOutDiscards
      - 1.3.6.1.2.1.31.1.1.1.6  #IF-MIB::ifHCInOctets
      - 1.3.6.1.2.1.31.1.1.1.7  #IF-MIB::ifHCInUcastPkts
      - 1.3.6.1.2.1.31.1.1.1.10 #IF-MIB::ifHCOutOctets
      - 1.3.6.1.2.1.31.1.1.1.11 #IF-MIB::ifHCOutUcastPkts
      - 1.3.6.1.2.1.31.1.1.1.15 #IF-MIB::ifHighSpeed

    lookups:
      - source_indexes: [ifIndex]
        lookup: 1.3.6.1.2.1.31.1.1.1.1  #ifName
        drop_source_indexes: true
      - source_indexes: [ifIndex]
        lookup: 1.3.6.1.2.1.2.2.1.3     #ifType
```

Используется только то, что нужно и что можно (строки нельзя, но ссылка внизу). Не требуется собирать вообще все и тем более обе таблицы iftable и ifxtable целиком.
Таблица с метриками:

что | зачем
---|---
sysUpTime | для определения времени работы оборудования
ifOperStatus | статус интерфейса up/down
ifInErrors | общий счетчик входящих ошибок
ifOutDiscards | дискарды на портах из-за speed mismatch, qos, и тп
ifHCInOctets | 64-битный счетчик входящих байт
ifHCInUcastPkts | 64-битный счетчик входящих юникастовых пакетов
ifHCOutOctets | 64-битный счетчик исходящих байт
ifHCOutUcastPkts | 64-битный счетчик исходящих юникастовых пакетов
ifHighSpeed | скорость интерфейса для расчета % утилизации

Таблица с label, которые дополнительно будут у каждой метрики предыдущей таблицы:

что | зачем
---|---
ifName | имя интерфейса из ifxtable вместо ifIndex, который удален `drop_source_indexes: true`
ifType | тип интерфейса для фильтрации метрик в vmagent'е, чтобы исключить сбор данных по логическим интерфейсам

Дропать индекс интерфейсов для некоторых маршрутизаторах cisco нельзя. Например, при опросе 7206VXR с модулями может возникнуть ошибка.

У snmp_exporter'а есть функционал overrides, который можеть дропать метрики. Это необходимо для удаления высококардинальных метрик, например, или мусора.
```yaml
  n3k:
    retries: 1
    walk:
      - 1.3.6.1.2.1.1.3             # sysUpTime
      - 1.3.6.1.2.1.2.2.1.7         # IF-MIB::ifAdminStatus
      - 1.3.6.1.2.1.2.2.1.8         # IF-MIB::ifOperStatus
      - 1.3.6.1.2.1.2.2.1.9         # IF-MIB::ifLastChange
      - 1.3.6.1.2.1.2.2.1.14        # IF-MIB::ifInErrors
      - 1.3.6.1.2.1.2.2.1.19        # IF-MIB::ifOutDiscards
      - 1.3.6.1.2.1.31.1.1.1.6      # IF-MIB::ifHCInOctets
      - 1.3.6.1.2.1.31.1.1.1.7      # IF-MIB::ifHCInUcastPkts
      - 1.3.6.1.2.1.31.1.1.1.10     # IF-MIB::ifHCOutOctets
      - 1.3.6.1.2.1.31.1.1.1.11     # IF-MIB::ifHCOutUcastPkts
      - 1.3.6.1.2.1.31.1.1.1.15     # IF-MIB::ifHighSpeed
      - 1.3.6.1.2.1.10.7.10         # EtherLike-MIB::dot3PauseTable
      - 1.3.6.1.2.1.15.3.1.2        # BGP4-MIB::bgpPeerState
      - 1.3.6.1.2.1.15.3.1.15       # BGP4-MIB::bgpPeerFsmEstablishedTransitions
      - 1.3.6.1.4.1.9.9.91.1.1.1.1.4    # CISCO-ENTITY-SENSOR-MIB::entSensorValue
      - 1.3.6.1.4.1.9.9.91.1.1.1.1.5    # CISCO-ENTITY-SENSOR-MIB::entSensorStatus
      - 1.3.6.1.4.1.9.9.109.1.1.1.1.7       # CISCO-PROCESS-MIB::cpmCPUTotal1minRev
      - 1.3.6.1.4.1.9.9.109.1.1.1.1.12      # CISCO-PROCESS-MIB::cpmCPUMemoryUsed
      - 1.3.6.1.4.1.9.9.109.1.1.1.1.13      # CISCO-PROCESS-MIB::cpmCPUMemoryFree
      - 1.3.6.1.4.1.9.9.117.1.1.2       # CISCO-ENTITY-FRU-CONTROL-MIB::cefcFRUPowerStatusTable
      - 1.3.6.1.4.1.9.9.117.1.2.1       # CISCO-ENTITY-FRU-CONTROL-MIB::cefcModuleTable
      - 1.3.6.1.4.1.9.9.117.1.4.1       # CISCO-ENTITY-FRU-CONTROL-MIB::cefcFanTrayStatusTable
      - 1.3.6.1.4.1.9.9.42.1.2.10.1.2   # CISCO-RTTMON-MIB::rttMonLatestRttOperSense

    lookups:
      - source_indexes: [ifIndex]
        lookup: 1.3.6.1.2.1.31.1.1.1.1  # ifName
        drop_source_indexes: true
      - source_indexes: [dot3StatsIndex]
        lookup: 1.3.6.1.2.1.31.1.1.1.1  # ifName
        drop_source_indexes: true
      - source_indexes: [ifIndex]
        lookup: 1.3.6.1.2.1.2.2.1.3     # ifType for eth=6
      - source_indexes: [entPhysicalIndex]
        drop_source_indexes: true
        lookup: 1.3.6.1.2.1.47.1.1.1.1.2    # entPhysicalDescr

    overrides:
      entSensorStatus:
        ignore: true
```

Есть возможность переписать строки в числа, иначе будет значение 1, а строковое значение, например, загрузка cpu или кол-во свободной памяти,
будет в label, что плохо скажется на производительности, т.к. увеличивается кардинальность метрик:
```yaml
  exos:
    retries: 1
    walk:
      - 1.3.6.1.2.1.1.3             # sysUpTime
      - 1.3.6.1.2.1.2.2.1.7         # IF-MIB::ifAdminStatus
      - 1.3.6.1.2.1.2.2.1.8         # IF-MIB::ifOperStatus
      - 1.3.6.1.2.1.2.2.1.9         # IF-MIB::ifLastChange
      - 1.3.6.1.2.1.2.2.1.14        # IF-MIB::ifInErrors
      - 1.3.6.1.2.1.2.2.1.19        # IF-MIB::ifOutDiscards
      - 1.3.6.1.4.1.1916.1.4.14.1.1   # EXTREME-PORT-MIB::extremePortCongDropPkts
      - 1.3.6.1.2.1.31.1.1.1.6      # IF-MIB::ifHCInOctets
      - 1.3.6.1.2.1.31.1.1.1.7      # IF-MIB::ifHCInUcastPkts
      - 1.3.6.1.2.1.31.1.1.1.10     # IF-MIB::ifHCOutOctets
      - 1.3.6.1.2.1.31.1.1.1.11     # IF-MIB::ifHCOutUcastPkts
      - 1.3.6.1.2.1.31.1.1.1.15     # IF-MIB::ifHighSpeed
      - 1.3.6.1.2.1.15.3.1.2        # BGP4-MIB::bgpPeerState
      - 1.3.6.1.2.1.15.3.1.15       # BGP4-MIB::bgpPeerFsmEstablishedTransitions
      - 1.3.6.1.4.1.1916.1.32.1.4.1.8   # EXTREME-SOFTWARE-MONITOR-MIB::extremeCpuMonitorSystemUtilization1min
      - 1.3.6.1.4.1.1916.1.32.2.2       # EXTREME-SOFTWARE-MONITOR-MIB::extremeMemoryMonitorSystemTable
      - 1.3.6.1.4.1.1916.1.1.1.27.1.2       # EXTREME-SYSTEM-MIB::extremePowerSupplyStatus
      - 1.3.6.1.4.1.1916.1.1.1.7        # EXTREME-SYSTEM-MIB::extremeOverTemperatureAlarm
      - 1.3.6.1.4.1.1916.1.1.1.8        # EXTREME-SYSTEM-MIB::extremeCurrentTemperature
      - 1.3.6.1.4.1.1916.1.1.1.9        # EXTREME-SYSTEM-MIB::extremeFanStatusTable

    lookups:
      - source_indexes: [ifIndex]
        lookup: 1.3.6.1.2.1.31.1.1.1.1  # ifName for ifmib and fcfemib
        drop_source_indexes: true
      - source_indexes: [ifIndex]
        lookup: 1.3.6.1.2.1.2.2.1.3     # ifType for eth=6
      - source_indexes: [entPhysicalIndex]
        drop_source_indexes: true
        lookup: 1.3.6.1.2.1.47.1.1.1.1.2    # entPhysicalDescr
    overrides:
      extremeCpuMonitorSystemUtilization1min:
        regex_extracts:
          '':
            - regex: '(.*)'
              value: '$1'
      extremeMemoryMonitorSystemTotal:
        regex_extracts:
          '':
            - regex: '(.*)'
              value: '$1'
      extremeMemoryMonitorSystemFree:
        regex_extracts:
          '':
            - regex: '(.*)'
              value: '$1'
      extremeMemoryMonitorSystemUsage:
        regex_extracts:
          '':
            - regex: '(.*)'
              value: '$1'
      extremeMemoryMonitorUserUsage:
        regex_extracts:
          '':
            - regex: '(.*)'
              value: '$1'
```

Есть возможность заменить несколько индексов на один. Например, было и стало
```text
# TYPE jnxOperatingTemp gauge
jnxOperatingTemp{jnxOperatingContentsIndex="1",jnxOperatingL1Index="0",jnxOperatingL2Index="0",jnxOperatingL3Index="0"} 0
jnxOperatingTemp{jnxOperatingContentsIndex="12",jnxOperatingL1Index="1",jnxOperatingL2Index="0",jnxOperatingL3Index="0"} 36
jnxOperatingTemp{jnxOperatingContentsIndex="2",jnxOperatingL1Index="1",jnxOperatingL2Index="0",jnxOperatingL3Index="0"} 29
jnxOperatingTemp{jnxOperatingContentsIndex="2",jnxOperatingL1Index="2",jnxOperatingL2Index="0",jnxOperatingL3Index="0"} 32
jnxOperatingTemp{jnxOperatingContentsIndex="4",jnxOperatingL1Index="1",jnxOperatingL2Index="1",jnxOperatingL3Index="0"} 0
jnxOperatingTemp{jnxOperatingContentsIndex="4",jnxOperatingL1Index="1",jnxOperatingL2Index="2",jnxOperatingL3Index="0"} 0
jnxOperatingTemp{jnxOperatingContentsIndex="4",jnxOperatingL1Index="2",jnxOperatingL2Index="1",jnxOperatingL3Index="0"} 0
jnxOperatingTemp{jnxOperatingContentsIndex="4",jnxOperatingL1Index="2",jnxOperatingL2Index="2",jnxOperatingL3Index="0"} 0
jnxOperatingTemp{jnxOperatingContentsIndex="4",jnxOperatingL1Index="3",jnxOperatingL2Index="1",jnxOperatingL3Index="0"} 0
jnxOperatingTemp{jnxOperatingContentsIndex="4",jnxOperatingL1Index="3",jnxOperatingL2Index="2",jnxOperatingL3Index="0"} 0
jnxOperatingTemp{jnxOperatingContentsIndex="7",jnxOperatingL1Index="1",jnxOperatingL2Index="0",jnxOperatingL3Index="0"} 0
jnxOperatingTemp{jnxOperatingContentsIndex="8",jnxOperatingL1Index="1",jnxOperatingL2Index="1",jnxOperatingL3Index="0"} 0
jnxOperatingTemp{jnxOperatingContentsIndex="8",jnxOperatingL1Index="1",jnxOperatingL2Index="2",jnxOperatingL3Index="0"} 0
jnxOperatingTemp{jnxOperatingContentsIndex="9",jnxOperatingL1Index="1",jnxOperatingL2Index="0",jnxOperatingL3Index="0"} 41


# TYPE jnxOperatingTemp gauge
jnxOperatingTemp{jnxOperatingDescr="CB 0"} 36
jnxOperatingTemp{jnxOperatingDescr="FPC: MPC @ 0/*/*"} 0
jnxOperatingTemp{jnxOperatingDescr="Fan Tray 0 Fan 0"} 0
jnxOperatingTemp{jnxOperatingDescr="Fan Tray 0 Fan 1"} 0
jnxOperatingTemp{jnxOperatingDescr="Fan Tray 1 Fan 0"} 0
jnxOperatingTemp{jnxOperatingDescr="Fan Tray 1 Fan 1"} 0
jnxOperatingTemp{jnxOperatingDescr="Fan Tray 2 Fan 0"} 0
jnxOperatingTemp{jnxOperatingDescr="Fan Tray 2 Fan 1"} 0
jnxOperatingTemp{jnxOperatingDescr="PEM 0"} 29
jnxOperatingTemp{jnxOperatingDescr="PEM 1"} 32
jnxOperatingTemp{jnxOperatingDescr="PIC: 4XQSFP28 PIC @ 0/0/*"} 0
jnxOperatingTemp{jnxOperatingDescr="PIC: 8XSFPP PIC @ 0/1/*"} 0
jnxOperatingTemp{jnxOperatingDescr="Routing Engine"} 41
jnxOperatingTemp{jnxOperatingDescr="midplane"} 0

```
Это достигается следующей конфигурацией
```yaml
  mx:
    retries: 1
    walk:
      - 1.3.6.1.2.1.1.3             # sysUpTime
      - 1.3.6.1.2.1.2.2.1.14        # IF-MIB::ifInErrors
      - 1.3.6.1.2.1.31.1.1.1.6      # IF-MIB::ifHCInOctets
      - 1.3.6.1.2.1.31.1.1.1.10     # IF-MIB::ifHCOutOctets
      - 1.3.6.1.2.1.15.3.1.2        # BGP4-MIB::bgpPeerState
      - 1.3.6.1.2.1.15.3.1.15       # BGP4-MIB::bgpPeerFsmEstablishedTransitions
      - 1.3.6.1.4.1.2636.3.5.2.1.4  # JUNIPER-FIREWALL-MIB::jnxFWCounterPacketCount
      - 1.3.6.1.4.1.2636.3.5.2.1.5  # JUNIPER-FIREWALL-MIB::jnxFWCounterByteCount
      - 1.3.6.1.4.1.2636.3.1.13     # https://kb.juniper.net/InfoCenter/index?page=content&id=KB17526
                                    # https://kb.juniper.net/InfoCenter/index?page=content&id=KB15467
    lookups:
      - source_indexes: [ifIndex]
        lookup: 1.3.6.1.2.1.31.1.1.1.1  # ifName
        drop_source_indexes: true
      - source_indexes: [ifIndex]
        lookup: 1.3.6.1.2.1.2.2.1.3     # ifType
      - source_indexes: [jnxOperatingContentsIndex,jnxOperatingL1Index,jnxOperatingL2Index,jnxOperatingL3Index]
        lookup: 1.3.6.1.4.1.2636.3.1.13.1.5
        drop_source_indexes: true
```

Бывает необходимость поменять тип, в противном случае данные в label отображаются в hex. Ниже пример для мониторинга гипервизора esxi
```yaml
  esxi:
    retries: 1
    walk:
      - 1.3.6.1.2.1.25.1.1              # hrSystemUptime
      - 1.3.6.1.2.1.25.2.3.1.5          # hrStorageSize
      - 1.3.6.1.2.1.25.2.3.1.6          # hrStorageUsed
      - 1.3.6.1.2.1.25.3.3              # hrProcessorTable
      - 1.3.6.1.2.1.25.5.1.1.2          # hrSWRunPerfMem
      - 1.3.6.1.2.1.25.4.2.1.7          # hrSWRunStatus

    lookups:
      - source_indexes: [hrStorageIndex]
        lookup: 1.3.6.1.2.1.25.2.3.1.2    # hrStorageType
        drop_source_indexes: false
      - source_indexes: [hrSWRunIndex]
        lookup: 1.3.6.1.2.1.25.4.2.1.2    # hrSWRunName
        drop_source_indexes: false
    overrides:
      hrSWRunName:
        type: DisplayString
```
RAM тут как диск
```text
hrStorageSize{hrStorageType="1.3.6.1.2.1.25.2.1.2"}
hrStorageUsed{hrStorageType="1.3.6.1.2.1.25.2.1.2"}
```
## проверка

```sh
curl localhost:9116/snmp?module=mx\&target=mx_host
```

Ссылки по теме:

[https://gtacknowledge.extremenetworks.com/articles/Q_A/Is-there-a-way-to-display-the-OID-1-3-6-1-4-1-1916-1-32-1-4-1-9-for-CPU-utilization-to-an-integer-instead-of-a-string](https://gtacknowledge.extremenetworks.com/articles/Q_A/Is-there-a-way-to-display-the-OID-1-3-6-1-4-1-1916-1-32-1-4-1-9-for-CPU-utilization-to-an-integer-instead-of-a-string)

[EXTREME-SOFTWARE-MONITOR-MIB](http://www.circitor.fr/Mibs/Html/E/EXTREME-SOFTWARE-MONITOR-MIB.php)

[Numbers from DisplayStrings with the snmp_exporter](https://www.robustperception.io/numbers-from-displaystrings-with-the-snmp_exporter)

[Why is my SNMP string showing as hexadecimal?](https://www.robustperception.io/why-is-my-snmp-string-showing-as-hexadecimal)

[EX How to check temperature, CPU/memory usage by SNMP OID](https://kb.juniper.net/InfoCenter/index?page=content&id=KB17526)

[Verifying some of the EX Switch chassis operating states using SNMP OID's](https://kb.juniper.net/InfoCenter/index?page=content&id=KB15467)
