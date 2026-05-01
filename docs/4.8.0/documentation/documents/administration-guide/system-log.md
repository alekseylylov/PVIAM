# Системный журнал

> Возможна интеграция с системой Platform V Monitor Журналирование (LOGA), при помощи встраивания Fluent Bit Sidecar
> в pod IAM Proxy (или с установленным IAM Proxy). Настройки интеграции описаны в разделе
> [Установка](../installation-guide/installation.md)

## Прокси сервер (IAM Proxy)

Состояние сервиса:

```
systemctl status iamproxy
```

### Логи запросов к IAM Proxy

`/opt/iamproxy/logs/access.log`

```
[22/Nov/2024:13:47:48 +0300]  -> 10.x.x.98:55231 -> 10.x.x.153:443 -> - - - "GET /MegaSystem/ HTTP/1.1" 302 138 - - "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/129.x.x.0 Safari/537.36 MyCompBrowser/21.x.x.0" "-" "-" "https://auth-psi.mycompany.ru/auth/realms/users/protocol/openid-connect/auth?state=35320e83a9991111538982abe266a143&scope=openid%20basic%20extended%20access&response_type=code&redirect_uri=https%3A%2F%2Ftvlos-sss00003.mycompany.ru%2Fopenid-connect-auth%2Fredirect_uri&nonce=a03bcd97d9999f57e0000728be723529&client_id=CI00890888" 4d458282c47504fdfd30dc5d8e022a44 1426 2899 "0cb8d7e0f80715506700004e3e91b06c"
[22/Nov/2024:13:47:49 +0300]  -> 10.x.x.105:42860 -> 10.x.x.153:10080 -> - -  "GET /metrics HTTP/1.1" 404 146 - - "-" "Prometheus/2.33.4" "-" "-" "-" 190480fca9e64a80e0465825bc2b3399 1425 11 ""
[22/Nov/2024:13:48:00 +0300]  -> 10.x.x.98:55231 -> 10.x.x.153:443 -> - - - "GET /openid-connect-auth/redirect_uri?state=35320e83a9991111538982abe266a143&session_state=f55fda9e-279a-0000-a346-1c1bf58d6c60&iss=https%3A%2F%2Fauth-psi.mycompany.ru%2Fauth%2Frealms%2Fusers&code=19b70019-1742-43cc-0000-5c4a336e1b7e.f55fda9e-279a-4fbb-0000-1c1bf58d6c60.13be11d5-2482-401c-afe2-9635b882bd47 HTTP/1.1" 302 138 - - "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/129.x.x.0 Safari/537.36 MyCompBrowser/21.x.x.0" "-" "-" "/MegaSystem/" e98000c7b9e7dd710d78b7817745bfbf 1426 2899 "0cb8d7e0f80715506700004e3e91b06c"
[22/Nov/2024:13:48:00 +0300]  -> 10.x.x.98:55231 -> 10.x.x.153:443 -> 30.x.x.89:8443 - ivanov-ii "GET /MegaSystem/ HTTP/1.1" 302 0 0.004 0.054 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/129.x.x.0 Safari/537.36 MyCompBrowser/21.x.x.0" "-" "-" "/MegaSystem/sc/workplace" 7781cb7d77c42f67f63f7111292b8b6a 1426 2899 "f55fda9e-279a-0000-a346-1c1bf58d6c60"
[22/Nov/2024:13:48:01 +0300]  -> 10.x.x.98:55231 -> 10.x.x.153:443 -> 30.x.x.33:8443 - ivanov-ii "GET /MegaSystem/sc/workplace HTTP/1.1" 200 46479 0.003 0.987 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/129.x.x.0 Safari/537.36 MyCompBrowser/21.x.x.0" "-" "-" "-" f7690b953434410c772359c8615624c5 1426 2899 "f55fda9e-279a-0000-a346-1c1bf58d6c60"
[22/Nov/2024:13:48:01 +0300]  -> 10.x.x.98:55231 -> 10.x.x.153:443 -> 30.x.x.89:8443 - ivanov-ii "GET /MegaSystem/classpath/js/jquery-1.7.2.min.js HTTP/1.1" 200 94840 0.003 0.005 "https://tvlos-sss00003.mycompany.ru/MegaSystem/sc/workplace" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/129.x.x.0 Safari/537.36 MyCompBrowser/21.x.x.0" "-" "-" "-" 21aca7f3e68d1376d28bbdd8a78715bc 1426 2899 "f55fda9e-279a-0000-a346-1c1bf58d6c60"
```

Формат access-log сервиса IAM Proxy (актуальный можно посмотреть в файле
`/opt/iamproxy/conf/common/access-log-format.http.conf`):

```
  log_format main_pp  '[$time_local] $forwarded_for -> $remote_addr:$remote_port -> $server_addr:$server_port -> $upstream_addr - '
  '$session_user '
  '"$request" $status $body_bytes_sent $upstream_connect_time $upstream_response_time '
  '"$http_referer" "$http_user_agent" "$http_x_forwarded_for" "$ssl_client_s_dn" "$sent_http_location" '
  '$request_id/$http_x_request_id $pid $connection "$session_id"';
```

Где переменные означают:

- `$time_local` - время записи в лог в текущей временной зоне (фактически время завершения отправки ответа);
- `$forwarded_for` - для запроса адрес из заголовка PROXY_PROTOCOL или HTTP заголовок `x-forwarded-for`;
- `$remote_addr` - IP-адрес сетевого клиента;
- `$remote_port` - IP-порт сетевого клиента;
- `$server_addr` - IP-адрес на сервера для текущего соединения;
- `$server_port` - IP-порт на сервера для текущего соединения;
- `$upstream_addr` - IP адрес+порт куда проксировали (бэкенд);
- `$session_user` - login пользователя;
- `$request` - первоначальная строка запроса целиком;
- `$status` - HTTP код ответа;
- `$body_bytes_sent` - число байт, переданное клиенту, без учета заголовка ответа;
- `$upstream_connect_time` - затраченное на установление соединения с сервером группы (в секундах с точностью до
  миллисекунд, в случае SSL, включает в себя время, потраченное на handshake);
- `$upstream_response_time` - затраченное на получение ответа от сервера группы; время хранится в секундах с точностью
  до миллисекунд;
- `$http_referer` - HTTP заголовок запроса  `referer`;
- `$http_user_agent` - HTTP заголовок запроса `user-agent`;
- `$http_x_forwarded_for` - HTTP заголовок запроса `x-forwarded-for`;
- `$ssl_client_s_dn` - строка “subject DN” клиентского сертификата для установленного SSL-соединения согласно RFC 2253 (
  1.11.6);
- `$sent_http_location` - HTTP заголовок ответа `location`;
- `$request_id` - уникальный идентификатор запроса, сформированный из 16 случайных байт, в шестнадцатеричном виде;
- `$http_x_request_id` - HTTP заголовок запроса `x-request-id`;
- `$pid` - номер (PID) рабочего процесса;
- `$connection` - порядковый номер соединения;
- `$session_id` - ID сессии пользователя;

> Для переменных `$upstream_*` могут быть указаны данные по нескольким соединениям. Если при обработке запроса были
> сделаны обращения к нескольким серверам, то их данные разделяются запятой, например, “192.x.x.1:80, 192.x.x.2:80”.

### Логи запросов к IAM Proxy отправляемые по syslog

При отправке syslog на удаленный сервер можно использовать параметр `proxy_to_syslog_format`, который задает разные форматы событий.

Формат access-log сервиса IAM Proxy (актуальный можно посмотреть в файле
`/opt/iamproxy/conf/common/access-log-format.http.conf.j2`)

Syslog-формат для совместимости:

```
  log_format main_syslog 'BR:"$request" TS:$time_iso8601 M:$request_method URL:"$request_uri" CA:"$remote_addr[$forwarded_for]" F:$request_time ST:$status B:$body_bytes_sent CTRESP:"$sent_http_content_type" SI:$session_id SU:$session_user LA:$server_addr JN:$jctroot S:$upstream_addr T:$request_id CSSL:"$ssl_client_s_dn" PID:$pid C:$connection';

```

Syslog-формат для структурированного логирования событий в формате JSON:

```
   log_format log_json escape=json '{'
  '"time":"$time_iso8601","timestamp":$msec'
  ',"hostname":"$hostname"'
  ',"remote_addr":"$remote_addr","remote_port":"$remote_port","remote_user":"$remote_user"'
  ',"server_addr":"$server_addr","server_port":"$server_port","server_protocol":"$server_protocol"'
  ',"upstream_addr":"$upstream_addr","proxy_host":"$proxy_host"'
  ',"ssl":"$ssl_protocol $ssl_cipher","ssl_client_s_dn":"$ssl_client_s_dn"'
  ',"request_method":"$request_method","request_uri":"$request_uri","scheme":"$scheme","request":"$request","status":"$status"'
  ',"request_filename": "$request_filename"'
  ',"executionTime":"$request_time"'
  ',"ipAddress":"$real_ip_addr"'
  ',"userLogin":"$session_user","sessionId":"$session_id","userName":"$session_username"'
  ',"http_host":"$http_host"'
  ',"http_x_forwarded_for":"$http_x_forwarded_for"'
  ',"httpReferer":"$http_referer","http_user_agent":"$http_user_agent"'
  ',"http_Content_type":"$http_Content_type","sent_http_Content_type":"$sent_http_Content_type"'
  ',"request_length":"$request_length","bytes":"$bytes_sent","body_bytes_sent":"$body_bytes_sent"'
  ',"upstream_connect_time":"$upstream_connect_time","upstream_response_time":"$upstream_response_time","upstream_response_length":"$upstream_response_length","upstream_status":"$upstream_status"'
  ',"tcp_info":"[rtt: $tcpinfo_rtt, rttvar: $tcpinfo_rttvar, cwnd: $tcpinfo_snd_cwnd, space: $tcpinfo_rcv_space]"'
  ',"request_id":"$request_id","rqUid":"$rquid","response_rqUid":"$sent_http_x_request_id"'
  ',"processId":"$pid","connection_id":"$connection"'
  ',"sent_http_location":"$sent_http_location"'
  ',"message":"$status $request_method $request_uri"'
'}'
```
Syslog-формат для структурированного и компактного логирования событий:
```
  log_format log_json_small escape=json
'{'
  '"time_local": "[$time_local:$msec]",'
  '"connection": "$connection",'
  '"host": "$host",'
  '"server_name": "$server_name",'
  '"hostname": "$hostname",'
  '"server_protocol": "$server_protocol",'
  '"request_method": "$request_method",'
  '"remote_addr": "$remote_addr",'
  '"status": "$status",'
  '"url": "$uri",'
  '"request_uri": "$request_uri",'
  '"request_filename": "$request_filename",'
  '"httpReferer": "$http_referer",'
  '"bytes": "$bytes_sent",'
  '"request_length": "$request_length",'
  '"body_bytes_sent": "$body_bytes_sent",'
  '"request_time": "[$request_time:$msec]",'
  '"tcp_info": "[rtt: $tcpinfo_rtt, rttvar: $tcpinfo_rttvar, cwnd: $tcpinfo_snd_cwnd, space: $tcpinfo_rcv_space]"'
'}';
```

Syslog-формат для аудирования:
```
log_format log_json_audit escape=json
'{'
{% if not (audit2_options is defined and audit2_options.http_api is defined and audit2_options.audit_type == 'audit2-gt') %}
  '"timestamp": $msec,'
  '"id": "$request_id",'
  '"requestId": "$rquid",'
{% endif %}
  '"createdAt": $msec_no_decimal,'
  '"metamodelVersion": "${project.version}",'  '"module": "{{ stend_abbr|default('auth',True) }}-iamproxy",'
  '"name": "IAMProxyHTTPRequest",'
  '"userNode": "$real_ip_addr",'
  '"userLogin": "$session_user",'
  '"userName": "$session_username",'
  '"session": "$session_id",'
  '"tags": ["product:IAM","component:AUTH","name:iamproxy","sourceSystem:{{ ( stend_display_name|default('Platform V') )|escape }}"],'
  '"params": ['
    '{'
      '"name": "authn.userName",'
      '"value": "$session_user"'
    '},'
    '{'
      '"name": "authn.userSession",'
      '"value": "$session_id"'
    '},'
    '{'
      '"name": "server.addr",'
      '"value": "$server_addr [$pid:$connection]"'
    '},'
    '{'
      '"name": "jct.root",'
      '"value": "$jctroot"'
    '},'
    '{'
      '"name": "jct.server",'
      '"value": "$upstream_addr"'
    '},'
    '{'
      '"name": "req.id",'
      '"value": "$request_id"'
    '},'
    '{'
      '"name": "req.rqUid",'
      '"value": "$rquid"'
    '},'
    '{'
      '"name": "req.head",'
      '"value": "$request"'
    '},'
    '{'
      '"name": "req.client_ip",'
      '"value": "$remote_addr[$forwarded_for]"'
    '},'
    '{'
      '"name": "req.date",'
      '"value": "$time_iso8601"'
    '},'
    '{'
      '"name": "req.time",'
      '"value": "$request_time"'
    '},'
    '{'
      '"name": "resp.bytes",'
      '"value": "$body_bytes_sent"'
    '},'
    '{'
      '"name": "resp.status",'
      '"value": "$status"'
    '},'
    '{'
      '"name": "resp.content_type",'
      '"value": "$sent_http_content_type"'
    '}'
  ']'
'}';
```

Syslog-формат для глубокого анализа, дебаггинга и аудита работы IAM Proxy:
```
log_format log_req_resp escape=none '{ '
'"time":"$time_iso8601" , "timestamp":"$msec" '
', "pid":"$pid" , "connection_id":"$connection" , "request_id":"$request_id" '
', "remote_addr":"$remote_addr" , "remote_user":"$remote_user" '
', "ssl":"$ssl_protocol $ssl_cipher" , "ssl_client_s_dn":"$ssl_client_s_dn" '
', "request":"$request" , "status":"$status" , "body_bytes_sent":"$body_bytes_sent" '
', "http_referer":"$http_referer" , "http_user_agent":"$http_user_agent" '
', "request_time":"$request_time" , "req_headers":$req_header , "req_body":$req_body '
', "upstream_response_time":"$upstream_response_time" , "upstream_response_length":"$upstream_response_length" , "upstream_status":"$upstream_status" '
', "resp_headers":$resp_header , "resp_body":$resp_body '
'}';
```

Где переменные означают:

- `$time_local` - время завершения ответа в локальной временной зоне (`[дд/ммм/гггг:чч:мм:сс +ЧЧММ]`);
- `$time_iso8601` - время в формате ISO 8601 (`YYYY-MM-DDTHH:MM:SS+TZ`);
- `$msec` -  текущее время в секундах с десятичной дробью;
- `$msec_no_decimal` - текущее время в секундах без десятичной дроби (целое число);
- `$hostname` - имя хоста сервера;
- `$remote_addr` - IP-адрес клиента;
- `$remote_port` - порт клиента;
- `$server_addr` - IP-адрес сервера;
- `$server_port` - порт сервера;
- `$server_protocol` - протокол сервера;
- `$upstream_addr` - адрес и порт бэкенд;
- `$proxy_host` - заголовок `Host`, отправленный прокси;
- `$ssl_protocol` - используемый SSL-протокол;
- `$ssl_cipher` - используемый SSL-шифр;
- `$ssl_client_s_dn`- "Subject DN" клиентского SSL-сертификата (RFC 2253);
- `$request_method` - метод HTTP-запроса (GET, POST и т.д.);
- `$request_uri` - полный URI запроса (включая параметры);
- `$scheme` - схема запроса (`http` или `https`);
- `$request` - полная строка запроса (`МЕТОД URI ПРОТОКОЛ`);
- `$status` - HTTP-код статуса ответа (например: `200`, `404`);
- `$body_bytes_sent` - количество байт, отправленных клиенту (без заголовков);
- `$bytes_sent` - общее количество байт, отправленных клиенту (включая заголовки);
- `$request_length` - длина запроса от клиента;
- `$request_time` - общее время обработки запроса (с точностью до миллисекунд);
- `$real_ip_addr` - реальный IP-адрес клиента (из заголовка `X-Forwarded-For` или PROXY-протокола);
- `$session_user` - логин пользователя;
- `$session_id` - уникальный идентификатор сессии;
- `$session_username` - имя пользователя (если отличается от логина);
- `$http_host` - заголовок `Host` из запроса;
- `$http_x_forwarded_for` - заголовок `X-Forwarded-For`;
- `$http_referer` - заголовок `Referer`;
- `$http_user_agent` - заголовок `User-Agent`;
- `$http_Content_type` - заголовок `Content-Type` из запроса;
- `$sent_http_Content_type` - заголовок `Content-Type` в ответе;
- `$sent_http_location` - заголовок `Location` в ответе (для редиректов);
- `$jctroot` - корневой контекст (jct-root);
- `$pid` - идентификатор рабочего процесса (PID);
- `$connection`- номер соединения;
- `$request_id` - уникальный идентификатор запроса (16 случайных байт в HEX);
- `$rquid` - идентификатор запроса из заголовка `X-Request-ID`;
- `$request_filename`- путь к файлу на сервере, соответствующий запросу;
- `$tcp_info` - информация о TCP-соединении: RTT, cwnd, rcv_space;
- `$upstream_connect_time` - время установки соединения с бэкенд (в секундах);
- `$upstream_response_time` - время получения ответа от бэкенд (в секундах);
- `$upstream_response_length` - размер ответа от бэкенд;
- `$upstream_status` - статус ответа от бэкенд (если есть ошибки);
- `$createdAt` - время события в миллисекундах (без десятичной части);
- `$metamodelVersion` - версия метамодели;
- `$module` - имя модуля;
- `$name` - тип события;
- `$userNode` - узел пользователя (IP-адрес);
- `$tags` - метки для классификации события;
- `$params` - список параметров с именами и значениями (описание ниже);
- `$req_header` - заголовки запроса (в формате JSON);
- `$req_body` - тело запроса (в формате JSON);
- `$resp_header` - заголовки ответа (в формате JSON);
- `$resp_body` - тело ответа (в формате JSON).

### Логи ошибок и других диагностических сообщений (детализация зависит от уровня логирования в iamproxy.conf)

`/opt/iamproxy/logs/error.log`

Сообщения имеют формат:

```
YYYY/MM/DD mm:hh:ss [level] NNNNN#TTTTT: *CCCC FUNCTION MESSAGE
```

где:

- `YYYY/MM/DD mm:hh:ss` время в локальной временной зоне;
- `level` уровень логирования для сообщения;
- `NNNNN#TTTTT` id worker-процесса и потока через `#` (обычно это два одинаковых id);
- `*CCCC` id tcp соединения, по которому произошло событие (может отсутствовать);
- `FUNCTION` наименование функции, при выполнении которой произошло событие (может отсутствовать);
- `MESSAGE` описание события;

```
2020/04/09 17:15:43 [warn] 2090#2090: the "user" directive makes sense only if the master process runs with super-user privileges, ignored in /opt/iamproxy/conf/iamproxy.conf:2
2020/04/09 17:15:47 [warn] 2322#2322: the "user" directive makes sense only if the master process runs with super-user privileges, ignored in /opt/iamproxy/conf/iamproxy.conf.rds:2
2020/04/09 17:15:47 [warn] 2356#2356: the "user" directive makes sense only if the master process runs with super-user privileges, ignored in /opt/iamproxy/conf/iamproxy.conf.rds:2
2020/04/09 17:36:03 [error] 2385#2385: send() failed (111: Connection refused)
2020/04/09 17:36:04 [crit] 2385#2385: *7 SSL_do_handshake() failed (SSL: error:140944E7:SSL routines:ssl3_read_bytes:reason(1255):SSL alert number 255) while SSL handshaking, client: 10.x.x.178, server: 0.0.0.0:10444
2020/04/09 17:36:04 [crit] 2385#2385: *9 SSL_do_handshake() failed (SSL: error:140944E7:SSL routines:ssl3_read_bytes:reason(1255):SSL alert number 255) while SSL handshaking, client: 10.x.x.178, server: 0.0.0.0:10444
2020/04/09 17:36:04 [crit] 2384#2384: *8 SSL_do_handshake() failed (SSL: error:140944E7:SSL routines:ssl3_read_bytes:reason(1255):SSL alert number 255) while SSL handshaking, client: 10.x.x.178, server: 0.0.0.0:10444
2020/04/09 17:36:04 [crit] 2385#2385: *10 SSL_do_handshake() failed (SSL: error:140944E7:SSL routines:ssl3_read_bytes:reason(1255):SSL alert number 255) while SSL handshaking, client: 10.x.x.178, server: 0.0.0.0:10444
2020/04/09 17:36:04 [crit] 2384#2384: *11 SSL_do_handshake() failed (SSL: error:140944E7:SSL routines:ssl3_read_bytes:reason(1255):SSL alert number 255) while SSL handshaking, client: 10.x.x.178, server: 0.0.0.0:10444
2020/04/09 17:36:04 [crit] 2384#2384: *19 SSL_do_handshake() failed (SSL: error:140944E7:SSL routines:ssl3_read_bytes:reason(1255):SSL alert number 255) while SSL handshaking, client: 10.x.x.178, server: 0.0.0.0:10444
2020/04/09 17:36:04 [crit] 2385#2385: *18 SSL_do_handshake() failed (SSL: error:140944E7:SSL routines:ssl3_read_bytes:reason(1255):SSL alert number 255) while SSL handshaking, client: 10.x.x.178, server: 0.0.0.0:10444
2020/04/09 17:36:04 [crit] 2385#2385: *20 SSL_do_handshake() failed (SSL: error:140944E7:SSL routines:ssl3_read_bytes:reason(1255):SSL alert number 255) while SSL handshaking, client: 10.x.x.178, server: 0.0.0.0:10444
2020/04/09 17:36:04 [crit] 2385#2385: *22 SSL_do_handshake() failed (SSL: error:140944E7:SSL routines:ssl3_read_bytes:reason(1255):SSL alert number 255) while SSL handshaking, client: 10.x.x.178, server: 0.0.0.0:10444
2020/04/09 17:36:05 [error] 2385#2385: send() failed (111: Connection refused)
```

Для изменения уровня логирования на самый подробный необходимо в `iamproxy.conf` изменить уровень во всех опциях
`error_log` на `debug`.

Пример повышения уровня до `debug`:

```
error_log /opt/iamproxy/logs/error.log debug;
```

> Журнал `/opt/iamproxy/logs/error.log` содержит сообщения как об ошибках, так и о различных других событиях
> (например о деталях вызова healthcheck).

### Клиент по конфигурированию маршрутов (RDS Client)

Состояние сервиса:

```
systemctl status rds-client
```

логи работы приложения
`/opt/iamproxy/rds-client/logs/log-2020-04-10.log`

```
16:35:15.096  INFO | [pool-3-thread-1]: Trying to send GET request to https://mycompany-auth-svc-idp1-dev2.mycompany.ru:7443/rds-for-proxy/active-conf-json
16:35:15.105  INFO | [pool-3-thread-1]: Response received.
16:35:20.097  INFO | [pool-3-thread-1]: Trying to send GET request to https://mycompany-auth-svc-idp1-dev2.mycompany.ru:7443/rds-for-proxy/active-conf-json
16:35:20.116  INFO | [pool-3-thread-1]: Response received.
16:35:25.097  INFO | [pool-3-thread-1]: Trying to send GET request to https://mycompany-auth-svc-idp1-dev2.mycompany.ru:7443/rds-for-proxy/active-conf-json
16:35:25.104  INFO | [pool-3-thread-1]: Response received.
16:35:29.823  INFO | [main]: RDS-client is started
16:35:30.287  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/cache/last-conf.json
16:35:30.375  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/cache/context.log
16:35:30.498  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/results/index.html
16:35:30.537  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/cache/context.log
16:35:30.546  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/results/sps-st.upstream.conf
16:35:30.550  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/cache/context.log
16:35:30.552  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/results/spsSt.upstream.conf
16:35:30.556  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/cache/context.log
16:35:30.558  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/results/prb-st.upstream.conf
16:35:30.562  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/cache/context.log
16:35:30.566  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/results/prb-st2.upstream.conf
16:35:30.567  INFO | [pool-1-thread-1]: Name for resulting file is empty. Skip. [/opt/iamproxy/rds-client/templates/jct.name-upstream.upstream.jinja2 , jct]
16:35:30.571  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/cache/context.log
16:35:30.574  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/results/aud-st.upstream.conf
16:35:30.576  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/cache/context.log
16:35:30.578  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/results/logger-st.upstream.conf
16:35:30.581  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/cache/context.log
16:35:30.583  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/results/rds-for-proxy.upstream.conf
16:35:30.586  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/cache/context.log
16:35:30.589  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/results/jct-snoop.upstream.conf
16:35:30.591  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/cache/context.log
16:35:30.595  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/results/snoop.upstream.conf
16:35:30.597  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/cache/context.log
16:35:30.600  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/results/stub.upstream.conf
16:35:30.602  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/cache/context.log
16:35:30.604  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/results/test-sutb2.upstream.conf
16:35:30.607  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/cache/context.log
16:35:30.613  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/results/sps-st.server.conf
16:35:30.615  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/cache/context.log
16:35:30.618  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/results/spsSt.server.conf
16:35:30.626  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/cache/context.log
16:35:30.628  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/results/prb-st.server.conf
16:35:30.631  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/cache/context.log
16:35:30.634  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/results/prb-st2.server.conf
16:35:30.641  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/cache/context.log
16:35:30.643  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/results/logger-st.server.conf
16:35:30.648  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/cache/context.log
16:35:30.651  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/results/aud-st.server.conf
16:35:30.653  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/cache/context.log
16:35:30.656  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/results/logger-st.server.conf
16:35:30.659  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/cache/context.log
16:35:30.661  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/results/rds-for-proxy.server.conf
16:35:30.664  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/cache/context.log
16:35:30.667  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/results/jct-snoop.server.conf
16:35:30.669  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/cache/context.log
16:35:30.672  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/results/snoop.server.conf
16:35:30.674  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/cache/context.log
16:35:30.678  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/results/stub.server.conf
16:35:30.680  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/cache/context.log
16:35:30.682  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/results/test-sutb2.server.conf
16:35:30.685  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/cache/context.log
16:35:30.687  INFO | [pool-1-thread-1]: Write to file: /opt/iamproxy/rds-client/cache/reload-nginx.sh
16:35:30.792  INFO | [pool-1-thread-1]:  --- SCRIPT OUTPUT BEGIN ---
16:35:30.793  INFO | [pool-1-thread-1]: nginx: [warn] the "user" directive makes sense only if the master process runs with super-user privileges, ignored in /opt/iamproxy/conf/iamproxy.conf.rds:2
16:35:30.793  INFO | [pool-1-thread-1]: OK - Тест новой конфигурации пройден
16:35:30.793  INFO | [pool-1-thread-1]: /opt/iamproxy/rds-client/results/test-sutb2.server.conf -> /opt/iamproxy/conf/jct/test-sutb2.server.conf
16:35:30.793  INFO | [pool-1-thread-1]: ok
16:35:30.793  INFO | [pool-1-thread-1]: /opt/iamproxy/rds-client/results/stub.server.conf -> /opt/iamproxy/conf/jct/stub.server.conf
16:35:30.793  INFO | [pool-1-thread-1]: ok
16:35:30.793  INFO | [pool-1-thread-1]: /opt/iamproxy/rds-client/results/sps-st.upstream.conf -> /opt/iamproxy/conf/jct/sps-st.upstream.conf
16:35:30.793  INFO | [pool-1-thread-1]: ok
16:35:30.793  INFO | [pool-1-thread-1]: /opt/iamproxy/rds-client/results/spsSt.upstream.conf -> /opt/iamproxy/conf/jct/spsSt.upstream.conf
16:35:30.793  INFO | [pool-1-thread-1]: ok
16:35:30.793  INFO | [pool-1-thread-1]: /opt/iamproxy/rds-client/results/logger-st.server.conf -> /opt/iamproxy/conf/jct/logger-st.server.conf
16:35:30.793  INFO | [pool-1-thread-1]: ok
16:35:30.793  INFO | [pool-1-thread-1]: /opt/iamproxy/rds-client/results/aud-st.upstream.conf -> /opt/iamproxy/conf/jct/aud-st.upstream.conf
16:35:30.793  INFO | [pool-1-thread-1]: ok
16:35:30.793  INFO | [pool-1-thread-1]: /opt/iamproxy/rds-client/results/snoop.server.conf -> /opt/iamproxy/conf/jct/snoop.server.conf
16:35:30.793  INFO | [pool-1-thread-1]: ok
16:35:30.793  INFO | [pool-1-thread-1]: /opt/iamproxy/rds-client/results/snoop.upstream.conf -> /opt/iamproxy/conf/jct/snoop.upstream.conf
16:35:30.793  INFO | [pool-1-thread-1]: ok
16:35:30.793  INFO | [pool-1-thread-1]: /opt/iamproxy/rds-client/results/jct-snoop.server.conf -> /opt/iamproxy/conf/jct/jct-snoop.server.conf
16:35:30.793  INFO | [pool-1-thread-1]: ok
16:35:30.793  INFO | [pool-1-thread-1]: /opt/iamproxy/rds-client/results/prb-st2.upstream.conf -> /opt/iamproxy/conf/jct/prb-st2.upstream.conf
16:35:30.793  INFO | [pool-1-thread-1]: ok
16:35:30.793  INFO | [pool-1-thread-1]: /opt/iamproxy/rds-client/results/aud-st.server.conf -> /opt/iamproxy/conf/jct/aud-st.server.conf
16:35:30.793  INFO | [pool-1-thread-1]: ok
16:35:30.794  INFO | [pool-1-thread-1]: /opt/iamproxy/rds-client/results/rds-for-proxy.server.conf -> /opt/iamproxy/conf/jct/rds-for-proxy.server.conf
16:35:30.794  INFO | [pool-1-thread-1]: ok
16:35:30.794  INFO | [pool-1-thread-1]: /opt/iamproxy/rds-client/results/stub.upstream.conf -> /opt/iamproxy/conf/jct/stub.upstream.conf
16:35:30.794  INFO | [pool-1-thread-1]: ok
16:35:30.794  INFO | [pool-1-thread-1]: /opt/iamproxy/rds-client/results/sps-st.server.conf -> /opt/iamproxy/conf/jct/sps-st.server.conf
16:35:30.794  INFO | [pool-1-thread-1]: ok
16:35:30.794  INFO | [pool-1-thread-1]: /opt/iamproxy/rds-client/results/logger-st.upstream.conf -> /opt/iamproxy/conf/jct/logger-st.upstream.conf
16:35:30.794  INFO | [pool-1-thread-1]: ok
16:35:30.794  INFO | [pool-1-thread-1]: /opt/iamproxy/rds-client/results/jct-snoop.upstream.conf -> /opt/iamproxy/conf/jct/jct-snoop.upstream.conf
16:35:30.794  INFO | [pool-1-thread-1]: ok
16:35:30.794  INFO | [pool-1-thread-1]: /opt/iamproxy/rds-client/results/prb-st.server.conf -> /opt/iamproxy/conf/jct/prb-st.server.conf
16:35:30.794  INFO | [pool-1-thread-1]: ok
16:35:30.794  INFO | [pool-1-thread-1]: /opt/iamproxy/rds-client/results/prb-st.upstream.conf -> /opt/iamproxy/conf/jct/prb-st.upstream.conf
16:35:30.794  INFO | [pool-1-thread-1]: ok
16:35:30.794  INFO | [pool-1-thread-1]: /opt/iamproxy/rds-client/results/rds-for-proxy.upstream.conf -> /opt/iamproxy/conf/jct/rds-for-proxy.upstream.conf
16:35:30.794  INFO | [pool-1-thread-1]: ok
16:35:30.794  INFO | [pool-1-thread-1]: /opt/iamproxy/rds-client/results/test-sutb2.upstream.conf -> /opt/iamproxy/conf/jct/test-sutb2.upstream.conf
16:35:30.794  INFO | [pool-1-thread-1]: ok
16:35:30.794  INFO | [pool-1-thread-1]: /opt/iamproxy/rds-client/results/spsSt.server.conf -> /opt/iamproxy/conf/jct/spsSt.server.conf
16:35:30.794  INFO | [pool-1-thread-1]: ok
16:35:30.794  INFO | [pool-1-thread-1]: /opt/iamproxy/rds-client/results/prb-st2.server.conf -> /opt/iamproxy/conf/jct/prb-st2.server.conf
16:35:30.794  INFO | [pool-1-thread-1]: ok
16:35:30.794  INFO | [pool-1-thread-1]: ‘/opt/iamproxy/rds-client/results/index.html’ -> ‘/opt/iamproxy/conf/../html/index.html’
16:35:30.794  INFO | [pool-1-thread-1]: Успешно запущен reload iamproxy [31147]
16:35:30.794  INFO | [pool-1-thread-1]:  --- SCRIPT OUTPUT END ---
16:35:30.796  INFO | [pool-3-thread-1]: Trying to send GET request to https://mycompany-auth-svc-idp1-dev2.mycompany.ru:7443/rds-for-proxy/active-conf-json
16:35:30.945  INFO | [pool-3-thread-1]: *** SSL *** Force set private key with alias: platform-devb.mycompany.ru
16:35:31.076  INFO | [pool-3-thread-1]: Response received.
16:35:35.796  INFO | [pool-3-thread-1]: Trying to send GET request to https://mycompany-auth-svc-idp1-dev2.mycompany.ru:7443/rds-for-proxy/active-conf-json
16:35:35.812  INFO | [pool-3-thread-1]: Response received.
16:35:40.797  INFO | [pool-3-thread-1]: Trying to send GET request to https://mycompany-auth-svc-idp1-dev2.mycompany.ru:7443/rds-for-proxy/active-conf-json
16:35:40.812  INFO | [pool-3-thread-1]: Response received.
16:35:45.797  INFO | [pool-3-thread-1]: Trying to send GET request to https://mycompany-auth-svc-idp1-dev2.mycompany.ru:7443/rds-for-proxy/active-conf-json
16:35:45.809  INFO | [pool-3-thread-1]: Response received.
16:35:50.797  INFO | [pool-3-thread-1]: Trying to send GET request to https://mycompany-auth-svc-idp1-dev2.mycompany.ru:7443/rds-for-proxy/active-conf-json
16:35:50.811  INFO | [pool-3-thread-1]: Response received.
16:35:55.797  INFO | [pool-3-thread-1]: Trying to send GET request to https://mycompany-auth-svc-idp1-dev2.mycompany.ru:7443/rds-for-proxy/active-conf-json
16:35:55.809  INFO | [pool-3-thread-1]: Response received.

```

### При использовании IAM Proxy как программного балансировщика

Состояние сервиса:

```
systemctl status lb-iamproxy
```

Расположение логов:

`/opt/iamproxy/lb-logs/access-lb-iamproxy.log`

`/opt/iamproxy/lb-logs/error-lb-iamproxy.log`

## Сервер аутентификации (Keycloak)

Состояние сервиса:

```
systemctl status keycloak
```

### Логи java

`/opt/keycloak-4.8.3.Final/standalone/log/server.log`

```
2020-04-09 14:27:23,362 WARN  [org.keycloak.events] (default task-2) type=LOGIN_ERROR, realmId=master, clientId=security-admin-console, userId=71xx73xx-0xx0-4xxe-bxx4-fxx96xx43xx8, ipAddress=10.x.x.63, error=invalid_user_credentials, auth_method=openid-connect, auth_type=code, redirect_uri=https://10.x.x.178:8444/auth/admin/master/console/, code_id=f3xxbcx6-3xxf-4xx4-bxxf-75xx2exx59x9, username=admin
2020-04-09 14:27:23,395 WARN  [org.keycloak.services] (Brute Force Protector) KC-SERVICES0053: login failure for user 71xx73xx-0xx0-4xxe-bxx4-fxx96xx43xx8 from ip 10.x.x.63
2020-04-09 14:27:26,568 WARN  [org.keycloak.events] (default task-2) type=LOGIN_ERROR, realmId=master, clientId=security-admin-console, userId=71xx73xx-0xx0-4xxe-bxx4-fxx96xx43xx8, ipAddress=10.x.x.63, error=invalid_user_credentials, auth_method=openid-connect, auth_type=code, redirect_uri=https://10.x.x.178:8444/auth/admin/master/console/, code_id=f3xxbcx6-3xxf-4xx4-bxxf-75xx2exx59x9, username=admin
2020-04-09 14:27:26,579 WARN  [org.keycloak.services] (Brute Force Protector) KC-SERVICES0053: login failure for user 71xx73xx-0xx0-4xxe-bxx4-fxx96xx43xx8 from ip 10.x.x.63
2020-04-09 14:30:56,725 WARN  [org.keycloak.events] (default task-5) type=LOGIN_ERROR, realmId=PlatformAuth, clientId=PlatformAuth-Proxy, userId=a1xx33xe-bxx8-4xx6-8xxe-60xxa6xx0exc, ipAddress=10.x.x.180, error=invalid_user_credentials, auth_method=openid-connect, auth_type=code, redirect_uri=https://platform-devb.mycompany.ru/openid-connect-auth/redirect_uri, code_id=89xx63xa-6xxa-4xxe-9xx8-c1xx21xx84x3, username=sudir-admin
2020-04-09 14:30:56,751 WARN  [org.keycloak.services] (Brute Force Protector) KC-SERVICES0053: login failure for user a1xx33xe-bxx8-4xx6-8xxe-60xxa6xx0exc from ip 10.x.x.180
```

## Сервис обработки логов (Fluent Bit)

Посмотреть состояние сервиса, команда:

```
systemctl status fluent-bit
```

логи сервиса в  `/var/log/messages`

## Уровни логирования

При необходимости повышения уровня логирования в IAM Proxy, можно включить `debug` режим. При выключенном `debug`,
установлен уровень логирования `warn`.

Для изменения уровня логирования при deploy, необходимо задать следующие опции:

- для VM - в профиле развертывания посредством переключения параметра `debug` в файле `proxy.yml`;
- для OSE/k8s - в переменных окружения (environment).

> Значения для переменных окружения описаны в разделе
> [Руководство по установке](../installation-guide/proxy-deploy-docker-description.md).

Для изменения уровня логирования вручную, зайдите в настройки deployment, добавьте переменную окружения `DEBUG` со
следующими значениями:

- `1` - включает `debug` логирование на SynGX;
- `2` - включает уровень логирования `debug` для модулей SynGX.

При необходимости изменения уровня логирования, использовать параметр `DEBUG`/`auth.k8s.debug.enabled`/`debug`,
подробнее описано в разделе
[Соответствие имен параметров для разных сред и инструментов установки](../installation-guide/params.md).

Использование уровня логирования `debug` в штатном режиме не рекомендуется из-за больших ресурсозатрат и падения
производительности.

> При установке на ПРОМ среду (тип стенда `prom`), средствами Platform V DevOps Tools (CDJE), возможность задания уровня
> логирования в `debug` отсутствует (отключено по соображениям безопасности).

## Ротация логов

В IAM Proxy реализована функциональность для осуществления ротации логов. Ротация логов осуществляется посредством
автоматического запуска logrotate. Ротация осуществляется каждые 10 минут, либо при достижении размера файла с логами 20
Мб.

> Параметры настройки функциональности ротации логов приведены в разделе
> [Соответствие имен параметров для разных сред и инструментов установки](../installation-guide/params.md).

## Настройка системного журнала

> Все события, собираемые во время работы, публикуются в локальные файлы логов и централизованную систему Platform V 
> Monitor Журналирование (LOGA) через интеграцию с Fluent Bit. Настройка интеграции описана в
> [Установка](../installation-guide/installation.md).

Журнал регистрации событий располагается в следующих директориях:

- IAM Proxy: `/opt/iamproxy/logs/access.log, /opt/iamproxy/logs/error.log`
- RDS Client: `/opt/iamproxy/rds-client/logs/log-<дата>.log`
- Keycloak: `/opt/keycloak/log/keycloak-json.log`
- Fluent Bit: `/var/log/messages`.

Настройка уровня журналирования выполняется:

- для IAM Proxy: изменение параметра error_log в конфигурационном файле `/opt/iamproxy/conf/iamproxy.conf`;
  -  уровни: debug, info, warn, error.
- для Keycloak: через переменные окружения (KEYCLOAK_LOG_LEVEL) или вручную в конфигурационных файлах;
- для RDS Client: через настройки логирования в коде (уровень INFO по умолчанию).

Правила ротации логов:

- автоматическая ротация каждые 10 минут или при достижении размера файла 20 Мб;
- параметры настройки описаны в [Соответствие имен параметров для разных сред и инструментов установки](../installation-guide/params.md).

> Уровни DEBUG и TRACE не рекомендуются для использования в промышленной среде из-за высокой нагрузки на производительность.

## Доступ к системному журналу

Порядок доступа к логам:
- локальный доступ:
  - использование команды journalctl для просмотра системных журналов;
  - прямой доступ к файлам логов через SSH:

  ```shell
  cat /opt/iamproxy/logs/access.log
  tail -f /opt/iamproxy/logs/error.log
  ```
- централизованный доступ:
  - через систему Platform V Monitor Журналирование (LOGA) с использованием Fluent Bit Sidecar;
  - интеграция настраивается в конфигурационных файлах Kubernetes или OpenShift. 
## Основные события

Формат логов IAM Proxy:
- логи запросов (access.log);

  ```
  [22/Nov/2024:13:47:48 +0300]  -> 10.x.x.98:55231 -> 10.x.x.153:443 -> - - - "GET /MegaSystem/ HTTP/1.1" 302 138 - - "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/129.x.x.0 Safari/537.36 MyCompBrowser/21.x.x.0" "-" "-" "https://auth-psi.mycompany.ru/auth/realms/users/protocol/openid-connect/auth?state=35320e83a9991111538982abe266a143&scope=openid%20basic%20extended%20access&response_type=code&redirect_uri=https%3A%2F%2Ftvlos-sss00003.mycompany.ru%2Fopenid-connect-auth%2Fredirect_uri&nonce=a03bcd97d9999f57e0000728be723529&client_id=CI00890888" 4d458282c47504fdfd30dc5d8e022a44 1426 2899 "0cb8d7e0f80715506700004e3e91b06c"
  ```
- логи ошибок (error.log).  
  ```
  2020/04/09 17:36:04 [crit] 2385#2385: *7 SSL_do_handshake() failed (SSL: error:140944E7:SSL routines:ssl3_read_bytes:reason(1255):SSL alert number 255) while SSL handshaking, client: 10.x.x.178, server: 0.0.0.0:10444
  ```
**Примеры записей логов** 

| Уровень | Текст/шаблон сообщения                                                                                         | перевод                                                                                          |
|---------|----------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------|
| ERROR   | SSL_do_handshake() failed (SSL: error:140944E7:SSL routines:ssl3_read_bytes:reason(1255):SSL alert number 255) | Ошибка SSL-рукопожатия: SSL alert number 255                                                     |
| WARN    | the "user" directive makes sense only if the master process runs with super-user privileges                    | Директива "user" имеет смысл только если мастер-процесс запущен с привилегиями суперпользователя |
| INFO    | Write to file: /opt/iamproxy/rds-client/results/sps-st.upstream.conf                                           | Запись конфигурационного файла sps-st.upstream.conf                                              |

**Примеры JSON-записей**
- уровень ERROR:
  ```
  {
   "eventdate": "2024-11-22T13:48:00+03:00",
   "source_message": "SSL_do_handshake() failed (SSL: error:140944E7:SSL routines:ssl3_read_bytes:reason(1255):SSL alert number 255)",
   "logger_name": "nginx:error",
   "level": "ERROR",
   "tier": "local",
   "app_id": "iamproxy"
  }
  ```
- уровень WARM:
  ```
  {
   "eventdate": "2024-11-22T13:47:48+03:00",
   "source_message": "the 'user' directive makes sense only if the master process runs with super-user privileges",
   "logger_name": "nginx:warn",
   "level": "WARN",
   "tier": "local",
   "app_id": "iamproxy"
  }
  ``` 
- уровень INFO:
  ```
  {
   "eventdate": "2024-11-22T16:35:30+03:00",
   "source_message": "Write to file: /opt/iamproxy/rds-client/results/sps-st.upstream.conf",
   "logger_name": "rds-client:info",
   "level": "INFO",
   "tier": "local",
   "app_id": "rds-client"
  }
  ```   