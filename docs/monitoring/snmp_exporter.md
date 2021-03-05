[vmagent](https://victoriametrics.github.io/) периодически опрашивает через [snmp_exporter](https://github.com/prometheus/snmp_exporter)
устройства по протоколу snmp и записывает полученные метрики в базу VictoriaMetrics.


## Установка go и зависимостей для генератора

[ubuntu golang wiki](https://github.com/golang/go/wiki/Ubuntu)
```sh
curl -LO https://get.golang.org/$(uname)/go_installer && chmod +x go_installer && ./go_installer && rm go_installer
source ${HOME}/.bash_profile
```
Зависимости, кроме snmp, который нужен все равно
```sh
sudo apt install unzip build-essential libsnmp-dev p7zip-full snmp
```

## Установка генератора

[generator readme](https://github.com/prometheus/snmp_exporter/tree/master/generator) ставить по инструкции, кроме установки мибов.
```sh
go get github.com/prometheus/snmp_exporter/generator
cd ${GOPATH-$HOME/go}/src/github.com/prometheus/snmp_exporter/generator
go build
cp generator ${HOME}/bin
```
Все мибы положить в домашний каталог ~/.snmp/mibs
Настроить переменные, чтобы самому пользоваться этими мибами с snmptranslate
```text
export MIBDIRS=$HOME/.snmp/mibs:/usr/share/snmp/mibs
export MIBS=ALL
```

## Запуск генератора

Запускается в каталоге с файлом generator.yml и создает snmp.yml
```text
(base) vit@abc-MS-1244:~/work/m2/ansible-snmp-exporter/files$ ls
generator.yml  snmp.yml
(base) vit@abc-MS-1244:~/work/m2/ansible-snmp-exporter/files$ generator generate
level=info ts=2020-08-12T06:08:15.911Z caller=net_snmp.go:142 msg="Loading MIBs" from=$HOME/.snmp/mibs:/usr/share/snmp/mibs:/usr/share/snmp/mibs/iana:/usr/share/snmp/mibs/ietf:/usr/share/mibs/site:/usr/share/snmp/mibs:/usr/share/mibs/iana:/usr/share/mibs/ietf:/usr/share/mibs/netsnmp
```

## generator.yml

[пример generator.yml](https://github.com/prometheus/snmp_exporter/blob/master/generator/generator.yml)

Удобнее, когда устройства однотипные, имеют одну и ту же community на чтение, иначе для каждого community писать отдельный модуль.
Каждому модулю в vmagent'е будет соответствовать отдельное задание (job) со своими настройками.
Отказался от отдельных модулей по функционалу в пользу модулей, где все описано единожды для конкретного оборудования, опрашивается один раз и для vmagent это будет один target.

Для примера простой модуль для коммутатора в generator.yml:
```yaml
modules:

  vendorname_l2:
    retries: 1
    walk:
      - 1.3.6.1.2.1.1.3         #sysUpTime
      - 1.3.6.1.2.1.2.2.1.8     #IF-MIB::ifOperStatus
      - 1.3.6.1.2.1.2.2.1.14    #IF-MIB::ifInErrors
      - 1.3.6.1.2.1.2.2.1.19    #IF-MIB::ifOutDiscards
      - 1.3.6.1.2.1.31.1.1.1.6  #IF-MIB::ifHCInOctets
      - 1.3.6.1.2.1.31.1.1.1.7  #IF-MIB::ifHCInUcastPkts
      - 1.3.6.1.2.1.31.1.1.1.10 #IF-MIB::ifHCOutOctets
      - 1.3.6.1.2.1.31.1.1.1.11 #IF-MIB::ifHCOutUcastPkts
      - 1.3.6.1.2.1.31.1.1.1.15 #IF-MIB::ifHighSpeed

    lookups:
      - source_indexes: [ifIndex]
        lookup: 1.3.6.1.2.1.31.1.1.1.1  #ifName
        drop_source_indexes: true
      - source_indexes: [ifIndex]
        lookup: 1.3.6.1.2.1.2.2.1.3     #ifType
```

Используется только то, что нужно и что можно (строки нельзя, но ссылка внизу). Не требуется собирать вообще все и тем более обе таблицы iftable и ifxtable целиком.
Таблица с метриками:

что | зачем
---|---
sysUpTime | для определения времени работы оборудования
ifOperStatus | статус интерфейса up/down
ifInErrors | общий счетчик входящих ошибок
ifOutDiscards | дискарды на портах из-за speed mismatch, qos, и тп
ifHCInOctets | 64-битный счетчик входящих байт
ifHCInUcastPkts | 64-битный счетчик входящих юникастовых пакетов
ifHCOutOctets | 64-битный счетчик исходящих байт
ifHCOutUcastPkts | 64-битный счетчик исходящих юникастовых пакетов
ifHighSpeed | скорость интерфейса для расчета % утилизации

Таблица с label, которые дополнительно будут у каждой метрики предыдущей таблицы:

что | зачем
---|---
ifName | имя интерфейса из ifxtable вместо ifIndex, который удален `drop_source_indexes: true`
ifType | тип интерфейса для фильтрации метрик в vmagent'е, чтобы исключить сбор данных по логическим интерфейсам

Дропать индекс интерфейсов для некторых маршрутизаторов cisco нельзя.

У snmp_exporter'а есть функционал overrides, который можеть дропать метрики:
```text
    overrides:
      entSensorStatus:
        ignore: true
```
Переписать строки в числа, иначе будет значение 1, а строковое значение, например, загрузка cpu или кол-во свободной памяти,
будет в label, что плохо скажется на производительности, т.к. увеличивается кардинальность метрик:
```text
    overrides:
      extremeCpuMonitorSystemUtilization1min:
        regex_extracts:
          '':
            - regex: '(.*)'
              value: '$1'
      extremeMemoryMonitorSystemTotal:
        regex_extracts:
          '':
            - regex: '(.*)'
              value: '$1'
      extremeMemoryMonitorSystemFree:
        regex_extracts:
          '':
            - regex: '(.*)'
              value: '$1'
      extremeMemoryMonitorSystemUsage:
        regex_extracts:
          '':
            - regex: '(.*)'
              value: '$1'
      extremeMemoryMonitorUserUsage:
        regex_extracts:
          '':
            - regex: '(.*)'
              value: '$1'
```
Ссылки по теме:
[https://gtacknowledge.extremenetworks.com/articles/Q_A/Is-there-a-way-to-display-the-OID-1-3-6-1-4-1-1916-1-32-1-4-1-9-for-CPU-utilization-to-an-integer-instead-of-a-string](https://gtacknowledge.extremenetworks.com/articles/Q_A/Is-there-a-way-to-display-the-OID-1-3-6-1-4-1-1916-1-32-1-4-1-9-for-CPU-utilization-to-an-integer-instead-of-a-string)
[EXTREME-SOFTWARE-MONITOR-MIB](http://www.circitor.fr/Mibs/Html/E/EXTREME-SOFTWARE-MONITOR-MIB.php)
[Numbers from DisplayStrings with the snmp_exporter](https://www.robustperception.io/numbers-from-displaystrings-with-the-snmp_exporter)

Примеры:
[github.com/prometheus/snmp_exporter/tree/master/generator](https://github.com/prometheus/snmp_exporter/tree/master/generator)
[github.com/v98765/ansible-snmp-exporter/blob/master/files/generator.yml](https://github.com/v98765/ansible-snmp-exporter/blob/master/files/generator.yml)

## Запуск snmp_exporter

Необходим конфигурационный файл /etc/snmp_exporter/snmp.yml и запустить snmp_exporter.

```text
$ systemctl status snmp_exporter
● snmp_exporter.service - SNMP Exporter
   Loaded: loaded (/usr/lib/systemd/system/snmp_exporter.service; enabled; vendor preset: disabled)
   Active: active (running) since Чт 2020-06-25 10:28:56 +07; 1 months 17 days ago
 Main PID: 6467 (snmp_exporter)
   CGroup: /system.slice/snmp_exporter.service
           └─6467 /usr/local/bin/snmp_exporter --config.file /etc/snmp_exporter/snmp.yml
``` 
Установку snmp_expoter делаю с ролью [ansible-snmp-exporter](https://github.com/v98765/ansible-snmp-exporter).

## vmagent

Вот vmagent.yml и job, для которого настроен file service discovery. Опрос хостов раз в минуту. У job'а в параметрах может быть только один module.

```yaml
global:
  scrape_interval: 60s
  scrape_timeout: 30s


scrape_configs:
  - file_sd_configs:
    - files:
      - targets/vendorname_l2_*.yml
    job_name: vendorname_l2
    metric_relabel_configs:
    - action: drop
      regex: ^[^6]$|\d{2,}
      source_labels:
      - ifType
    metrics_path: /snmp
    params:
      module:
      - vendorname_l2
    relabel_configs:
    - source_labels:
      - __address__
      target_label: __param_target
    - source_labels:
      - __param_target
      target_label: instance
    - replacement: 127.0.0.1:9116
      target_label: __address__
```
В metric_relabel_configs правило, которое дропает все метрики, где iftype!=6.
Т.о. можно будет суммировать метрики по физическим интерфейсам, не хранить данные по счетчикам логических интерфейсов.

Файл с таргетами vendorname_l2_region1.yml:
```yaml
- targets:
    - switch1
    - switch2
  labels:
    obj: 'region1'
```
Статус таргетов можно в vmagent'е запросить:
```text
curl http://localhost:8429/targets
```
Установку vmagent делаю с ролью [ansible-vmagent](https://github.com/v98765/ansible-vmagent).

Для маршрутизаторов нужны в т.ч. логические интерфейсы, поэтому для цисок удаляются какие-то конкретные iftype:
```text
    metric_relabel_configs:
    - action: drop
      regex: ^1$|22|24|32|33|39|53|63|77
      source_labels:
      - ifType
```

Мониторинг huawei nqa осложняется наличием меняющихся индексов истории, которые надо удалить:
```text
    metric_relabel_configs:
    - action: labeldrop
      regex: nqaResultsIndex|nqaResultsHopIndex
```

mkdir snmp
https://raw.githubusercontent.com/prometheus/snmp_exporter/master/generator/Dockerfile
podman build -t snmp-generator .

## links

[https://www.robustperception.io/numbers-from-displaystrings-with-the-snmp_exporter](https://www.robustperception.io/numbers-from-displaystrings-with-the-snmp_exporter)
