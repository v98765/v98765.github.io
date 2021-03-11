## syslog format

[rfc5424](https://tools.ietf.org/html/rfc5424), старый [3164](https://tools.ietf.org/html/rfc3164)

## vector.dev

https://vector.dev/docs/setup/installation/package-managers/apt/
Создать каталог /var/log/vector/ с правами на запись пользователю vector
```bash
mkdir -p /var/log/vector
chown vector /var/log/vector
```

## vector.yaml

Для версии не ниже 0.12.
Конфиг для запуска и проверки работы с фильтром по `debug`.

```yaml
data_dir: /var/lib/vector
sources:
  syslog:
    type: "syslog"
    address: "0.0.0.0:514"
    mode: "udp"
transforms:
  filter_severity:
    type: "filter"
    inputs:
      - "syslog"
    condition: |
      !includes(["debug"], .severity)
sinks:
  file_syslog:
    type: "file"
    inputs:
      - "filter_severity"
    healthcheck: true
    path: "/var/log/vector/vector-%Y-%m-%d.log"
    encoding: "ndjson"
```

## логирование в ubuntu

создать файл /etc/rsyslog.d/44-vector.conf
```text
*.* @[vector.hostname]:514;RSYSLOG_SyslogProtocol23Format
```
## centos

```bash
yum install rsyslog
systemctl enable rsyslog
systemctl start rsyslog
systemctl status rsyslog
```
создать файл /etc/rsyslog.d/44-vector.conf
```text
*.* @[vector.hostname]:514;RSYSLOG_SyslogProtocol23Format
```

## cisco

По умолчанию rfc3164. В XR есть `logging format rfc5424`, а в nx-os кое-где и то с ошибками

nexus
```text
logging server [vector.ip] 7
logging source-interface loopback0
logging level authpri 6
login on-success log
```
либо проще если старый nx-os или mds, где нет лупбека
```text
logging server [vector.ip] 7
logging level authpri 6
```

## juniper

По умолчанию rfc3164.

The structured-data format complies with Internet standard RFC 5424, The Syslog Protocol, which is at rfc5424.
The RFC establishes a standard message format regardless of the source or transport protocol for logged messages.
To output messages to a file in structured-data format, include the structured-data statement at the .. hierarchy level

```text
set system syslog host [vector.ip] any any
set system syslog host [vector.ip] explicit-priority
set system syslog host [vector.ip] structured-data brief
```

## exos

По умолчанию rfc5424.

Удалить старое
```text
configure syslog delete [old-ip]:514 vr VR-Default local0
```
Добавить так как будет в конфиге в итоге. Вообще `local7 match Any` добавяет предыдущую и следующую строку автоматом
```text
configure syslog add [vector-ip]:514 vr VR-Default local7
enable log target syslog [vector-ip]:514 vr VR-Default local7
configure log target syslog [vector-ip]:514 vr VR-Default local7 filter DefaultFilter severity Debug-Data
configure log target syslog [vector-ip]:514 vr VR-Default local7 match Any
configure log target syslog [vector-ip]:514 vr VR-Default local7 format timestamp seconds date Mmm-dd event-name none priority host-name tag-name
```
