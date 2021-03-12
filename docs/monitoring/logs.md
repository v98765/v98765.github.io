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

Для версии не ниже 0.12. возможна настройка конфига в yaml формате.
Конвертер [toml-yaml](https://www.convertsimple.com/convert-toml-to-yaml/) для преобразования примеров из документации.

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

Валидация конфига
```sh
vector validate --config-yaml vector.yaml
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

ОС | формат логов | предлагаемый facility
---|---|---
nxos | 3164 | local6
exos | 5424 | local5
junos | 5424 | local4

В junos 3164 по умолчанию, однако надо настроить 5424. Логи cisco придется переделывать.

## ansible коллекции

Необходимые коллекции для exos, cisco. В марте 2021 вышла ansible.netcommon 2.0.
Для junos нужен ncclient, [junos_netconf_module](https://docs.ansible.com/ansible/latest/collections/junipernetworks/junos/junos_netconf_module.html), настроеный netconf на оборудовании.

Все ставится в [окружении](https://v98765.github.io/work/venv/) пользователя.
```sh
pip install wheel
pip install ansible scp ncclient
ansible-galaxy collection install -f ansible.netcommon junipernetworks.junos cisco.nxos

```
Сконвертировать старый inventory в yaml формат, поверить переменные.
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

## проверка playbook

```sh
ansible-playbook syslog.yml --syntax-check
ansible-playbook -v syslog.yml  -u [username] -k --limit [test_device] --check
```

## exos playbook

При логирования через OOBM меняется VR-Default на VR-Mgmt.
Идемпотентности тут нет, как в прочем и других playbook для exos, т.к. удаление/отключение команд,
которые в конфигурации не отображаюстся все равно приводит к модификации, поэтому тут запись, а не handler.
Команды должны быть написаны в том же виде, как и в конфигурации, без сокращений.

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

## cisco playbook

Цитата при запуске о том, что команды должны быть в том же виде, что и в текущем конфиге. Только в этом случае гарантируется идемпотентность.

> [WARNING]: To ensure idempotency and correct diff the input configuration lines should be similar to how they appear if present in
the running configuration on device including the indentation

with_item тут не применить из-за ограничений модуля, поэтому jinja шаблон.

```yaml
---
- name: configure syslog for nexus
  hosts: cisco_n3k
  gather_facts: false
  vars:
    Facility: local6
    LogServers:
      - 10.1.0.1
      - 10.2.0.1

  tasks:

    - name: syslog servers for n3k
      ansible.netcommon.cli_config:
        config: |
            #jinja2: lstrip_blocks: True
            {% for ip in LogServers %}
            logging server {{ ip }} 7 use-vrf default facility {{ Facility }}
            {% endfor %}
            logging source-interface loopback0
            logging level authpri 6
            login on-success log
      notify:
        - SAVE CONFIGURATION

  handlers:

    - name: SAVE CONFIGURATION
      cli_command:
        command: copy running-config startup-config
```

Для mds команда попроще
```text
logging server {{ ip }} 7 facility {{ Facility }}
```
В итоге в логах все равно нет facility.


## junos playbook

Какие-то специфичные модули не работали, поэтому пока так, как и в предыдущих примерах.
Из-за команды delete будут изменения в конфигурации всегда и коммит, поэтому для примера добавил теги и
ограничение на последовательное выполнение таска по хостам с паузой на 60 секунд между удалением и добавлением строк,
т.к. коммиты каждый раз, как показано ниже.
```text
0   2021-03-12 00:35:09 MSK by [username] via netconf
    set syslog hosts
1   2021-03-12 00:35:06 MSK by [username] via netconf
    set syslog hosts
2   2021-03-12 00:34:01 MSK by [username] via netconf
    delete syslog hosts
3   2021-03-12 00:33:58 MSK by [username] via netconf
    delete syslog hosts
```
Ниже запуск плея с тегом junos_set. Коммита не будет, если такие команды уже есть в конфигурации,
т.е. идемпотентность обеспечена.
```sh
ansible-playbook syslog_junos.yml -u [username] -k -t junos_set
```
playbook
```yaml
---
- name: configure syslog for juniper
  hosts: junos
  serial: 1
  gather_facts: false
  vars:
    Facility: local4
    LogServers:
      - 10.1.0.1
      - 10.2.0.1

  tasks:

    - name: delete syslog servers for junos
      junipernetworks.junos.junos_config:
        lines:
          - "delete system syslog host {{ item }}"
        comment: delete syslog hosts
      with_items: "{{ LogServers }}"
      tags:
        - junos_delete

    - name: pause for 60 seconds
      pause:
        seconds: 60
      tags:
        - junos_delete

    - name: syslog servers for junos
      junipernetworks.junos.junos_config:
        lines:
          - "set system syslog host {{ item }} any any"
          - "set system syslog host {{ item }} facility-override {{ Facility }}"
          - "set system syslog host {{ item }} explicit-priority"
          - "set system syslog host {{ item }} structured-data brief"
        comment: set syslog hosts
      with_items: "{{ LogServers }}"
      tags:
        - junos_set
```
Ничего страшного в `any any` нет, т.к. debug удаляется vector'ом.
Потом эти правила могут быть расширены только для сообщений с local4, например.
