web,tftp-сервера

Установить сервера

```text
apt install lighttpd dnsmasq
```

Чтобы показывало листинг файлов в каталоге, создать `/etc/lighttpd/conf-enabled/10-dir-listing.conf`
```text
dir-listing.encoding = "utf-8"
server.dir-listing   = "enable"
```
Создать каталог
```text
mkdir /var/www/html/fw
```
Перезапустить сервис
```text
systemctl restart lighttpd
```

Настроить tftp в dnsmasq, создав новый конфигурационный файл `/etc/dnsmasq.d/tftp.conf`
```text
enable-tftp
tftp-root=/var/www/html/fw
```
Перезапустить сервис
```text
systemctl restart dnsmasq
```
