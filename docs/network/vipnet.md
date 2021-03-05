## hw50

Настраивать с консоли на скорости 38400. По умолчанию пользователь user и пароль user. Выбрать cli нажатием "1"
```text
1) command line interface                                   
2) full-screen interface                                    
Please select setup wizard operating mode :             

Welcome to the ViPNet Coordinator HW50 4.3.2-3421!
You must install keys or restore saved configuration
Would you like to start installing keys or restoring configuration? [Yes/No] : 

Would you like to install keys from TFTP, USB or CD storage device? [t/u/c] : u

Insert USB storage device with DST or VBE file and press <Enter>

Try to mount /dev/sdb1 as vfat
```
Следом вводится пароль пользователя RemoteOffceN. Пароль есть в дистрибутиве ключей вместе с паролем администратора.
```text
Enter password: 
Host name: RemoteOffceN
User ID: 1167006D
User password successfully checked
Hardware platform HW50-N1 is detected
Releasing all resources...
Configure interface eth0? [Yes/No]:Yes

Use dhcp on the interface eth0? [Yes/No] : No

Enter interface IP-address : 893.888.43.125

Enter interface netmask : 255.255.255.224

Configure interface eth1? [Yes/No]:No

Configure interface eth2? [Yes/No]:No

Enter ip-address of the default gateway : 893.888.43.97

Do you want to use DNS server? [Yes/No] : No

Do you want to use NTP daemon to synchronize the time? [Yes/No] : Yes

This node will use public NTP servers for time synchronization by default.
Do you want to add custom NTP server? [Yes/No] : Yes

Enter IP-address or domain name of the custom NTP server : 194.190.168.1

Enter hostname (default hw50-1167006a)
Only latin letters, digits and '-' symbols are allowed (2-63 symbols)
Or press Enter to leave default value : 

The current virtual IP address range is: 11.0.0.1/8
Do you want to specify custom virtual IP address range? [Yes/No] : No

Do you want to probe VPN-connection with some host in order to verify the configuration you've just made? [Yes/No] : No

Do you want to start VPN services before leaving the installation wizard? [Yes/No] : Yes

Do you want to start the command shell now? [Yes/No] : Yes
```
На этом начальная настройка завершена.

Правила фильтрации по умолчанию:
```text 
hw50-1167006a# firewall rules show
 Service Vpn Rules:
================================================================================
Num   Name                                               Option        Schedule 
Act   Protocol                  Source                 > Destination            
================================================================================
1     Block not original udp port                        Generated              
drop  udp:                      @local                 > @any                   
       from 0-2045                                                              
       to 2046,                                                                 
      udp:                                                                      
       from 2047-65535                                                          
       to 2046                                                                  
--------------------------------------------------------------------------------
 Vpn Rules:
================================================================================
Num   Name                                               Option        Schedule 
Act   Protocol                  Source                 > Destination            
================================================================================
1     Allow DHCP Service                                 User                   
pass  udp:                      @any                   > @any                   
       from 67                                                                  
       to 68                                                                    
--------------------------------------------------------------------------------
2     Allow DHCP Service                                 User                   
pass  udp:                      @any                   > @any                   
       from 68                                                                  
       to 67                                                                    
--------------------------------------------------------------------------------
3     Allow DHCP-Relay service                           User                   
pass  udp:                      @any                   > @any                   
       from 67                                                                  
       to 67                                                                    
--------------------------------------------------------------------------------
4     Allow ViPNet base services                         User                   
pass  udp:                      @any                   > @any                   
       from 2048                                                                
       to 2048,                                                                 
      udp:                                                                      
       from 2050                                                                
       to 2050                                                                  
--------------------------------------------------------------------------------
5     Allow ViPNet base services                         User                   
pass  udp:                      @any                   > @any                   
       to 2046                                                                  
--------------------------------------------------------------------------------
6     Allow ViPNet StateWatcher                          User                   
pass  tcp:                      @any                   > @any                   
       to 5100,                                                                 
      tcp:                                                                      
       to 10092                                                                 
--------------------------------------------------------------------------------
7     Allow ViPNet DBViewer                              User                   
pass  tcp:                      @any                   > @any                   
       to 2047                                                                  
--------------------------------------------------------------------------------
8     Allow ViPNet MFTP                                  User                   
pass  tcp:                      @any                   > @any                   
       to 5000-5003                                                             
--------------------------------------------------------------------------------
9     Allow ViPNet WebGui                                User                   
pass  tcp:                      @any                   > @local                 
       to 8080                                                                  
--------------------------------------------------------------------------------
10    Allow ICMP Ping                                    User                   
pass  icmp: 8                   @any                   > @any                   
--------------------------------------------------------------------------------
11    Allow SSH                                          User                   
pass  tcp:                      @any                   > @any                   
       to 22                                                                    
--------------------------------------------------------------------------------
12    Allow DNS                                          User                   
pass  udp:                      @any                   > @any                   
       to 53,                                                                   
      tcp:                                                                      
       to 53                                                                    
--------------------------------------------------------------------------------
13    Allow NTP                                          User                   
pass  udp:                      @any                   > @any                   
       to 123                                                                   
--------------------------------------------------------------------------------
14    Allow UPS service                                  User                   
pass  tcp:                      @any                   > @any                   
       to 3493                                                                  
--------------------------------------------------------------------------------
15    Allow syslog outgoing                              User                   
pass  udp:                      @local                 > @any                   
       to 514                                                                   
--------------------------------------------------------------------------------
16    Allow SNMP                                         User                   
pass  udp:                      @any                   > @local                 
       to 161                                                                   
--------------------------------------------------------------------------------
17    Allow SNMP traps                                   User                   
pass  udp:                      @local                 > @any                   
       to 162                                                                   
--------------------------------------------------------------------------------
 Vpn Default:
================================================================================
Num   Name                                               Option        Schedule 
Act   Protocol                  Source                 > Destination            
================================================================================
1     Block All Traffic                                  User                   
drop  @any                      @any                   > @any                   
--------------------------------------------------------------------------------
empty rule for  Nat Rules:
 Tunnel Rules:
================================================================================
Num   Name                                               Option        Schedule 
Act   Protocol                  Source                 > Destination            
================================================================================
1     To all tunnel nodes                                User                   
pass  @any                      @any                   > @tunneledip            
--------------------------------------------------------------------------------
2     From all tunnel nodes                              User                   
pass  @any                      @tunneledip            > @any                   
--------------------------------------------------------------------------------
 Tunnel Default:
================================================================================
Num   Name                                               Option        Schedule 
Act   Protocol                  Source                 > Destination            
================================================================================
1     Block All Traffic                                  User                   
drop  @any                      @any                   > @any                   
--------------------------------------------------------------------------------
 Service Local Rules:
================================================================================
Num   Name                                               Option        Schedule 
Act   Protocol                  Source                 > Destination            
================================================================================
1     ViPNet Service Common In                           Generated              
drop  tcp/udp:                  @any                   > @local                 
       to 2046,                                                                 
      tcp/udp:                                                                  
       to 2047,                                                                 
      tcp/udp:                                                                  
       to 10096,                                                                
      tcp/udp:                                                                  
       to 5100,                                                                 
      tcp/udp:                                                                  
       to 10092                                                                 
--------------------------------------------------------------------------------
2     ViPNet Service Common Out                          Generated              
drop  tcp/udp:                  @local                 > @any                   
       from 2046                                                                
--------------------------------------------------------------------------------
 Local Rules:
================================================================================
Num   Name                                               Option        Schedule 
Act   Protocol                  Source                 > Destination            
================================================================================
1     Allow DHCP Service                                 User                   
pass  udp:                      @any                   > @any                   
       from 67                                                                  
       to 68                                                                    
--------------------------------------------------------------------------------
2     Allow DHCP Service                                 User                   
pass  udp:                      @any                   > @any                   
       from 68                                                                  
       to 67                                                                    
--------------------------------------------------------------------------------
3     Allow DHCP-Relay service                           User                   
pass  udp:                      @any                   > @any                   
       from 67                                                                  
       to 67                                                                    
--------------------------------------------------------------------------------
4     Block ICMP timestamp response                      User                   
drop  icmp: 14                  @local                 > @any                   
--------------------------------------------------------------------------------
5     Allow ICMP Ping                                    User                   
pass  icmp: 8                   @local                 > @any                   
--------------------------------------------------------------------------------
6     Allow ICMP type 3 code 4 'fragmentation needed ... User                   
pass  icmp: 3/4                 @local                 > @any                   
--------------------------------------------------------------------------------
7     Allow DNS                                          User                   
pass  udp:                      @local                 > @any                   
       to 53,                                                                   
      tcp:                                                                      
       to 53                                                                    
--------------------------------------------------------------------------------
8     Allow NTP                                          User                   
pass  udp:                      @local                 > @any                   
       to 123                                                                   
--------------------------------------------------------------------------------
 Local Default:
================================================================================
Num   Name                                               Option        Schedule 
Act   Protocol                  Source                 > Destination            
================================================================================
1     Block All Traffic                                  User                   
drop  @any                      @any                   > @any                   
--------------------------------------------------------------------------------
empty rule for  Forward Rules:
 Forward Default:
================================================================================
Num   Name                                               Option        Schedule 
Act   Protocol                  Source                 > Destination            
================================================================================
1     Block All Traffic                                  User                   
drop  @any                      @any                   > @any                   
--------------------------------------------------------------------------------
```

Дополнительные временные правила для удаленного доступа по ssh и через web http://host:8080.
Для этого необходимо перейти в привилегированный режим командой enable, ввести пароль администратора
```text
firewall local add src @any dst @local tcp dport 22 pass
firewall local add src @any dst @local tcp dport 8080 pass
```

Проверка подключения на публичный адрес пользователем user
```text
ssh user@893.888.43.125
user@893.888.43.125's password: 
Last login: Fri Mar  5 13:42:46 MSK 2021 on ttyS0
Last login: Fri Mar  5 14:34:21 2021 from 893.888.43.97
Product: ViPNet Coordinator HW
Platform: HW50 N1
License: HW50 AU
Software version: 4.3.2-3421
(C) InfoTeCS JSC, 2020; website: www.infotecs.ru, email: soft@infotecs.ru; phone (Russia): 8 800 250-0-260, phone (Moscow): +7 495 737-61-92
Loading command shell, please wait...
Starting the command line interface of Platform: HW50
hw50-1167006a> 
```
Выключение производится кратковременным нажатием на кнопку питания. Далее как обычном ПК происходит автоматическое отключение с выключением сервисов linux

