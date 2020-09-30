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
