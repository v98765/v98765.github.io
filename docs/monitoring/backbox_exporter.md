[Параметры в конфигурации](https://github.com/prometheus/blackbox_exporter/blob/master/CONFIGURATION.md)
[Отладка с примерами](https://www.robustperception.io/debugging-blackbox-exporter-failures)
[Пример конфига](https://github.com/prometheus/blackbox_exporter/blob/master/example.yml)

## почта

Проверка ssl клиентом, чтобы составить правила c send/expect для blackbox. 143,993 - это порты IMAP. 587 - SMTP

```sh
openssl s_client -starttls imap -crlf -connect domain:143
openssl s_client -crlf -connect domain:993
openssl s_client -starttls smtp -crlf -connect domain:587
```

Конфигурация blackbox_exporter

```yaml
  imap_starttls:
    prober: tcp
    tcp:
      preferred_ip_protocol: ipv4
      query_response:
      - starttls: true
      - expect: ^* OK (.+)$
      - send: ". capability\r"
      - expect: ^* CAPABILITY IMAP(.+)$
      - send: "logout\r"
      tls: false
    timeout: 5s
  smtp_starttls:
    prober: tcp
    tcp:
      preferred_ip_protocol: ipv4
      query_response:
      - expect: ^220 (.+) ESMTP (.+)$
      - send: "EHLO prober\r"
      - expect: ^250-STARTTLS
      - send: "STARTTLS\r"
      - expect: ^220
      - starttls: true
      - send: "EHLO prober\r"
      - expect: ^250-AUTH
      - send: "QUIT\r"
    timeout: 5s
```

## пинги

Конфигурация blackbox_exporter

```yaml
  icmp:
    icmp:
      preferred_ip_protocol: ip4
    prober: icmp
```

## http и https

Конфигурация blackbox_exporter

```yaml
  http_2xx:
    http:
      fail_if_not_ssl: false
      fail_if_ssl: true
      method: GET
      no_follow_redirects: false
      preferred_ip_protocol: ipv4
    prober: http
    timeout: 5s
  https_2xx:
    http:
      fail_if_not_ssl: true
      fail_if_ssl: false
      method: GET
      no_follow_redirects: false
      preferred_ip_protocol: ipv4
    prober: http
    timeout: 5s
```
Для самоподписанного сертификата и списком допустимых/валидных кодов ответа
```text
  https_self:
    prober: http
    timeout: 5s
    http:
      method: GET
      no_follow_redirects: false
      fail_if_ssl: false
      fail_if_not_ssl: true
      preferred_ip_protocol: "ipv4"
      valid_status_codes:
        - 200
        - 401
        - 403
      tls_config:
        insecure_skip_verify: true
```
