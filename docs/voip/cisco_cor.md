Ограничение межгорода - наиболее востребованная функция, когда применяют COR-листы на голосовых шлюзах cisco.
Необходимо четко понимать, когда они работают и в каком случае не работают в принципе.
По-умолчанию все звонки с неопределенным номером, не попавшие ни в какой входящий dial-peer, используют системный dial-peer 0 , который нельзя изменить.
Такой вызов будет игнорировать все corlist'ы на любом исходящем dial-peer'е. Потому что:

COR List on Incoming dial-peer | COR List on Outgoing dial-peer	| Result | Reason
---|---|---|---
No COR. | No COR. | Call succeeds. | COR is not in the picture.
No COR.	| COR list applied for outgoing calls. | Call succeeds.	| The incoming dial-peer, by default, has the highest COR priority when no COR is applied. Therefore, if you apply no COR for an incoming call leg to a dial-peer, then this dial-peer can make calls out of any other dial-peer, irrespective of the COR configuration on the outgoing dial-peer.
The COR list applied for incoming calls. | No COR. | Call succeeds.	| The outgoing dial-peer, by default, has the lowest priority. Since there are some COR configurations for incoming calls on the incoming/originating dial-peer, it is a super set of the outgoing call COR configurations on the outgoing/terminating dial-peer.
The COR list applied for incoming calls (super set of COR lists applied for outgoing calls on the outgoing dial-peer). | The COR list applied for outgoing calls (subset of COR lists applied for incoming calls on the incoming dial-peer.)	| Call succeeds. | The COR list for incoming calls on the incoming dial-peer is a super set of COR lists for outgoing calls on the outgoing dial-peer
The COR list applied for incoming calls (subset of COR lists applied for outgoing calls on the outgoing dial-peer).	| The COR list applied for outgoing calls (super set of COR lists applied for incoming calls on the incoming dial-peer).	| Call cannot be completed using this outgoing dial-peer. | COR lists for incoming calls on the incoming dial-peer are not a super set of COR lists for outgoing calls on the outgoing dial-peer.

Пусть имеется голосовой шлюз Сisco. Для corlist'а непринципиально применение на voip или pots dial-peer'ах.
Для начала необходимо определиться с ограничениями:

1. Ограничить "межгород", набор которого производится через 98
2. Разрешить выход в город, набор которого производится через 9
3. Разрешить выход на корпоративные номера, начинающиеся на 2 и 3.

Имеем ip-телефоны Cisco в нумерации 211x, одному из которых (2111) надо закрыть "межгород" и удаленный шлюз с номерами 311х, где только номеру 3111 необходимо открыть "межгород".
Понадобится всего два corlist'а. Будем проще и назовем их Limit и NoLimit, так как название не имеет никакого значения вообще.
```text
dial-peer cor custom
 name Limit
 name NoLimit
!
!
dial-peer cor list Limit
 member Limit
!
dial-peer cor list NoLimit
 member Limit
 member NoLimit
```
Исходя из правила, что при отсутствии входящих corlist'ов на dial-peer'ах все вызовы будут беспрепятственно осуществляться, "вешаем" исходящие corlist'ы
```text
dial-peer voice 98 voip
 corlist outgoing NoLimit
 description MejGorod
 destination-pattern 98T
 session target ipv4:10.0.0.100
 no vad
dial-peer voice 9 voip
 corlist outgoing Limit
 description Gorod
 destination-pattern 9[012345679]T
 session target ipv4:10.0.0.100
 no vad
dial-peer voice 2000 voip
 corlist outgoing Limit
 description Corporate
 destination-pattern [23]...
 session target ipv4:10.0.0.100
 no vad
```
Теперь необходимо применить входящие corlist'ы. Помним, что отсутствие листа равноценно максимальному приоритету, а несовпадение листов хотя бы одним признаком приведет к отказу в выполнении вызова.
```text
ephone-dn 10
 number 2110
!
ephone-dn 11
 number 2111
 corlist incoming Limit
!
ephone-dn 12
 number 2112
!
ephone-dn 13
 number 2113
!
ephone-dn 14
 number 2114
!
ephone-dn 15
 number 2115
!
ephone-dn 16
 number 2116
!
ephone-dn 17
 number 2117
!
ephone-dn 18
 number 2118
!
ephone-dn 19
 number 2119
!
dial-peer voice 3110 voip
 corlist incoming Limit
 description Remote Gateway
 huntstop
 destination-pattern 311.
 session target ipv4:10.100.0.100
 no vad
!
dial-peer voice 3111 voip
 description Happy User on Remote Gateway
 huntstop
 destination-pattern 3111
 session target ipv4:10.100.0.100
 no vad
```
На ephone-dn и dial-peer'ах выше отсутствуют любые "corlist outgoing", а это значит, что 2111 и 311. 
могут беспрепятственно звонить на все номера (211х и 311х) на данном шлюзе. Потому что так задумано и описано в таблице.
