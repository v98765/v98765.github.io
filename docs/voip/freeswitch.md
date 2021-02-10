## sbin

При установке fusionpbx в debian до запуска скрипта прописать PATH до /sbin

## ssl cert

Обновление сертификата letsencrypt
```sh
systemctl stop nginx

certbot -q renew

mkdir -p /etc/freeswitch/tls

rm /etc/freeswitch/tls/*

cat /etc/letsencrypt/live/[domain]/fullchain.pem > /etc/freeswitch/tls/all.pem
cat /etc/letsencrypt/live/[domain]/privkey.pem >> /etc/freeswitch/tls/all.pem

cp /etc/letsencrypt/live/[domain]/cert.pem /etc/freeswitch/tls
cp /etc/letsencrypt/live/[domain]/chain.pem /etc/freeswitch/tls
cp /etc/letsencrypt/live/[domain]/fullchain.pem /etc/freeswitch/tls
cp /etc/letsencrypt/live/[domain]/privkey.pem /etc/freeswitch/tls

ln -s /etc/freeswitch/tls/all.pem /etc/freeswitch/tls/agent.pem
ln -s /etc/freeswitch/tls/all.pem /etc/freeswitch/tls/tls.pem
ln -s /etc/freeswitch/tls/all.pem /etc/freeswitch/tls/wss.pem
ln -s /etc/freeswitch/tls/all.pem /etc/freeswitch/tls/dtls-srtp.pem

systemctl start nginx

systemctl restart freeswitch
```

## reboot

По расписанию в 3 ночи
```sh
/sbin/shutdown -r 03:00
```
Отмена
```sh
/sbin/shutdown -c
```

## codec

Только 711a. В vars
```text
<X-PRE-PROCESS cmd="set" data="outbound_codec_prefs=PCMA" />
<X-PRE-PROCESS cmd="set" data="global_codec_prefs=PCMA@20i,H264" />
```
greedy, чтоб было только с указанным ptime 20, и не было попыток с 30. В sip_profiles/internal
```text
<!--set to 'greedy' if you want your codec list to take precedence -->
<param name="inbound-codec-negotiation" value="greedy"/>
```

## fs_cli

Пароль в файле `autoload_configs/event_socket.conf.xml`
```text
<configuration name="event_socket.conf" description="Socket Client">
  <settings>
    <param name="listen-ip" value="127.0.0.1"/>
    <param name="listen-port" value="8021"/>
    <param name="password" value="123"/>
  </settings>
</configuration>
```
Подключение `fs_cli -p 123`
 
