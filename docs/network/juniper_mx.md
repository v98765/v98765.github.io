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
