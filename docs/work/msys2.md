[msys2](https://www.msys2.org) нужен для установки ansible в windows

```text
$ pacman -S python3-pip
$ pip3 install --upgrade pip
$ pacman -S gcc make libffi-devel openssl-devel libcrypt-devel
$ pacman -S sshpass
$ pip install wheel
$ pip install ansible
$ pip install scp
```
tnx [источник](https://titanwolf.org/Network/Articles/Article?AID=82172110-76b5-4a41-b107-2466bbbf4d9e#gsc.tab=0)

Установка коллекций
```text
$ ansible-galaxy collection install ansible.netcommon
$ ansible-galaxy collection install cisco.nxos
```
с последним пока проблема из-за ошибок с симлинками. похоже на отсутствие прав
[https://stackoverflow.com/questions/6260149/os-symlink-support-in-windows/8464306#8464306](https://stackoverflow.com/questions/6260149/os-symlink-support-in-windows/8464306#8464306)
