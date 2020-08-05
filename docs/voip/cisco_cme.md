## CME 7.x

Это вообще 2010 год.

Для начала необходимо в 2800 или 3800 серию "залить" соответствующий IOS. Это может быть 12.4Т voice или advipservices.
PVDM требуется только при подключении к телефонной станции потоком E1, либо при организации на cisco ip-телефонах hardware конференций,
т.е. конференций, где участников больше чем 3.
Если маршрутизатор не подключается к телефонной станции, к нему планируются подключаться еще и телефонные SIP ip-шлюзы с FXS-портами,
то необходимо разрешить устанавливать соединения между ними.
Предполагается, что для локального использования будет достаточно трехзначной нумерации и со всеми другими маршрутизаторами
передача голоса будет осуществляться по H323-протоколу
```text
voice rtp send-recv
!
voice service voip 
 allow-connections h323 to h323
 allow-connections h323 to sip
 allow-connections sip to h323
 allow-connections sip to sip
 supplementary-service h450.12
 fax protocol t38 nse ls-redundancy 0 hs-redundancy 0 fallback cisco
 h323
 sip
```
Отсутствие строк allow-connections ** to ** в конфигурационном файле означает, что взаимодействие voip-устройств возможно только через dial-peer pots.

Пусть имеем некий интерфейс:
```text
interface FastEthernet0/1.100
 description LAN
 encapsulation dot1Q 100
 ip address 10.10.9.254 255.255.255.0

interface FastEthernet0/1.110
 description Voice LAN
 encapsulation dot1Q 110
 ip address 10.10.10.254 255.255.255.0
 h323-gateway voip bind srcaddr 10.10.10.254
```
Отсутствие лубкека и наличие желания фиксировать адрес, с которого будет устанавливаться rtp-сессия с другими маршрутизаторами,
подразумевает наличие строки h323-gateway voip bind srcaddr в интерфейсе.

Настраивается сервис телефонии для ip-телефонов Сisco.
```text
telephony-service
 ip source-address 10.10.10.254 port 2000
 calling-number initiator
 timeouts interdigit 3
 system message MyCompany
 cnf-file location flash:
 time-zone 42
 time-format 24
 date-format dd-mm-yy
 dialplan-pattern 1 ... extension-length 3
 max-conferences 6 gain -6
 call-forward pattern .T
 call-forward system redirecting-expanded
 transfer-system full-consult
 transfer-pattern .T
```
В настройках указан часовой пояс, который никак не связан с настройками часового пояса в маршрутизаторе.
Текущее время телефон получает только при загрузке конфигурации и самостоятельно никак не синхронизируется.
Поэтому важно синхронизировать маршрутизатор и не забывать осуществлять руками в сервисе телефонии переход на летнее/зимнее время.
Понятно, что смена часового пояса должна в итоге сопровождаться командой:
```text
router(config)#telephony-service 
router(config-telephony)#reset all
```
Разные модели маршрутизаторов поддерживают разное количество телефонов.
Таким образом, в зависимости от модели, оперативной памяти и еще чего-нибудь, задается количество телефонов и DN (что-то вроде профиля для номера).
```text
telephony-service
 max-ephones 30
 max-dn 100
```
Далее создаются телефонные номера. Например, хочу создать 10 номеров и чтобы все они раздались автоматически при подключении телефона.
```text
ephone-dn  10  dual-line
 number 210
!
!
ephone-dn  11  dual-line
 number 211
!
!
ephone-dn  12  dual-line
 number 212
!
!
ephone-dn  13  dual-line
 number 213
!
!
ephone-dn  14  dual-line
 number 214
!
!
ephone-dn  15  dual-line
 number 215
!
!
ephone-dn  16  dual-line
 number 216
!
!
ephone-dn  17  dual-line
 number 217
!
!
ephone-dn  18  dual-line
 number 218
!
!
ephone-dn  19  dual-line
 number 219

telephony-service
 auto assign 10 to 19
```
Настраиваются DHCP пулы, который будет раздавать адреса рабочим станциям и телефонам. Наличие опции 150 укажет им с какого адреса они будут скачивать конфигурационный файл в формате xml по протоколу tftp. В качестве tftp-сервера выступает сам маршрутизатор.
Часть адресов резервируется для голосовых шлюзов, которые должны иметь статические адреса.
```text
ip dhcp excluded-address 10.10.10.1 10.10.10.50

ip dhcp pool LAN
   network 10.10.9.0 255.255.255.0
   default-router 10.10.9.254 
   dns-server 10.0.0.1 10.0.1.1
   domain-name mycompany.com
   netbios-node-type h-node

ip dhcp pool VoiceLAN
   network 10.10.10.0 255.255.255.0
   default-router 10.10.10.254 
   dns-server 10.0.0.1 10.0.1.1 
   option 150 ip 10.10.10.254
```
Предполагается, что использоваться будет только кодек G729, который поддерживается всеми телефонами Cisco, в том числе и софтфоном Cisco Communicator. Поэтому после регистрации телефонов в каждом ephone прописывается:
```text
ephone  1
..
 codec g729r8
..
```
Если будут использоваться различные сервисы на телефонах, например конференции, то необходимо включить sccp.
```text
sccp local FastEthernet0/1.110
sccp
```

## Ad-Hoc конференция

Существуют два типа конференций:

1. Ad-Hoc, когда конференция создается с телефона cisco путем нажатия кнопки Confrn. Без настройки конференций (conference hardware) на маршрутизаторе максимальное число участников такой конференции равно трем.
2. Meetme, когда конференция создается с телефона cisco путем дозвона на предварительно созданный номер на маршрутизаторе. После чего все остальные участники с любого телефона могут звонить на тот же номер.

Сейчас будет рассматриваться только Ad-Hoc конференция. 
Считается, что один DSP поддерживает 8 конференций по 8 участников при использовании кодека G711 и 2 конференции по 8
участников при использовании кодека G729. Так же допускается настроить конференцию на 16 участников.
Мне пока неясно, как использовать больше чем один DSP. Пока все настраивается так, что больше чем один не может использоваться.
Соответственно, получаю лимит на 2 по 8.

В маршрутизаторе вставлен PVDM2, который необходимо настроить, включив dspfarm:
```text
router(config)#voice-card 0
router(config-voicecard)#?
Voice-card configuration commands:
  codec         Manage codec configuration parameters for voice card
  default       Set a command to its defaults
  dsp           Manage DSP configuration for the voice card
  dspfarm       Enable dspFarm feature for this voice card (command will be deprecated in future releases, use the new command 'dsp tdm pooling')
  exit          Exit from voice card configuration mode
  local-bypass  Enable TDM hairpinning
  no            Negate a command or set its defaults

router(config-voicecard)#dspfarm ?
  

router(config-voicecard)#dsp ?   
  allocation  dsp allocation scheme
  services    Manage DSP services configuration for the voice card
  tdm         Manage TDM configuration for the DSP on voice card

router(config-voicecard)#dsp services ?
  dspfarm  Enable dspfarm services on the Voice Card


router(config-voicecard)#dsp services dspfarm ?
  

router(config-voicecard)#dsp services dspfarm 
router(config-voicecard)#
```
В профиле dspfarm будут указываться тоны, которые проигрываются при входе и выходе из конференции:
```text
voice class custom-cptone jointone
 dualtone conference
  frequency 1200 1200
  cadence 150 50 150 50
!
voice class custom-cptone leavetone
 dualtone conference
  frequency 900 900
  cadence 150 50 150 50
```
Далее настраивается сам профиль. Обратите внимание, что в конфигурационном файле профиль по-умолчанию выключен:
```text
!
dspfarm profile 1 conference  
 codec g711ulaw
 codec g711alaw
 codec g729ar8
 codec g729abr8
 codec g729r8
 codec g729br8
 maximum sessions 2
 conference-join custom-cptone jointone
 conference-leave custom-cptone leavetone
 associate application SCCP
 shutdown
!
```
Чтобы задать профиль для sccp необходим идентификатор.
```text
sccp ccm 10.10.10.254 identifier 1 version 7.0 

sccp ccm group 1
 bind interface FastEthernet0/1.110
 associate ccm 1 priority 1
 associate profile 1 register mtp002414de5631
 keepalive retries 16
 keepalive timeout 10
```
Здесь mtp002414de5631 случайный набор понятных букв для указания их в сервисе телефонии. Напомню, что ранее sccp был уже включен:
```text
sccp local FastEthernet0/1.110
sccp
```
Правим сервис телефонии, чтобы указать там используемый профиль sccp и включить конференции:
```text
telephony-service
 sdspfarm units 1
 sdspfarm tag 1 mtp002414de5631
 conference hardware
```
Включаем профиль dspfarm
```text
router(config)#dspfarm profile 1 conference  
router(config-dspfarm-profile)#no sh
router(config-dspfarm-profile)#
```
Смотрим, что получилось:
```text
router#sh sccp all
SCCP Admin State: UP
Gateway Local Interface: FastEthernet0/1.110
        IPv4 Address: 10.10.10.254
        Port Number: 2000
IP Precedence: 5
User Masked Codec list: None
Call Manager: 10.10.10.254, Port Number: 2000
  Priority: N/A, Version: 7.0, Identifier: 1
  Trustpoint: N/A

SCCP application services(dspfarm profiles) are not enabled 
or haven't associated to SCCP CCM group

CCM Group Identifier: 1
 Description: None
 Binded Interface: FastEthernet0/1.110
        IPv4 Address: 10.10.10.254
 Associated CCM Id: 1, Priority in this CCM Group: 1
 Associated Profile: 1, Registration Name: mtp002414de5631
 Registration Retries: 3, Registration Timeout: 10 sec
 Keepalive Retries: 16, Keepalive Timeout: 10 sec
 CCM Connect Retries: 3, CCM Connect Interval: 10 sec
 Switchover Method: GRACEFUL, Switchback Method: GRACEFUL_GUARD
 Switchback Interval: 10 sec, Switchback Timeout: 7200 sec
 Signaling DSCP value: cs3, Audio DSCP value: ef
[skip]

router#sh dspfarm all
Dspfarm Profile Configuration

 Profile ID = 1, Service = CONFERENCING, Resource ID = 1  
 Profile Description :  
 Profile Service Mode : Non Secure 
 Profile Admin State : UP 
 Profile Operation State : RESOURCE ALLOCATED 
 Application : SCCP   Status : NOT ASSOCIATED 
 Resource Provider : FLEX_DSPRM   Status : UP 
[skip]
```
Есть ошибки, при наличии которых создать конференции нельзя.
Документация описывает причину, по которым такой статус возможен - не включен созданный профиль dspfarm.
Но тут необходимо обратить внимание на sccp, где, как оказывается, тоже важна последовательность команд.
```text
sccp local FastEthernet0/1.110
sccp ccm 10.10.10.254 identifier 1 version 7.0 
sccp
```
Любое изменение (определили идентификатор в данном случае) sccp должно сопровождаться выключением/включением sccp:
```text
router(config)#no sccp
router(config)#sccp
router(config)#^Z
router#sh dspfarm all
Dspfarm Profile Configuration

 Profile ID = 1, Service = CONFERENCING, Resource ID = 1  
 Profile Description :  
 Profile Service Mode : Non Secure 
 Profile Admin State : UP 
 Profile Operation State : ACTIVE 
 Application : SCCP   Status : ASSOCIATED 
 Resource Provider : FLEX_DSPRM   Status : UP
[skip] 
```
Для Ad-Hoc конференций необходимо создать DN-ки. Количество штук определяется максимальным количеством участников.
Если DN-ки dual-line , то необходимо вполовину меньше.
Пусть будет на 32 участника 16 DN dual-line:
```text
ephone-dn  100  dual-line
 number A0001
 name conference
 conference ad-hoc
 preference 1
 no huntstop
!
!
ephone-dn  101  dual-line
 number A0001
 name conference
 conference ad-hoc
 preference 2
 no huntstop
!
!
ephone-dn  102  dual-line
 number A0001
 name conference
 conference ad-hoc
 preference 3
 no huntstop
!
!
ephone-dn  103  dual-line
 number A0001
 name conference
 conference ad-hoc
 preference 4
 no huntstop
!
!
ephone-dn  104  dual-line
 number A0001
 name conference
 conference ad-hoc
 preference 5
 no huntstop
!
!
ephone-dn  105  dual-line
 number A0001
 name conference
 conference ad-hoc
 preference 6
 no huntstop
!
!
ephone-dn  106  dual-line
 number A0001
 name conference
 conference ad-hoc
 preference 7
 no huntstop
!
!
ephone-dn  107  dual-line
 number A0001
 name conference
 conference ad-hoc
 preference 8
 no huntstop
!
!
ephone-dn  108  dual-line
 number A0001
 name conference
 conference ad-hoc
 preference 9
 no huntstop
!
!
ephone-dn  109  dual-line
 number A0001
 name conference
 conference ad-hoc
 preference 9
 no huntstop
!
!
ephone-dn  110  dual-line
 number A0001
 name conference
 conference ad-hoc
 preference 9
 no huntstop
!
!
ephone-dn  111  dual-line
 number A0001
 name conference
 conference ad-hoc
 preference 9
 no huntstop
!
!
ephone-dn  112  dual-line
 number A0001
 name conference
 conference ad-hoc
 preference 9
 no huntstop
!
ephone-dn  113  dual-line
 number A0001
 name conference
 conference ad-hoc
 preference 9
 no huntstop
!
!
ephone-dn  114  dual-line
 number A0001
 name conference
 conference ad-hoc
 preference 9
 no huntstop
!
!
ephone-dn  115  dual-line
 number A0001
 name conference
 conference ad-hoc
 preference 9
 huntstop
!
```
Интерестно, что указывается number A0001. Непривычно. Для meetme необходимо указывать нормальный рабочий номер.
По настройкам - все.
Проверить работу конференции можно с зарегистрированного на CME телефона. Например, с Cisco IP Communicator'а, который поддерживает G711 и G729 кодеки.

Смотрим тестовую конференцию из трех участников:
```text
router#sh sccp all
SCCP Admin State: UP
Gateway Local Interface: FastEthernet0/1.110
        IPv4 Address: 10.10.10.254
        Port Number: 2000
IP Precedence: 5
User Masked Codec list: None
Call Manager: 10.10.10.254, Port Number: 2000
  Priority: N/A, Version: 7.0, Identifier: 100
  Trustpoint: N/A

Conferencing Oper State: ACTIVE - Cause Code: NONE
Active Call Manager: 10.10.10.254, Port Number: 2000
TCP Link Status: CONNECTED, Profile Identifier: 1
Reported Max Streams: 16, Reported Max OOS Streams: 0
Supported Codec: g711ulaw, Maximum Packetization Period: 30
Supported Codec: g711alaw, Maximum Packetization Period: 30
Supported Codec: g729ar8, Maximum Packetization Period: 60
Supported Codec: g729abr8, Maximum Packetization Period: 60
Supported Codec: g729r8, Maximum Packetization Period: 60
Supported Codec: g729br8, Maximum Packetization Period: 60
Supported Codec: rfc2833 dtmf, Maximum Packetization Period: 30
Supported Codec: rfc2833 pass-thru, Maximum Packetization Period: 30
Supported Codec: inband-dtmf to rfc2833 conversion, Maximum Packetization Period: 30


SCCP Application Service(s) Statistics:

Profile Identifier: 1, Service Type: Conferencing
TCP packets rx 1835, tx 1819
Unsupported pkts rx 0, Unrecognized pkts rx 0
Register tx 1, successful 1, rejected 0, failed 0
Unregister tx 0, successful 0
KeepAlive tx 1808, successful 1808, failed 0
OpenReceiveChannel rx 6, successful 6, failed 0
CloseReceiveChannel rx 3, successful 0, failed 0
StartMediaTransmission rx 6, successful 3, failed 0
StopMediaTransmission rx 3, successful 0, failed 0
PortReq rx 0
PortRes tx 0, successful 0, failed 0
PortClose rx 0
QosListen rx 0
QosPath rx 0
QosTeardown rx 0, send 0, recv 0, sendrecv 0
QosResvNotify tx 0, send 0, recv 0, sendrecv 0
QosErrorNotify tx 0, send 0, recv 0, sendrecv 0
   err0 0, err1 0, err2 0, err3 0, err4 0, err5 0,
   err6 0, err7 0, err8 0, err9 0, err10 0, err11 0,
   err12 0
QosModify rx 0, send 0, recv 0, sendrecv 0
UpdateDscp rx 0
Reset rx 0, successful 0, failed 0
MediaStreamingFailure rx 6
MediaStreamingFailure tx 0
Switchover 0, Switchback 0

CCM Group Identifier: 1
 Description: None
 Binded Interface: FastEthernet0/1.110
        IPv4 Address: 10.10.10.254
 Associated CCM Id: 1, Priority in this CCM Group: 1
 Associated Profile: 1, Registration Name: mtp002414de5631
 Registration Retries: 3, Registration Timeout: 10 sec
 Keepalive Retries: 16, Keepalive Timeout: 10 sec
 CCM Connect Retries: 3, CCM Connect Interval: 10 sec
 Switchover Method: GRACEFUL, Switchback Method: GRACEFUL_GUARD
 Switchback Interval: 10 sec, Switchback Timeout: 7200 sec
 Signaling DSCP value: cs3, Audio DSCP value: ef

sess_id    conn_id      stype mode     codec   sport rport ripaddr

3221356546 65542        conf  sendrecv g729    17742 2000  10.10.10.254
3221356546 65541        conf  sendrecv g729    17264 2000  10.10.10.254
3221356546 65540        conf  sendrecv g729    17148 2000  10.10.10.254

Total number of active session(s) 1, and connection(s) 3

sess_id    conn_id    st ev orc_ts     orca_ts    crc_ts     stmt_ts    spmt_ts    dis_ind_ts preq_ts    pc_ts      qosl_ts    qost_r_ts  qosp_ts    qost_s_ts 

3221356546 65542      -  -  61980160   61980200   0          61980212   0          0          0          0          0          0          0          0         
3221356546 65541      -  -  61978476   61978492   0          61978504   0          0          0          0          0          0          0          0         
3221356546 65540      -  -  61978388   61978440   0          61978452   0          0          0          0          0          0          0          0         

Total number of active session(s) 1, and connection(s) 3

          
bridge-info(bid, cid) - Normal bridge information(Bridge id, Calleg id)
mmbridge-info(bid, cid) - Mixed mode bridge information(Bridge id, Calleg id)

sess_id    conn_id    call-id    codec   pkt-period dtmf_method    type        bridge-info(bid, cid)   mmbridge-info(bid, cid) srtp_cryptosuite          dscp      

3221356546 -          1249       N/A     N/A        none              confmsp   All RTPSPI Callegs      All MM-MSP Callegs      N/A                       N/A       

3221356546 65542      1255       g729    20         none              rtpspi    (390,1249)               N/A                     N/A                       0         

3221356546 65541      1252       g729    20         rfc2833_pthru     rtpspi    (388,1249)               N/A                     N/A                       0         

3221356546 65540      1248       g729    20         rfc2833_pthru     rtpspi    (386,1249)               N/A                     N/A                       0         


Total number of active session(s) 1, connection(s) 3, and callegs 4

SCCP Application Service(s) Statistics Summary:
Total Conferencing Sessions: 1, Connections: 3
Total Transcoding Sessions: 0, Connections: 0
Total MTP Sessions: 0, Connections: 0
Total ALG-Phone Sessions: 0, Connections: 0
Total BRI-Phone Sessions: 0, Connections: 0
Total SCCP Sessions: 1, Connections: 3


sess_id    conn_id    rsvp_id    dir  local ip       :port  remote ip      :port 


 Total active sessions 1, connections 3, rsvp sessions 0
Statistic                  Count
-------------------------  -----------
Send queue enqueue error   0
Socket send error          1
Msgs discarded upon error  0
```

Есть комментарии в выводе
```text
router#sh dspfarm all
Dspfarm Profile Configuration

 Profile ID = 1, Service = CONFERENCING, Resource ID = 1  
 Profile Description :  
 Profile Service Mode : Non Secure 
 Profile Admin State : UP 
 Profile Operation State : ACTIVE 
 Application : SCCP   Status : ASSOCIATED 
 Resource Provider : FLEX_DSPRM   Status : UP 
 Number of Resource Configured : 2 
 Number of Resource Available : 2         # как увеличить это число?
 Codec Configuration 
 Codec : g711ulaw, Maximum Packetization Period : 30 , Transcoder: Not Required 
 Codec : g711alaw, Maximum Packetization Period : 30 , Transcoder: Not Required 
 Codec : g729ar8, Maximum Packetization Period : 60 , Transcoder: Not Required 
 Codec : g729abr8, Maximum Packetization Period : 60 , Transcoder: Not Required 
 Codec : g729r8, Maximum Packetization Period : 60 , Transcoder: Not Required 
 Codec : g729br8, Maximum Packetization Period : 60 , Transcoder: Not Required


SLOT DSP VERSION  STATUS CHNL USE   TYPE    RSC_ID BRIDGE_ID PKTS_TXED PKTS_RXED

0    1   24.3.2   UP     1    USED  conf    1      0x182       6165      6174     # используется 01 канал (см. ниже)
0    1   24.3.2   UP     1    USED  conf    1      0x184       6165      6172     
0    1   24.3.2   UP     1    USED  conf    1      0x186       6086      6007     
0    1   24.3.2   UP     N/A  FREE  conf   1      -         -         -        

Total number of DSPFARM DSP channel(s) 2





router#sh voice dsp voice 
 
DSP  DSP                DSPWARE CURR  BOOT                         PAK     TX/RX
TYPE NUM CH CODEC       VERSION STATE STATE   RST AI VOICEPORT TS ABORT  PACK COUNT
==== === == ======== ========== ===== ======= === == ========= == ===== ============
edsp 0001 01 g729r8    0.1 IDLE  50/0/1.1     
edsp 0002 02 g729r8 p  0.1 IDLE  50/0/1.2     
[skip]  
edsp 0010 01 g729r8    0.1 busy  50/0/40.1 # телефон с которого организуется конференция   
[skip]
edsp 0030 01 g729r8    0.1 busy  50/0/100.1  # 
edsp 0031 02 g729r8    0.1 busy  50/0/100.2  # используются 3 линии Ad-Hoc
edsp 0032 01 g729r8    0.1 busy  50/0/101.1  # 
edsp 0033 02 g729r8 p  0.1 IDLE  50/0/101.2   
edsp 0034 01 g729r8 p  0.1 IDLE  50/0/102.1   
edsp 0035 02 g729r8 p  0.1 IDLE  50/0/102.2   
[skip]

 
----------------------------FLEX VOICE CARD 0 ------------------------------
                           *DSP VOICE CHANNELS*

CURR STATE : (busy)inuse (b-out)busy out (bpend)busyout pending 
LEGEND     : (bad)bad    (shut)shutdown  (dpend)download pending

DSP   DSP                 DSPWARE CURR  BOOT                         PAK   TX/RX
TYPE  NUM CH CODEC        VERSION STATE STATE   RST AI VOICEPORT TS ABRT PACK COUNT
===== === == ========= ========== ===== ======= === == ========= == ==== ============
# в списке отсутствует канал 01
C5510 001 02 None          24.3.2 idle  idle      0  0                 0          0/0 
C5510 001 03 None          24.3.2 idle  idle      0  0                 0          0/0
C5510 001 04 None          24.3.2 idle  idle      0  0                 0          0/0
C5510 001 05 None          24.3.2 idle  idle      0  0                 0          0/0
C5510 001 06 None          24.3.2 idle  idle      0  0                 0          0/0
C5510 001 07 None          24.3.2 idle  idle      0  0                 0          0/0
C5510 001 08 None          24.3.2 idle  idle      0  0                 0          0/0
C5510 001 09 None          24.3.2 idle  idle      0  0                 0          0/0
C5510 001 10 None          24.3.2 idle  idle      0  0                 0          0/0
C5510 001 11 None          24.3.2 idle  idle      0  0                 0          0/0
C5510 001 12 None          24.3.2 idle  idle      0  0                 0          0/0
C5510 001 13 None          24.3.2 idle  idle      0  0                 0          0/0
C5510 001 14 None          24.3.2 idle  idle      0  0                 0          0/0
C5510 001 15 None          24.3.2 idle  idle      0  0                 0          0/0
C5510 001 16 None          24.3.2 idle  idle      0  0                 0          0/0
------------------------END OF FLEX VOICE CARD 0 ----------------------------
```

## Meetme конференция

CUCME позволяет реализовать что-то наподобие сервера конференций.
Функционал называется Meetme, когда можно позвонить на определенный номер и попасть в конференцию, в отличии от Ad-hoc конференций,
когда до каждого участника вручную необходимо дозвониться.

Настройки маршрутизатора остаются прежними. Добавляется только ephone-dn:
```text
ephone-dn  111  dual-line
 number 2222
 conference meetme
 preference 2
 no huntstop
!
!
ephone-dn  112  dual-line
 number 2222
 conference meetme
 preference 3
 no huntstop
!
!
ephone-dn  113  dual-line
 number 2222
 conference meetme
 preference 4
 no huntstop
!
!
ephone-dn  114  dual-line
 number 2222
 conference meetme
 preference 5
 no huntstop
!
!
ephone-dn  115  dual-line
 number 2222
 conference meetme
 preference 6
 no huntstop
!
!
ephone-dn  116  dual-line
 number 2222
 conference meetme
 preference 7
 no huntstop
!
!
ephone-dn  117  dual-line
 number 2222
 conference meetme
 preference 8
 no huntstop
!
!
ephone-dn  118  dual-line
 number 2222
 conference meetme
 preference 9
 no huntstop
!
!
ephone-dn  119  dual-line
 number 2222
 conference meetme
 preference 10
 no huntstop
!
!
ephone-dn  120  dual-line
 number 2222
 conference meetme
 preference 10
 no huntstop
!
!
ephone-dn  121  dual-line
 number 2222
 conference meetme
 preference 10
 no huntstop
!
!
ephone-dn  122  dual-line
 number 2222
 conference meetme
 preference 10
 no huntstop
!
!
ephone-dn  123  dual-line
 number 2222
 conference meetme
 preference 10
 no huntstop
!
!
ephone-dn  124  dual-line
 number 2222
 conference meetme
 preference 10
```
Количество dn определяет число участников, как и в Ad-hoc конференциях. Понятно, используются ресурсы DSP.
Конференция создается только при наличии строки conference meetme хотя бы в одной DN.
И только с телефона или ip-коммуникатора Cisco имеющего программные клавиши.
Их необходимо создать и применить к телефону, который будет создавать конференцию. В примере ниже такое проделано для ip-коммуникатора:
```text
ephone-template  1
 softkeys hold  Newcall Resume Select Join
 softkeys idle  Cfwdall ConfList Dnd Gpickup HLog Join Login Newcall Pickup Redial RmLstC
 softkeys seized  Redial Pickup Gpickup HLog Meetme Endcall
 softkeys connected  Acct ConfList Confrn Endcall Flash HLog Hold Join Park RmLstC Select Trnsfer
ephone  89
 device-security-mode none
 mac-address 0025.B311.E0D0
 ephone-template 1
 type CIPC
 button  1:10
```
Последовательность создания конференции:

* снимаем "трубку" на телефоне.
* выбираем программную клавишу Meetme (для русифицированных это Конф№). Она активна, если есть DN c conference meetme. Услышите тоновый сигнал по нажатию.
* набираем номер конференции 2222.

Все остальные могут звонить на 2222 без ограничений как на обычный телефонный номер.
Подключение и отключение участников будет сопровождаться join и leave тоновыми сигналами.
К недостатку можно отнести невозможность управления участниками конференции из-за ошибок русификации. Все меню с участниками в кракозябрах.
К ограничениям можно отнести возможность создания конференции только с аппаратов, использующих sccp сигнализацию.
Cisco IP-Phone с прошивками SIP, а так же телефонные шлюзы ATA не могут создавать конференции Meetme.
