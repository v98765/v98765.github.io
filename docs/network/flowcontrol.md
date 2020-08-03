# Польза от flowcontrol
Когда буфер сетевой карты или коммутатора переполнен, в сеть отправляется фрейм PAUSE и удаленная сторона прекращает передачу данных,
накапливает в своей очереди данные.
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
