## Установка

Установка коллектора netflow, sflow с ролью [v98765_nfdump](https://github.com/v98765/v98765_nfdump). Пакеты есть для ubuntu,debian. Роль только для этих ОС с systemd.

## Примеры использования

`man nfdump` для справки.

Индекс интерфейса пишется тоже по умолчанию. Выбрать top 1 srcip, где snmp-индекс исходящего интерфейса 12, чтобы посмотреть откуда качают. Заменить на dstip, чтобы посмотреть с каких адресов.
```sh
nfdump -r [filename] 'out if 12' -s srcip -O bytes -n 1
```

Проверка маркировки телефонной сигнализации
```sh
nfdump -r [filename] -c 1 'tos > 0 and port 5060'  -o long
```

Проверка отсутствия маркировки на пакетах от/до АТС.
```sh
nfdump -r [filename] 'host 10.00.00.0 and tos 0'  -o long
```

Top10 за 1 апреля
```sh
nfdump -R 2021/04/01 -s srcip -O bytes
```

Фрагменты
```sh
nfdump -r [filename] '(proto tcp and flags F and dst ip in [ 999.888.777.0/22 ] and dst port in [ 80 445 443 ]) %nf_fragment ' -c 10
```
TCP syn. В `man nfdump` прямо так по флагам.
```sh
nfdump -r [filename]  '(proto tcp and flags S and not flags ARFPU and dst ip in [ 999.888.777.0/22 ] and port in [ 80 443 445 ]) %nf_syn' -c 10
```
UDP
```sh
nfdump -r [filename]   '(proto udp and dst ip in [ 999.888.777.0/22 ] and port in [ 53 123 161 80 3389 ] )' -c 10
```
ICMP
```sh
nfdump -r [filename]  '(proto icmp and dst ip in [  999.888.777.0/22 ] )' -A dstip
```

## Место на диске

Есть утилита `nfexpire` но она только для файлов с именем `nfdump.*`. Если переименовать файлы, то бесполезна.

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

## агрегация статистики

С сохранением в файл в формате nfdump, агрегацией и сжатием `-A srcip,dstip -j`.
```sh
#!/bin/bash

cd /var/cache/nflow
filecounter=0
for file in `find . -name 'nfcapd.[0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9]'` ; do
    aggfile=$(echo ${file} | sed s/nfcapd/ajnfcapd/)
    nfdump -A srcip,dstip -j -a -r ${file} -w ${aggfile}
    [ -f ${aggfile} ] && rm -f ${file} && let filecounter=${filecounter}+1
done
date >> add.log
echo " $filecounter" >> add.log
```

## поиск по файлам за период

Для случаев когда нужно выбрать конкретные данные

```sh
#!/bin/bash

cd /var/cache/nflow
mkdir -p reports

echo 'enter ip address'
read varip

# создание каталогов для отчета по Ip-адресу

mkdir -p reports/${varip}

# поиск данных по каталогам статистики

date
for m in $(find ./20* -maxdepth 1 -mindepth 1 -type d -name "[0-9][0-9]"); do
  reportdata=$( echo $m | tr -d './')
  set -x bash
  targetnfdump="reports/${varip}/nfdump.${reportdata}"
  nfdump -R $m "host ${varip}" -w $targetnfdump
  nfdump -r $targetnfdump -o "fmt:%ts;%sa;%da" > reports/${varip}/${reportdata}.txt
  set +x bash
done
date
```

Лучше запускать в `screen`, потому что долго.

## просмотр текущих данных

Файлы ротируются с интервалом раз в 15 минут, поэтому долго ждать актуальных данных. Просмативать можно и текущий временный файл. Время должно быть синхронизировано.
```sh
#!/bin/bash -x

cd /var/cache/nflow

filecurr=$(ls nfcapd.current*)
timewin=$(date -d "-1 minutes" +%Y/%m/%d.%H:%M)
nfdump -r $filecurr -t $timewin
```

## vmware

Помимо телекоммуникационного оборудования, для сбора статистики по трафику можно использовать возможности vmware
[Configure the NetFlow Settings of a vSphere Distributed Switch](https://docs.vmware.com/en/VMware-vSphere/6.7/com.vmware.vsphere.networking.doc/GUID-55FCEC92-74B9-4E5F-ACC0-4EA1C36F397A.html)
