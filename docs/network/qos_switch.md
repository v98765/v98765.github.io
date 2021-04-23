QoS на коммутаторах

## flow-control

Нужно отключать flow-control на коммутаторах. Подробнее [тут](https://v98765.github.io/network/flowcontrol/).

## cos или dscp

Dscp для маршрутизируемых сетей, т.к. является частью поля TOS в заголовке ip-пакета. Сos для L2 в пределах площадки, т.к. является частью L2 заголовка.
Очереди для cos настраиваются, когда из-за ограничений нет возможности проанализировать ip-заголовок. Такое может быть из-за mpls, например.
Чаще всего хватает dscp, поэтому там, где это требуется, настраивается dscp based qos.

## dscp таблица

DSCP Class | DSCP (bin) | DSCP (hex) | DSCP (dec) | ToS (dec) |ToS (hex)
---|---|---|---|---|---
none | 000000 | 0×00 | 0  | 0   | 0×00
cs1  | 001000 | 0×08 | 8  | 32  | 0×20
af11 | 001010 | 0×0A | 10 | 40	| 0×28
af12 | 001100 | 0×0C | 12 | 48	| 0×30
af13 | 001110 | 0×0E | 14 | 56	| 0×38
cs2	 | 010000 | 0×10 | 16 | 64	| 0×40
af21 | 010010 | 0×12 | 18 | 72	| 0×48
af22 | 010100 | 0×14 | 20 | 80	| 0×50
af23 | 010110 | 0×16 | 22 | 88	| 0×58
cs3	 | 011000 | 0×18 | 24 | 96	| 0×60
af31 | 011010 | 0×1A | 26 | 104	| 0×68
af32 | 011100 | 0×1C | 28 | 112	| 0×70
af33 | 011110 | 0×1E | 30 | 120	| 0×78
cs4  | 100000 | 0×20 | 32 | 128	| 0×80
af41 | 100010 | 0×22 | 34 | 136	| 0×88
af42 | 100100 | 0×34 | 36 | 144	| 0×90
af43 | 100110 | 0×26 | 38 | 152	| 0×98
cs5  | 101000 | 0×28 | 40 | 160	| 0xA0
ef   | 101110 | 0×2E | 46 | 184	| 0xB8
cs6	 | 110000 | 0×30 | 48 | 192	| 0xC0
cs7	 | 111000 | 0×38 | 56 | 224	| 0xE0

## очереди и приоритеты

У коммутаторов 4 или 8 очередей. При отключенном qos очереди обычно 2:

 * неприоритетная, где 95-100% буферов
 * приоритетная

Есть два режима работы шедулера, который передает данные из очередей:

 * Strict-Priority. Это оычно одна очередь (8ая), откуда данные передаются в первую очередь.
 * Weighted Round Robin (WRR). Все остальные очереди, откуда данные передаются в соотв. с весами, заданными явно или в пропорциях от bandwidth.

При переполнении неприоритетной очереди, данные дропаются, что фиксируется в счетчиках.
Настроить qos - выделить часть передаваемых данных в определенную очередь коммутатора, где потерь данных не будет.
Из-за уменьшения буферов в неприоритетной очереди количество потерь там увеличится.

Juniper EX
```text
ex2200> show class-of-service scheduler-map
Scheduler map: , Index: 2

  Scheduler: , Forwarding class: best-effort, Index: 21
    Transmit rate: 95 percent, Rate Limit: none, Buffer size: 95 percent, Buffer Limit: none, Priority: low
    Excess Priority: low
    Drop profiles:
      Loss priority   Protocol    Index    Name
      High            non-TCP         1          
      High            TCP             1          

  Scheduler: , Forwarding class: network-control, Index: 23
    Transmit rate: 5 percent, Rate Limit: none, Buffer size: 5 percent, Buffer Limit: none, Priority: low
    Excess Priority: low
    Drop profiles:
      Loss priority   Protocol    Index    Name
      High            non-TCP         1          
      High            TCP             1          
```
Extreme 670
```text
extreme # show qosprofile
    QP1    Weight =   1     Max Buffer Percent = 100
    QP8    Weight =   1     Max Buffer Percent = 100
extreme # show qosscheduler

Configured Scheduler (global) : Strict-Priority
```
Cisco 3750 при включенном mls qos с настройками по умолчанию. Делолтовые значения не посмотреть
```text
c3750#sh mls qos queue-set 1
Queueset: 1
Queue     :       1       2       3       4
----------------------------------------------
buffers   :      25      25      25      25
threshold1:     100     200     100     100
threshold2:     100     200     100     100
reserved  :      50      50      50      50
maximum   :     400     400     400     400
```

## где маркировать

Маркировать в корпсетях нужно на самих устройствах. Для телефонов это делается через провиженинг, а для видеокодеков - руками.
На коммутаторе включается qos, на портах `trust dscp`, настраивается mapping и размер очередей.
Для корпоративной сети вот такой вариант маркировки:

dscp dec | dscp class | desc
---|---|---
46 | ef | медиа (rtp)
24 | cs3 | сигнализация (sip,h323,unistim)
32 | cs4 | видео c кодеков(rtp)
34 | af41 | видео с телефона, софтового клиента, кодека (rtp)
0 | be |весь остальной трафик без маркировки.


Вариант маркировки для передачи данных между серверыми коммутаторами, когда подключены гипервизоры vmware esxi с TEP'ами NSX'а.

dscp dec | dscp class | desc
---|---|---
8 | cs1 | vmotion
24 | cs3 | iscsi
32 | cs4 | vxlan,geneve
0 | be |весь остальной трафик без маркировки

Маркировать на гипервизоре может и получится, но нет. Поэтому маркировка на leaf-коммутаторах по acl.
По ссылке [порты vmware](https://kb.vmware.com/s/article/2039095).
iscsi, vmotion - это tcp, a туннели vxlan,geneve - udp.
udp необходим более высокий приоритет. В случае vmotion можно немного потерять, подождать.

## класифиры

По умолчанию не все коммутаторы анализируют dscp в ip-заголовки и настроены на работу с полем cos. Например, у juniper ex
```text
ex2200> show class-of-service interface ge-0/0/1  
Physical interface: ge-0/0/1, Index: 130
Queues supported: 8, Queues in use: 4
  Scheduler map: , Index: 2
  Congestion-notification: Disabled

  Logical interface: ge-0/0/1.0, Index: 70
Object                  Name                   Type                    Index
Classifier              ieee8021p-default      ieee8021p                  11
```
На старых cisco и прочих c похожим cli коммутаторах выбор между 802.1p и dscp производится одной командой, похожей на `trust qos dscp`.
В extreme необходимо отключить 802.1p, иначе этот выбор будет приоритетнее, чем dscp
```text
disable dot1p examination ports all
enable diffserv examination ports all
```
В eltex вовсе делать ничего не нужно
```text
eltex-2324#sh qos 
Qos: Basic mode
Basic trust: dscp
CoS to DSCP mapping: disabled
DSCP to CoS mapping: disabled
```
В huawei необходимо на портах включить `trust dscp`, а на routed-интефейсах дополнительно еще `undo qos phb marking enable`.

## dscp mapping to queue

Нет какого-то универсального правила, в соотв. с которым dscp мапятся в очереди.
У всех вендоров это разные значения на разных моделях. Делать по своему усмотрению. Пример:

Очередь | dot1p | DSCP
---|---|---
1 | 0 | 0
2 | 1 | 8
3 | 2 | 16
4 | 3 | 24
5 | 4 | 32
6 | 5 | 40
7 | 6 | 48
8 | 7 | 56

extreme 670 по умолчанию:
```text
# show diffserv examination
CodePoint->QOSProfile mapping:
        00->QP1 01->QP1 02->QP1 03->QP1 04->QP1 05->QP1 06->QP1 07->QP1
        08*>QP1 09*>QP1 10*>QP1 11*>QP1 12*>QP1 13*>QP1 14*>QP1 15*>QP1
        16*>QP1 17*>QP1 18*>QP1 19*>QP1 20*>QP1 21*>QP1 22*>QP1 23*>QP1
        24*>QP1 25*>QP1 26*>QP1 27*>QP1 28*>QP1 29*>QP1 30*>QP1 31*>QP1
        32*>QP1 33*>QP1 34*>QP1 35*>QP1 36*>QP1 37*>QP1 38*>QP1 39*>QP1
        40*>QP1 41*>QP1 42*>QP1 43*>QP1 44*>QP1 45*>QP1 46*>QP1 47*>QP1
        48*>QP1 49*>QP1 50*>QP1 51*>QP1 52*>QP1 53*>QP1 54*>QP1 55*>QP1
        56->QP8 57->QP8 58->QP8 59->QP8 60->QP8 61->QP8 62->QP8 63->QP8
```
Т.е. нет никаких предустановленных правил, да и очереди всего две. Поэтому необходимо создать новые очереди, описать mapping.
```text
create qosprofile "qp2"
create qosprofile "qp3"
create qosprofile "qp4"
configure diffserv examination code-point 24 qosprofile qp3
configure diffserv examination code-point 32 qosprofile qp4
configure diffserv examination code-point 8 qosprofile qp2
```
eltex
```text
eltex-3324#sh qos map dscp-queue
Dscp-queue map:
     d1 : d2 0  1  2  3  4  5  6  7  8  9
     -------------------------------------
      0 :   01 01 01 01 01 01 01 01 01 02
      1 :   02 02 02 02 02 02 06 03 03 03
      2 :   03 03 03 03 06 04 04 04 04 04
      3 :   04 04 07 05 05 05 05 05 05 05
      4 :   06 07 07 07 07 07 07 07 06 06
      5 :   06 06 06 06 06 06 06 06 06 06
      6 :   06 06 06 06
eltex-2324#sh qos map dscp-queue
Dscp-queue map:
     d1 : d2 0  1  2  3  4  5  6  7  8  9
     -------------------------------------
      0 :   02 01 01 01 01 01 01 01 01 03
      1 :   03 03 03 03 03 03 07 04 04 04
      2 :   04 04 04 04 07 05 05 05 05 05
      3 :   05 05 07 06 06 06 06 06 06 06
      4 :   07 08 08 08 08 08 08 08 07 07
      5 :   07 07 07 07 07 07 07 07 07 07
      6 :   07 07 07 07
```
Juniper EX
```text
ex2200> show class-of-service classifier name dscp-default
Classifier: dscp-default, Code point type: dscp, Index: 7
  Code point         Forwarding class                    Loss priority
  000000             best-effort                         low
  000001             best-effort                         low
  000010             best-effort                         low
  000011             best-effort                         low
  000100             best-effort                         low
  000101             best-effort                         low
  000110             best-effort                         low
  000111             best-effort                         low
  001000             best-effort                         low
  001001             best-effort                         low
  001010             best-effort                         low
  001011             best-effort                         low
  001100             best-effort                         low
  001101             best-effort                         low
  001110             best-effort                         low
  001111             best-effort                         low
  010000             best-effort                         low
  010001             best-effort                         low
  010010             best-effort                         low
  010011             best-effort                         low
  010100             best-effort                         low
  010101             best-effort                         low
  010110             best-effort                         low
  010111             best-effort                         low
  011000             best-effort                         low
  011001             best-effort                         low
  011010             best-effort                         low
  011011             best-effort                         low
  011100             best-effort                         low
  011101             best-effort                         low
  011110             best-effort                         low
  011111             best-effort                         low
  100000             best-effort                         low
  100001             best-effort                         low
  100010             best-effort                         low
  100011             best-effort                         low
  100100             best-effort                         low
  100101             best-effort                         low
  100110             best-effort                         low
  100111             best-effort                         low
  101000             best-effort                         low
  101001             best-effort                         low
  101010             best-effort                         low
  101011             best-effort                         low
  101100             best-effort                         low
  101101             best-effort                         low
  101110             best-effort                         low
  101111             best-effort                         low
  110000             network-control                     low
  110001             network-control                     low
  110010             network-control                     low
  110011             network-control                     low
  110100             network-control                     low
  110101             network-control                     low
  110110             network-control                     low
  110111             network-control                     low
  111000             network-control                     low
  111001             network-control                     low
  111010             network-control                     low
  111011             network-control                     low
  111100             network-control                     low
  111101             network-control                     low
  111110             network-control                     low
  111111             network-control                     low
```
Cisco 3750, где 4 очереди (01..04) и еще у каждой очереди есть пара трешхолдов
```text
SWITCH#sh mls qos maps dscp-output-q
   Dscp-outputq-threshold map:
     d1 :d2    0     1     2     3     4     5     6     7     8     9 
     ------------------------------------------------------------
      0 :    02-01 02-01 02-01 02-01 02-01 02-01 02-01 02-01 02-01 02-01 
      1 :    02-01 02-01 02-01 02-01 02-01 02-01 03-01 03-01 03-01 03-01 
      2 :    03-01 03-01 03-01 03-01 03-01 03-01 03-01 03-01 03-01 03-01 
      3 :    03-01 03-01 04-01 04-01 04-01 04-01 04-01 04-01 04-01 04-01 
      4 :    01-01 01-01 01-01 01-01 01-01 01-01 01-01 01-01 04-01 04-01 
      5 :    04-01 04-01 04-01 04-01 04-01 04-01 04-01 04-01 04-01 04-01 
      6 :    04-01 04-01 04-01 04-01
```

## перемаркировка

Она может быть выключена или включена по умолчанию. Где-то ее можно включить на конкретных портах, где-то она работает только глобально.

На Cisco 3750 включена. Если нет политик для маркировки пакетов, то отключить
```text
no mls qos rewrite ip dscp
```
На extreme по умолчанию выключена
```text
disable dot1p examination ports all
```
На современных cisco делается все политиками, а не специфичными командами, поэтому проще стало с одной стороны. Достаточно в политике выставить `set dscp xx`.
Перемаркировка на extreme на примере, где все данные в дефолтовой очереди порта 47 перемаркируются:
```text
configure diffserv replacement qosprofile qp1 code-point 0
configure ports 47 qosprofile qp1
enable diffserv replacement port 47
```

## пример 1 для nexus 3000

[Cisco Nexus 3000 Series NX-OS QoS Configuration Guide](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/nexus3000/sw/qos/7x/b_3k_QoS_Config_7x/b_3k_QoS_Config_7x_chapter_010.html)
В коммутатор включены сервера с установленными гипервизорами vmware esxi.
Для примера приводится в виде тасков ансибла. Команды и таски описываются в том виде и в том порядке, в каком они есть в конфигурации устройства.
Поэтому это вместо листинга тоже подойдет.
Настройки выполнены по таблице ниже:

dscp dec | dscp class | qos-group | bw,% | desc
---|---|---|---|---
8 | cs1 | 1 | 25 |vmotion
24 | cs3 | 3 | 10 | iscsi
32 | cs4 | 4 | 40 | vxlan,geneve
0 | be | 0 | 25 | other

Весь трафик перемаркировывается, ему присвается соотвествующая qos-group'а, у которой приоритеты определены процентами от полосы.
```yaml
---
- name: configure qos for nexus
  hosts: cisco_n3k
  gather_facts: false
  serial: 1

  tasks:

    - name: acl
      ansible.netcommon.cli_config:
        config: |
          ip access-list acl-iscsi
            10 permit tcp any 10.0.0.0/8 eq 3260
            20 permit tcp 10.0.0.0/8 eq 3260 any
          ip access-list acl-vmotion
            10 permit tcp any 10.0.0.0/8 eq 8000
            20 permit tcp 10.0.0.0/8 eq 8000 any
          ip access-list acl-vxlan
            10 permit udp any 10.0.0.0/8 eq 4789
            20 permit udp any 10.0.0.0/8 eq 6081
            30 permit udp 10.0.0.0/8 eq 4789 any
            40 permit udp 10.0.0.0/8 eq 6081 any
      notify:
        - SAVE CONFIGURATION

    - name: class-map qos
      ansible.netcommon.cli_config:
        config: |
          class-map type qos match-all ISCSI
            match access-group name acl-iscsi
          class-map type qos match-all VXLAN
            match access-group name acl-vxlan
          class-map type qos match-all VMOTION
            match access-group name acl-vmotion
      notify:
        - SAVE CONFIGURATION

    - name: class-map type queuing
      ansible.netcommon.cli_config:
        config: |
          class-map type queuing QG1
            match qos-group 1
          class-map type queuing QG3
            match qos-group 3
          class-map type queuing QG4
            match qos-group 4
      notify:
        - SAVE CONFIGURATION

    - name: policy-map type qos SET-QG
      ansible.netcommon.cli_config:
        config: |
          policy-map type qos SET-QG
            class ISCSI
              set qos-group 3
              set dscp 24
            class VXLAN
              set qos-group 4
              set dscp 32
            class VMOTION
              set qos-group 1
              set dscp 8
            class class-default
              set dscp 0
      notify:
        - SAVE CONFIGURATION

    - name: policy-map type queuing DS (datastore iscsi)
      ansible.netcommon.cli_config:
        config: |
          policy-map type queuing DS
            class type queuing QG3
              bandwidth percent 90
            class type queuing class-default
              bandwidth percent 10
      notify:
        - SAVE CONFIGURATION

    - name: policy-map type queuing ESXI
      ansible.netcommon.cli_config:
        config: |
          policy-map type queuing ESXI
            class type queuing QG1
              bandwidth percent 25
            class type queuing QG3
              bandwidth percent 10
            class type queuing QG4
              bandwidth percent 40
            class type queuing class-default
              bandwidth percent 25
      notify:
        - SAVE CONFIGURATION

    - name: class-map type network-qos
      ansible.netcommon.cli_config:
        config: |
          class-map type network-qos cn_1
            match qos-group 1
          class-map type network-qos cn_2
            match qos-group 2
          class-map type network-qos cn_3
            match qos-group 3
          class-map type network-qos cn_4
            match qos-group 4
          class-map type network-qos cn_5
            match qos-group 5
          class-map type network-qos cn_6
            match qos-group 6
          class-map type network-qos cn_7
            match qos-group 7
      notify:
        - SAVE CONFIGURATION

    - name: policy-map type network-qos cq_system
      ansible.netcommon.cli_config:
        config: |
          policy-map type network-qos cq_system
            class type network-qos cn_1
              mtu 9216
            class type network-qos cn_2
              mtu 9216
            class type network-qos cn_3
              mtu 9216
            class type network-qos cn_4
              mtu 9216
            class type network-qos cn_5
              mtu 9216
            class type network-qos cn_6
              mtu 9216
            class type network-qos cn_7
              mtu 9216
            class type network-qos class-default
              mtu 9216
      notify:
        - SAVE CONFIGURATION

    - name: system qos
      ansible.netcommon.cli_config:
        config: |
          system qos
            service-policy type qos input SET-QG
            service-policy type queuing output ESXI
            service-policy type network-qos cq_system
      notify:
        - SAVE CONFIGURATION

    - name: clear counters
      ansible.netcommon.cli_command:
        command: clear counters

    - name: clear qos statistics
      ansible.netcommon.cli_command:
        command: clear qos statistics

  handlers:

    - name: SAVE CONFIGURATION
      cli_command:
        command: copy running-config startup-config
```

## пример 1 для extreme 670

В коммутатор включены сервера с установленными гипервизорами vmware esxi.
Для примера приводится в виде последовательности команд. Непонятно как автоматизировать редактирование acl.
Настройки выполнены по таблице ниже:

dscp dec | dscp class | qosprofile | desc
---|---|---|---|---
8 | cs1 | 2 | vmotion
24 | cs3 | 3 |  iscsi
32 | cs4 | 4 | vxlan,geneve
0 | be | 1 | other

```text
create qosprofile "qp2"
create qosprofile "qp3"
create qosprofile "qp4"
configure diffserv examination code-point 24 qosprofile qp3
configure diffserv examination code-point 32 qosprofile qp4
configure diffserv examination code-point 8 qosprofile qp2
configure diffserv replacement qosprofile qp1 code-point 0
configure diffserv replacement qosprofile qp2 code-point 8
configure diffserv replacement qosprofile qp3 code-point 24
configure diffserv replacement qosprofile qp4 code-point 32
disable dot1p examination ports all
enable diffserv examination ports all
enable diffserv replacement ports all
```

Весь трафик перемаркировывается политикой, ему присвается соотвествующий qosprofile.
Создать acl/политику `vi qacl.pol`
```text
entry QP2-vmotionsrc {
    if {
        source-address 10.0.0.0/8;
        protocol tcp;
        source-port 8000;
    } then {
        Qosprofile qp2;
        replace-dscp;
   }
}
entry QP2-vmotiondst {
    if {
        destination-address 10.0.0.0/8;
        protocol tcp;
        destination-port 8000;
    } then {
        Qosprofile qp2;
        replace-dscp;
   }
}
entry QP3-iscsisrc {
    if {
        source-address 10.0.0.0/8;
        protocol tcp;
        source-port 3260;
    } then {
        Qosprofile qp3;
        replace-dscp;
   }
}
entry QP3-iscsidst {
    if {
        destination-address 10.0.0.0/8;
        protocol tcp;
        destination-port 3260;
    } then {
        Qosprofile qp3;
        replace-dscp;
   }
}
entry QP4-vxlansrc {
    if {
        source-address 10.0.0.0/8;
        protocol udp;
        source-port 4789;
    } then {
        Qosprofile qp4;
        replace-dscp;
   }
}
entry QP4-vxlandst {
    if {
        destination-address 10.0.0.0/8;
        protocol udp;
        destination-port 4789;
    } then {
        Qosprofile qp4;
        replace-dscp;
   }
}
entry QP4-genevesrc {
    if {
        source-address 10.0.0.0/8;
        protocol udp;
        source-port 6081;
    } then {
        Qosprofile qp4;
        replace-dscp;
   }
}
entry QP4-genevedst {
    if {
        destination-address 10.0.0.0/8;
        protocol udp;
        destination-port 6081;
    } then {
        Qosprofile qp4;
        replace-dscp;
   }
}
```
Проверить синтаксис, включить для всех портов
```text
check policy qacl
configure access-list qacl any
```

## пример 2 для extreme 670

Коммутатор является транзитным. Перемаркировка трафика не производится, кроме недоверенных портов, на которых она сбрасывается в 0.
Для примера приводится в виде последовательности команд.
Настройки выполнены по таблице ниже:

dscp dec | dscp class | qosprofile
---|---|---|---|---
8 | cs1 | 2
24 | cs3 | 3
32 | cs4 | 4
0 | be | 1

```text
create qosprofile "qp2"
create qosprofile "qp3"
create qosprofile "qp4"
configure diffserv examination code-point 24 qosprofile qp3
configure diffserv examination code-point 32 qosprofile qp4
configure diffserv examination code-point 8 qosprofile qp2
configure diffserv replacement qosprofile qp1 code-point 0
disable dot1p examination ports all
enable diffserv examination ports all
```

Недоверенный порт
```text
configure ports 47 qosprofile qp1
enable diffserv replacement port 47
```
