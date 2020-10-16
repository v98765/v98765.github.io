## venv в ubuntu

```sh
$ sudo apt install python3-pip python3-venv
```

## использование

[Installing packages using pip and virtual environments](https://packaging.python.org/guides/installing-using-pip-and-virtual-environments/)

```sh
$ python3 -m venv env
$ source env/bin/activate
(env) ~$ pip install ansible scp
...
Successfully installed MarkupSafe-1.1.1 PyYAML-5.3.1 ansible-2.9.13 bcrypt-3.2.0 cffi-1.14.2 cryptography-3.1 jinja2-2.11.2 paramiko-2.7.2 pycparser-2.20 pynacl-1.4.0 scp-0.13.2 six-1.15.0
```
Проверка
```sh
(env) ~$ ansible --version | grep exe
  executable location = /home/user/env/bin/ansible
```
Выход из env `deactivate`

Установка нескольких пакетов [using-requirements-files](https://packaging.python.org/guides/installing-using-pip-and-virtual-environments/#using-requirements-files) c `pip install -r requirements.txt`

## centos 8

Установить python3
```sh
dnf install python3
```
Настроить и активировать окружение
```sh
python3 -m venv base
source base/bin/activate
```
В окружении обновить pip и установить пакеты
```sh
pip install --upgrade pip
pip install ansible scp
```

