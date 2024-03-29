## DNS-сервера НСДИ

Информация взята из документа "Инструкция по подключению операторов связи и владельцев АС к Национальной системе доменных имен (НСДИ)"
Для использования публичных серверов (резолверов) НСДИ владельцу автономной системы необходимо изменить и/или добавить в список DNS-серверов для конечного клиента адреса резолверов НСДИ.

имя | a.res-nsdi.ru | b.res-nsdi.ru
---|---|---
ipv4 | 195.208.4.1 | 195.208.5.1
ipv6 | 2a0c:a9c7:8::1 | 2a0c:a9c7:9::1

Любой из указанных адресов может быть использован в качестве основного или дополнительного DNS-сервера.

## Другое

[Популярные DNS-провайдеры](https://kb.adguard.com/ru/general/dns-providers)

В корпоративных сетях, сервисах, которые разворачиваются с использованием MS AD, dns-сервером выступают контроллеры домена.

## DoH

Для DNS over HTTPS (DoH) нужен [dnscrypt-proxy](https://github.com/DNSCrypt/dnscrypt-proxy/wiki/installation), либо настройка в браузерах.

## Кто на 53

Нужны административные права, чтобы посмотреть процесс

```text
sudo ss -lp 'sport = :domain'
```
Или
```text
sudo netstat -lp | grep ':domain'
```

## Утилиты

Все записи
```text
dig domain ANY
```
Либо почти все
```text
nslookup -q=any domain
```
В ubuntu
```text
resolvectl query -t any domain
```
Кратко
```text
dig domain +short
host domain
```
Тупо
```text
ping domain
```
whois
```text
whois domain
```
