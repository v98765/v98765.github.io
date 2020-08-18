# learning

Коммутатор вносит в таблицу mac-адресов src адреса из каждого полученного фрейма.
При отсутствии в таблице записи о dst mac-адресе, фрейм будет отправлен во все порты, включенные в vlan, указанный в заголовке фрейма.
Во все, кроме порта с которого получен фрейм.

## ethernet frame

Field | Field Length in Bytes | Description
---|---|---
Preamble | 7 | Synchronization
Start Frame Delimiter (SFD) | 1 | Signifies that the next byte begins the Destination MAC field
Destination MAC address | 6 | Identifies the intended recipient of this frame
Source MAC address | 6 | Identifies the sender of this frame
Length | 2 | Defines the length of the data field of the frame (either length or type is present, but not both)
Type | 2 | Defines the type of protocol listed inside the frame (either length or type is present, but not both)
Data and Pad | 46–1500 | Holds data from a higher layer, typically a Layer 3 PDU (generic), and often an IP packet
Frame Check Sequence (FCS) | 4 | Provides a method for the receiving NIC to determine whether the frame experienced transmission errors

## switching

Два режима: Store-and-Forward и Cut-Through. Во втором случае все фреймы с ошибками, например с FCS, передаются на следующее устройство.

[Цитата](https://community.mellanox.com/s/article/switch-forwarding---160---store-and-forward--vs---cut-through-x)

>Store-and-Forward is used to describe a functionality where a switch receives a complete packet, stores it, and only then forwards it.
>since the switch make forwarding decisions based on the destination address which is at the header of the packet, the switch can make the forwarding decision before receiving the complete packet, this process is called cut-through, the switch forwards part of the packet before receiving the complete packet.
>Cut-through allows lower latency and saves buffer space, but if an error occurred in the packet while utilizing cut-through, the packet will be forwarded with an error, alternatively, utilizing store-and-forward allows the switch to drop erroneous packets.

## flow-control

![flow-control](/img/flow.gif) "flow-contol")

## aging

Время хранения хэша в памяти по умолчанию 5 минут. В хеше обычно хранятся только юникастовые адреса, а для мультикастовых групповых адресов есть igmp-таймеры.
В cisco можно сделать статическую запись для мультикастового адреса, но это исключение.
Когда на коммутаторе включен протокол STP, то при получении TCN таблица mac-адресов очищается.
Использование port-security на портах доступа автоматически создает статические записи mac-адресов без aging-time.
Если коллизий не возникает, то для уменьшения случаев unknown unicast необходимо увеличить aging-time и
настроить должным образом STP для предотвращения очистки записей из "таблицы".

# forwarding

### multicast

Для зарезервированных dst групповых адресов диапазон 0100.5E00.0000 по 0100.5E7F.FFFF.
Многоадресная рассылка во все порты vlan'а, если нет ограничений со стороны протокола igmp. Кроме порта с которого получен фрейм, конечно.
Если в 8 бите первого октета адреса есть 1, то это мультикастовый адрес. Если 0, то юникастовый. Перевести hex в bin, например, 01 - 1, 03 - 11.
Для группового адреса 0100.5E00.0000 протокол igmp может отработать и скопировать фреймы в разные порты, подписавшиеся на группу по igmp протоколу,
а для адреса Microsoft NLB кластера 03bf.0a00.0002 нет и коммутатор отправит копии фрейма во все порты vlan'а. 
Чтобы ограничить рассылку несколькими портами в cisco в нарушение rfc можно делать статические записи на мультикастовые адреса.
Подробнее в заметке по [NLB](/network/nlb.md).

### unknown unicast

Запись хранится "в таблице mac-адресов" не более aging-time в случае, когда фреймов с подобным src адресом не передавалось. 
Фактически запись хранится в памяти ASIC'а в виде хэша, что определяет максимально допустимое число mac-адресов в памяти коммутатора.
Поскольку хеш - это не только mac-адрес, но и номер vlan id, то существует вероятность коллизии хешей, особенно когда много похожих адресов в одном vlan,
например, адресов ТВ-приставок. В этом случае src адрес не может быть изучен коммутатором, т.е. возникает unknown unicast.
Другими словами, unknown unicast это рассылка фрейма с юникастовым dst адресом во все порты vlan'а, кроме порта с которого получен фрейм,
когда dst адрес не мог быть изучен ранее.

### broadcast

Dst адрес ffff.ffff.ffff. Коммутатор делает рассылку во все порты vlan'а, кроме порта с которого получен фрейм.

### unicast

Dst адрес изучен, есть в "таблице mac-адресов" коммутатора и ему соответствует определенный физический или логический интерфейс коммутатора, aging-time таймер не истек. Коммутатор при получении фрейма с известным dst передает его в порт согласно "таблице".

## port isolation
Так или switchport protected или еще как называется.
Фреймы, полученные из порта с такой настройкой, не будут передаваться в порт с аналогичной настройкой.
Довольно часто так настраивают операторы downlink порты, чтобы исключить любое взаимодействие между абонентами в пределах коммутатора.
Тогда вся коммутация возможна только между портом абонента и магистральным портом.
