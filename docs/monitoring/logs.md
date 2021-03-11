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

## facility

Т.к cisco не поддерживает по большей части rfc5424, то необходимо назначить разным устройствам различные facility.
local7 оставить для ненастроенного оборудования

rfc-soft | facility
---|---
3164-nxos | local6
5424-exos | local5
5424-junos | local4

## ansible коллекции

Необходимые коллекции для exos, cisco. В марте 2021 вышла ansible.netcommon 2.0.
Для junos нужен [junos_netconf_module](https://docs.ansible.com/ansible/latest/collections/junipernetworks/junos/junos_netconf_module.html), настроеный netconf на оборудовании.
```sh
ansible-galaxy collection install -f ansible.netcommon junipernetworks.junos cisco.nxos
```
Сконвертировать старый инветнори в yaml формат, поверить переменные.
```sh
ansible-inventory -i inventory -y --list > inventory.yaml
```
Файл ansible.cfg в текущем каталоге
```text
[defaults]
host_key_checking = False
#forks = 20
inventory=./inventory.yaml
roles_path = ~/.ansible/roles
[ssh_connection]
pipelining = true
```

## exos playbook

При логирования через OOBM меняется VR-Default на VR-Mgmt

```yaml
---
- name: configure syslog for extreme summit
  hosts: extreme
  gather_facts: false
  vars:
    Facility: local5
    FacilityOld: local7
    LogServers:
      - 10.1.0.1
      - 10.2.0.1

  tasks:

    - name: Delete old syslog
      community.network.exos_config:
        lines:
          - "configure syslog delete {{ item }}:514 vr VR-Default {{ FacilityOld }}"
      with_items: "{{ LogServers }}"

    - name: Add new syslog
      community.network.exos_config:
        lines:
          - "configure syslog add {{ item }}:514 vr VR-Default {{ Facility }}"
          - "enable log target syslog {{ item }}:514 vr VR-Default {{ Facility }}"
          - "configure log target syslog {{ item }}:514 vr VR-Default {{ Facility }} filter DefaultFilter severity Debug-Data"
          - "configure log target syslog {{ item }}:514 vr VR-Default {{ Facility }} match Any"
          - "configure log target syslog {{ item }}:514 vr VR-Default {{ Facility }} format timestamp seconds date Mmm-dd event-name none priority host-name tag-name"
      with_items: "{{ LogServers }}"

    - name: Save running to startup when modified
      community.network.exos_config:
        save_when: modified
```
