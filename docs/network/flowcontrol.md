## Польза от flowcontrol
Когда буфер сетевой карты или коммутатора переполнен, в сеть отправляется фрейм PAUSE и удаленная сторона прекращает передачу данных,
накапливает в своей очереди данные.

![flow-control](/img/flow.gif)

Включенный на коммутаторах flowcontrol или рекомендации по его включению от производителя могут косвенно
свидетельствовать об аппаратных ограничениях сетевого оборудования.
Таким образом, flowcontrol позволяет частично избежать потери данных.
Dell для iSCSI рекомендует коммутаторы с буфером не менее 9MB и возможностью включения flow-control rx.

[dell : Using Ethernet Pause Frames for Flow Control ](https://www.dell.com/support/manuals/ru/ru/rubsdt1/force10-s4048-on/s4048_on_9.9.0.0_config_pub-v1/using-ethernet-pause-frames-for-flow-control?guid=guid-3f29e829-1674-4a4b-8a5a-2605b26678b9&lang=en-us)

> Ethernet Pause Frames allow for a temporary stop in data transmission. A situation may arise where a sending device may transmit data faster than a destination device can accept it. The destination sends a PAUSE frame back to the source, stopping the sender’s transmission for a period of time.
> An Ethernet interface starts to send pause frames to a sending device when the transmission rate of ingress traffic exceeds the egress port speed. The interface stops sending pause frames when the ingress rate falls to less than or equal to egress port speed.
> The globally assigned 48-bit Multicast address 01-80-C2-00-00-01 is used to send and receive pause frames. To allow full-duplex flow control, stations implementing the pause operation instruct the MAC to enable reception of frames with destination address equal to this multicast address.
> The PAUSE frame is defined by IEEE 802.3x and uses MAC Control frames to carry the PAUSE commands. Ethernet pause frames are supported on full duplex only.
> If a port is over-subscribed, Ethernet Pause Frame flow control does not ensure no-loss behavior.
> Restriction: Ethernet Pause Frame flow control is not supported if PFC is enabled on an interface.

## Вред от flowcontrol
Еще старый cisco 3550 предупреждал об этом:
```text
cat3550(config)#mls qos
QoS: ensure flow-control on all interfaces are OFF for proper operation.
```
QoS и flowcontrol - это взаимоисключающие настройки сетевого оборудования.
Отключение может привести к деградации прочих сервисов, если окажется что буфера коммутатора недостаточно.


## monitoring

[EtherLike-MIB::dot3PauseTable](http://www.circitor.fr/Mibs/Html/E/EtherLike-MIB.php#dot3PauseTable). Не во всех коммутаторах есть эти счетчики.
При получении pause от устройства увеличивается счетчик dot3InPauseFrames и dot3HCInPauseFrames

## рекомендации netapp

[vmware-netapp-and-vmware-vsphere-storage-best-practices.pdf](https://www.vmware.com/content/dam/digitalmarketing/vmware/en/pdf/partners/netapp/vmware-netapp-and-vmware-vsphere-storage-best-practices.pdf) стр 25. 

> For modern network equipment, especially 10GbE equipment, NetApp recommends turning off flow control and allowing congestion management to be performed higher in the network stack.
> For older equipment, typically GbE with smaller buffers and weaker buffer management, NetApp recommends configuring the endpoints, ESX servers, and NetApp arrays with the flow control set to "send."


[Configuring your network for best performance](https://docs.netapp.com/ontap-9/index.jsp?topic=%2Fcom.netapp.doc.exp-iscsi-esx-cpg%2FGUID-BE5FD8D6-C2E4-49F3-9040-DF1373A2E325.html)

> Connect the host and storage ports to the same network.
> It is best to connect to the same switches. Routing should never be used.

> Select the highest speed ports available, and dedicate them to iSCSI.
> 10 GbE ports are best. 1 GbE ports are the minimum.

> Disable Ethernet flow control for all ports.
> You should see the ONTAP 9 Network Management Guide for using the CLI to configure Ethernet port flow control.

> Enable jumbo frames (typically MTU of 9000).
> All devices in the data path, including initiators, targets, and switches, must support jumbo frames. Otherwise, enabling jumbo frames actually reduces network performance substantially.
