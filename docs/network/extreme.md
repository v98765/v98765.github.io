[Документация extreme](https://www.extremenetworks.com/support/documentation/) и [ExtremeXOS 30.7](https://www.extremenetworks.com/support/documentation/extremexos-30-7/) 

## reset

Сбросить конфигурацию. Сразу после ввода команды требуется перезагрузка.

```text
unconfigure switch all
```

## vlan

Создать
```text
create vlan myvlan tag 10
```
Переименовать
```text
configure vlan myvlan name notmyvlan
```
Добавить в порт нетегированным. untagged по умолчанию
```text
configure vlan myvlan add port 1 [untagged]
```
Добавить тегированным
```text
configure vlan myvlan add port 1 tagged
```
Посмотреть в каких портах влан
```text
show vlan myvlan
```
Посмотреть какие вланы в порту
```text
show vlan port 1
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
configure jumbo-frame-size 9216
enable jumbo-frame ports all
configure ip-mtu 9216 vlan p2p
Warning: The IP-MTU size 9216 may be too large for jumbo-frame-size 9216
configure ip-mtu 9194 vlan p2p
```

## forwarding
```text
enable ipforwarding
enable ipforwarding p2p
```

## bgp

[How to enable Bidirectional Forwarding Detection (BFD) protection of BGP peering sessions](https://gtacknowledge.extremenetworks.com/articles/How_To/How-to-enable-Bidirectional-Forwarding-Detection-BFD-protection-of-BGP-peering-sessions)
Настроить bgp, bfd, включить пира.
```text
enable bfd vlan p2p
configure bfd vlan p2p receive-interval 250 transmit-interval 250
```

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

The following are limitations for EVPN:

* EVPN functionality is not supported between switches running ExtremeXOS 30.2 and switches running earlier ExtremeXOS versions. The earlier versions rely on auto-creation of IBGP peers, which is disabled functionality in ExtremeXOS 30.2. However, the proprietary AFI are supported and can be used to establish tunnels to RTEPs so that native VXLAN functionality using data plane learning functions is supported.
* A maximum of 1,024 EVI instances are supported.
* IPv6 Type 2 routes are not supported.
* Stacking is not supported.
* ExtremeXOS only supports asymmetric routing model.
* Configuring VMANs as VXLAN tenant VLANs is not supported.
* Anycast gateway is not supported.
* ExtremeXOS does not advertise default gateway extended community.
* Multi-hop BFD is not supported.
* Peer-group configuration for L2VPN-EVPN address family is not supported.
* If silent hosts are expected, static ARP/FDB should be created on tenant VLANs for these hosts. To configure static ARP entries it is necessary to configure IP address on tenant VLANs.

For Type 5 routes, the following are not supported:
* Switching through a VXLAN tunnel to a remote L3 Anycast gateway.
* Default VRs.

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
```text
Error: The VLAN cannot be added to the virtual network due to the following incompatibilities:
At least one of the member ports has MAC Locking feature enabled
```

## vxlan flood mode
Цитата:

In ExtremeXOS, a virtual network operates in one of the following flood modes:

* Flood mode standard (default)
* Flood mode explicit

These modes determine the way the remote endpoints are created/learned, and the way the tenant BUM traffic is handled.

Flood Mode Standard

The remote endpoints dynamically are learned using OSPF VXLAN extensions (see OSPFv2 VXLAN Extensions) or Multiprotocol BGP (MBGP) support for VXLAN (see Multiprotocol Border Gateway Protocol (MBGP) Support for VXLAN). Alternately, remote endpoints can be statically configured and attached to a virtual network using the CLI.
The tenant BUM traffic is head-end replicated to each of the configured/learned remote endpoints on that virtual network.

Flood Mode Explicit

The remote endpoints can only be statically configured by attaching them to BUM FDB entries using the CLI.
The tenant BUM traffic is head-end replicated to each of the configured remote endpoints for the corresponding FDB entry.
By explicitly adding BUM FDB entries, this mode provides flexibility in splitting the BUM entries to individual broadcast, unknown unicast, and unknown multicast entries.


Точка А
```text
create virtual-network "vni2" flooding explicit-remotes
configure virtual-network "vni2" vxlan vni 101
configure virtual-network "vni2" add vlan tenant_a_b
create fdb broadcast vlan tenant_a_b vxlan ipaddress 10.0.0.Б
create fdb unknown-multicast vlan tenant_a_b vxlan ipaddress 10.0.0.Б
create fdb unknown-unicast vlan tenant_a_b vxlan ipaddress 10.0.0.Б
```
Точка Б
```text
create virtual-network "vni2" flooding explicit-remotes
configure virtual-network "vni2" vxlan vni 101
configure virtual-network "vni2" add vlan tenant_a_b
create fdb broadcast vlan tenant_a_b vxlan ipaddress  10.0.0.А
create fdb unknown-multicast vlan tenant_a_b vxlan ipaddress 10.0.0.А
create fdb unknown-unicast vlan tenant_a_b vxlan ipaddress 10.0.0.А
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

## passwd

Для информации, что подобное настраивается
```text
configure account all password-policy lockout-on-login-failures off
configure account all password-policy char-validation none
configure account all password-policy min-length none
```
Для смены пароля локального пользователя необходимо знать текущий пароль, либо удалить учетку целиком.
При этом на коммутаторе должна до этого быть как минимум еще одна учетка с административными правами.
```text
extreme# configure account admin
Current user's password:
New password:
Reenter password:
```

## radius

```text
configure radius mgmt-access primary server [radius-ip] 1812 client-ip [my-ip] vr VR-Default
```
Ввести ключ
```text
configure radius mgmt-access primary shared-secret
Note: Shared secrets created with this version of EXOS are not compatible with EXOS 21.x and earlier.
Secret:
Reenter Secret:
```
Включить радиус для mgmt-access
```text
enable radius mgmt-access
```

## ntp

Через какой влан доступен ntp сервер. Проще через все

```text
enable ntp vlan all
configure ntp server add [ntp1-ip] vr VR-Default
configure ntp server add [ntp2-ip] vr VR-Default
```

## sntp

```text
configure sntp-client primary [ntp1-ip] vr VR-Default
configure sntp-client secondary [ntp2-ip] vr VR-Default
enable sntp-client


## ansible

[Manage Extreme Networks EXOS](https://docs.ansible.com/ansible/latest/collections/community/network/exos_config_module.html)

Инвентори файл inventory
```text
[extreme:vars]
ansible_network_os=exos
ansible_become=no
ansible_connection=network_cli
ansible_python_interpreter=~/base/bin/python

[extreme]
extreme_switch1 ansible_host=10.9.8.7
```
Playbook для проверки разного в качестве примера.
```yaml
---
- name: must have configuration
  hosts: extreme
  gather_facts: false

  tasks:

    - name: sntp clinet
      community.network.exos_config:
        lines:
          - configure sntp-client primary 10.0.0.1 vr VR-Default
          - configure sntp-client secondary 10.0.0.2 vr VR-Default
          - enable sntp-client

    - name: timezone
      community.network.exos_config:
        lines: configure timezone name MSK 3 autodst

    - name: Save running to startup when modified
      community.network.exos_config:
        save_when: changed
```

`gather_facts: false` т.к. если true, то проверяет вланы и тп.
