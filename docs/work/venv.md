## venv в ubuntu

```sh
$ sudo apt install python3-pip python3-venv
```
## centos 8

Установить python3
```sh
dnf install python3
```

## Установка

[Installing packages using pip and virtual environments](https://packaging.python.org/guides/installing-using-pip-and-virtual-environments/)

```sh
python3 -m venv base
source base/bin/activate
```
Настроить и активировать окружение
```sh
python3 -m venv base
source base/bin/activate
```
В prompt шелла появится `(base)`. Добавить строку `source base/bin/activate` в конец файла `~/.bashrc`
Выход из env `deactivate`

В окружении обновить pip и установить пакеты
```sh
base/bin/python3 -m pip install --upgrade pip
pip -V
```
Покажет версию pip, установленную в окружении.


Установка ansible
```sh
pip install ansible scp
```
Установка коллекций
```sh
ansible-galaxy collection install ansible.netcommon
ansible-galaxy collection install cisco.nxos
```
Коллекции устанавливаются в `~/.ansible/collections/` , а роли копировать в `~/.ansible/roles/`. В локальном конфиге `ansible.cfg` прописать `roles_path = ~/.ansible/roles`.

Установка нескольких пакетов [using-requirements-files](https://packaging.python.org/guides/installing-using-pip-and-virtual-environments/#using-requirements-files) c `pip install -r requirements.txt`
