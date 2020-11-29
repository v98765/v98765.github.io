Проверка номера телефона

```sh
curl https://api.bmi.io/bdpn/v2/check/full?number=79110000000
{"code":200,"error":false,"Transfered":false,"OperatorID":"14","OperatorCODE":"mMTS","OperatorMCC":"250","OperatorMNC":"01","OperatorNAME":"\u041f\u0410\u041e \"\u041c\u043e\u0431\u0438\u043b\u044c\u043d\u044b\u0435 \u0422\u0435\u043b\u0435\u0421\u0438\u0441\u0442\u0435\u043c\u044b\"","OperatorAreaID":"1839","OperatorAreaNAME":"\u0433.\u0421\u0430\u043d\u043a\u0442-\u041f\u0435\u0442\u0435\u0440\u0431\u0443\u0440\u0433 \u0438 \u041b\u0435\u043d\u0438\u043d\u0433\u0440\u0430\u0434\u0441\u043a\u0430\u044f \u043e\u0431\u043b\u0430\u0441\u0442\u044c","OperatorAreaTIMEZONE":"3","OperatorAreaGEOLAT":"59.93913130","OperatorAreaGEOLON":"30.31590040","PhoneFormat":"79110000000","PhoneTemplate":"+7 911 000-00-00","UTC":"3"}
```
Или бот в телеге @bmi_np_bot

[num.voxlink.ru](http://num.voxlink.ru/)

```sh
curl http://num.voxlink.ru/get/?num=79000000000
{"code": "900", "num": "0000000", "full_num": "9000000000", "operator": "МегаФон", "old_operator": "Теле2", "region": "Краснодарский край"}
```
