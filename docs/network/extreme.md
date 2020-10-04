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
