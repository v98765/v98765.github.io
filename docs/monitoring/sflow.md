## Установка

Установка коллектора netflow, sflow с ролью [v98765_nfdump](https://github.com/v98765/v98765_nfdump). Пакеты есть для ubuntu,debian. Роль только для этих ОС с systemd.
Каталоги в примерах ниже от старой роли.

## Примеры использования

`man nfdump` для справки.

Индекс интерфейса пишется тоже по умолчанию. Выбрать top 1 srcip, где snmp-индекс исходящего интерфейса 12, чтобы посмотреть откуда качают в заданном интервале времени. Заменить на dstip, чтобы посмотреть с каких адресов.
```text
root@flows:/var/lib/flows/router1/2020/09/08# nfdump -R nfcapd.202009080701:nfcapd.202009080756 'out if 12' -s srcip -O bytes -n 1
Top 1 Src IP Addr ordered by bytes:
Date first seen          Duration Proto       Src IP Addr    Flows(%)     Packets(%)       Bytes(%)         pps      bps   bpp
2020-09-08 07:00:35.800  3655.194 any         10.06.00.30   111331(26.6)    2.7 M(39.0)    3.0 G(50.5)      726    6.6 M  1141

Summary: total flows: 418762, total bytes: 6003467232, total packets: 6816316, avg bps: 13128750, avg pps: 1863, avg bpp: 880
Time window: 2020-09-08 07:00:31 - 2020-09-08 08:01:30
Total flows processed: 835676, Blocks skipped: 0, Bytes read: 46799764
Sys: 0.119s flows/second: 7016060.9  Wall: 0.117s flows/second: 7100232.0 
```

Проверка маркировки телефонной сигнализации
```text
root@flows:/var/lib/flows/router1/2020/09/08# nfdump -R nfcapd.202009080701:nfcapd.202009080756 -c 1 'tos > 0 and port 5060'  -o long
Date first seen          Duration Proto      Src IP Addr:Port          Dst IP Addr:Port   Flags Tos  Packets    Bytes Flows
2020-09-08 07:01:19.647     0.000 UDP         10.00.00.0:5060  ->      10.00.20.50:5062  ......  96        1      413     1
Summary: total flows: 1, total bytes: 413, total packets: 1, avg bps: 0, avg pps: 0, avg bpp: 0
Time window: 2020-09-08 07:00:31 - 2020-09-08 07:06:30
Total flows processed: 18706, Blocks skipped: 0, Bytes read: 1047612
Sys: 0.004s flows/second: 3882523.9  Wall: 0.002s flows/second: 7536664.0 
```

Без маркировки нет.
```text
root@flows:/var/lib/flows/router1/2020/09/08# nfdump -R nfcapd.202009080701:nfcapd.202009080756 'host 10.00.00.0 and tos 0'  -o long
Date first seen          Duration Proto      Src IP Addr:Port          Dst IP Addr:Port   Flags Tos  Packets    Bytes Flows
Summary: total flows: 0, total bytes: 0, total packets: 0, avg bps: 0, avg pps: 0, avg bpp: 0
Time window: 2020-09-08 07:00:31 - 2020-09-08 08:01:30
Total flows processed: 835676, Blocks skipped: 0, Bytes read: 46799764
Sys: 0.082s flows/second: 10137516.1 Wall: 0.080s flows/second: 10409516.7
```

Top10 за 8 число
```text
root@flows:/var/lib/flows/router1/2020/09# nfdump -R 08 -s srcip -O bytes
Top 10 Src IP Addr ordered by bytes:
...
```

## Место на диске

Есть утилита

```text
root@flows:/var/lib/flows# nfexpire -h
usage nfexpire [options] 
-h      This text
-l datadir  List stat from directory
-e datadir  Expire data in directory
-r datadir  Rescan data directory
-u datadir  Update expire params from collector logging at <datadir>
-s size     max size: scales b bytes, k kilo, m mega, g giga t tera
-t lifetime maximum life time of data: scales: w week, d day, H hour, M minute
-w watermark    low water mark in % for expire.

root@flows:/var/lib/flows# for i in `ls` ; do echo -n "$i has " ; nfexpire -l $i | grep Status ; done 
router1 has Status:    OK
router2 has Status:    OK
switch1 has Status:    OK
switch2 has Status:    OK

root@flows:/var/lib/flows# for i in `ls` ; do nfexpire -u $i -s 5g -t 8w ; done
```

Можно ставить node_exporter с collector.textfile и генерить метрики

## Ограничение доступа

На ubuntu есть ufw.

Коллектор | Протокол | Порт по умолчанию
---|---|---
netflow | udp | 2055
sflow | udp | 6343

```text
# cat /etc/ufw/applications.d/nfdump 
[nfdump]
title=nfdump
description=nfdump
ports=2055/udp

# ufw app update nfdump
Rules updated for profile 'nfdump'
Skipped reloading firewall

# ufw allow from any to any app nfdump
```

## агрегация статистики

С сохранением в файл в формате nfdump
```sh
for day in `ls` ; do cd $day;
for file in `find . -name 'nfcapd.[0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9]'` ; do  nfdump -A srcip,dstip -a -r ${file} -w ${file}.a ; rm -f ${file} ; done;
cd ../;
done
```

## vmware

Помимо телекоммуникационного оборудования, для сбора статистики по трафику можно использовать возможности vmware
[Configure the NetFlow Settings of a vSphere Distributed Switch](https://docs.vmware.com/en/VMware-vSphere/6.7/com.vmware.vsphere.networking.doc/GUID-55FCEC92-74B9-4E5F-ACC0-4EA1C36F397A.html)
