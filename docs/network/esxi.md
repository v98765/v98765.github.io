## list

```text
esxcli network nic list
```

## flowcontrol

По умолчанию включен на всех интерфейсах
```text
esxcli network nic pauseParams list
```
Добавить в /etc/rc.local.d/local.sh
```text
/bin/esxcli network nic pauseParams set --nic-name=vmnicX --rx=false --tx=false
```

## rx buffer

[Troubleshooting network receive traffic faults and other NIC errors in ESXi (50121760)](https://kb.vmware.com/s/article/50121760)

```text
[root@exsi:~] esxcli network nic stats get -n vmnic2  | grep Receive
   Receive packets dropped: 0
   Receive length errors: 0
   Receive over errors: 50571
   Receive CRC errors: 0
   Receive frame errors: 0
   Receive FIFO errors: 748049
   Receive missed errors: 0
```
Посмотреть
```text
esxcli network nic ring preset get -n vmnic2
esxcli network nic ring current get -n vmnic2
```
Дропы фиксируются в т.ч. в счетчиках ifInErrors, которые можно опросить по snmp.

## snmp

```text
esxcli system snmp set --communities public
esxcli system snmp set --enable true
```
