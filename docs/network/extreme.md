[Документация extreme](https://www.extremenetworks.com/support/documentation/) и [ExtremeXOS 30.7](https://www.extremenetworks.com/support/documentation/extremexos-30-7/) 

## reset

Сбросить конфигурацию. Сразу после ввода команды требуется перезагрузка.

```text
unconfigure switch all
```


## erps

Прокотол защиты от петель ERPS. Чаcть документации взято с сайта [h3c ERPS configuration](http://www.h3c.com/en/Support/Resource_Center/Technical_Documents/Home/Switches/00-Public/Configure/Configuration_Guides/H3C_S5560S-EI_S5560S-SI_S5500V3-SI_CG-6W102/10/201909/1227821_294551_0.htm)

Своими словами про обычное кольцо без interconnection nodes:

* RPL - ring protection links. Это резервная оптика, через которую данные в обычном режиме работы не передаются.
* RPL Owner - коммутатор с одной стороны резервного линка.
* RPL Neighbor - коммутатор с другого конца резервного линка до RPL owner
* Ring node - остальные коммутаторы в кольце. 

Состояния кольца erps отображается идентично на всех нодах:

* idle - все работает, резервный линк не используется, т.к. блокируется RPL Neighbor'ом
* protected - обрыв линка между нодами, используется линк RPL.
* pending - линк восстановлен, но переходное состояние на время заданного таймаута.

Как добавить vlan в кольцо

На примере extreme, где есть проверка ввода команд с описанием
```text
extreme # create vlan "v100"
* extreme # configure vlan v100 tag 100
* extreme # configure vlan v100 add ports 1,48 tagged
Make sure v100 is protected by ERPS. Adding ERPS ring ports to a VLAN could cause a loop in the network.
Do you really want to add these ports? (y/N) No
Operation cancelled.
* extreme # configure erps RING add protected vlan v100
Warning: Ring ports already configured are not added to VLAN
```

Т.е. сначала настроить, добавить в protected vlan, а потом прописать в порты кольца. На всех коммутаторах кольца:
```text
extreme # create vlan "v100"
* extreme # configure vlan v100 tag 100
* extreme # configure erps RING add protected vlan v100
Warning: Ring ports already configured are not added to VLAN
```

Добавить в порты на всех коммутаторах кольца. В примере это порты 1 и 48 на всех 4 коммутаторах:
```text
extreme1 # configure vlan v100 add ports 1,48 tagged
extreme2 # configure vlan v100 add ports 1,48 tagged
extreme3 # configure vlan v100 add ports 1,48 tagged
extreme4 # configure vlan v100 add ports 1,48 tagged
```

Сохранить конфигурацию
```text
* extreme1 # save
* extreme2 # save
* extreme3 # save
* extreme4 # save
```

## redundant port

Первый порт активный, второй отключен.
```text
extreme #configure port 1 redundant 2 link off
```

Ниже нет линка с 1 портом, потом 1 порт включается, а 2 автоматом отключается.
```text
extreme # sh ports redundant

Primary: 1,             Redundant: *2,   Link on/off option: OFF
        Flags: (*)Active, (!) Disabled, (g) Load Share Group

extreme # sh ports redundant

Primary: *1,            Redundant: 2,    Link on/off option: OFF
        Flags: (*)Active, (!) Disabled, (g) Load Share Group
```

## mtu

```text
extreme # configure jumbo-frame-size 9216
* extreme # enable jumbo-frame ports all
* extreme # configure ip-mtu 9216 vlan p2p
Warning: The IP-MTU size 9216 may be too large for jumbo-frame-size 9216
IP packets larger than 9194 bytes may be lost.
configure ip-mtu 9194 vlan p2p
```

## forwarding
```text
enable ipforwarding
```

## bgp

[How to enable Bidirectional Forwarding Detection (BFD) protection of BGP peering sessions](https://gtacknowledge.extremenetworks.com/articles/How_To/How-to-enable-Bidirectional-Forwarding-Detection-BFD-protection-of-BGP-peering-sessions)
Настроить bgp, bfd, включить пира.
```text
configure bgp AS-number 65010
configure bgp routerid 10.10.10.10
configure bgp maximum-paths 2
configure bgp add network 10.10.10.10/32
configure bgp add network 10.100.0.0/30
configure bgp add network 10.200.0.0/30
create bgp neighbor 10.100.0.1 remote-AS-number 65100
configure bgp neighbor 10.100.0.1 bfd on
enable bgp neighbor 10.100.0.1
create bgp neighbor 10.200.0.1 remote-AS-number 65200
configure bgp neighbor 10.200.0.1 bfd on
enable bgp neighbor 10.200.0.1
configure bgp neighbor 10.100.0.1 next-hop-self
configure bgp neighbor 10.200.0.1 next-hop-self
enable bgp
```
Смотреть
```
extreme # show bfd session
Neighbor         Interface      Clients  Detection  Status       VR
=============================================================================
10.100.0.1       p2p_100        -b----     750      Up           VR-Default
10.200.0.1       p2p_200        -b----     750      Up           VR-Default
=============================================================================
Clients Flag: b - BGP, m - MPLS, o - OSPF, s - Static, t - OTM
NOTE: All timers in milliseconds.
```
Команды для bgp
```text
show bgp neighbor
show bgp neighbor [peer_ip] received-routes all
show bgp neighbor [peer_ip] transmitted-routes all
```

## ecmp

Включить, указав нужный vr
```text
enable iproute sharing vr VR-Default
```

## evpn

[EVPN with iBGP Configuration Example](https://documentation.extremenetworks.com/exos_30.7/GUID-2E5C4051-51F8-4B1C-B4ED-760D1BB9C494.shtml)

Для случая, когда надо только L2 и не нужен mlag

 * `enable jumbo-frame ports all`
 * создать p2p вланы, включить для них bfd
 * включить ipforwarding и ecmp
 * `configure ip-mtu 9194 vlan p2p`
 * настроить ospf + bfd
 * настроить ibgp с capability l2vpn-EVPN
 * создать влан tenant, ipaddress для него не нужен
 * отключить igmp snooping для tenant
 * `virtual-network local-endpoint ipaddress` указать адрес на лупбеке.
 * добавить tenant в vni
 * `configure vlan tenant suppress arp-only` только после добавления в vni возможно
 * добавить tenant в порты

Смотреть
```text
show iproute
show ospf neighbor
show bgp neighbor
show bgp evpn evi
show bgp evpn mac
```

Несовместимо с mac-locking
```
Error: The VLAN cannot be added to the virtual network due to the following incompatibilities:
At least one of the member ports has MAC Locking feature enabled
```


## packet drops

```text
show port congestion
```

[Prevent packet drops](https://gtacknowledge.extremenetworks.com/articles/Solution/Prevent-packet-drops)


## edp

Помимо cdp и lldp есть протокол edp, который позволяет посмотреть соседей (ports all), а так же список vlan, которые прописаны на удаленном коммутаторе extreme (ports 1 detail).
```text
show edp ports all
show edp ports 1 detail
```

## mac-lock

[По инструкции](https://gtacknowledge.extremenetworks.com/articles/How_To/How-to-bind-a-single-mac-address-to-a-port)
ограничить кол-во мак-адресов на порту, указать статикой нужный

```text
enable mac-locking
enable mac-locking ports 5
configure mac-locking ports 5 first-arrival limit-learning 0
configure mac-locking ports 5 static add station d4:ca:6d:88:f5:50
```

## elrp

[По инструкции](https://gtacknowledge.extremenetworks.com/articles/How_To/How-to-configure-ELRP-to-disable-ports)
Отключить блокировку аплинков ( порт 1 ), включить рассылку фреймов для обнаружение петли в 10 влане

```text
enable elrp-client
configure elrp-client periodic vlan VLAN-10 ports 5 disable-port duration 30
configure elrp-client disable-port exclude 1
```
