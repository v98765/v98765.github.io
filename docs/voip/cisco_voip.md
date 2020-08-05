## cisco phone reset

Сброс Cisco IP Phone. Отключить питание телефона.Зажав  # на телефоне, включить питание и отпустить после индикации на трубе.
Нажать все кнопки поочередно 123456789*0#

Перезагрузка Cisco IP Phone/ Перейти в меню "Настройка", нажать ##*##



## fxs sip password

Есть некоторые особенности при использовании маршрутизатора Cisco в качестве FXS-шлюза. Например, в пароле не может быть символа "?".
```text
dial-peer voice 220 pots
 destination-pattern 220
 authentication username 220 password 7 02110C4218091C245E47060C16
 port 0/0/0
 forward-digits 0
```

## connection-reuse

При регистрации учетных данных со стороны маршрутизатора будет по умолчанию использоваться порт, отличный от 5060.
Несмотря на успешную регистрацию, использовать отладку debug ccsip не получится, т.к. данная команда,
как оказалось, может показывать сообщения принятые только на порт 5060. Эдакий tcpdump port 5060.
Предлагается решить эту задачу через connection-reuse
```text
sip-ua 
 registrar dns:pbx.a.b expires 3600
 sip-server dns:pbx.a.b
 connection-reuse
```

## white list, source-interface

Обязательно требуется указать список доверенных хостов. Так как INVITE придет от хоста pbx.a.b, необходимо учитывать возможную смену адреса ip-атс.
```text
voice service voip
 ip address trusted list
  ipv4 100.99.999.1
 fax protocol t38 version 0 ls-redundancy 0 hs-redundancy 0 fallback none
 sip
  bind control source-interface Loopback0
  bind media source-interface Loopback0
```
