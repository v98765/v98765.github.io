## podman

Ниже все для убунты 2.10, где можно установить podman.
Чтобы обновить 20.04 необходимо в /etc/update-manager/release-upgrades указать `normal`. Далее в `screen` выполнить `do-release-upgrade`.
В 20.10 выполнить:
```text
sudo apt install slirp4netns podman runc fuse-overlayfs
```
В `$HOME/.config/containers/storage.conf`
```text
graphroot="$HOME/.local/share/containers/storage"
runroot="$XDG_RUNTIME_DIR/containers"
```

## venv

Настроить окружение [v98765.github.io/work/venv/](https://v98765.github.io/work/venv/)

## molecule driver podmap

Установка
```text
pip install molecule ansible-lint yamllint podman molecule-podman
```
Проверка
```text
$ molecule --version
molecule 3.2.3 using python 3.8
    ansible:2.10.5
    delegated:3.2.3 from molecule

$ molecule init role --help
Usage: molecule init role [OPTIONS] ROLE_NAME

  Initialize a new role for use with Molecule.

Options:
  --dependency-name [galaxy]      Name of dependency to initialize. (galaxy)
  -d, --driver-name [delegated|podman]
                                  Name of driver to initialize. (delegated)
  --lint-name [yamllint]          Name of lint to initialize. (yamllint)
  --provisioner-name [ansible]    Name of provisioner to initialize. (ansible)
  --verifier-name [ansible|testinfra]
                                  Name of verifier to initialize. (ansible)
  --help                          Show this message and exit.
```

## роли

Роли в домашнем каталоге
```text
cd .ansible
mkdir roles
cd roles
```
Имя роли `^[a-z][a-z0-9_]+$`
```text
molecule init role ansible_distkontrol --driver-name=podman
```
Поправить файл meta/main.yml , что будет понятно после запуска
```text
ansible-lint ansible_distkontrol
```
Список образов:
```text
platforms:
  - name: rhel8
    image: registry.access.redhat.com/ubi8/ubi-init
    tmpfs:
      - /run
      - /tmp
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    capabilities:
      - SYS_ADMIN
    command: "/usr/sbin/init"
    pre_build_image: true
  - name: ubuntu
    image: geerlingguy/docker-ubuntu2004-ansible
    tmpfs:
      - /run
      - /tmp
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    capabilities:
      - SYS_ADMIN
    command: "/lib/systemd/systemd"
    pre_build_image: true
```
Выбрал ubuntu из-за неудач установить usbip в centos8
```text
platforms:
  - name: ubuntu
    image: geerlingguy/docker-ubuntu2004-ansible
    tmpfs:
      - /run
      - /tmp
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    capabilities:
      - SYS_ADMIN
    command: "/lib/systemd/systemd"
    pre_build_image: true
```
Указать нужное в `molecule/default/molecule.yml` и добавить в конец
```text
lint: |
  set -e
  yamllint .
  ansible-lint .
```
Там же
```text
provisioner:
  name: ansible
  config_options:
    defaults:
      interpreter_python: auto_silent
      callback_whitelist: profile_tasks, timer, yaml
    ssh_connection:
      pipelining: false
```
В каталоге роли выполнить
```text
molecule lint
```
Если все пакеты podman и тд установлены корректно, файлы конфигураций сделаны, то
```text
molecule create
```
Будет создан имадж
```text
TASK [Create molecule instance(s)] *********************************************
Saturday 13 February 2021  18:55:43 +0000 (0:00:00.124)       0:00:02.145 *****
changed: [localhost] => (item={'capabilities': ['SYS_ADMIN'], 'command': '/lib/systemd/systemd', 'image': 'geerlingguy/docker-ubuntu2004-ansible', 'name': 'ubuntu', 'pre_build_image': True, 'tmpfs': ['/run', '/tmp'], 'volumes': ['/sys/fs/cgroup:/sys/fs/cgroup:ro']})
...
Saturday 13 February 2021  19:01:00 +0000 (0:05:16.421)       0:05:19.320 *****
===============================================================================
Wait for instance(s) creation to complete ----------------------------- 316.42s
Discover local Podman images -------------------------------------------- 1.27s
Create molecule instance(s) --------------------------------------------- 0.75s
Check presence of custom Dockerfiles ------------------------------------ 0.45s
Determine the CMD directives -------------------------------------------- 0.12s
Log into a container registry ------------------------------------------- 0.10s
Create Dockerfiles from image names ------------------------------------- 0.10s
Build an Ansible compatible image --------------------------------------- 0.07s
Playbook run took 0 days, 0 hours, 5 minutes, 19 seconds
INFO     Running default > prepare
WARNING  Skipping, prepare playbook not configured.
$ molecule list
INFO     Running default > list
                ╷             ╷                  ╷               ╷         ╷
  Instance Name │ Driver Name │ Provisioner Name │ Scenario Name │ Created │ Converged
╶───────────────┼─────────────┼──────────────────┼───────────────┼─────────┼───────────╴
  ubuntu        │ podman      │ ansible          │ default       │ true    │ false
                ╵             ╵                  ╵               ╵         ╵
$ podman ps
CONTAINER ID  IMAGE                                                   COMMAND               CREATED        STATUS            PORTS   NAMES
a9d4b2366016  docker.io/geerlingguy/docker-ubuntu2004-ansible:latest  /lib/systemd/syst...  3 minutes ago  Up 3 minutes ago          ubuntu
```
При разработке роли Molecule использует запущенные инстансы для ее тестирования.
Если тест проваливается или какая-то ошибка приводит к необратимым изменениям, из-за которых все надо начинать сначала,
вы можете в любое время убить эти инстансы командой `molecule destroy` и создать их заново командной `molecule create`.

Подключиться
```text
$ molecule login -h ubuntu
INFO     Running default > login
root@ubuntu:/#
```
Применить роль `molecule converge` и подключиться повторно, проверить работу установленных приложений.
Если все работает, то `molecule test`.

## molecule driver docker

```sh
pip install molecule-docker
```
Устанавливал докер через snap
```sh
sudo snap install docker
```
Создание роли
```sh
molecule init role v98765_vmalert --driver-name=docker
```
Настройка молекулы для запуска докера от root, т.к. прав не хватит. molecule/default/molecule.yml
```yaml
---
dependency:
  name: galaxy
driver:
  name: docker
platforms:
  - name: instance_vmalert
    privileged: true
    image: docker.io/pycontribs/centos:8
    tmpfs:
      - /run
      - /tmp
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    capabilities:
      - SYS_ADMIN
    command: "/lib/systemd/systemd"
    pre_build_image: true
provisioner:
  name: ansible
  config_options:
    defaults:
      interpreter_python: auto_silent
      callback_whitelist: profile_tasks, timer, yaml
    ssh_connection:
      pipelining: false
verifier:
  name: ansible
lint: |
  set -e
  yamllint .
  ansible-lint .
```

## Ссылки
[Developing and Testing Ansible Roles with Molecule and Podman - Part 1](https://www.ansible.com/blog/developing-and-testing-ansible-roles-with-molecule-and-podman-part-1), 
[Developing and Testing Ansible Roles with Molecule and Podman - Part 2](https://www.ansible.com/blog/developing-and-testing-ansible-roles-with-molecule-and-podman-part-2), 
[Разработка и тестирование Ansible-ролей с использованием Molecule и Podman](https://habr.com/ru/company/redhatrussia/blog/519452/), 
[Podman Installation Instructions](https://podman.io/getting-started/installation), 
[Basic Setup and Use of Podman](https://github.com/containers/podman/blob/master/docs/tutorials/podman_tutorial.md), 
[Basic Setup and Use of Podman in a Rootless environment](https://github.com/containers/podman/blob/master/docs/tutorials/rootless_tutorial.md)
