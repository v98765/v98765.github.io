## Фильтр pvst на коммутаторах huawei

Фреймы BPDU pvst протокола прозрачно передаются коммутаторами, на которых не настроен pvst.
Поэтому, когда в сети настроен протокол mstp, необходимо pvst фильтровать на коммутаторах отличных от cisco.
Например, есть схема, где root msti 0 и mst 1 - это cisco.
К cisco подключен коммутатор huawei и на нем настроен mstp как и на cisco.
Подключается новый коммутатор cisco к коммутатору huawei, но его дефолтовые настройки stp не менялись.
Т.о. через вновь созданный транковый порт в нативном влане передается pvst bpdu через huawei далее на root'а.
Root (Cisco) переводит порт из режима P2p в режим совместимости с pvst, что выглядит как P2p Bound(PVST).
Root остается прежним для msti 0 (cist), но не для msti 1. В итоге huawei в msti 1 остался без root'а.
При кольцевой топологии такие изменения могут иметь печальные последствия, поэтому pvst надо фильтровать.
Взято с [https://forum.huawei.com/enterprise/en/how-to-filter-pvst-bpdus/thread/394511-861](https://forum.huawei.com/enterprise/en/how-to-filter-pvst-bpdus/thread/394511-861).
```text
acl number 4000
 rule 10 permit destination-mac 0100-0ccc-cccd

traffic classifier c1 operator or
 if-match acl 4000

traffic policy p1 match-order config
 classifier c1 behavior b1

traffic-policy p1 global inbound
```

## Подключение телефонов cisco 7940

Телефоны без прошивки с lldp и с poe несовместимы, не включаются. Для настройки voice vlan есть костыли для совместимости с cdp. Команды `voice-vlan legacy enable` и `poe legacy enable`
```text
interface GigabitEthernet0/0/11           
 port link-type hybrid
 voice-vlan 110 enable
 voice-vlan legacy enable
 port hybrid pvid vlan 520
 port hybrid tagged vlan 110
 port hybrid untagged vlan 520
 poe legacy enable
 lldp compliance cdp txrx
 lldp compliance cdp receive

>display cdp neighbor brief 
Local Intf    Neighbor Dev             Neighbor Intf             Exptime(s)
GE0/0/1      SEP081FF363712B          Port 1                    134    
```
