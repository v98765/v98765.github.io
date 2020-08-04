## установка

[tac_plus](https://github.com/facebookarchive/tac_plus), поэтому не нужно искать старые репы для centos. Для ubuntu 16(xenial),18 (bionic) есть в пакетах
```text
tacacs+/xenial 4.0.4.27a-1 amd64
  TACACS+ authentication daemon

tacacs+/bionic 4.0.4.27a-3 amd64
  TACACS+ authentication daemon
```
## пароли

```text
$ tac_pwd 
Password to be encrypted: test
Q3SdrVnvYFMnM
```
В конфиге будет так
```text
login = des Q3SdrVnvYFMnM
```
или так
```text
login = cleartext test
```

## настройка

Кофигурационный файл /etc/tac_plus.conf. При его изменении релоад сервису `systemctl reload tac_plus`

Ключ в открытом виде, указанный в оборудовании. Один для всех устройств.
```text
key = "ключ шифрования"
```
Группа администраторов с максимальным уровнем привилегий 15 на cisco, huawei, eltex, dlink и тп.
Почитать про [tacacs в junos](https://www.juniper.net/documentation/en_US/junos/topics/topic-map/user-access-tacacs-authentication.html) и в [KB Configuration for TACACS Plus](https://kb.juniper.net/InfoCenter/index?page=content&id=KB24265&actp=METADATA)
А вот ограниченные привилегии работают не везде, возможно из-за некорректных настроек.
Можно еще команды ограничивать, примеры в интернетах, но проверять как оно будет работать на оборудовании НЕ cisco.
Поэтому имеет смысл отказаться от таких учетных записей с якобы ограничением по списку выполняемых команд вовсе.

service у разных вендоров отличается. Большинству подходит exec.

```
group = admins {
        default service = permit
        login = PAM
        service = exec {
                priv-lvl = 15
                idletime = 600
                }
        }
```
Списки для ограничения доступа по адресу оборудования. Разрешено авторизоваться (tacacs source ip) на устройствах с адресом 10.9.8.*
```text
acl = acl8 {
    permit = ^10\.9\.8\.
    deny   = ^10\.
    }
```
Учетка username1 с acl
```text
user = username1 {
    member = admins
    login = des secret
    acl = voip
    }
user = username2 {
    member = admins
    login = des secret
    }
```
