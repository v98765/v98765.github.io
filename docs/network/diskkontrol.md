[Документация v3](http://distkontrol.ru/upload/RukovodstvoUSBoverIPv3.pdf).
[История изменений](http://distkontrol.ru/index.php/history-dk).
Информация по прошивке:

---|---
name | DistKontrolUSB.3.14
url | http://www.distkontrol.ru/img/distkontrolusb.img
description | USB over IP / v.3.14
sha1 | `4ba84db362ed11f105aaad742d0ea2776f66357c`
size | 688017440

Настройка отказоустойчивого включения [diskkintrol](http://www.distkontrol.ru/) двумя портами.

нерабочий вариант:

* создать на eth0 и eth1 интерфейсы vlan.
* создать интерфейс bond, указать там в slaves оба созданных интерфейса, выбрать один в качестве мастера, прописать ip-адресацию.

Ошибка:

> slaves: The value 'eth0.10,eth1.10' doesn't match the pattern '^(((eth|venet|wlan)\d+|(en|veth|wl)\S+),)*((eth|venet|wlan)\d+|(en|veth|wl)\S+)$' (code=0).

рабочий вариант:

* удалить один из интерфейсов и не применяя конфигурацию создать bond интерфейс с удаленным интерфейсом
* создать интерфейс vlan, выбрав parent interface `bond0`, прописать адрес, применить конфигурацию
* подключиться на новый адрес, удалить eth, применить конфигурацию, добавить интерфейс в bond в режиме active-backup

По невыясненным обстоятельствам резервирование работает когда в качестве активного eth выбран eth0.
Иногда помогает применить настройки корректно отключение по питанию.

В режиме active-backup mac-адреса сетевых карт устройства одинаковые. Интерфейсу vlan присвается тот же mac-адрес.

