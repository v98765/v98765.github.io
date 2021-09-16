web,tftp-сервера

Установить сервера

```text
apt install lighttpd atftpd
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

Указать для tftpd рабочий каталог
```text
echo 'tftp          dgram   udp     wait    nobody /usr/sbin/tcpd /usr/sbin/in.tftpd --tftpd-timeout 300 --retry-timeout 5 --mcast-port 1758 --mcast-addr 239.239.239.0-255 --mcast-ttl 1 --maxthread 100 --verbose=5 /var/www/html/fw' |  inetd2rlinetd --force-overwrite --add-from-comment
```
Перезапустить сервис
```text
systemctl restart rlinetd
```
Проверка
```text
systemctl status rlinetd | grep tftp
```
В логах будет `rlinetd[x]:service tftp_udp enabled`
