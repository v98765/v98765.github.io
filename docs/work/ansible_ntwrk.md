## best practices

[Network Best Practices for Ansible 2.5](https://acozine.github.io/html/network/user_guide/network_best_practices_2.5.html)
Модули net_get в 2.6, что написано в [Red Hat Ansible Network Automation Updates](https://www.ansible.com/blog/red-hat-ansible-network-automation-updates)

## Установка ansible в ubuntu

Есть так себе причины, почему не через pip, а через apt 
[installing-ansible-on-ubuntu](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-ansible-on-ubuntu)

```sh
$ sudo apt update
$ sudo apt install software-properties-common
$ sudo apt-add-repository --yes --update ppa:ansible/ansible
$ sudo apt install ansible
```

Для работы модулей [net_get](https://docs.ansible.com/ansible/latest/modules/net_get_module.html), [net_put](https://docs.ansible.com/ansible/latest/modules/net_put_module.html) необходим scp.
```sh
$ sudo apt install python-scp
```

В centos 7 c python-scp.noarch 0:0.7.1-3.el7 не работает, поэтому pip `pip install scp` или miniconda.

## Сохранение конфигурации

Скорость работы зависит от кол-ва памяти, когда можно увеличить кол-во forks. По умолчанию forks = 5

ansible.cfg в текущем каталоге
```text
[defaults]
host_key_checking = False
#forks = 20
inventory=./inventory
[ssh_connection]
pipelining = true
```

В inventory. Можно задать пароль, можно вводить каждый раз
```text
[all:children]
cisco_l2

[all:vars]
ansible_python_interpreter=/usr/bin/python3

[cisco_l2]
cisco-switch1 ansible_host=10.0.0.1
cisco-switch2 ansible_host=10.0.0.2

[cisco_l2:vars]
ansible_become=no
ansible_network_os=ios
ciscocopy_dir=/var/lib/cisco
```

playbook ciscocopy.yml. проверки на создание каналога нет, в переменной и ниже написано для примера
```yaml
---
- name: copy config
  hosts: cisco_l2
  connection: network_cli
  gather_facts: false

  tasks:
    - name: set scp server
      cli_config:
        config: ip scp server enable
      notify:
        - SAVE CONFIGURATION
      tags:
        - ciscocopy_set

    - name: copy file from the network device to localhost
      net_get:
        src: startup-config
        dest: "{{ ciscocopy_dir }}/{{ inventory_hostname_short }}.cfg"
        protocol: scp
      tags:
        - ciscocopy_backup

  handlers:

    - name: SAVE CONFIGURATION
      cli_command:
        command: write memory
```

Запуск, где -k означает ввод пароля для учетки ciscologin
```sh
$ ansible-playbook ciscocopy.yml -u ciscologin -k
``` 

Когда на всех коммутаторах уже включен scp server, то запустить с тегом
```sh
$ ansible-playbook ciscocopy.yml -u ciscologin -k --tag ciscocopy_backup
```

Можно запустить плей индивидуально для устройства
```sh
$ ansible-playbook ciscocopy.yml -u ciscologin -k --limit cisco-switch2
```
