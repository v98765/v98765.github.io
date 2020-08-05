CUBE - это что-то вроде маршрутизатора в телефонии или ip-to-ip шлюза, как писали ранее. [Cisco Unified Border Element Configuration Guide](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/voice/cube/configuration/cube-book.html).
Ниже настройки для IOS 15.x.
Пусть будет шлюзом между двумя телефонными станциями host1 и host2.
Каждая станция имеет маршрут, доступ до адреса на Loopback0 CUBE, но не до адресов друг друга.
```text
voice rtp send-recv
!
voice service voip
 ip address trusted list
  ipv4 <host1> 255.255.255.255
  ipv4 <host2> 255.255.255.255
 allow-connections sip to sip
 fax protocol t38 version 0 ls-redundancy 0 hs-redundancy 0 fallback none
 sip
  bind control source-interface Loopback0
  bind media source-interface Loopback0
```
диалпиры на host1 и host2. Универсальные правила создавать лучше не надо, чтобы вызовы не петляли между шлюзами, если какой-то номер недоступен.
Это касается не только cube, но и станций host1 и host2. huntstop - прекратить поиск альтернативных маршрутов.
Возможна ситуация, когда sip-транк до host2 по протоколу udp, а до host1 по tcp и порт нестандартный. Будет работать и так.
На станциях настраиваются транки на адрес lo0 cube без авторизации. Хосты станций указаны в trusted листе на cube.
```text
dial-peer voice 100 voip
 huntstop
 destination-pattern [567]..
 session protocol sipv2
 session target ipv4:<host1>:5080
 session transport tcp
 codec g711alaw
!
dial-peer voice 200 voip
 huntstop
 destination-pattern [234]..
 session protocol sipv2
 session target ipv4:<host2>
 session transport udp
 dtmf-relay rtp-nte
 codec g711alaw
```
Когда необходимо сделать резервный диалпир на host3, то убрать `huntstop` в основном и прописать `preference 10` на резервном диалпире.
```text
dial-peer voice 200 voip
 destination-pattern [234]..
 session protocol sipv2
 session target ipv4:<host2>
 session transport udp
 dtmf-relay rtp-nte
 codec g711alaw
dial-peer voice 300 voip
 huntstop
 preference 10
 destination-pattern [234]....
 session protocol sipv2
 session target ipv4:<host3>
 session transport udp
 dtmf-relay rtp-nte
 codec g711alaw
```
Обычно на cube шлюзах не делают транскодинг, ну только если он не подключен к CUBE в качестве фермы. Можно не указывать в явном виде кодек, указав:
```text
 codec transparent
```
Просмотр звонков
```text
sh voip rtp connections
```
Отладка
```text
debug voip ccapi inout
debug ccsip
```
