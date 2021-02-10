[Документация vyos](https://docs.vyos.io/). Ниже для VyOS 1.3-rolling

The most up-do-date Rolling Release for AMD64 can be accessed using the following URL:
[https://downloads.vyos.io/rolling/current/amd64/vyos-rolling-latest.iso](https://downloads.vyos.io/rolling/current/amd64/vyos-rolling-latest.iso)
из документации по
[обновлению](https://docs.vyos.io/en/latest/image-mgmt.html#update-vyos)

## interface

Адрес на интерфейс и сабинтерфейсы:
```text 
set interfaces ethernet eth0 address '10.0.0.2/24'
set interfaces ethernet eth1 vif 100 address '10.0.100.1/24'
set interfaces ethernet eth1 vif 200 address '10.0.200.1/24'
set interfaces loopback lo address '10.9.8.1/32'
```

Смотреть
```text
show int
```

## arp

`arp -n`, но в документации `show arp`.

## vrrp

Через high-availability
```text
set high-availability vrrp group 1 authentication password '1'
set high-availability vrrp group 1 authentication type 'plaintext-password'
set high-availability vrrp group 1 interface 'eth0'
set high-availability vrrp group 1 virtual-address '10.0.0.1/24'
set high-availability vrrp group 1 vrid '1'
```
Смотреть, но нет адресов, только backup/master и таймеры
```text
show vrrp
```

## dhcp server

```text
set service dhcp-server shared-network-name [poolname] subnet 10.0.200.0/24 default-router '10.0.200.1'
set service dhcp-server shared-network-name [poolname] subnet 10.0.200.0/24 dns-server '8.8.8.8'
set service dhcp-server shared-network-name [poolname] subnet 10.0.200.0/24 dns-server '8.8.4.4'
set service dhcp-server shared-network-name [poolname] subnet 10.0.200.0/24 range 1 start '10.0.200.10'
set service dhcp-server shared-network-name [poolname] subnet 10.0.200.0/24 range 1 stop '10.0.200.100'
```

Смотреть
```text
sh dhcp server leases
```


## bw limits

Шейпер 1мбит
```text
set traffic-policy shaper 1m default bandwidth '1mbit'
set interfaces ethernet eth1 vif 100 traffic-policy out '1m'
```

Полисер на out
```text
set traffic-policy rate-control 10m bandwidth '10mbit'
set interfaces ethernet eth1 vif 200 traffic-policy out '10m'
```

Полисер на in
```text
set traffic-policy limiter in10m default bandwidth '10mbit'
set interfaces ethernet eth1 vif 200 traffic-policy out 'in10m'
```
Смотреть
```text
sh int ethernet [ifname] queue
```

## flow-accounting

netflow v5

```text
set system flow-accounting netflow engine-id '[id]'
set system flow-accounting netflow server [ip] port [port]
set system flow-accounting netflow version '5'
```

Интерфейсы. Собирается ingress, поэтому списком все что нужны.
```text
set system flow-accounting interface [ifname]
```

Смотреть
```
sh flow-accounting
sh flow-accounting interface [ifname]
```

## prefix-list

Дефолт
```text
set policy prefix-list DEFAULT rule 10 action 'permit'
set policy prefix-list DEFAULT rule 10 prefix '0.0.0.0/0'
```

Заказчик со спецификами из блока оператора
```text
set policy prefix-list CUSTOMER rule 10 action 'permit'
set policy prefix-list CUSTOMER rule 10 ge '25'
set policy prefix-list CUSTOMER rule 10 prefix '10.9.8.0/23'
```

Собственные префиксы
```text
set policy prefix-list MYSELF rule 10 action 'permit'
set policy prefix-list MYSELF rule 10 prefix '10.9.8.0/23'
```

## route-map

Для FRR в bgp нужен, чтоб отдать дефолт. Взято из
[https://blog.donatas.net/blog/2018/07/27/frr-bgp-default-originate/](https://blog.donatas.net/blog/2018/07/27/frr-bgp-default-originate/)
```text
set policy route-map PM-DEFAULT rule 10 action 'permit'
set policy route-map PM-DEFAULT rule 10 set as-path-prepend '[as] [as]'
set policy route-map PM-DEFAULT rule 10 set metric '200'
```

## bgp

Агрегированный маршрут для анонса операторам. Хотя бы один адрес с лупбека прописать в network.
```text
set protocols bgp [as] address-family ipv4-unicast aggregate-address 10.9.8.0/23 summary-only
set protocols bgp [as] address-family ipv4-unicast network 10.9.8.1/32
set protocols bgp [as] parameters router-id '10.9.8.1'
```

Настройка пира с домру, принять все префиксы без ограничений.
```text
set protocols bgp [as] neighbor [isp-ip] address-family ipv4-unicast prefix-list export 'MYSELF'
set protocols bgp [as] neighbor [isp-ip] address-family ipv4-unicast soft-reconfiguration inbound
set protocols bgp [as] neighbor [isp-ip] remote-as '9049'
comment protocols bgp [as] neighbor [isp-ip] 'DOMru'
```

Настройка пира с заказчиком, до которого две линии связи.
```text
set protocols bgp [as] neighbor [peer-ip-1] peer-group 'CUSTOMER1'
comment protocols bgp [as] neighbor 1[peer-ip-1] 'CUSTOMER1-link1'
set protocols bgp [as] neighbor [peer-ip-2] peer-group 'CUSTOMER1'
comment protocols bgp [as] neighbor [peer-ip-2] 'CUSTOMER1-link2'
set protocols bgp [as] peer-group CUSTOMER1 address-family ipv4-unicast default-originate route-map 'PM-DEFAULT'
set protocols bgp [as] peer-group CUSTOMER1 address-family ipv4-unicast maximum-prefix '10'
set protocols bgp [as] peer-group CUSTOMER1 address-family ipv4-unicast nexthop-self
set protocols bgp [as] peer-group CUSTOMER1 address-family ipv4-unicast prefix-list export 'DEFAULT'
set protocols bgp [as] peer-group CUSTOMER1 address-family ipv4-unicast prefix-list import 'CUSTOMER'
set protocols bgp [as] peer-group CUSTOMER1 address-family ipv4-unicast soft-reconfiguration inbound
set protocols bgp [as] peer-group CUSTOMER1 remote-as '[customer1-as]'
```

Смотреть
```text
show ip bgp summary
sh ip bgp neighbors [peer-ip] advertised-routes
sh ip bgp neighbors [peer-ip] received-routes
sh ip bgp neighbors [peer-ip] received-routes
sh ip route [prefix]
sh ip bgp [prefix]
```

## ntp

dns нужен в тч для обновления
```text
set system name-server '8.8.8.8'
set system ntp server 0.ru.pool.ntp.org
set system ntp server 1.ru.pool.ntp.org
set system ntp server 2.ru.pool.ntp.org
set system ntp server 3.ru.pool.ntp.org
set system time-zone 'Asia/Krasnoyarsk'
```

Смотреть
```text
sh ntp
```

## snmp

Мибы линукса, ничего специфичного нет.
```text
set service snmp community public authorization 'ro'
set service snmp community public client '[nms-ip]
set service snmp contact '[contact]'
set service snmp location '[location]'
```

## ssh

В документации рекомендуют по ключам, а не паролям.

## rpf-check

Для статических адресов, не для vrrp и dhcp.
```text
set interfaces ethernet [ifname] ip source-validation 'strict'
```

## ansible

[vyos.vyos.vyos_config](https://docs.ansible.com/ansible/latest/collections/vyos/vyos/vyos_config_module.html)

## fastnetmon

(краткое описание)[https://fastnetmon.com/fastnetmon-community-on-vyos-rolling-1-3/]
