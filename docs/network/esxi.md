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


Intel-i40en                     Network driver for Intel(R) X710/XL710/XXV710/X722 Adapters                 1.14.1.0-1OEM.700.1.0.15843807       i40en-1.14.1.0                                       Intel                       07-16-2021     VMwareCertified
https://www.intel.com/content/www/us/en/embedded/products/networking/nvm-update-tool-vmware-esx-quick-usage-guide.html
https://www.intel.com/content/www/us/en/download/18190/non-volatile-memory-nvm-update-utility-for-intel-ethernet-network-adapter-700-series.html

cd /var/tmp
wget http://flow.local/fw/700Series_NVMUpdatePackage_v8_50_ESX.tar.gz
tar zxvf 700Series_NVMUpdatePackage_v8_50_ESX.tar.gz
cd 700Series/ESXi_x64
./nvmupdaten64e -h


wget http://flow.local/fw/700Series_NVMUpdatePackage_v8_30_ESX.tar.gz
mkdir 83
tar zxvf 700Series_NVMUpdatePackage_v8_30_ESX.tar.gz -C 83
cd 83/700Series/ESXi_x64/
./nvmupdaten64e -v

Intel(R) Ethernet NVM Update Tool
NVMUpdate version 1.37.1.1
Copyright(C) 2013 - 2021 Intel Corporation.

Version:
QV SDK    - 2.37.01.00
igb       - unknown
ixgbe     - unknown
i40e      - unknown
igbn      - 1.5.2.0-1OEM.700.1.0.15843807
ixgben    - 1.10.3.0-1OEM.700.1.0.15843807
i40en     - 1.14.1.0-1OEM.700.1.0.15843807
icen      - 1.6.2.0-1OEM.700.1.0.15843807

 ./nvmupdaten64e -b

Intel(R) Ethernet NVM Update Tool
NVMUpdate version 1.37.1.1
Copyright(C) 2013 - 2021 Intel Corporation.


WARNING: To avoid damage to your device, do not stop the update or reboot or power off the system during this update.
Inventory in progress. Please wait [****-.....]


Num Description                          Ver.(hex)  DevId S:B    Status
=== ================================== ============ ===== ====== ==============
01) Intel(R) Ethernet Converged         4.37(4.25)   1572 00:064 Update
    Network Adapter X710-4                                       available

Options: Adapter Index List (comma-separated), [A]ll, e[X]it
Enter selection: X

Tool execution completed with the following status: Update available for one or more adapters.
Press any key to exit.


esxcli network nic list
 vmware -v
VMware ESXi 7.0.3 build-18905247


[root@h0062:~] esxcli network nic list
Name    PCI Device    Driver      Admin Status  Link Status  Speed  Duplex  MAC Address         MTU  Description
------  ------------  ----------  ------------  -----------  -----  ------  -----------------  ----  -----------
vmnic0  0000:1c:00.0  i40en       Up            Up            1000  Full    3c:ec:ef:05:48:18  1500  Intel(R) Ethernet Connection X722 for 1GbE
vmnic1  0000:1c:00.1  i40en       Up            Down             0  Half    3c:ec:ef:05:48:19  1500  Intel(R) Ethernet Connection X722 for 1GbE
vmnic2  0000:5e:00.0  nmlx5_core  Up            Up           10000  Full    b8:ce:f6:dc:74:22  1500  Mellanox Technologies ConnectX-4 Lx EN NIC; 25GbE; dual-port SFP28; (MCX4121A-ACA)
vmnic3  0000:5e:00.1  nmlx5_core  Up            Up           10000  Full    b8:ce:f6:dc:74:23  1500  Mellanox Technologies ConnectX-4 Lx EN NIC; 25GbE; dual-port SFP28; (MCX4121A-ACA)

[root@h0062:~] vmkchdev -l | grep vmnic
0000:1c:00.0 8086:37d1 15d9:37d1 vmkernel vmnic0
0000:1c:00.1 8086:37d1 15d9:37d1 vmkernel vmnic1
0000:5e:00.0 15b3:1015 15b3:0003 vmkernel vmnic2
0000:5e:00.1 15b3:1015 15b3:0003 vmkernel vmnic3
[root@h0062:~] esxcli network nic get -n vmnic3
   Advertised Auto Negotiation: true
   Advertised Link Modes: Auto, 1000BaseCX-SGMII/Full, 10000BaseKR/Full, 25000BaseTwinax/Full
   Auto Negotiation: true
   Cable Type: FIBRE
   Current Message Level: -1
   Driver Info:
         Bus Info: 0000:5e:00:1
         Driver: nmlx5_core
         Firmware Version: 14.30.1004
         Version: 4.21.71.101
