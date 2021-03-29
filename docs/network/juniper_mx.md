## arp policer

[Example: Configuring ARP Policer](https://www.juniper.net/documentation/en_US/junos/topics/example/example-configuring-arp-policer.html)

Проблема в дефолтовом полисере. На бордерах, подключенных к точкам обмена, его необходимо настраивать
```text
> show policer
Policers:
Name                                                Bytes              Packets
__default_arp_policer__                       74695272896           1656698750
```

По инструкции создать фильтр, применить к интерфейсу, почистить счетчики `clear firewall`. Проверить еще раз и при необходимости увеличить.
```text
> show policer
Policers:
Name                                                Bytes              Packets
__default_arp_policer__                                 0                    0
arp_limit-xe-0/0/0.100-inet-arp                        46                    1
```

## ddos

```text
show ddos-protection protocols statistics terse
show ddos-protection protocols culprit-flows
show ddos-protection protocols violations
```

## sflow

[Example: Configuring sFlow Technology to Monitor Network Traffic on MX Series Routers
](https://www.juniper.net/documentation/en_US/junos/topics/example/sflow-configuring-mx-series.html)

Смотреть индексы интерфейсов c дескрипшенами, чтобы добавить их в описание `annotate interfaces ..`
```text
show snmp mib walk ifAlias | match =
```
Есть ограничение в один сабинтерфейс на один физический интерфейс.
```text
> show configuration protocols sflow
polling-interval 60;
sample-rate {
    ingress 2048;
    egress 2048;
}
source-ip 10.9.8.7;
collector 10.0.0.1 {
    forwarding-class best-effort;
}
/* 550.peer1 */
interfaces xe-0/1/1.100;
/* 560.peer2 */
inactive: interfaces xe-0/1/1.200;
/* 570.peer3 */
inactive: interfaces xe-0/1/1.300;
/* 580.peer4 */
interfaces xe-0/1/2.100;
```

В 20.2
```text
Alarm cleared: License id=0, color=YELLOW, class=CHASSIS, reason=Secure Flow usage requires a license
Alarm set: License id=0, color=YELLOW, class=CHASSIS, reason=Secure Flow usage requires a license
```

## load-balancing

> You can configure Junos OS so that, for the active route, all next-hop addresses for a destination are installed in the forwarding table. This feature is called per-packet load balancing. The naming may be counter-intuitive. However, Junos per-packet load balancing is functionally equivalent to what other vendors may term per-flow load balancing. You can use load balancing to spread traffic across multiple paths between routers.

> To enable per-flow load balancing, you must set the load-balance per-packet action in the routing policy configuration.

Полиси
```text
> show configuration policy-options policy-statement balance
from protocol bgp;
then {
    load-balance per-packet;
}
> show configuration routing-options forwarding-table
export balance;
```
В группу bgp peer-ов добавить `multipath multiple-as`
```text
> show configuration protocols bgp group upstream
type external;
multipath {
    multiple-as;
}
...
```

## pic

[pic-mode](https://www.juniper.net/documentation/en_US/junos/topics/reference/configuration-statement/pic-mode-edit-chassis-mx-series.html)
[Port Checker](https://apps.juniper.net/home/port-checker/index.html)
[Port Speed](https://www.juniper.net/documentation/en_US/junos/topics/topic-map/port-speed-configuration.html#id-configuring-rate-selectability)

## ЦМУ

Делается в рамках инструкции для передачи данных в ЦМУ. Написал ниже cmo, переделывать уже не стану. Адрес ЦМУ реальный из инструкции. Интернет в инстансе для примера `as1234`.

Для snmp хотят:

> IP-адрес системы (на стороне ЦМУ) для SNMP  (от данного адреса будут приходит запросы);77.95.132.140 Запросы направляются на стандарт-ный порт (161).
> Опрашиваются IF-MIB, IP-FORWARD-MIB,  BGP4-MIB,  SNMPv2-MIB. Перио-дичность опросов по умолчанию 60 минут (может быть скорректирована).

```text
set snmp view CMO oid 1.3.6.1.2.1.2.2 include
set snmp view CMO oid 1.3.6.1.2.1.15 include
set snmp view CMO oid 1.3.6.1.6.3.1 include
set snmp view CMO oid 1.3.6.1.2.1.4.24 include
set snmp community cmo view CMO
set snmp community cmo authorization read-only
set snmp community cmo routing-instance as1234 clients 77.95.132.140/32
set snmp community cmo routing-instance as1234 clients [test-ip]/32
set snmp routing-instance-access
```
С тестового хоста с адресом `test-ip` проверить доступность
```sh
snmpwalk -c as1234@cmo -v2c router
```

Для bgp хотят:

> IP-адрес системы (на стороне ЦМУ) для BGP соединений;	Необходимо использовать eBGP Multihop соединение.
> Если для установления BGP соединения НЕ используется MD5 (рекомендуется), для конфигурации использовать адрес: 77.95.132.140
> Если для установления BGP соединения используется MD5 (НЕ рекомендуется), адрес для конфигурации необходимо запросить и согласовать с ЦМУ (будет предложен адрес из диапазона 185.224.228.0/24)
> Внимание! В связи с техническими возможностями ЦМУ используемый в данном документе адрес со временем будет меняться.
> В связи с этим, при передаче заполненной таблицы следует запросить у ЦМУ корректный на момент отправки таблицы  IP-адресс.
> Обратите внимание, ПО на стороне ЦМУ не является маршрутизатором, расположено за межсетевым экраном и NAT трансляцией, поэтому установка сес-сии может быть инициирована только со стороны ЦМУ,
> соединение устанавливается на 179 порт с порта из диапазона >1024. Большая просьба учиты-вать данную информации при конфигурировании сессии BGP и фильтров (ACL) на Вашей стороне.
> Ес-ли поддерживается маршрутизатором, конфигури-руйте сессию с ЦМУ в режиме passive.
> Номер AS на стороне ЦМУ 	По умолчанию 65211 либо может быть любым, предполагается использование Private Space Number.

```text
set routing-instances as1234 protocols bgp group cmo type external
set routing-instances as1234 protocols bgp group cmo neighbor 77.95.132.140 multihop
set routing-instances as1234 protocols bgp group cmo neighbor 77.95.132.140 local-address [local-ip]
set routing-instances as1234 protocols bgp group cmo neighbor 77.95.132.140 passive
set routing-instances as1234 protocols bgp group cmo neighbor 77.95.132.140 import RF-reject
set routing-instances as1234 protocols bgp group cmo neighbor 77.95.132.140 export RF-default-export
set routing-instances as1234 protocols bgp group cmo neighbor 77.95.132.140 peer-as 65211
```

Хотят netflow. Есть sflow с PR1487876. В 20.2 хочет лицензию какую-то, чего в 18 и в 19 нет.

```text
Alarm cleared: License id=0, color=YELLOW, class=CHASSIS, reason=Secure Flow usage requires a license
Alarm set: License id=0, color=YELLOW, class=CHASSIS, reason=Secure Flow usage requires a license
```

> 77.95.132.140. Порт UDP 9080

```text
set protocols sflow polling-interval 60
set protocols sflow sample-rate ingress 10000
set protocols sflow sample-rate egress 10000
set protocols sflow source-ip [local-ip]
set protocols sflow collector 77.95.132.140 udp-port 9080
set protocols sflow interfaces xe-0/1/4
```

By default, Junos will use the default routing instance to send sFlow packets. 
```text
set routing-options static route 77.95.132.140/32 next-table as1234.inet.0
```

Фильтр с каунтером для проверки отправки flows
```text
set firewall family inet filter cmo term cmo from destination-address 77.95.132.140/32
set firewall family inet filter cmo term cmo from destination-port 9080
set firewall family inet filter cmo term cmo then count cmo
set firewall family inet filter cmo term cmo then accept
set firewall family inet filter cmo term all then accept
```

## ipfix

Взято из [github.com/VerizonDigital/vflow](https://github.com/VerizonDigital/vflow/blame/master/docs/junos_integration.md) без изменений

sampling на интерфейсах
```
set interfaces xe-1/0/0.0 family inet sampling input
set interfaces xe-1/0/0.0 family inet sampling output
```

темплейт
```
set services flow-monitoring version-ipfix template vflow flow-active-timeout 10
set services flow-monitoring version-ipfix template vflow flow-inactive-timeout 10
set services flow-monitoring version-ipfix template vflow template-refresh-rate packets 1000
set services flow-monitoring version-ipfix template vflow template-refresh-rate seconds 10
set services flow-monitoring version-ipfix template vflow option-refresh-rate packets 1000
set services flow-monitoring version-ipfix template vflow option-refresh-rate seconds 10
set services flow-monitoring version-ipfix template vflow ipv4-template
```

```
set chassis fpc 0 sampling-instance vflow
set chassis fpc 1 sampling-instance vflow
set forwarding-options sampling instance ipfix input rate 100
set forwarding-options sampling instance ipfix family inet output flow-server 192.168.0.10 port 4739
set forwarding-options sampling instance ipfix family inet output flow-server 192.168.0.10 version-ipfix template vflow
set forwarding-options sampling instance ipfix family inet output inline-jflow source-address 192.168.0.1
```

## junos upgrade

[VM Host Overview](https://www.juniper.net/documentation/en_US/junos/topics/topic-map/vm-host-overview.html), [vm-host-operations-management](https://www.juniper.net/documentation/en_US/junos/topics/concept/vm-host-operations-management.html), 
[installation_upgrade](https://www.juniper.net/documentation/en_US/junos/topics/concept/installation_upgrade.html)

С 17.x до 20.2 обновиться сразу не получится.
```text

ERROR: estimate of space required: 4035098 Kbytes, available: 3994154 Kbytes
```
Обновление с 17.x до 19.4 возможно, но потребуется дополнительный ребут после первой загрузки 19.4.
```text
Chassis control process: <xnm:warning xmlns="http://xml.juniper.net/xnm/1.1/xnm" xmlns:xnm="http://xml.juniper.net/xnm/1.1/xnm">
Chassis control process: <source-daemon>chassisd</source-daemon>
Chassis control process: <edit-path>[edit groups junos-defaults]</edit-path>
Chassis control process: <statement>chassis</statement>
Chassis control process: <message>Chassis configuration for network services has been changed. A system reboot is mandatory.  Please reboot *ALL* routing engines NOW. >
Chassis control process: </xnm:warning>
```
Предварительная копия в /var/tmp/
```text
file copy http://[host]/fw/junos/junos-vmhost-install-mx-x86-64-19.4R3-S2.2.tgz /var/tmp/
```
Обновление с перезагрузкой
```text
request vmhost software add no-validate reboot /var/tmp/junos-vmhost-install-mx-x86-64-19.4R3-S2.2.tgz
```
Перезагрузка RE по причине выше
```text
request vmhost reboot
```
Обновление с 19.4 до 20.2 штатно
```text
request vmhost software add no-validate http://[host]/fw/junos/junos-vmhost-install-mx-x86-64-20.2R2-S2.6.tgz
```
После сообщения выполнить указанную команду
```text
A REBOOT IS REQUIRED TO LOAD THIS SOFTWARE CORRECTLY.
Use the 'request vmhost reboot' command to reboot the system.
```

Обновление проводить предпочтительно через OOBM подключение, т.к. после перезагрузки некоторое время будут недоступны физические интерфейсы
из-за обновлений линейных карт. Это же касается mx204, где несколько минут после загрузки будут отсутствовать `xe-` интерфейсы.

## vmhost rollback

```text
mx> show vmhost version
Current root details,           Device sda, Label: jrootp_P, Partition: sda3
Current boot disk: Primary
Current root set: p
UEFI    Version: CBEP_P_SUM1_00.13.01

Primary Disk, Upgrade Time: Wed Mar 17 12:30:11 UTC 2021

Version: set p
VMHost Version: 5.2268
VMHost Root: vmhost-x86_64-20.2R2-S1-20201210_1323_builder
VMHost Core: vmhost-core-x86-64-20.2R2-S2.6
kernel: 4.8.28-rt10-WR9.0.0.24_ovp
Junos Disk: junos-install-mx-x86-64-20.2R2-S2.6

Version: set b
VMHost Version: 5.2225
VMHost Root: vmhost-x86_64-19.4R3-S2-20210118_0229_builder
VMHost Core: vmhost-core-x86-64-19.4R3-S2.2
kernel: 4.8.28-rt10-WR9.0.0.20_ovp
Junos Disk: junos-install-mx-x86-64-19.4R3-S2.2
```


```text
request vmhost software rollback
```

```text
mx> show vmhost version
Current root details,           Device sda, Label: jrootb_P, Partition: sda4
Current boot disk: Primary
Current root set: b
UEFI    Version: CBEP_P_SUM1_00.13.01

Primary Disk, Upgrade Time: Wed Mar 17 12:30:11 UTC 2021

Version: set p
VMHost Version: 5.2268
VMHost Root: vmhost-x86_64-20.2R2-S1-20201210_1323_builder
VMHost Core: vmhost-core-x86-64-20.2R2-S2.6
kernel: 4.8.28-rt10-WR9.0.0.24_ovp
Junos Disk: junos-install-mx-x86-64-20.2R2-S2.6

Version: set b
VMHost Version: 5.2225
VMHost Root: vmhost-x86_64-19.4R3-S2-20210118_0229_builder
VMHost Core: vmhost-core-x86-64-19.4R3-S2.2
kernel: 4.8.28-rt10-WR9.0.0.20_ovp
Junos Disk: junos-install-mx-x86-64-19.4R3-S2.2

mx> show version
Model: mx204
Junos: 19.4R3-S2.2
```
