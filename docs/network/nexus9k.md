Устройство n9k необходимо рассматривать в схемах Multi-site evpn. Предполагается, что фабрика настраивается с помощью таких продуктов как:

* Nexus Dashboard, включающий в себя продукты из списка ниже
* DataCenter Network Manager (DCNM)
* Multi-Site Orcestrator (MSO)

Вебинары на тему Cisco EVPN. Пароль там где просит `Engage2021`. Какие-то ссылки могут быть нерабочими.
[ЦОД: вычислительные мощности, сеть и управление](https://www.cisco.com/c/ru_ru/training-events/on-demand-webinars.html)

## Основные материалы


* [Cisco Nexus 9000 Series NX-OS VXLAN Configuration Guide, Release 9.3(x)](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/nexus9000/sw/93x/vxlan/configuration/guide/b-cisco-nexus-9000-series-nx-os-vxlan-configuration-guide-93x/b-cisco-nexus-9000-series-nx-os-vxlan-configuration-guide-93x_chapter_01001.html)
* [Cisco Nexus 9000 Series NX-OS Layer 2 Switching Configuration Guide, Release 9.3(x)](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/nexus9000/sw/93x/layer-2-switching/configuration/guide/b-cisco-nexus-9000-nx-os-layer-2-switching-configuration-guide-93x.html)
* [VXLAN BGP EVPN  based Multi-Site Extended](https://www.ciscolive.com/c/dam/r/ciscolive/us/docs/2019/pdf/5eU6DfQV/TECDCN-2110.pdf)
* [VXLAN EVPN Multi-Site Design and Deployment White Paper](https://www.cisco.com/c/en/us/products/collateral/switches/nexus-9000-series-switches/white-paper-c11-739942.html)

## DCNM

Аплаенс с виртуальной машиной подключается на каждом сайте в OOBM сеть. При возникновении проблем [tshoot](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/sw/11_x/troubleshooting/cisco_dcnm_troubleshooting_guide/device_discovery.html)
Настраивать по [инструкции](https://www.cisco.com/c/en/us/td/docs/dcn/dcnm/1151/configuration/lanfabric/cisco-dcnm-lanfabric-configuration-guide-1151/managing-greenfield-vxlan-fabric.html).

> Enable Bootstrap - Select this check box to enable the bootstrap feature. Bootstrap allows easy day-0 import and bring-up of new devices into an existing fabric. Bootstrap leverages the NX-OS POAP functionality.

## storm-control

Начиная с 9.3.6 работает иначе. Более не учитываются интерфейсы `fabric-tracking`. Включится, когда сумма рейта входящего броадкаста по трем интерфейсам с `dci-tracking` за интервал 3.9мс превысит установленный трешхолд.

> Cisco NX-OS Release 9.3(6) and later releases optimize rate granularity and accuracy. Bandwidth is calculated based on the accumulated DCI uplink bandwidth, and only interfaces tagged with DCI tracking are considered. (Prior releases also include fabric-tagged interfaces.) In addition, granularity is enhanced by supporting two digits after the decimal point. These enhancements apply to the Cisco Nexus 9300-EX, 9300-FX/FXP/FX2/FX3, and 9300-GX platform switches.

## Таймеры

Есть таймер vpс `delay restore interface-vlan`, который смотреть в `show vpc brief`.
Есть `Multi-Site delay-restore` таймер, который смотреть для BGW в `show nve interface nve 1 detail`
```text
Multi-Site delay-restore time: 180 seconds
Multi-Site delay-restore time left: 0 seconds
```

## hmm

host mobility manager (hmm). ?

[Chapter: Configuring VXLAN BGP EVPN](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/nexus9000/sw/93x/vxlan/configuration/guide/b-cisco-nexus-9000-series-nx-os-vxlan-configuration-guide-93x/b-cisco-nexus-9000-series-nx-os-vxlan-configuration-guide-93x_chapter_0101.html)
Пример есть в более старой версии документации:
```text
/*** Advertise host tenant routes as evpn type-5 routes for interoperability. ***/
switch(config)# route-map permitall permit 10 
switch(config)# router bgp 1001
switch(config-router)# vrf vni-491830
switch(config-router-vrf)# address-family ipv4 unicast
switch(config-router-vrf-af)# advertise l2vpn evpn
switch(config-router-vrf-af)# redistribute hmm route-map permitall
```

## bgw backup

Есть ограничение на выбор оборудования для border gateway (BGW), на котором обязательно необходимо делать [backup через vpc peer-link](https://www.cisco.com/c/en/us/support/docs/switches/nexus-9000-series-switches/214624-configure-system-nve-infra-vlans-in-vxla.html)

> Ensure the system nve infra-vlans command is in place on Nexus 9000 platforms with CloudScale ASIC (Tahoe) like the Nexus 9300 Switches which end in EX, FX and FX2 to specify the VLAN can act as an uplink and properly forward the frames with VXLAN encapsulation over the vPC peer-link."

В мануале это кратко описано как Configuring Peer Link as Transport in Case of Link Failure, возможное только на указанном оборудовании из списка в [Guidelines and Limitations for VXLAN EVPN Multi-Site](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/nexus9000/sw/93x/vxlan/configuration/guide/b-cisco-nexus-9000-series-nx-os-vxlan-configuration-guide-93x/b-cisco-nexus-9000-series-nx-os-vxlan-configuration-guide-93x_chapter_01001.html#reference_imd_jvs_sgb)
