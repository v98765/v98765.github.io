# switch mellanox с sonic community

## readme

Sonic community это софт, который можно установить на коммутатор и использовать на свой страх и риск. Установка ежедневной сборки master уже риск. Лучше собрать самому другую версию софта.
Функционала, который есть в sonic enterpise от производителей оборудования, в community версии нет и скорее всего не будет. Читать документацию вендоров и искать такой же функционал, например, sonic-cli, SAG, VRRP бессмысленно.
Фактически такой коммутатор с sonic community можно использовать как платформу, где работает FRR c BGP и где можно реализовать то, что можно сделать в линунксе стандартными пакетами программ для debian.

```
https://opencomputeproject.github.io/onie/developers/building.html
https://thedxt.ca/2023/08/onie-and-onyx-mlnx-os-install/
https://github.com/sonic-net/sonic-buildimage/
https://github.com/sonic-net/sonic-utilities/blob/master/doc/Command-Reference.md
https://qiita.com/masru0714
https://github.com/iMasaruOki/sonic-custom-build
https://techblog.rtbhouse.com/2021/11/25/L2-EVPN-to-L3-concept-part2/
https://github.com/sonic-net/SONiC/blob/master/doc/mgmt/sonic_stretch_management_vrf_design.md
https://netbergtw.com/top-support/netberg-sonic/virtual-routing-and-forwarding-vrf/
```

## serial

speed: 115200, databit: 8m, stop: 1, parity:none, flowcontrol: none.

## ONIE

Нужен для случаев, когда неудачное обновление ОС на коммутаторе удалило загрузчик.
Собирается по [инструкции командами](https://opencomputeproject.github.io/onie/developers/building.html) в [докере](https://github.com/opencomputeproject/onie/tree/master/contrib/build-env).
Каких-то пакетов может не хватать, их понадобится устрановить дополнительно и обязательно указать переменную `PATH`.
В итоге получается файл `onie/build/images/onie-recovery-x86_64-mlnx_x86-r0.iso`. Его нужно записать на флешку.

## BIOS

Чтобы загрузиться с флешки с ONIE, понадобиться при загрузке нажать Ctrl+B, ввести пароль bios `admin` и выбрать загрузку с флешки в меню Save \ Boot Override.
Далее выбрать ONIE: Embed ONIE и дождаться установки.

## build sonic

Ниже замечания к [инструкции](https://github.com/sonic-net/sonic-buildimage/), т.к нужна ubuntu 22.x, не 20. Как-то более-менее еще ветка 202305. В ОС создан пользователь `sonic`.

Установить докер
```
curl -fsSL https://get.docker.com -o install-docker.sh
sudo sh install-docker.sh
sudo gpasswd -a ${USER} docker
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
sudo reboot
```

Установить venv, j2cli
```
sudo apt install python3-pip python3-venv
python3 -m venv base
source base/bin/activate
pip3 install wheel j2cli
```

Отредактировать `rules/config` чтобы выключить разное.
```
diff --git a/rules/config b/rules/config
index 8ac564dea..3fd9b2e16 100644
--- a/rules/config
+++ b/rules/config
@@ -55,7 +55,7 @@ DEFAULT_PASSWORD = YourPaSsWoRd
 # ENABLE_DHCP_GRAPH_SERVICE = y

 # ENABLE_ZTP - installs Zero Touch Provisioning support.
-# ENABLE_ZTP = y
+ENABLE_ZTP = n

 # INCLUDE_PDE - Enable platform development enviroment
 # INCLUDE_PDE = y
@@ -92,8 +92,8 @@ ENABLE_ORGANIZATION_EXTENSIONS = y
 # as includes symbols information. Given that 'profiling' option is a superset
 # of 'debugging' one, user should only enable either one option or the other --
 # if both options are enabled, the 'profiling' one will prevail.
-#SONIC_DEBUGGING_ON = y
-#SONIC_PROFILING_ON = y
+SONIC_DEBUGGING_ON = n
+SONIC_PROFILING_ON = n

 # DEFAULT_KERNEL_PROCURE_METHOD - default method for obtaining kernel
 #   build:    build kernel from source
@@ -129,13 +129,13 @@ DEFAULT_VS_PREPARE_MEM = yes
 INCLUDE_SYSTEM_EVENTD = y

 # INCLUDE_SYSTEM_TELEMETRY - build docker-sonic-telemetry for system telemetry support
-INCLUDE_SYSTEM_TELEMETRY = y
+INCLUDE_SYSTEM_TELEMETRY = n

 # INCLUDE_ICCPD - build docker-iccpd for mclag support
 INCLUDE_ICCPD = n

 # INCLUDE_SFLOW - build docker-sflow for sFlow support
-INCLUDE_SFLOW = y
+INCLUDE_SFLOW = n

 # ENABLE_SFLOW_DROPMON - support of drop packets monitoring feature for sFlow deamon
 ENABLE_SFLOW_DROPMON = n
@@ -151,16 +151,16 @@ ENABLE_HOST_SERVICE_ON_START = y
 INCLUDE_RESTAPI = n

 # INCLUDE_NAT - build docker-nat for nat support
-INCLUDE_NAT = y
+INCLUDE_NAT = n

 # INCLUDE_DHCP_RELAY - build and install dhcp-relay package
-INCLUDE_DHCP_RELAY = y
+INCLUDE_DHCP_RELAY = n

 # INCLUDE_P4RT - build docker-p4rt for P4RT support
 INCLUDE_P4RT = n

 # ENABLE_AUTO_TECH_SUPPORT - Enable the configuration for event-driven techsupport & coredump mgmt feature
-ENABLE_AUTO_TECH_SUPPORT = y
+ENABLE_AUTO_TECH_SUPPORT = n

 # ENABLE_TRANSLIB_WRITE - Enable translib write/config operations via the gNMI interface.
 # Uncomment to enable:
@@ -171,7 +171,7 @@ ENABLE_AUTO_TECH_SUPPORT = y
 ENABLE_NATIVE_WRITE = y

 # INCLUDE_MACSEC - build docker-macsec for macsec support
-INCLUDE_MACSEC = y
+INCLUDE_MACSEC = n

 # INCLUDE_GBSYNCD - build docker-gbsyncd-* for gearbox support
 INCLUDE_GBSYNCD ?= y
```

Когда в составе ОС нужны дополнительные пакеты, то отредактировать `build_debian.sh`
```
diff --git a/build_debian.sh b/build_debian.sh
index f8de45c7f..ecaad2a51 100755
--- a/build_debian.sh
+++ b/build_debian.sh
@@ -352,6 +352,7 @@ fi
 ## Note: don't install python-apt by pip, older than Debian repo one
 ## Note: fdisk and gpg are needed by fwutil
 sudo LANG=C DEBIAN_FRONTEND=noninteractive chroot $FILESYSTEM_ROOT apt-get -y install      \
+    keepalived              \
     file                    \
     ifmetric                \
     iproute2                \
@@ -557,6 +558,7 @@ sudo https_proxy=$https_proxy LANG=C chroot $FILESYSTEM_ROOT pip3 install 'docke

 # Install scapy
 sudo https_proxy=$https_proxy LANG=C chroot $FILESYSTEM_ROOT pip3 install 'scapy==2.4.4'
+sudo https_proxy=$https_proxy LANG=C chroot $FILESYSTEM_ROOT pip3 install 'j2cli'
```

Сборка.
```
sudo mkdir /var/cache/sonic
sudo chown sonic /var/cache/sonic
git clone --recurse-submodules  https://github.com/sonic-net/sonic-buildimage.git -b 202305
cd sonic-buildimage/
sudo modprobe overlay

make init
make configure PLATFORM=mellanox
make SONIC_BUILD_JOBS=4 all
```
Если предыдущие попытки неудачные, то удалить староe `sudo docker rmi -f $(docker images -a -q)`


## install sonic

Когда ОС не установлена автоматически запускается ONIE-install. Включить менедж в eth0, прописать адрес, установить с веб-сервера собранный софт.
```
ONIE:/ # onie-discovery-stop
ONIE:/ # ifconfig eth0 10.0.0.100 netmask 255.255.255.0
ONIE:/ # ip route add default via 10.0.0.1
ONIE:/ # onie-nos-install http://10.1.2.3/sonic-mellanox-202305.bin
```

## default password

Логин `admin` пароль `YourPaSsWoRd`. Смена пароля без проверки на символы `sudo passwd admin`

## feature

Ниже есть iccpd, т.к. это попытка сборки с mclag
```
$ sudo show feature status
Feature         State            AutoRestart
--------------  ---------------  --------------
bgp             enabled          enabled
database        always_enabled   always_enabled
dhcp_relay      disabled         enabled
eventd          enabled          enabled
iccpd           enabled          enabled
lldp            enabled          enabled
mgmt-framework  enabled          enabled
mux             always_disabled  enabled
pmon            enabled          enabled
radv            enabled          enabled
snmp            enabled          enabled
swss            enabled          enabled
syncd           enabled          enabled
teamd           enabled          enabled
```

## config save

```
sudo config save -y
```

## hostname

Требуется перезагрузка
```
sudo config hostname sonic01
```

## management

```
sudo config vrf add mgmt
sudo config save -y
sudo config interface ip add Ethernet0 172.18.0.10/24
sudo config route add prefix vrf mgmt 0.0.0.0/0 nexthop 172.18.0.1
show mgmt-vrf
show mgmt-vrf routes
sudo ip vrf exec mgmt ping 172.18.0.1
sudo config save -y
```

## date ntp

Дата руками
```
sudo config clock timezone Europe/Moscow
sudo config clock date 2024-02-09 14:00:00
```
ntp
```
sudo config ntp add 10.0.0.10
sudo config ntp add 10.0.1.10
```
Проверка при условии настройки vrf mgmt
```
sudo  ip vrf exec mgmt ntpq -p
```

## monitorig

```
https://github.com/tynany/frr_exporter
https://github.com/mehdy/keepalived-exporter
https://github.com/vinted/sonic-exporter
```

Скопировать frr_exporter в докер
```
root@sonic:/etc/sonic/frr# docker cp /tmp/frr_exporter 70113ad4f31b:/usr/local/bin/
root@sonic:/etc/sonic/frr# docker exec -it 70113ad4f31b bash
root@sonic:/# /usr/local/bin/frr_exporter
```

При наличии sonic-exporter мониторить по snmp смысла мало
```
sudo config feature state snmp disable
```

## vrrp

По умолчанию никаких команд нет, пакета нет. Только в проекте использование keepalived. Поэтому, по совету masru0714 ставил пакет, настраивал примерно так.
С мультикастовым mac-адресом ничего не получилось.

```
$ cat /etc/keepalived/keepalived.conf
global_defs
{
  router_id sonic01
}
vrrp_instance i25 {
  state BACKUP
  interface Vlan25
  priority 99
  advert_int 1
  virtual_router_id 25
  virtual_ipaddress { 10.10.25.254 }
  unicast_src_ip 10.10.25.252
  unicast_peer { 10.10.25.251 }
}
```

## mclag

Настроить можно, но если указывать интерфейс mclag как в примере ниже, то на одном из коммутаторов периодически будет сломан LAG.
```
sudo config mclag add 1 192.168.10.2 192.168.10.1 PortChannel00
```

Вывод, где LAG сломан.
```
admin@sonic01:~$ mclagdctl -i 1 dump state
The MCLAG's keepalive is: OK
MCLAG info sync is: completed
Domain id: 1
Local Ip: 192.168.10.2
Peer Ip: 192.168.10.1
Peer Link Interface: PortChannel01
Keepalive time: 1
sesssion Timeout : 15
Peer Link Mac: 1c:34:da:f2:17:00
Role: Standby
MCLAG Interface:
Loglevel: NOTICE
admin@sonic01:~$ show interfaces portchannel
Flags: A - active, I - inactive, Up - up, Dw - Down, N/A - not available,
       S - selected, D - deselected, * - not synced
  No.  Team Dev       Protocol    Ports
-----  -------------  ----------  -------
   01  PortChannel01  N/A
admin@sonic01:~$

admin@sonic02:~$ mclagdctl -i 1 dump state
The MCLAG's keepalive is: OK
MCLAG info sync is: completed
Domain id: 1
Local Ip: 192.168.10.1
Peer Ip: 192.168.10.2
Peer Link Interface: PortChannel01
Keepalive time: 1
sesssion Timeout : 15
Peer Link Mac: 04:3f:72:2f:34:80
Role: Active
MCLAG Interface:
Loglevel: NOTICE
admin@sonic02:~$ show interfaces portchannel
Flags: A - active, I - inactive, Up - up, Dw - Down, N/A - not available,
       S - selected, D - deselected, * - not synced
  No.  Team Dev       Protocol     Ports
-----  -------------  -----------  -----------------------------
   01  PortChannel01  LACP(A)(Up)  Ethernet196(S) Ethernet192(S)
admin@sonic02:~$
```

## ipv6

Отключение.
```
sudo config ipv6 disable link-local
```
