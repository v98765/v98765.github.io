# Про juniper ex

## vlan, trunk, access

Группы удобны. Настроенный порт в транк без вланов оказывается включен во все вланы.
Если указан список, то только в них. Поэтому есть vlan default, в котором acl дропает все.
```text
groups {
    trunk {
        interfaces {
            <*> {
                mtu 9216;
                ether-options {
                    no-flow-control;
                }
                unit 0 {
                    family ethernet-switching {
                        port-mode trunk;
                        vlan {
                            members default;
                        }
                    }
                }
            }
        }
    }
    access {
        interfaces {
            <*> {
                mtu 9216;
                ether-options {
                    no-flow-control;
                }
                unit 0 {
                    family ethernet-switching {
                        port-mode access;
                    }
                }
            }
        }
    }
}
```
Нужен access
```text
set interfaces ge-0/0/0 apply-groups access
```
Нужен trunk
```text
set interfaces ge-0/0/1 apply-groups trunk
```
Фильтр для vlan default
```text
> show configuration vlans default    
vlan-id 1;
filter {
    input Drop;
}

{master:0}
> show configuration firewall family ethernet-switching 
filter Drop {
    term all {
        then discard;
    }
}

{master:0}
```
Создать vlan. Название всегда с буквы и для простоты пишу так, а описание в описании.
```text
set vlans v30 vlan-id 30 desciption "dmz_1"
set vlans v40 vlan-id 40 desciption "dmz_2"
```
Для access порта, который уже в группе access. Вланы числами удобнее.
```text
set interfaces ge-0/0/0 unit 0 family ethernet-switching vlan members 30
```
Для порта в группе trunk
```text
set interfaces ge-0/0/1 unit 0 family ethernet-switching vlan members 30
set interfaces ge-0/0/1 unit 0 family ethernet-switching vlan members 40
```

## lacp
Группы можно делать, но тут проще руками.
Нужно какое-то количество aggregated-devices указать, указать в физических интерфейсах логический ae, настроить ae.
Забегая вперед, для dhcp-snooping ae не является доверенным портом по умолчанию.
```text
> show configuration chassis                 
aggregated-devices {
    ethernet {
        device-count 6;
    }
}
alarm {
    management-ethernet {
        link-down ignore;
    }
}

{master:0}
> show configuration interfaces ge-0/0/10 
ether-options {
    802.3ad ae0;
}

{master:0}
> show configuration interfaces ae0          
mtu 9216;
aggregated-ether-options {
    no-flow-control;
    lacp {
        active;
        periodic slow;
    }
}
unit 0 {
    family ethernet-switching {
        port-mode trunk;
        vlan {
            members default;
        }
    }
}

{master:0}
> show lacp interfaces 
Aggregated interface: ae0
    LACP state:       Role   Exp   Def  Dist  Col  Syn  Aggr  Timeout  Activity
      ge-0/0/10      Actor    No   Yes    No   No   No   Yes     Slow    Active
      ge-0/0/10    Partner    No   Yes    No   No   No   Yes     Fast   Passive
      ge-1/0/10      Actor    No   Yes    No   No   No   Yes     Slow    Active
      ge-1/0/10    Partner    No   Yes    No   No   No   Yes     Fast   Passive
    LACP protocol:        Receive State  Transmit State          Mux State 
      ge-0/0/10           Port disabled     No periodic           Detached
      ge-1/0/10           Port disabled     No periodic           Detached

{master:0}
```

## virtual-chassis

В основном стекируются через sfp+ порты 2300 и 3300, либо через медные в ex2200. Бывают stack интерфейсы у 4200 серии.
Читайте документацию на модель.
Для 2200 написано "We recommend that you physically cable the ports as the final step of this procedure."
Т.е. настроить порты, vc, включить кабели. Как удобно в cisco и eltex. В последнем даже софт накатит до версии мастера автоматом.
Ниже порты 3 и 4 настроены как vcp и в конфигурации их нет ни в каком виде.

```text
> show virtual-chassis vc-port    
fpc0:
--------------------------------------------------------------------------
Interface   Type              Trunk  Status       Speed        Neighbor
or                             ID                 (mbps)       ID  Interface
PIC / Port
0/3         Configured          5    Up           1000         1   vcp-255/0/3
0/4         Configured          5    Up           1000         1   vcp-255/0/4

fpc1:
--------------------------------------------------------------------------
Interface   Type              Trunk  Status       Speed        Neighbor
or                             ID                 (mbps)       ID  Interface
PIC / Port
0/3         Configured          5    Up           1000         0   vcp-255/0/4
0/4         Configured          5    Up           1000         0   vcp-255/0/4

{master:0}
> show configuration virtual-chassis 
preprovisioned;
no-split-detection;
member 0 {
    role routing-engine;
    serial-number CW0218430000;
}
member 1 {
    role routing-engine;
    serial-number CW0218430001;
}

{master:0}
```

## software add

В базе знаний советую ознакомиться с EX/SRX Recovering from file system corruption during a system reboot,
NAND media utility checks for bad blocks in the NAND flash memory
и EX Switch boots from backup root patition after file system corruption on the primary root partition.

### Подготовка к обновлению

Для ex4200 cтал копировать в шеле в /var/tmp/.
Если делать из cli с указанием /var/tmp/ в качестве назначения, то сначала он будет копировать в раздел на котором места нет,
а потом якобы должен будет переместить в /var/tmp/.
Ниже показана очистка флешки и проверка свободного места в разделах.
```text
ex2200> request system storage cleanup    
Please check the list of files to be deleted using the dry-run option. i.e.
request system storage cleanup dry-run
Do you want to proceed ? [yes,no] (no) yes   

fpc0:
--------------------------------------------------------------------------

List of files to delete:
[пропущено]
   708B Dec 17  2012 /var/lost+found/#00045
[пропущено]


ex2200> start shell

% df -h | grep -v mnt
Filesystem       Size    Used   Avail Capacity  Mounted on
/dev/da0s1a      184M    105M     64M    62%    /
devfs            1.0K    1.0K      0B   100%    /dev
/dev/md9         126M     12K    116M     0%    /tmp
/dev/da0s3e      123M    3.2M    110M     3%    /var
/dev/da0s3d      369M    132K    339M     0%    /var/tmp
/dev/da0s4d       62M    274K     57M     0%    /config
/dev/md10         59M     17M     37M    31%    /var/rundb
procfs           4.0K    4.0K      0B   100%    /proc
%
```
Проверить целостность флешки.
```text
% su
Password:
root@ex2200:RE:0% nand-mediack -C
Media check on da0 on ex platforms
root@ex2200:RE:0%exit 
% exit
exit

{master:0}
```
Ошибок нет. На всякий случай еще раз переформатировать раздел для последующей установки.
```text
ex2200> request system snapshot slice alternate 
fpc0:
--------------------------------------------------------------------------
Formatting alternate root (/dev/da0s2a)...
Copying '/dev/da0s1a' to '/dev/da0s2a' .. (this may take a few minutes)
The following filesystems were archived: /
```
Скачать с ftp и проверить md5
```text
% cd /var/tmp/
% fetch ftp://999.1.1.1/junos/jinstall-ex-2200-12.3R12.4-domestic-signed.tgz
jinstall-ex-2200-12.3R12.4-domestic-signed.tgz  3% of   95 MB  643 kBps
% md5 jinstall-ex-2200-12.3R12.4-domestic-signed.tgz
MD5 (jinstall-ex-2200-12.3R12.4-domestic-signed.tgz) = 73759d24a949181dcfe6111c0df20d1d
```
Копирование с usb-флешки
```text
% clear
% mount_msdosfs /dev/da1s1 /mnt
% cp /mnt/jinstall-ex-2200-12.3R3.4-domestic-signed.tgz /var/tmp/
% umount /mnt
```


### Обновление 

Вернуться в cli и запустить установку с ключем force.
Как-то обновлял коммутатор ex4200, а он в непонятный режим вывалился без части файлов
и хорошо еще удаленно смогли консолью подключиться и перезагрузить с рабочего слайса командой `request system reboot slice alternate`.
Со второй попытки и с ключем force обновиться получилось.
```text
ex2200> request system software add no-copy delay-restart force /var/tmp/jinstall-ex-2200-12.3R12.4-domestic-signed.tgz
```
Посмотреть текущее время.
```text
ex2200> show system uptime
```
Перезагрузиться через 840 минут.
```text
ex2200> request system reboot in 840 
```
Отменить перезагрузку, если не устраивает время.
```text
ex2200>clear system reboot
```
После перезагрузки выполнить и дождаться(!) выполнения команды:
```text
ex2200> request system snapshot slice alternate 
fpc0:
--------------------------------------------------------------------------
Formatting alternate root (/dev/da0s2a)...
Copying '/dev/da0s1a' to '/dev/da0s2a' .. (this may take a few minutes)
The following filesystems were archived: /
```

## autosnapshot

При повреждении файловой системы и автоматической загрузке с альтернативного слайса будет сделан снапшот.
Однако, возможное повторное отключение питания до завершения операции сделает устройство неработоспособным,
т.к. контроля успешности завершения снапшота нет.
```text
set system auto-snapshot

ex> show system auto-snapshot    
Auto-snapshot Configuration:   Enabled
Auto-snapshot State:  Disabled
```

## snmp

В коммутаторах EX-серии по snmp запросу ifName (1.3.6.1.2.1.31.1.1.1.1) ответы в том числе с юнитами. Можно фильтровать юниты и прочие логические интерфейсы.
```text
ex> show configuration snmp filter-interfaces 
interfaces {
    lsi;
    bme;
    dsc;
    ipip;
    mtun;
    pimd;
    pime;
    tap;
    lt;
    "^ge-[0-9]+/[0-9]+/[0-9]+.0$";
    "^xe-[0-9]+/[0-9]+/[0-9]+.0$";
    "^ae[0-9]+.0$";
}
```

## tacacs

```text
system {
   authentication-order [ tacplus password ];
    tacplus-server {
        10.8.7.6 {
            secret "key";
            single-connection;
        }
    }
    accounting {
        events [ login interactive-commands ];
        destination {
            tacplus {
                server {
                    10.8.7.6 {
                        secret "key";
                        single-connection;
                    }
                }
            }
        }
    }
}
```

## dhcp-snooping на Juniper EX, кроме EX9200

Чем чреваты настройки dhcp-snooping'а на джуниперах:

* не работает для vlan'ов, которые маршрутизируются данным коммутатором, т.е. имеется RVI интерфейс и настроен dhcp-relay.
* На ex9200 работает вроде, но я забыл уже. Там зато был регулярный косяк с dhcp-relay, который правили раза два минимум. Просто переставал релаить и привет;
* локальную базу хранить небезопасно, так как при внеплановой перезагрузке она может быть недоступа из-за загрузки с альтернативного раздела. Это специфика junos;
* по умолчанию доверенными являются все транковые порты, а недоверенными - access-порты. Нигде не описывается, что транковый агрегированный интерфейс (ae) является недоверенным по умолчанию. В случае, когда в такой порт прописан vlan с настройкой dot1q-tunneling, то настроить порт ae доверенным не получится. Почитать про EX/QFX Error message: DHCP trusted configuration cannot be specified on a dot1q tunneled interfaces. Получить такой ответ при создании тикета я совсем не ожидал. Не работает и всё.

Настройка для всех vlan. Доверенные порты по умолчанию - это все транки, кроме ae.
```text
ex2200> show configuration ethernet-switching-options secure-access-port 
vlan all {
    examine-dhcp;
}
dhcp-snooping-file {
    location /путь/имя_файла;
    write-interval 300;
    timeout 10;
}
```
## CoS EX2200
Рассматривается приоритезация трафика ip-телефонии и видеоконференцсвязи на коммутаторе EX2200 путем изменения дефолтовых настроек class-of-service (CoS).
К коммутатору подключены ip-телефоны. Для маркировки ip-пакетов от ip-PBX и других ip-телефонов используются следующие значения dscp:

dscp dec | dscp class | desc
---|---|---
46 | ef | медиа (rtp)
24 | cs3 | сигнализация (sip,h323,unistim)
32 | cs4 | видео c кодеков(rtp)
34 | af41 | видео с телефона, софтового клиента, кодека (rtp)
0 | be |весь остальной трафик без маркировки.

### Работа коммутатора EX2200 с дефолтовым CoS

Без лишней необходимости включать qos на коммутаторе не нужно.
Как минимум, необходимо убедиться в корректной маркировке пакетов телефонами или кодеками, иначе будет только хуже.
Если qos на коммутаторе не настраивался:

* с высокой вероятностью дропов на портах нет;
* все порты доверяют меткам cos и dscp по умолчанию;
* для всех портов одинаковые настройки очередей, где для всего трафика, кроме network control выделено 95% буферов.

Разделение происходит по маркировке, которую можно посмотреть в настройках CoS по умолчанию.
```text
ex2200> show class-of-service forwarding-class 
Forwarding class                    ID                      Queue  Policing priority    SPU priority
  best-effort                             0                       0           normal           low    
  expedited-forwarding                    1                       5           normal           low    
  assured-forwarding                      2                       1           normal           low    
  network-control                         3                       7           normal           low   
  
ex2200> show class-of-service interface ge-0/0/1  
Physical interface: ge-0/0/1, Index: 130
Queues supported: 8, Queues in use: 4
  Scheduler map: , Index: 2
  Congestion-notification: Disabled

  Logical interface: ge-0/0/1.0, Index: 70
Object                  Name                   Type                    Index
Classifier              ieee8021p-default      ieee8021p                  11

ex2200> show class-of-service classifier name ieee8021p-default 
Classifier: ieee8021p-default, Code point type: ieee-802.1, Index: 11
  Code point         Forwarding class                    Loss priority
  000                best-effort                         low         
  001                best-effort                         low         
  010                best-effort                         low         
  011                best-effort                         low         
  100                best-effort                         low         
  101                best-effort                         low         
  110                network-control                     low         
  111                network-control                     low          
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

ex2200> show class-of-service code-point-aliases dscp
Code point type: dscp
  Alias              Bit pattern
  af11               001010         
  af12               001100         
  af13               001110         
  af21               010010         
  af22               010100         
  af23               010110         
  af31               011010         
  af32               011100         
  af33               011110         
  af41               100010         
  af42               100100         
  af43               100110         
  be                 000000         
  cs1                001000         
  cs2                010000         
  cs3                011000         
  cs4                100000         
  cs5                101000         
  cs6                110000         
  cs7                111000         
  ef                 101110         
  nc1                110000         
  nc2                111000         

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
### Настройка СoS на коммутаторе EX2200

Необходимо dscp ef и af поставить в соответствие с внутренними классами expedited-forwarding и assured-forwarding.
Для этого настраивается т.н. classifiers. Можно создать и новые классы, но для примера пусть будет так.
Перечислять все остальные биты dscp, как это сделано в класифире по умолчанию, смысла нет.
```text
ex2200> show configuration class-of-service classifiers 
dscp custom-dscp {
    forwarding-class network-control {
        loss-priority low code-points [ cs6 cs7 ];
    }
    forwarding-class expedited-forwarding {
        loss-priority low code-points ef;
    }
    forwarding-class assured-forwarding {
        loss-priority low code-points [ cs3 cs4 af41 ];
    }
}
```
Для каждого класса необходимо настроить шедулеры:
```text
ex2200> show configuration class-of-service schedulers  
sc-ef {
    buffer-size percent 10;
    priority strict-high;
}
sc-af {
    shaping-rate 20m;
    buffer-size percent 10;
}
sc-nc {
    buffer-size percent 5;
    priority strict-high;
}
sc-be {
    shaping-rate percent 80;
    buffer-size {
        remainder;
    }
}
```
Названия выбираются произвольно, а процент выделенных буферов - по потребностям.
Для шедулеров sc-ef и sc-nc включается механизм strict-priority. Для шедулера sc-af полоса ограничивается 20мбит/с, а для sc-be - 80% от полосы интерфейса.
Далее шедулеры надо поставить в соответствие с внутренними классами:
```text
ex2200> show configuration class-of-service scheduler-maps    
custom-maps {
    forwarding-class network-control scheduler sc-nc;
    forwarding-class expedited-forwarding scheduler sc-ef;
    forwarding-class assured-forwarding scheduler sc-af;
    forwarding-class best-effort scheduler sc-be;
}
```
В итоге, scheduler-map и classifier необходимо применить ко всем интерфейсам, описав их как шаблон, например:
```text
ex2200> show configuration class-of-service interfaces 
ge-* {
    scheduler-map custom-maps;
    unit 0 {
        classifiers {
            dscp custom-dscp;
        }
    }
}
ae* {
    scheduler-map custom-maps;
    unit 0 {
        classifiers {
            dscp custom-dscp;
        }
    }
}
```
На интерфейсы можно применить одинаковые или индивидуальные настройки, т.е. можно написать разные scheduler и scheduler-maps для разных интерфейсов.
В этом отношении коммутаторы juniper лучше, чем cisco 2960/3750, где всего два queue-set.
Итоговая конфигурация выглядит вот так:
```text
ex2200> show configuration class-of-service 
classifiers {
    dscp custom-dscp {
        forwarding-class network-control {
            loss-priority low code-points [ cs6 cs7 ];
        }
        forwarding-class expedited-forwarding {
            loss-priority low code-points ef;
        }
        forwarding-class assured-forwarding {
            loss-priority low code-points [ cs3 cs4 af41 ];
        }
    }
}
host-outbound-traffic {
    forwarding-class network-control;
}
interfaces {
    ge-* {
        scheduler-map custom-maps;
        unit 0 {
            classifiers {
                dscp custom-dscp;
            }
        }
    }
    ae* {
        scheduler-map custom-maps;
        unit 0 {
            classifiers {
                dscp custom-dscp;
            }
        }
    }                                   
}
scheduler-maps {
    custom-maps {
        forwarding-class network-control scheduler sc-nc;
        forwarding-class expedited-forwarding scheduler sc-ef;
        forwarding-class assured-forwarding scheduler sc-af;
        forwarding-class best-effort scheduler sc-be;
    }
}
schedulers {
    sc-ef {
        buffer-size percent 10;
        priority strict-high;
    }
    sc-af {
        shaping-rate 20m;
        buffer-size percent 10;
    }
    sc-nc {
        buffer-size percent 5;
        priority strict-high;
    }
    sc-be {
        shaping-rate percent 80;
        buffer-size {
            remainder;
        }
    }
}
```
Перед применением конфигурации необходимо ее проверить командой commit check. У меня возникала следующая ошибка:
```text
[edit class-of-service interfaces]
  'ge-*'
    One or more "strict-high" priority queues have lower queue-numbers than priority "low" queues in custom-maps for ge-*. Ifd ge-* supports strict-high priority only on higher numbered queues.
error: configuration check-out failed
```
Это буквально означает, что нельзя указать priority "strict-high" только для 5ой очереди, когда у 7ой останется priority "low".
Поэтому тут или переписать классы и изменить очередь у network-control, либо настроить для network-control priority "strict-high", что и было сделано.
После применения конфигурации часть фреймов будет в очередях потеряна, поэтому надо почистить счетчики и спустя короткое время проверить счетчики дропов,
где значение отлично от нуля.
```text
clear interfaces statistics all
show interfaces queue | match dropped | except " 0$"
```
Если счетчики дропов нарастают, то имеется ошибка в конфигурации.
Например, если какой-то интерфейс не описан в class-of-service interfaces шаблоном или в явном виде, то трафик в классах дропнется.
Ниже пример нормальной работы без потерь.
```text
ex2200> show interfaces queue ge-0/0/22    
Physical interface: ge-0/0/22, Enabled, Physical link is Up
  Interface index: 151, SNMP ifIndex: 531
Forwarding classes: 16 supported, 4 in use
Egress queues: 8 supported, 4 in use
Queue: 0, Forwarding classes: best-effort
  Queued:
  Transmitted:
    Packets              :                320486
    Bytes                :             145189648
    Tail-dropped packets :                     0
    RL-dropped packets   :                     0
    RL-dropped bytes     :                     0
Queue: 1, Forwarding classes: assured-forwarding
  Queued:
  Transmitted:
    Packets              :                   317
    Bytes                :                169479
    Tail-dropped packets :                     0
    RL-dropped packets   :                     0
    RL-dropped bytes     :                     0
Queue: 5, Forwarding classes: expedited-forwarding
  Queued:
  Transmitted:
    Packets              :                   624
    Bytes                :                138260
    Tail-dropped packets :                     0
    RL-dropped packets   :                     0
    RL-dropped bytes     :                     0
Queue: 7, Forwarding classes: network-control
  Queued:
  Transmitted:                          
    Packets              :                   674
    Bytes                :                243314
    Tail-dropped packets :                     0
    RL-dropped packets   :                     0
    RL-dropped bytes     :                     0
```
Классы можно мониторить по snmp.

## spanning-tree

На разных моделях в разных версиях может отличаться. Например в 2300 stp работает на указанных в настройках портах.
В старых версиях и моделях 2200 3300 4200 работает везде, надо в конфиге наоборот отключать.
```text
> show configuration protocols mstp 
configuration-name abc;
revision-level 1;
bridge-priority 24k;
interface ge-0/0/1.0 {
    disable;
}
interface ge-0/0/2.0 {
    mode point-to-point;
}
msti 1 {
    bridge-priority 24k;
    vlan 1-4094;
}

{master:0}
```

## bpdufilter, bpduguard

Я не знаю, фильтруют ли производители своими bpdufilter'ами цисковский 01:00:0C:CC:CC:CD или нет.
В общем случае, рекомендуется написать фильтр с указанием двух мак-адресов и повесить по входу, например, такой:
```text
firewall {
    family inet {
        filter bpdufilter {
            term discard-bpdu {
                from {
                    destination-mac-address {
                        01:80:c2:00:00:00/48;
                        01:00:0c:cc:cc:cd/48;
                    }
                }
                then {
                    discard;
                    count BPDU_FILTER;
                }
            }
            term allow-other {
                then accept;
            }
    
        }
    }
}
```
Согласно документации, на мелких коммутаторах, кроме ex9200, можно глобально включить фильтр для портов, где spanning-tree отключен.
В примере ниже для всех интерфейсов отключен spanning-tree и включена фильтрация bpdu, но это работало в старых версиях junos.
В 12.3R12.4 надо в явном виде в bpdu-block прописывать интерфейсы, где stp работает, иначе зафильтрует bpdu на всех интерфейсах. Нежданчик.
```text
ex2200> show configuration protocols mstp                
configuration-name abc;
revision-level 1;
bridge-priority 28k;
interface all {
    disable;
}
msti 1 {
    bridge-priority 28k;
    vlan 1-4094;
}

{master:0}
ex2200> show configuration ethernet-switching-options bpdu-block 
interface ge-0/0/5.0 {
    disable;
}

interface xstp-disabled {
    drop;
}
{master:0}
```
Настроить bpduguard на ex2200 можно прописав пользовательские порты диапазоном,
в настройках используемого протокола stp указать для созданного диапазона портов тип edge и там же включить функционал bpdu-block-on-edge.
```text
ex2200# show | compare 
[edit interfaces]
+   interface-range Users {
+       member-range ge-0/0/20 to ge-0/0/30;
+       unit 0 {
+           family ethernet-switching {
+               port-mode access;
+               vlan {
+                   members 200;
+               }
+           }
+       }
+   }
[edit protocols mstp]
+    interface Users {
+        edge;
+    }
[edit protocols mstp]
+   bpdu-block-on-edge;
[edit ethernet-switching-options bpdu-block]
+   disable-timeout 120;
```

## igmp-snooping
Если не оператор связи, то выключить совсем. Если оператор, то настраивать исключения для конкретных вланов и тп.
```text
> show configuration protocols igmp-snooping 
vlan all {
    disable;
}

{master:0}
```

## storm-control

Хорошая штука, но на своих аплинках надо выключать
```text
ethernet-switching-options {
    storm-control {
        action-shutdown;
        interface ge-1/0/1.0 {
            no-broadcast;
            no-unknown-unicast;
            no-multicast;
        }
}
```

## SPAN

Количество dst интерфейсов ограничено. От 1 в до 4.
```text
ethernet-switching-options {
    analyzer span {
        input {
            ingress {
                vlan 100;
                vlan 200;
                vlan 300;
            }
        }
        output {
            interface {
                ge-0/0/47.0;
            }
        }
    }

}
```
