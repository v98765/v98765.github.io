# Установка AdGuardHome на домашний raspberry pi

Нужно написать ansible роль, запустить сервис и автоматизировать обновление.
pi установлена с нуля, адрес зарезервирован в dhcp или статический, включен ssh.
[Инструкция по установке AdGuardHome](https://github.com/AdguardTeam/AdGuardHome/wiki/Getting-Started).


## Удаленный доступ на pi

Сгенерить ключ. Дома без пароля сойдет
```sh
(base) vit@a:~$ ssh-keygen 
Generating public/private rsa key pair.
Enter file in which to save the key (/home/vit/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/vit/.ssh/id_rsa.
Your public key has been saved in /home/vit/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:Zvzpu/8ANQ1+ldUlsTr9AtDj/uHFAnsyXAKmHr3L5r0 vit@a
The key's randomart image is:
+---[RSA 2048]----+
|            . o.O|
|           o o =.|
|          + * +  |
|       . + = *   |
|        S o B o  |
|       + o * B o |
|        . + B = +|
|         o.o B = |
|         oB+Eo+  |
+----[SHA256]-----+

```
Скопировать ключ и проверить доступ.
```sh
(base) vit@a:~$ ssh-copy-id -i pi@192.168.2.254
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/home/vit/.ssh/id_rsa.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
pi@192.168.2.254's password: 

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'pi@192.168.2.254'"
and check to make sure that only the key(s) you wanted were added.

(base) vit@a:~$ ssh pi@192.168.2.254
Linux raspberrypi 5.4.51-v7+ #1327 SMP Thu Jul 23 10:58:46 BST 2020 armv7l
```
## Подготовка ansible

В каталоге ~/work/home будет роль и настройки ansible, который установлен через [miniconda](/work/miniconda/).
```sh
(base) vit@a:~/work/home$ cat ansible.cfg 
[defaults]
host_key_checking = False
inventory=./inventory
(base) vit@a:~/work/home$ cat inventory 
[all:children]
home

[home]
pi ansible_host=192.168.2.254

[home:vars]
ansible_user=pi
ansible_become=true

(base) vit@a:~/work/home$ ansible all --list-hosts
  hosts (1):
    pi

(base) vit@a:~/work/home$ ansible all -m ping
[WARNING]: Platform linux on host pi is using the discovered Python interpreter at /usr/bin/python, but future installation of another Python interpreter could change
this. See https://docs.ansible.com/ansible/2.9/reference_appendices/interpreter_discovery.html for more information.
pi | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
```
## Роль ansible

Посмотреть [https://github.com/v98765/ansible-adguardhome](https://github.com/v98765/ansible-adguardhome). Скопировать
```sh
$ git clone https://github.com/v98765/ansible-adguardhome
```


## Проверка 

Корректность синтаксиса yamllint
```sh
(base) vit@a:~/work/home$ yamllint roles
roles/adguardhome/tasks/install.yml
  29:81     error    line too long (158 > 80 characters)  (line-length)
  30:81     error    line too long (117 > 80 characters)  (line-length)
  41:81     error    line too long (116 > 80 characters)  (line-length)
```
Ошибок в плее нет
```sh
(base) vit@a:~/work/home$ ansible-playbook play-pi.yml --syntax-check

playbook: play-pi.yml
```
Проверка на pi
```sh
(base) vit@a:~/work/home$ ansible-playbook play-pi.yml --check
```

## Запуск

```sh
$ ansible-playbook play-pi.yml
```
