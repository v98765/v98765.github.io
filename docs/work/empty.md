Актуально, если сейчас 2014 год, либо эксплуатируемое оборудование - дрова.
Для скриптов нужен список хостов, поэтому в текущем каналоге файл LIST, где будут построчно перечислены адреса/имена устройств:
```text
1.2.3.4
5.6.7.8
```

## sshpass

Простой скрипт для копирования конфигураций с оборудования cisco, где предварительно включен scp server.
```bash
#!/bin/bash

read -p "Username: "  username
echo -n 'Password:'
read -s SSHPASS
export SSHPASS
mkdir -p cfg
cat LIST | while read line ; do
    if ping -q -c 2 $line > /dev/null 2>&1; then
        sshpass -e scp -q $username@${line}:startup-config cfg/${line}.cfg
        rc=$?; echo "instance: $line, code: $rc"
    fi
done

unset SSHPASS
echo $SSHPASS
```

## empty-expect

В ubuntu установить пакет [empty-expect](http://empty.sourceforge.net/)
```text
apt install empty-expect
```
Можно подключаться telnet'ом, ssh. Практически универсальное решение, если у устройства нет никаких проблем с эхом,
как у коммутаторов tfortis,например.
Пользоваться просто и относительно удобно, если сделать все через функции, чтобы к каждому коммутатору было индивидуальное подключение со своим логом и тп.
Так по размеру лога можно судить об успешности выполнения скрипта, например.
Чтобы empty не тупило при запуске необходимо подготовить список хостов и предварительно проверить их доступность.

Если список команд для выполнения на устройстве постоянно меняется и нужен более универсальный скрипт, то такие команды писал в отдельный файл CMD.
Первые строки - это всегда отключение пейжера и прочие команды, снимающие ограничения по выводу текста.
Можно делать так, как удобнее.

Juniper Junos
```text
set cli terminal xterm
set cli screen-length 0
set cli screen-width 0
show system alarm
show chassis alarms
show chassis hardware | no-more
```

Cisco IOS
```text
terminal length 0
show int status
```

Eltex
```text
terminal datadump
show startup
```

Цикл для последовательного выполнения команд из файла CMD. `sleep` добавить, если нужно.
```text
cat ./CMD | while read line ; do
    CmdStr=$(echo $line | tr -d "\r\n")
    empty -w -i fifoputacl/out.$Host -o fifoputacl/inc.$Host "#" "${CmdStr}\n"
done
```

Учетные записи для подключения можно вводить при запуске скрипта, можно указывать в скрипте, либо брать из файла. Два варианта на выбор
```text
empty -w -i fifostatus/out.$Host -o fifostatus/inc.$Host "assword:" "${Password}\n"
empty -s -i fifostatus/out.$Host -o fifostatus/inc.$Host < ./mypass
```

Ниже скрипт. Сначала описываются функции.
```bash
#!/bin/bash

# проверка доступности хоста. вернуть 1, если не пингуется.
function PingIt {
    Host=$1    
    if ! ping -q -c 2 $Host > /dev/null 2>&1 ; then
        echo "$Host down"
        return 1
    fi
}

# telnet на каждый хост из списка.
# функции передается хост, имя пользователя и пароль
function ReadStatus {
    Host=$1
    Username=$2
    Password=$3
    #удаление данных от предыдущего запуска empty
    rm -f logstatus/$Host
    rm -f fifostatus/inc.$Host
    rm -f fifostatus/out.$Host
    #login
    empty -f -i fifostatus/inc.$Host -o fifostatus/out.$Host -p pidstatus/$Host -L logstatus/$Host telnet $Host
    sleep 2
    empty -w -i fifostatus/out.$Host -o fifostatus/inc.$Host "name" "${Username}\n"
    sleep 2
    empty -w -i fifostatus/out.$Host -o fifostatus/inc.$Host "assword:" "${Password}\n"
    sleep 2
    empty -w -i fifostatus/out.$Host -o fifostatus/inc.$Host "#" "\n"
    # если после ввода пароля не получен символ #, то выйти
    if [ $? -eq 255 ] ; then
        empty -k `cat pidstatus/$Host`
        echo "$Host Incorrect password or priv lvl"
        return
    fi
    echo ''
    echo "Connected to $Host"
    #последовательный ввод команд. Тут можно сделать while для файла CMD
    empty -w -i fifostatus/out.$Host -o fifostatus/inc.$Host "#" "terminal length 0\n"
    empty -w -i fifostatus/out.$Host -o fifostatus/inc.$Host "#" "show int status\n"
    empty -w -i fifostatus/out.$Host -o fifostatus/inc.$Host "#" "exit\n"
    sleep 2
    #отключение от коммутатора, empty отработал и теперь удалить из логов все, кроме необходимого
    echo "got iface name logstatus/$Host"
    #del all but ports
    sed -i '/\(^Fa\|^Gi\|^Te\)/!d; ' logstatus/$Host
}

#каталоги для empty
mkdir -p logstatus
mkdir -p fifostatus
mkdir -p pidstatus

#ввод учетки
echo -n Username:
read Username

echo -n Password:
read -s Password

#запуск в цикле функции PingIt для всех хостов. Если успех, то запуск ReadStatus с &.
for Host in `cat LIST` ; do
   echo "$Host start"
   PingIt "$Host" && ReadStatus "$Host" "$Username" "$Password" &
done
```
