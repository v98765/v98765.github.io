[MetricQL](https://victoriametrics.github.io/MetricsQL.html) и [PromQL](https://prometheus.io/docs/prometheus/latest/querying/functions/)

[Expand VictoriaMetrics WITH templates to canonical PromQL](https://play.victoriametrics.com/promql/expand-with-exprs)

Все описанные метрики получены vmagent'ом с snmp_exporter'а, записаны в базу VictoriaMetrics.

## Список метрик

В grafana есть возможность делать запросы в базы через Explore. Для вывода всех метрик, полученных по job'у job_name выполнить:
```text
max({job="job_name"}) by (__name__)
``` 

## Варианты запросов в базу

В редакторе панели есть query inspector. Он позволяет посмотреть запрос в базу со всеми параметрами. Можно разделить запросы на два типа, описанных в API:

* [instant-queries](https://prometheus.io/docs/prometheus/latest/querying/api/#instant-queries). Включается галочкой instant.
* [range-queries](https://prometheus.io/docs/prometheus/latest/querying/api/#range-queries). Используется по умолчанию.

[Цитата](https://t.me/VictoriaMetrics_ru1/15540) из чата:

>Instant запросы идут к /api/v1/query, а не к /api/v1/query_range.
>Там вообще нет параметра step - см. [https://prometheus.io/docs/prometheus/latest/querying/api/#instant-queries](https://prometheus.io/docs/prometheus/latest/querying/api/#instant-queries),
>т.к. такой запрос должен возвращать только по одной точке на ряд в момент времени, переданном в параметре time. ВМ принимает step в запросе к /api/v1/query из-за особенностей реализации.
>В большинстве случаев его туда передавать не нужно.

## Переменные для дашбордов

Для дашборда с выводом информации по маршрутизаторам и коммутаторам понадобится несколько переменных: хост (ip) и интерфейс (iface).
Имя хоста (instance) из метрики ifHCInOctets в переменную ip.
```text
label_values(ifHCInOctets{},instance)
```
Для списка интерфейсов конкретного хоста взял ifName из ifHCInOctets, где label instance - это выбранный ранее хост:
```text
label_values(ifHCInOctets{instance="$ip"},  ifName)
```
ifDescr из ifTable не опрашиваю, поэтому имена только из ifXTable.

## Uptime

```text
sysUpTime{instance="$ip"}*10
```

instant | panel | format | interval | relative time
---|---|---|---|---
yes | Stat |  | 1m | 15m

Field -> Standard options -> Units -> Time -> duration (ms), Field -> Standard options -> Decimals -> 3,
Panel -> Display -> Fields - Numeric Fields

## Таблица для определения утечки памяти на cisco

В vmagent'е job'у соотвествует модуль snmp_exporter'а, созданный для определенного типа устройств: asa, router, l2 switch, l3 switch.
Т.е. выбирается по 5 для каждого типа устройств.

```text
bottomk by (job) (5, (delta(ciscoMemoryPoolFree{ciscoMemoryPoolType!="20"}[7d])))
```

instant | panel | format | interval | relative time
---|---|---|---|---
yes | table | table | 7d | 7d


## Ошибки на интерфейсах

Устройства имеют label "obj" для идентификации места установки и удобства при составлении запросов.

```text
topk(10, (increase(ifInErrors{obj="$object"}[8h]))) > 0
topk(5, (increase(ifOutDiscards{obj="$object"}[8h]))) > 0
```

instant | panel | format | interval | relative time
---|---|---|---|---
yes | table | table | 8d | 8d

Для вывода ошибок по конкретному хосту удобнее видеть таблицу с текущими ошибками и суммой за период:
```text
increase(ifInErrors{instance="$ip"}[5m])
```

instant | panel | format | interval | relative time
---|---|---|---|---
no | table | timeseries | 5m | 1h

Для timeseries нужно в таблице сделать Transform Reduce с Calculation Last, Total.
В поле Legend указать `{{ ifName }}` чтобы не отображать все имеющиеся лейблы временного ряда.

## Отчет по доступности

Анализируются статусы недоступности в cisco ip sla (4) и huawei nqa (2).
В дашборде создана переменная ivl тип Interval.
Для каждого пробера в одной таблице создается отдельный запрос, чтобы понятными буквами в легенде описать название канала связи.
Каждая строка - отдельный запрос: A,B и т.д.
```text
count_gt_over_time(rttMonLatestRttOperSense{instance="cisco-router-1",rttMonCtrlAdminIndex="1"}[$ivl], 3)
count_gt_over_time(rttMonLatestRttOperSense{instance="cisco-router-1",rttMonCtrlAdminIndex="2"}[$ivl], 3)
..
count_gt_over_time(nqaResultsCompletions{instance="huawei-1",nqaAdminCtrlOwnerIndex="1",nqaAdminCtrlTestName="1"}[$ivl], 1)
count_gt_over_time(nqaResultsCompletions{instance="huawei-1",nqaAdminCtrlOwnerIndex="1",nqaAdminCtrlTestName="2"}[$ivl], 1)
```

instant | panel | format | interval | relative time
---|---|---|---|---
yes | table | timeseries | $ivl |

Для timeseries нужно в таблице сделать Transform Reduce с Calculation Max или Last.

## График свободной памяти

Нужно вывести %, а значит нужно вычислить:
```text
ciscoMemoryPoolFree{instance="$ip",ciscoMemoryPoolType!="20"}*100/(ciscoMemoryPoolFree{instance="$ip",ciscoMemoryPoolType!="20"}+ciscoMemoryPoolUsed{instance="$ip",ciscoMemoryPoolType!="20"})
```

instant | panel | format | interval | relative time
---|---|---|---|---
no | graph | timeseries | |

В поле Legend указать `{{ciscoMemoryPoolName}}` чтобы не отображать все имеющиеся лейблы временного ряда. Panel -> Axes -> Left Y -> Unit -> Misc -> percent (0-100)

## График загрузки cpu

В метрике есть %, поэтому:
```text
cpmCPUTotal5min{instance="$ip"}
```

instant | panel | format | interval | relative time
---|---|---|---|---
no | graph | timeseries | |

В поле Legend указать `{{cpu5min}}` чтобы не отображать все имеющиеся лейблы временного ряда. Panel -> Axes -> Left Y -> Unit -> Misc -> percent (0-100)

## График ifOutDiscards ошибок

В панели будет график с двумя линиями: выбранный интерфейс, сумма ошибок по всем интерфейсам.
Там будут записи только для ifType=6, потому что есть правило для удаление иных значений в vmagent'е. Для выбранного интерфейса:
```text
increase(ifOutDiscards{instance="$ip",ifName="$iface"}[5m])
```
Сумма ошибок:
```text
sum(increase(ifOutDiscards{instance="$ip"}[5m]))
```

instant | panel | format | interval | relative time
---|---|---|---|---
no | graph | timeseries | |

В поле Legend указать `$iface` для первого запроса и `Total` для второго.

## График IfInErrors ошибок

В панели будет график с двумя линиями: выбранный интерфейс, сумма ошибок по всем интерфейсам.
Там будут записи только для ifType=6, потому что есть правило для удаление иных значений в vmagent'е. Для выбранного интерфейса:
```text
increase(ifInErrors{instance="$ip",ifName="$iface"}[5m])
```
Сумма ошибок:
```text
sum(increase(ifInErrors{instance="$ip"}[5m]))
```

instant | panel | format | interval | relative time
---|---|---|---|---
no | graph | timeseries | |

В поле Legend указать `$iface` для первого запроса и `Total` для второго.

## График BPS

Сумма по всем физическим интерфейсам хоста:
```text
8 * sum(rate(ifHCInOctets{instance="$ip",ifType="6"}[5m]))
```
BPS по выбранному интерфейсу по приему:
```text
8 * rate(ifHCInOctets{instance="$ip",ifName="$iface"}[5m])
```
BPS по выбранному интерфейсу по передаче:
```text
8 * rate(ifHCOutOctets{instance="$ip",ifName="$iface"}[5m])
```

instant | panel | format | interval | relative time
---|---|---|---|---
no | graph | timeseries | |

В поле Legend указать `$iface recv` и `$iface send` для второго и третьего запроса и `Total recv` для первого.
Panel -> Axes -> Left Y -> Unit -> Data rate -> bits/sec.

MetricQL позволяет конструировать один запрос для приема и передачи. Ниже сумма счетчиков с двух интерфейсов Gi0/1.100 разных маршрутизаторов для построения одного графика:
```text
WITH (
	cf = {instance=~"border-1|border-2", ifName="Gi0/1.100" },
	netUsage(name, m) = label_set(sum(rate(m{cf})*8), "name", name),
)
union (
	netUsage("recv", ifHCOutOctets),
	netUsage("send", ifHCInOctets),
)
```
Когда используется шаблон `instance=~` для того чтобы указать какое-то регулярное выражение, то необходимо функцию `rate()` поместить в `sum()`.
Это сделано выше и ниже в примере, где BPS показан из счетчиков политики huawei c двух разных устройств. Счетчики политики для приема (direction=1):
```text
sum(rate(hwCBQoSPolicyStatClassifierMatchedPassBytes{hwCBQoSIfApplyPolicyDirection="1",hwCBQoSIfApplyPolicyIfIndex="6",hwCBQoSIfVlanApplyPolicyVlanid1="0",hwCBQoSPolicyStatClassifierName="0xA",instance=~"huawei-[12]"}[5m]))*8
```
Счетчики политики для передачи (direction=2):
```text
sum(rate(hwCBQoSPolicyStatClassifierMatchedPassBytes{hwCBQoSIfApplyPolicyDirection="2",hwCBQoSIfApplyPolicyIfIndex="6",hwCBQoSIfVlanApplyPolicyVlanid1="0",hwCBQoSPolicyStatClassifierName="0xB",instance=~"huawei-[12]"}[5m]))*8
```
Счетчики bps есть в политиках на маршрутизаторах cisco.
В примере ниже сумма счетчиков класса двух политик с двух разных маршрутизаторов.
Для сокращения напишу пример только для одного направления. Как узнать индексы я не буду тут писать.
```text
8 * (sum(rate(cbQosCMPostPolicyByte{cbQosObjectsIndex="13816289",cbQosPolicyIndex="240",instance="border-1"}))
 + 
sum(rate(cbQosCMPostPolicyByte{cbQosObjectsIndex="1341969",cbQosPolicyIndex="3680",instance="border-2"})))
```
Функция offset из MetricQL позволяет сравнивать текущие данные с историческими. Например, разница в BPS одного интерфейса сейчас и неделю назад:
```text
8 * (sum(rate(ifHCOutOctets{instance="switch",ifName="Gi1/0/1"})) - sum(rate(ifHCOutOctets{instance="switch",ifName="Gi1/0/1"}) offset 7d))
```

## Таблица BPS

Снова MetricQL. Top 10 загруженных интерфейсов по всей сети за 5 минут по счетчику ifHCOutOctets, переданному как параметр PortUtilization из topk.
Дополнительно используется функция ru() из MetricQL для определения % утилизации порта:
```text
WITH (
    maxRate = ifHighSpeed{ifType="6"}[$__interval] * 1e6 / 8,
    commonFilters = {ifType="6"},
    PortUtilization(ifHCOctets, maxRate) = ru(maxRate - rate(ifHCOctets{commonFilters}[$__interval]), maxRate)
)topk(10,PortUtilization(ifHCOutOctets, maxRate))
```

instant | panel | format | interval | relative time
---|---|---|---|---
yes | table | timeseries | 5m | 5m

В поле Legend указать `{{instance}},{{ifName}}`.
Такую же таблицу сделать для ifHCInOctets.

## График PPS

PPS по выбранному интерфейсу по приему:
```text
rate(ifHCInUcastPkts{instance="$ip",ifName="$iface"}[5m])
```
PPS по выбранному интерфейсу по передаче:
```text
rate(ifHCOutUcastPkts{instance="$ip",ifName="$iface"}[5m])
```

instant | panel | format | interval | relative time
---|---|---|---|---
no | graph | timeseries | |

В поле Legend указать `$iface recv` и `$iface send` для первого и второго запроса.
Panel -> Axes -> Left Y -> Unit -> Data rate -> packets/sec.

## Таблица c top по flapping interface

Первая таблица по кол-ву изменений оперативного статуса интерфейса.
```
topk(10,sum_over_time(changes(ifOperStatus{}[1m])[1h]))
```
Вторая таблица по изменнию скорости подключения. В топе будут принтеры, которые в режиме ожидания переключаются в 10мбит периодически.
```text
topk(10,sum_over_time(changes(ifHighSpeed{}[1m])[1h]))
```

instant | panel | format | interval | relative time
---|---|---|---|---
yes | table | timeseries | 1m | 1h

В поле Legend указать `{{instance}},{{ifName}}`.
Для timeseries нужно в таблице сделать Transform Reduce с Calculation Max.

## Таблицы по spanning-tree

Определение root'а в регионе.
```text
bottomk(1,dot1dStpPriority) by (obj) 
```

instant | panel | format | interval | relative time
---|---|---|---|---
yes | table | timeseries | 1m | 5m

В поле Legend указать `{{instance}}`.
Для timeseries нужно в таблице сделать Transform Reduce с Calculation Max.

Таймеры TCN. Если изменился недавно, то значит произошло изменение топологии из-за переключений или из-за некорректных настроек.
```text
10 * bottomk(1,dot1dStpTimeSinceTopologyChange != 0) by (obj)
```

instant | panel | format | interval | relative time
---|---|---|---|---
yes | table | timeseries | 1m | 5m

В поле Legend указать `{{instance}}`.
Для timeseries нужно в таблице сделать Transform Reduce с Calculation Max. Для колонки с временем выбрать Units -> Time -> duration (ms)

Сколько раз на коммутаторе менялся root port за час.
```text
sum_over_time(changes(dot1dStpRootPort{})[1h]) > 
```

instant | panel | format | interval | relative time
---|---|---|---|---
yes | table | timeseries | 1m | 1h

В поле Legend указать `{{instance}}`.
Для timeseries нужно в таблице сделать Transform Reduce с Calculation Max.

## Cisco ErrDisableIfStatus

Это не метрики, а события. С одной стороны неправильно для этого использовать tsdb. С другой стороны, когда recovery на коммутаторах выключен, то вполне.
В таблице будут числа, а Enum можно посмотреть в snmp.yml или в мибах. Так 2 - это отработал bpduguard.
```text
cErrDisableIfStatusCause{}[1m]
```

instant | panel | format | interval | relative time
---|---|---|---|---
yes | table | table | 1m | 5m

## Проверка протоколов маршрутизации

Для проверки состояния протоколов маршрутизации можно использовать доступные метрики, создавать алерты средствами grafana или vmalerts+alertmanager.

BGP peer status равен 6, если established. Если не 6, то алерт.
```text
bgpPeerState{bgpPeerRemoteAddr="10.9.8.7"}[1m]
```
Huawei OSPF nbr state. 8 - OK. Если меньше, то алерт.
```text
hwOspfv2NbrState{hwOspfv2NbrIpAddrIndex="10.9.8.7"}[1m]
```

## Проверка проберов SLA, NQA

NQA для huawei с icmp пробером. 1 - ОК. Если больше, то алерт.
```text
nqaResultsCompletions{instance="huawei",nqaAdminCtrlTestName="1"}[1m]`
```
Cisco SLA с icmp пробером. 1 - OK. 3 - OK, но сработал трешхолд по rtt. 4 - FAIL.
```text
rttMonLatestRttOperSense{instance="cisco",rttMonCtrlAdminIndex="1"}[1m]
```
Наличие потерь на канале связи, когда статус периодически изменяется. Если более 4 раз за 15 минут, то алерт.
```
changes(rttMonLatestRttOperSense{instance="cisco",rttMonCtrlAdminIndex="1"}[15m])
```

## Проверка статуса интерфейса

1 - OK. Если больше, то алерт.
```text
ifOperStatus{ifName="Gi1/0/1",instance="cisco"}[1m]
```

## Проверка температуры

Есть датчики на оборудовании Cisco и у большинства старых моделей одинаковый oid, но разные TemperatureStatusIndex для разных моделей, стеков.
Либо датчиков более одного и нужно выбрать конкретный.
```text
ciscoEnvMonTemperatureStatusValue{ciscoEnvMonTemperatureStatusIndex="1",instance="cisco"}[10m]
```
Проверка температуры батарей ИБП.
```text
upsAdvBatteryTemperature{instance="apc_ups"}
```

## Проверка ИБП APC

Статус. 2 - ОК. 3 - на батареях. 12 - напряжение выше 250В на входе.
```text
upsBasicOutputStatus{instance="apc_ups"}
```

Enum
```text
  1: unknown
  2: onLine
  3: onBattery
  4: onSmartBoost
  5: timedSleeping
  6: softwareBypass
  7: "off"
  8: rebooting
  9: switchedBypass
  10: hardwareFailureBypass
  11: sleepingUntilPowerReturn
  12: onSmartTrim
```

## alerts ifmib

flapping физического интерфейса, который в up, но его состояние изменилось до этого
```text
(changes(ifLastChange{ifType="6"}[15m]) > 0) and (ifOperStatus == 1) and ON(instance) (sysUpTime > 150000)
```
Нет линка на административно включенном интерфейсе
```text
(ifAdminStatus == 1) and (ifOperStatus == 2)
```
Включился flowcontrol. Полезно для портов, куда подключены стораджи
```text
rate(dot3HCInPauseFrames[5m]) > 5
```
Входящие ошибки
```text
rate(ifInErrors[15m]) > 5
```
Дискарды
```text
rate(ifOutDiscards[15m]) > 100
```
Утилизация пропускной полосы в зависимости от указанной bw или физической скорости интерфейса. 62500 = 1e6 / 8 / 2, где 2 - rate 50%.
В примере ниже 72% потому что круглое число
```text
rate(ifHCOutOctets{ifType="6"}[10m]) > ((ifHighSpeed{ifType="6"})*90000)
```

## alerts fiber channel

Редко. не видел
```text
rate(fcIfCreditLoss[5m]) > 0
rate(fcIfTxWaitCount[5m]) > 0
```
Физика, например, вкл/выкл
```text
rate(fcIfLinkResetIns[5m]) > 0
rate(fcIfLinkResetOuts[5m]) > 0
```
Нормальная работа FC сети, когда заканчиваются bbcredits
```text
rate(fcHCIfBBCreditTransistionToZero[5m]) > 500000
rate(fcHCIfBBCreditTransistionFromZero[5m]) > 500000
```
Серьезные проблемы. Что-то отвалилилось и данные не были доставлены.
```text
rate(fcIfTimeOutDiscards[5m]) > 0
rate(fcIfOutDiscards[5m]) > 0
```
Более 100мс не было ресурсов на порту. Переделать порт-группу, добавить кредитов на порт.
```text
rate(fcIfTxWtAvgBBCreditTransitionToZero[5m]) > 0
```
Не хватает полосы. Нужен port-channel, дополнительные линки
```
rate(fcIfC3InOctets[10m]) > ((ifHighSpeed)*90000)
rate(fcIfC3OutOctets[10m]) > ((ifHighSpeed)*90000)
```

## alerts exos

Питание
```text
extremePowerSupplyStatus !=2
```
Блок вентиляторов
```text
extremeFanOperational > 1
```
Перегрев
```text
extremeOverTemperatureAlarm < 2
```
cpu
```text
extremeCpuMonitorSystemUtilization1min > 15
```
