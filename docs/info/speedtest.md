## сайты для проверки скорости скачивания

[fast.com](https://fast.com) удобен для тестирования интернета на спутниковых каналах.
После первого измерения скорости доступны настройки (Show more/Settings), где можно указать минимальное количество параллельных соединений.

[speed.cloudflare.com](https://speed.cloudflare.com/)

[yandex.ru/internet](https://yandex.ru/internet)

[speedtest.net](https://speedtest.net)

[librespeed.org](https://librespeed.org)

[fireprobe.net](https://www.fireprobe.net/) в тч [ping-test.ru](http://ping-test.ru/)

## сайты для прочих проверок

[help.kontur.ru/check](https://help.kontur.ru/check) для windows и требует установки плагинов. Проверяет наличие mitm сертификатов в антивирусе, прокси.
Пишет список доменов-исключений в настройках.

## консольные утилиты для проверки скорости

[iperf](https://iperf.fr/). Есть в пакетах. 3я версия сервера работает в tcp и udp режиме одновременно. Удобнее пользоваться второй версией.

Запуск сервера для тестирования udp пакетами:
```text
iperf -s -u
```
Запуск клиента:
```text
iperf -u -c <ip> -b 1M -t 300 -i 10
```
Для клиента iperf2 необязательно наличие сервера. Так, например, можно проверить доступную полосу при аренде канала.
Запустить клиента, указать адрес маршрутизатора или хоста за ним и посмотреть загрузку порта на коммутаторе, куда включен канал от оператора связи.

[fast.com](https://fast.com) есть консольная утилита. Команды: установить, проверить скорость скачивания, проверить аплоад.
```text
$ npm install --global fast-cli
$ fast
$ fast -u
```

[speedtest.net](https://speedtest.net) есть консольная утилита. Команды: установка, запуск.
```text
$ apt install speedtest-cli
$ speedtest
```

## калькулятор tcp

Без TCP win scaling  максимальный размер окна равен 65535 байт. Из rfc1323:

>(1)  Window Size Limit
>
>           The TCP header uses a 16 bit field to report the receive
>           window size to the sender.  Therefore, the largest window
>           that can be used is 2**16 = 65K bytes.

[https://wintelguy.com/wanperf.pl](https://wintelguy.com/wanperf.pl) калькулятор скорости скачивания с учетом в тч размера окна.
Можно оценить примерную скорость приложения в зависимости от rtt при окне 65535
и тогда будет понятна разница при работе с московскими серверами в Москве и в регионах, в т.ч. через спутниковые каналы связи с rtt 600мс.
От буфера приложения тоже зависит, например, проводник на ftp будет передавать данные на скорости не более 1мбит/с.
Информация об этом есть на сайте MS.
