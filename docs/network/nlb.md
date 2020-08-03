На серверых ОС Microsoft'а имеется решение NLB с рекоменациями по настройке коммутаторов.
Прочитать подробности можно на странице [Configuring network infrastructure to support the NLB operation mode](https://support.microsoft.com/en-us/help/4494444/configuring-network-infrastructure-to-support-the-nlb-operation-mode).

## Cisco 3750 c версией IOS 15.х

Вроде всё что надо получается. Работу в режиме igmp multicast под нагрузкой не проверял.
В режиме multicast работает, но так как мультикастовый адрес 03bf не попадает в диапазон мультикастовых адресов igmp snooping'а,
фреймы будут рассылаться во все порты данного vlan'а, где подобных статических записей не будет.
Это может стать проблемой на транспортной сети, на blade коммутаторах в составе шасси, где какие-либо ограничения обычно отсутствуют.

```text
cisco3750(config)#arp  10.0.0.1 0100.5e7f.0001 arpa
cisco3750(config)#arp  10.0.0.2 03bf.0a00.0002 arpa
cisco3750(config)#^Z
cisco3750#sh ip arp
Protocol  Address          Age (min)  Hardware Addr   Type   Interface
Internet  10.0.0.1                -   0100.5e7f.0001  ARPA
Internet  10.0.0.2                -   03bf.0a00.0002  ARPA
Internet  10.0.0.10               -   6400.f154.1bea  ARPA   Vlan10

cisco3750#sh run | in static
mac address-table static 03bf.0a00.0002 vlan 10 interface GigabitEthernet1/0/9 GigabitEthernet1/0/10

cisco3750#sh mac address-table static | in 0002
 10    03bf.0a00.0002    STATIC      Gi1/0/9 Gi1/0/10
```

## Cisco 4900M

Все работает в режиме igmp multicast,но L3 коммутатор не следует делать igmp querier'ом.
Для этого понадобится рядом стоящий L2 коммутатор. В противном случае загрузка cpu будет пропорциональна pps до кластера.
Есть и другие ограничения, с которыми можно ознакомиться, изучив ссылки на сайт cisco.
Например, muticast mode тоже приведет к высокой загрузке cpu на 4500 шасси, в VSS на 4500Х может некорректно работать NLB из-за ошибок в ПО.

## Juniper EX2200
Прописать статикой mac-адрес в пару портов для multicast mode не выйдет, т.к. при второй записи удаляется первая и указывать мультикастовый mac тоже нельзя. Arp-запись для работы в режиме igmp multicast создать можно, но есть ограничение по схеме включения серверов в коммутаторы. Описаны в KB по ссылкам.

```text
{master:0}[edit ethernet-switching-options static vlan v32 mac 03:bf:0a:00:00:01]
user@ex2200# show
next-hop ge-0/0/11.0;

{master:0}[edit ethernet-switching-options static vlan v32 mac 03:bf:0a:00:00:01]
user@ex2200# set next-hop ge-0/0/10.0

{master:0}[edit ethernet-switching-options static vlan v32 mac 03:bf:0a:00:00:01]
user@ex2200# show
next-hop ge-0/0/10.0;

{master:0}[edit ethernet-switching-options static vlan v32 mac 03:bf:0a:00:00:01]
user@ex2200# commit check
[edit ethernet-switching-options static vlan v32]
  'mac 03:bf:0a:00:00:01'
    multicast mac address 03:bf:0a:00:00:01 is not allowed
error: configuration check-out failed

user@ex2200# show | compare
[edit interfaces vlan]
+    unit 32 {
+        family inet {
+            address 10.0.0.10/24 {
+                arp 10.0.0.1 multicast-mac 01:00:5e:7f:00:01;
+            }
+        }
+    }
[edit vlans v32]
+   l3-interface vlan.32;

{master:0}[edit]
user@ex2200# commit check
configuration check succeeds

user@ex2200> show arp
MAC Address       Address         Name                      Interface           Flags
01:00:5e:7f:00:01 10.0.0.1        10.0.0.1                  vlan.32             permanent
```

## Eltex 2324
В 4.0.11 не получится собрать корректно работающий nlb в режимах multicast и igmp multicast. Далее не проверял.
```text
eltex2324#sh run int vlan1
interface vlan 1
 ip address 10.0.0.10 255.255.255.0
 no ip address dhcp
!
eltex2324#conf t
eltex2324(config)#arp 10.0.0.1 01:00:5e:7f:00:01 vlan 1
ARP MAC address must be Unicast address.
eltex2324(config)#

eltex2324(config)#$mac address-table static 03bf.0a00.0001 vlan 1 interface te1/0/2
Port:te1/0/2 , MAC:03:bf:0a:00:00:01 : Illegal MAC address.
```

## Huawei 5720EI

Можно указать в arp-ах любой мультикастовый mac, нельзя сделать статическую запись в таблице mac-адресов. Всё как в Juniper EX2200, в общем.
```text
[huawei5720]display arp | include 10.0.0.
IP ADDRESS      MAC ADDRESS     EXPIRE(M) TYPE        INTERFACE   VPN-INSTANCE
                                          VLAN/CEVLAN
------------------------------------------------------------------------------
10.0.0.10       4cf9-5d79-aa12            I -         Vlanif1
10.0.0.1        03bf-0a00-0001            S--
10.0.0.2        0100-5e7f-0002            S--
------------------------------------------------------------------------------
[huawei5720]mac-address static 03bf-0a00-0001 GigabitEthernet0/0/4 vlan 1
Error: Multicast or broadcast does not support the configuration of the MAC address.
```
