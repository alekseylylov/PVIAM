# Описание настройки IAM Proxy при запуске в контейнере

## Каталоги и файлы для настройки

Файлы используемые для настройки:

- `/certs/server.crt.pem` - открытый ключ сертификата на HTTPs;
- `/certs/server.key.pem` - закрытый ключ сертификата на HTTPs;
- `/certs/server_backend.crt.pem` - открытый ключ клиентского сертификата на HTTPs с бэкенд;
- `/certs/server_backend.key.pem` - закрытый ключ клиентского сертификата на HTTPs с бэкенд;
- `/certs/trusted_*.crt.pem` - файлы с доверенными сертификатами для бэкенд;
- `/iamproxy/conf/custom.d/*.main.conf` - кастомные настройки (конфигурации) сервера, включаются в секцию `main` (
  корневой уровень);
- `/iamproxy/conf/custom.d/*.events.conf` - кастомные настройки (конфигурации) сервера, включаются в секцию `events`;
- `/iamproxy/conf/custom.d/*.http.conf` - кастомные настройки (конфигурации) сервера, включаются в секцию HTTP;
- `/iamproxy/conf/custom.d/*.server.conf` - кастомные настройки (конфигурации) сервера, включаются в секцию `server`;
- `/iamproxy/conf/custom.d/*.location.conf` - кастомные настройки для ответвления, файл включается в секцию `location`
  если добавлен в опции ответвления `applyJctRequestFilter`;
- `/iamproxy/conf/custom.d/*.upstream.conf` - кастомные настройки для ответвления, файл включается в секцию `upstream`
  если добавлен в опции ответвления applyJctRequestFilter;
- `/iamproxy/conf/custom.d/whitelist-useragent*.map-rules.conf` - кастомные правила проверки заголовка User-Agent на
  принадлежность к белому списку (имеет приоритет над черным списком), включаются в секцию `map`;
- `/iamproxy/conf/custom.d/blacklist-useragent*.map-rules.conf` - кастомные правила проверки заголовка User-Agent на
  принадлежность к черному списку, включаются в секцию `map`;
- `/iamproxy/conf/custom.d/blacklist-useragent.txt` - список строк для проверки заголовка User-Agent на принадлежность к
  черному списку (для применения необходимо задать путь до этого файла в опции `PROXY_BLACKLIST_USERAGENT_FILE`);
- `/iamproxy/rds-client/start-conf.json` - конфигурация ответвлений с содержимым аналогичным опции
  `RDS_START_CONF` (опция `RDS_START_CONF` в переменных окружения имеет приоритет над данным файлом).

## Переменные, используемые для настройки

**Все** параметры указанные ниже являются **не обязательными**, и имеют либо значения по умолчанию, либо опциональны и
при их отсутствии функциональность будет отключена.

### Параметры основной функциональности компонента IAM Proxy

- `STEND_NAME` - название стенда/сервиса для отображения в UI;
- `STEND_ABBR` - используется для обозначения стенда в технических названиях (латиница в нижнем регистре);
- `STEND_TYPE` - тип стенда. Допустимы следующие значения: `dev`, `ift`, `psi`, `prom`, `nt`. По умолчанию
  `prom`;
- `PROXY_SESSION_SECRET` - секретный ключ для шифрования сессии прокси (произвольный набор символов, рекомендуется
  длинна не менее 512 символов);
- `PROXY_SESSION_IDLETIME` - опциональный. По умолчанию имеет значение 240. Время жизни сессии прокси по не активности в
  минутах;
- `PROXY_SESSION_NAME` - опциональный. По умолчанию имеет значение `PLATFORM_SESSION`. Имя сессионного `cookie`;
- `PROXY_SESSION_DOMAIN` - имя домена для сессионной cookie (значение '..' позволяет задать родительский домен от
  фронтового fqdn);
- `PROXY_SESSION_CHECK_ADDR` - `True`\ `False`. По умолчанию имеет значение `False`. Включает привязку сессии к
  клиентскому IP;
- `PROXY_WORKER_PROCESSES` - количество запускаемых процессов прокси для обработки соединений (по умолчанию`auto`, и
  будет равно кол-ву CPU);
- `SYSLOG_SERVER` - сервер:порт для отправки событий аудита по протоколу syslog;
- `PROXY_TO_SYSLOG_FORMAT` - значение по умолчанию `main_syslog`. Наименование формата отправляемого события
  access-лога (имеющиеся форматы из "коробки" - `main_pp`, `main_syslog`, `log_json`, `log_json_small`,
  `log_json_audit`, `log_req_resp`);
- `PROXY_TO_SYSLOG_FILTERED` - `True`/`False`, по умолчанию `True`. `True` - фильтровать события и отправлять только те,
  где в `Content-Type` ответа есть одна из строк `text/html`, `application/json`, `application/x`, `False` - не
  фильтровать события перед отправкой;
- `PROXY_KEEPALIVE_TIMEOUT` - `timeout [header_timeout]`, опциональный, default `180 170`, параметр (timeout);
  ограничивающий время клиентских keepalive соединений, второй параметр `header_timeout` попадет в заголовок ответа в
  формате: `Keep-Alive: timeout=header_timeout` (по умолчанию будет заголовок `Keep-Alive: timeout=170`);
- `PROXY_KEEPALIVE_BACKEND_CONNECTIONS` - по умолчанию отключен. Опциональный, параметр устанавливает максимальное число
  неактивных постоянных соединений с серверами группы (в сторону бэкенд), которые будут сохраняться в кеше каждого
  рабочего процесса (при превышении этого числа наиболее давно не используемые соединения закрываются);
- `PROXY_KEEPALIVE_BACKEND_TIMEOUT` - по умолчанию `60s`. опциональный, задает тайм-аут, в течение которого неактивное
  постоянное соединение с сервером группы (в сторону бэкенд) не будет закрыто;
- `PROXY_STICKY_BACKEND_BY_SESSION_ENABLE` - Опциональный. `True`/`False`, по умолчанию `False`, настройка использования
  привязки подключения клиента к серверу группы (в сторону бэкенд) по hash от `session_id+real_ip`;
- `PROXY_SSL_SESSION_CACHE` - опциональный. По умолчанию имеет значение `none`. Настройка использования кеша SSL сессий
  клиентских подключений, в параметре указывается объем памяти выделенный под кеш в мегабайтах (в 1 мегабайте может
  поместиться около 4000 SSL сессий), так же можно указать `none`(разрешение использования кеша на клиенте) или `off`(
  запрет использования Cache);
- `PROXY_SSL_STAPLING_ENABLE` - Опциональный. `True`/`False`, По умолчанию `False`, настройка использования ssl
  stapling (кеширование/прикрепление ответов от OCSP на стороне сервера, при установке TLS);
- `PROXY_JCT_SSL_NAME` - по умолчанию при проверке сертификата на проксируемом сервере считаем валидные CN/SAN (из
  сертификатов бэкенд) c таким доменом/host `.ru`;
- `PROXY_MTLS_FRONT_VERIFY_DN_REGEX` - Проверка regexp строки `subject DN` клиентского сертификата для установленного на
  фронте SSL-соединения (`*` или пустая строка допускает любой DN), пример
  `^(CN=SMDEVPROFILEPRODusr,OU=MyCompdevices,O=MyCompany,C=RU|CN=SMCONSOLEGWPRODusr,OU=MyCompdevices,O=MyCompany,C=RU)$`.
- `PROXY_REDIRECT_FROM_ROOT_TO_URL` - Redirect на относительный URL, при запросе на корень. Значение по умолчанию ` `.
- `PROXY_METRICS_ENABLE` - опциональный. По умолчанию имеет значение `False`. `True` - активировать сбор метрик и
  публикацию их в формате Prometheus на `http://127.0.0.1:10080/metrics/`. Метрики приведены в разделе
  [Выводимые метрики и их описание](../administration-guide/monitoring.md);
- `PROXY_HEALTCHECK_ENABLE` - опциональный. По умолчанию имеет значение `False`, `True` - включение активного
  `healthcheck` до серверов бэкенд;
- `PROXY_SUPPORT_ISAM_HEADERS` - `True`/`False`. Опциональный. По умолчанию имеет значение `True`. `True` - добавлять в
  запросы HTTP-заголовки аналогично ISAM/WebSeal (`iv-user`, `iv-groups`, `iv-remote-address`); Для точечного отключения
  на ответвлении следует использовать опцию `common/rds-disable-support-isam-headers.location.conf` (в
  `applyJctRequestFilter`) (deprecated);
- `PROXY_SUPPORT_CUSTOM_CONFIGS` - `True`/`False`. Опциональный. По умолчанию имеет значение `True`. `True` - разрешить
  подключение/include файлов конфигурации из каталога `custom.d`;
- `PROXY_X_FORWARDED_PORT` - auto/номер порта. Опциональный. По умолчанию имеет значение `auto`. Значение `auto` -
  получить номер порта из заголовка `X-Forwarded-Port` или из `$proxy_protocol_port` или из заголовка Host или из
  заголовка `X-Forwarded-Proto` или из `$scheme` (указано в порядке получения);
- `PROXY_X_FORWARDED_PROTO` - `https`/`http`. Опциональный. По умолчанию будет `scheme` из `listener`;
- `WAIT_EXIST_FILES` - опциональный. Список файлов через пробел, появления которых необходимо дождаться перед запуском
  прокси. К примеру это может понадобиться, когда файлы сертификатов доставляются по pod отдельным
  процессом/контейнером/sidecar, если на момент запуска IAM Proxy, файлов еще не будет, то нужно будет дождаться их
  появления, а потом начать старт IAM Proxy. Пример: `/certs/server.key.pem /certs/server.crt.pem`;
- `J2_IGNORE_NAMES` - опциональный. Задать список шаблонов имен j2-файлов, которые нужно исключить из обработки
  j2-шаблонизатором. Пример: `my.skip-files.*.j2 oidc-auth-secrets.server.conf.j2`;
- `PROXY_LOG_TO_FLUENT_BIT_ENABLE` - True/False, опциональный. default False, включение логирования в
  `/var/log/logsshare/*.log`, используется при отправке журналов через fluent-bit;
- `PROXY_LOG_TO_FLUENT_BIT_ACCESS_LOG` - опциональный, путь до файла access-логов отправляемых через Fluent Bit (можно
  указать сервер syslog, например `syslog:server=127.0.0.1:15140`);
- `PROXY_LOG_TO_FLUENT_BIT_ERROR_LOG` - опциональный, путь до файла error-логов отправляемых через fluent-bit (можно
  указать сервер syslog, например `syslog:server=127.0.0.1:15141`);
- `PROXY_ROTATE_FILES` - опциональный, полные пути до файлов логов (через запятую), которые надо ротировать;
- `PROXY_ROTATE_FILE_SIZE` - опциональный, default 4000, максимальный размер одного файла в кб;
- `PROXY_ROTATE_TOTAL_SIZE` - опциональный, default 40000, максимальный размер всех файлов в кб, которые под ротацией;
- `PROXY_REQUEST_TO_UI_REGEX` - опциональный, default `/$`, регулярное выражение применяемое к URI запроса, при
  совпадении считается что тип запроса - запрос к UI. Для UI запросов в случае отсутствия аутентификации будет
  перенаправление на аутентификацию в IDP провайдер, для других типов запросов (API, JS, PNG, ...) в таком случае будет
  ответ с кодом 401;
- `PROXY_REQUEST_TO_API_REGEX` - опциональный, default `/api/`, регулярное выражение применяемое к URI запроса, при
  совпадении считается что тип запроса - запрос к API. Для API запросов в случае отсутствия аутентификации будет ответ с
  кодом 401;;
- `PROXY_LIMIT_REQ_JCT_IS_PER_SESSION` - значение по умолчанию `False`, при True ограничения по ответвлениям (заданные в
  limitRequests)
  будут действовать по каждой сессии аутентификации отдельно;
- `PROXY_AUDIT_TO_FLUENT_BIT_ENABLE` - значение по умолчанию `False`. Включение логирования в
  `/var/log/logsshare/*.log`, используется при отправке аудита через fluent-bit (можно указать сервер syslog, например
  `syslog:server=127.0.0.1:15142`);
- `PROXY_BLACKLIST_ENABLE` - по умолчанию `False`, `True` - включение блокировки по черному списку, по заголовку
  User-Agent (запросы ботов и сканеров);
- `PROXY_BLACKLIST_USERAGENT_FILE` - по умолчанию `/opt/iamproxy/conf/blacklist/blacklist-useragent.txt`, путь до
  текстового файла с черным списком строк (вхождение строки в User-Agent приводит к разрыву соединения и коду 444 в
  access-логе). Этот параметр задается **только в том случае**, если используется файл расположенный не по стандартному
  пути `/opt/iamproxy/conf/blacklist/blacklist-useragent.txt`.
- `DEBUG` - не пустое значение включает расширенное логирование (как в SynGX, так и на уровне bash-скриптов), значение
  `1` включает `debug` логирование только на уровне Lua-модуля SynGX, значение `2` включает уровень логирования `debug`
  для всех модулей SynGX, значение `wait` дополнительно включает ожидание в 1 час после завершения всех процессов в
  контейнере.

### Параметры подключения провайдеру аутентификации платформы (Platform V IAM KeyCloak.SE)

- `KEYCLOAK_FRONT_DNS_NAME` - DNS-имя сервиса Keycloak, используемое с front-end/браузера (LB);
- `KEYCLOAK_HTTPS_PORT_ON_FRONTEND` - порт к KEYCLOAK_FRONT_DNS_NAME;
- `KEYCLOAK_ADMIN_DNS_NAME` - DNS-имя сервиса Keycloak для административного интерфейса, используемое с
  front-end/браузера (LB);
- `KEYCLOAK_HTTPS_ADMIN_PORT` - Отдельный бэкенд порт для работы с административной консолью на сервере приложений;
- `KEYCLOAK_REALM` - используемое при аутентификации имя realm в Keycloak (по умолчанию "PlatformAuth");
- `OIDC_POST_LOGON_BY_TOKEN_CALL_URI` - вызвать endpoint на `IDP` после восстановления по токену сессии на прокси.

### Параметры для доступа сетевых клиентов с front-end (браузер/агент)

- `PROXY_DNSNAME` - DNS-имя сервиса IAM Proxy, используемое с front-end/браузера (LB);
- `LB_HTTPS_PORT` - порт к `PROXY_DNSNAME`.

### Параметры ограничения скорости запросов

#### Описание

Реализация Rate Limiter на стороне IAM Proxy подразумевает под собой защиту от аномального объема трафика как в сторону
IDP-провайдера аутентификации, так и в сторону бэкенд-приложения. Механизм защиты основан на конфигурируемом
ограничении скорости по количеству запросов в единицу времени. В случае превышения количества запросов они будут
задерживаться для соблюдения заданного лимита, или пользователю будет показано сообщение, о необходимости повторения
попытки через некоторое время.

Так как обычно клиентом является браузер, а проксирование производится на web-приложения, то нагрузка от одного клиента
в большинстве случаев будет со значительными всплесками (например при первичном открытии страницы или переходе на
какой-то новый раздел в UI), поэтому при ограничениях допускаются всплески в рамках одного клиента, чтобы пропустить без
задержек разовую пиковую нагрузку.

Запросы, превышающие объем всплеска, будут отклонены (с кодом ответа 509). Для запросов не связанных с аутентификацией
этот лимит увеличен в 4 раза (другими словами, если время ожидания запроса в очереди будет превышать 4 секунды, этот
запрос будет отклонен, что является штатным поведением).

![](resources/schema_limit_by_requests.png)

#### Настройка

Настройка на IAM Proxy производится параметрами:

- `PROXY_LIMIT_PER_IP_ENABLE` - `True`/`False`, опциональный, default `False`, включение ограничения скорости запросов,
  определение источника/клиента производится по его IP (на основе нескольких источников - X-Forwarded-For, PROXY
  Protocol, TCP ip.src);
- `PROXY_LIMIT_PER_SESSION_ENABLE` - `True`/`False`, опциональный, default `False`, включение ограничения скорости
  запросов, определение источника/клиента производится по его сессии аутентификации;
- `PROXY_LIMIT_REQ_PER_SEC` - запросов/секунду, опциональный, default `60`, ограничение скорости запросов на бэкенд в
  количестве запросов допустимых в секунду (допускается всплеск запросов по кол-ву в 4 раза превышающий заданный лимит
  по r/s, запросы в пределах всплеска будут на время задерживаться, чтобы соблюсти заданную скорость, а по превышению
  всплеска запросы будут отклоняться с HTTP кодом 509);
- `PROXY_LIMIT_REQ_OIDC_PER_MIN` - запросов/минуту, опциональный, default `2`, ограничение скорости запросов по сессии
  прокси к функциональности `OIDC`(обращения к провайдеру `Open ID Connect`) в максимально допустимом количестве
  запросов в минуту (допускается всплеск в кол-ве 10 запросов, по превышению всплеска запросы будут отклоняться с HTTP
  кодом 509).

> Примечание: Лимиты действуют в рамках одного экземпляра IAM Proxy, что следует учитывать при расчете возможной
> постоянной максимальной нагрузки на бэкенд, которую пропустит IAM Proxy.

#### Пример расчета параметров

Например, при включении ограничений по `PROXY_LIMIT_PER_IP_ENABLE` и `PROXY_LIMIT_PER_SESSION_ENABLE`, и с
`PROXY_LIMIT_REQ_PER_SEC`=60, пройдут 240(60*4) запросов без ограничений (допустимый всплеск), потом они начнут
искусственно задерживаться до соблюдения ограничения 60 запросов в секунду. Следующие запросы в рамках всплеска будут
возможны, когда очередь ограничения запросов станет пустой (если полностью остановится нагрузка, то через 4 секунды
можно будет повторить всплеск в 240 запросов).

Отдельное дополнительное ограничение будет для запросов связанных с IDP, если `PROXY_LIMIT_REQ_OIDC_PER_MIN`=2, то
запросов приводящих к вызову IDP (например новая аутентификация) можно будет выполнить разово не более 10(допустимый
всплеск), потом же таких запросов можно будет сделать не более 2 в минуту, иначе будет выдано сообщение повторить запрос
позже.

### Параметры подключения к Identity Provider OIDC

- `PROXY_OIDC_CLIENT_ID` - `CLIENT_ID` используемый для аутентификации по `OIDC` на `IDP`;
- `PROXY_OIDC_CLIENT_SECRET` - секрет к CLIENT_ID;
- `PROXY_OIDC_CLIENT_ID_ALT` - альтернативный `CLIENT_ID` используемый для аутентификации по `OIDC` на `IDP`;
- `OIDC_CLIENT_RSA_PRIVATE_KEY` - аутентификация на IDP методом "private_key_jwt", и тут указывается "закрытый
  клиентский ключ":

```
-----BEGIN PRIVATE KEY-----
MIIEvgIBADANBgkqhkiG9w0BAQEFAASCBKgwggSkAgEAAoIBAQCOfXFnv2XWPEpX
...
-----END PRIVATE KEY-----
```

- `OIDC_SSL_VERIFY` - True/False, проверять сертификат на endpoint OIDC;
- `OIDC_DISCOVERY_URL` - задание `URL` метаданных `OIDC` `IDP` (`https://all-sh-suddwsa02app.mycompany.mycompany.
  ru/mga/sps/oauth/oauth20`);
- `OIDC_LOGOUT_URI` - задание `URL` на который делать redirect при logout (`https://all-sh-suddwsa02app.mycompany.
  mycompany.ru/pkmslogout`);
- `OIDC_POST_LOGOUT_REDIRECT_PATH` - значение по умолчанию "/openid-connect-auth/logoutSuccessful.html", относительный
  путь, на который будет перенаправление после успешного logout;
- `OIDC_SCOPE` - задание дополнительных scope `OIDC` при аутентификации `basic access`;
- `OIDC_USE_IDP_PROVIDER` - вход на Keycloak через заранее указанного внешнего провайдера `esia`;
- `OIDC_USE_CLIENT_CERT` - `True`/`False`, использовать клиентский сертификат на endpoints OIDC;
- `OIDC_HOST_GRAY` - использовать для подключения к `OIDC` `IDP` отдельный ip:port, а не тот который в `URL`
  из `OIDC_DISCOVERY_URL` (может потребоваться при необходимости использования серых адресов IDP, для включения также
  необходимо выставить `OIDC_USE_CLIENT_CERT` в `True`). Пример "1.1.1.1:8443";
- `PROXY_TO_BACKEND_ACCESS_TOKEN` - опциональный, `True`/`False`. Значение по умолчанию `False`, настройка использования
  отправки `access-token` вместо `id-token` в сторону бэкенд;
- `OIDC_USE_PKCE` - `True`/`False`, значение по умолчанию `False`. Использовать `PKCE` при аутентификации по OIDC;
- `OIDC_USERNAME_ATTR` - какой атрибут из jwt-токена использовать в качестве имени/логина пользователя, и будет
  использоваться в логах, в header `iv-user` и тому подобное. Значение по умолчанию `preferred_username`;
- `OIDC_CHANGE_ORG_ENDPOINT` - ссылка на endpoint смены организации на IDP (реализован на KCSE в провайдере ЕСИА). При
  его вызове передаются параметры `client_uuid`, `redirect_uri`, `org_id`. Значение по умолчанию `keycloak_base_url`
  + `/auth/realms/PlatformAuth/broker/esia/endpoint/change-org` - переменная `keycloak_base_url` формируется на основе
    параметров `KEYCLOAK_*`;
- `OIDC_POST_CHANGE_ORG_ENDPOINT` - ссылка, на которую будет перенаправление после успешной смены организации на IDP.
  Пример: `/portal/start-page`;
- `OIDC_MAP_HEADERS` - добавить HTTP-заголовки по токену. Пример: `header-with-user-id=sub, user_roles=realm_access.
  roles, user.audiences=aud, User-FIO=name,user-access-systems=resource_access`. В значении использовать модификаторы,
  детали смотрите в разделе "Использование модификаторов HTTP-заголовков";
- `OIDC_MAP_ROLES` - задать mapping получения ролей из токена; пример `'realm_roles=realm_access.roles, root_roles=roles, client_roles=resource_access.${client_id}.roles, esia.
  roles.from_userinfo=userinfo@esia.roles'`;
- `OIDC_USERINFO_MAP` - задать список атрибутов из `userinfo` (полученных из URI userinfo API IDP) для сохранения(
  кэширования) их в контексте сессии IAM Proxy, например
  `OIDC_USERINFO_MAP: 'esia.roles=esia.roles , org=upper@organization.org_name, client_roles=resource_access.${client_id}.role'` (
  потом к сохраненной части можно будет обратиться через модификатор `userinfo` в параметрах `OIDC_MAP_HEADERS` и
  `OIDC_MAP_ROLES`, например `OIDC_MAP_HEADERS: 'org_from_userinfo=userinfo@org'`);
- `OIDC_ACCESS_TOKEN_EXPIRES_LEEWAY_RAND` - за сколько секунд до истечения `access-token` обновить токены (реальное
  количество секунд будет выбрано случайным образом от этого числа, а ошибка обновления будет проигнорирована если
  `access-token` действителен). Можно указать проценты от времени действия `access-token`, например `20%` или `0.2` ;
- `OIDC_REVOKE_TOKENS_ON_LOGOUT` - `True`/`False`. Значение по умолчанию `True`, включить процедуру отзыва токенов при
  logout (отзываются токены refresh и/или access при их наличии, и при условии, что IDP поддерживает отзыв/revoke
  токенов);
- `OIDC_DISABLE_LOGOUT_IN_IDP` - `True`/`False`. Значение по умолчанию `False`. Не производить logout сессии
  пользователя на IDP (logout на IDP обычно закрывает сессии всех клиентов/АС, созданных на данном IDP от имени
  пользователя);
- `OIDC_IGNORE_SSO_ON_IDP` - True/False, default False, включить принудительный запрос логин/пароля при каждой
  аутентификации на IDP (игнорируем SSO);
- `OIDC_REUSE_REFRESH_COUNT` - опциональный, значение по умолчанию `-1`, определяет количество повторных использований
  одного refresh токена (значение менее 0 снимает ограничение);
- `OIDC_ACCESS_TOKEN_EXPIRES_IN` - опциональный, значение по умолчанию `0`, период в секундах, после которого access
  токен считается недействительным. Значение 0 отключает периодическую проверку;
- `OIDC_INTROSPECTION_INTERVAL` - опциональный, значение по умолчанию `0`, период в секундах, с которым производится
  проверка access токена на отзыв (вызова introspect API IDP при авторизации запроса). Значение `0` отключает
  периодическую проверку;
- `OIDC_DIRECT_AUTH_ENABLED` - `True`/`False`, значение по умолчанию `False`. Включить аутентификацию direct-auth по
  умолчанию на всех ответвлениях;
- `OIDC_DIRECT_AUTH_HEADER` - значение по умолчанию `Authorization`. Заголовок из запроса, который надо передать на IDP
  при аутентификации direct-auth;
- `OIDC_DIRECT_AUTH_GRANT_TYPE` - значение по умолчанию `password`. Значение `grant_type`, используемое при вызове IDP,
  при аутентификации direct-auth;
- `OIDC_DIRECT_AUTH_CHALLENGE_TEXT` - опциональный, значение `challenge` (HTTP заголовка "WWW-Authenticate"), которое
  добавляется при ошибке аутентификации direct-auth (пример для BA: `Basic realm="PlatformAuth Users`).

### Использование модификаторов HTTP-заголовков

Цель применения модификаторов состоит в изменении информации, передаваемой в заголовках HTTP-запросов, определяемых
опциями `OIDC_MAP_HEADERS`, `OIDC_MAP_ROLES`, `OIDC_USERINFO_MAP`.

Синтаксис использования: <модификатор>@<параметры передаваемые в модификатор>

Список модификаторов, которые можно использовать для преобразования содержимого HTTP-заголовков:

- `qp` - преобразует данные в строку, содержащую только ASCII символы; модификатор заменяет не ASCII символы
  специальными последовательностями, состоящими из символа "=" и двух символов шестнадцатеричного кода;\ пример:
  значение "Hello, привет, мир!", будет преобразовано в
  "Hello, =D0=9F=D1=80=d0=B8=D0=B2=D0=B5=D1==82, =D0=BC=D0=B8=D1=80!";
- `decode_qp` - декодирование текста, закодированного в формате quoted-printable;\
  пример: "Hello, =D0=9F=D1=80=d0=B8=D0=B2=D0=B5=D1==82, =D0=BC=D0=B8=D1=80!" будет преобразован в "Hello, привет, мир!"
  ;
- `base64` - кодирование данных в формате base64;
- `decode_base64` - декодирование данных, закодированных в формате base64;
- `uri` - кодирование текста в формат uri (стандарт https://docs.racket-lang.org/net/uri-codec.html), позволяющий
  использовать специальные символы в URI;\
  пример, значение "my page" будет преобразовано в "my%20page";
- `unescape_uri` - декодирование текста, закодированного в формате uri;\
  пример, значение "my%20page" будет преобразовано в "my page";
- `md5` - вычисление хэш-суммы md5 от текста или данных;
- `upper` - преобразование текста в верхний регистр;
- `lower` - преобразование текста в нижний регистр;
- `trunc` - обрезка текста до длины в 255 символов;
- `userinfo` - позволяет получить данные из информации, полученной из endpoint userinfo API IDP
  (получение этой информации предварительно настраивается опцией `OIDC_USERINFO_MAP`),\
  пример: `userinfo@org` передаст в http-заголовок атрибут `org` указанный в опции `OIDC_USERINFO_MAP`;
- `joinStrBySpace` - объединяет в строку значения полученные из id-token через пробел (не ASCII символы будут
  закодированы используя escape_uri), имена атрибутов id-token (которые необходимо объединить) задаются как параметры
  через ":",\
  пример: joinStrBySpace@firstname:surname вернет "Ivan Ivanov";
- `appendKey` - используется для добавления имени атрибута источника к значению, в виде "ключ:значение".\
  Например, параметр `SYS=appendKey@systems` в `OIDC_MAP_ROLES`, приведет к добавлению в массив ролей из атрибута токена
  `systems` ролей со значениями `systems:name-system1`, `systems:name-system2` ...;
- `appendName` - используется для добавления имени атрибута результата к значению, в виде "имя:значение".\
  Например, параметр `SYS=appendName@systems` в `OIDC_MAP_ROLES`, приведет к добавлению в массив ролей из атрибута
  токена `systems` ролей со значениями `SYS:name-system1`, `SYS:name-system2` ....

### Использование реаутентификации

Для защиты произвольного `GET` вызова явной аутентификацией на IDP (как пример какой-то важный вызов на подтверждение
финансовой операции), есть два независимых варианта запустить реаутентификацию на IAM Proxy:

- добавить в аргументы GET вызова аргумент `requiredReauthentification=true` (например
  `https://<iamproxy>/mypath/mypage?requiredReauthentification=true`)
- вызвать специальный endpoint на IAM Proxy `https://<iamproxy>/openid-connect-auth/reauth` с аргументом
  `redirect_uri=<защищаемый URL>` (например
  `https://<iamproxy>/openid-connect-auth/reauth?redirect_uri=/mypath/mypage`)

В результате при вызове произойдет перенаправление на IDP за аутентификацией с указанием не использовать SSO на IDP
(будет добавлен аргумент `prompt=login` в вызов на аутентификацию, при этом IDP провайдер должен его поддерживать).
После успешной реаутентификации будет перенаправление на защищаемый URL с добавлением/заменой в нем аргумента
`requiredReauthentification=<начало реаутентикации в unixtime>` (например
`https://<iamproxy>/mypath/mypage?requiredReauthentification=1732739926`).

> При реализации на бэкенд следует так же проверять в аргументах `GET` вызова наличие **не пустого**
> аргумента `requiredReauthentification`, чтобы удостовериться, что на IAM Proxy использовалась реаутентификация.

### Принудительный запуск аутентификации заново

При необходимости выполнить аутентификацию повторно, есть возможность запустить сценарий аутентификации OIDC заново, для
этого в произвольном `GET` запросе нужно указать аргумент `needReauthentification=true` (например
`https://<iamproxy>/mypath/mypage?needReauthentification=true`). После успешного завершения аутентификации будет
перенаправление на защищаемый URL с удалением в нем аргумента `needReauthentification`(например
`https://<iamproxy>/mypath/mypage`).

> Аргумент `needReauthentification` будет работать только на ответвлениях, где **не отключена** необходимость
> аутентификации.
>
> Пока не завершена новая аутентификация, текущая аутентификация и сессия будут продолжать действовать, и например
> запросы в соседних вкладках браузера будут проходить нормально.

### Параметры подключения к `Авторизация`

Функциональность является устаревшей (deprecated). Описание использования функциональности авторизации `Авторизация`
смотрите в разделе
[Функциональность аутентификации на основе прав из Авторизации](../administration-guide/developer-proxy-azgt-description.md).

- `AUTHZ_SPAS_URL` - пример "https://10.x.x.10:8443/spas/rest", url для вызова `API Авторизации`;
- `AUTHZ_SPAS_SECRET` - пример 123456, секрет для вызова `API Авторизации`;
- `AUTHZ_SPAS_TICKET_LIFETIME` - пример 3600, частота обновления запроса Авторизации, в секунду;
- `AUTHZ_SPAS_TICKET_FAILED_LIFETIME` - пример 5, частота получения запроса Авторизации, если ранее попытка была
  неуспешной, в секунду;
- `AUTHZ_SPAS_RIGHTS_LIFETIME` - пример 60, частота обновления полномочий из `Авторизации`, в секунду;
- `AUTHZ_SPAS_RIGHTS_FAILED_LIFETIME` - пример 5, частота обновления полномочий из `Авторизации`, если ранее попытка
  была неуспешной, в секунду;
- `AUTHZ_SPAS_SSL_VERIFY` - пример False, проверять сертификат на endpoint `authz_spas_url`.

Для включения авторизации в настройках ответвления (в applyJctRequestFilter) задаем одну из опций:

- `common/rds-use-authz-by-url.location.conf`;
- `common/rds-use-authz-by-url-bridge.location.conf`;
- `common/rds-use-authz-by-url-and-method.location.conf`.

### Авторизации HTTP-запроса по аргументам (POST/GET) запроса или по его заголовкам

`AUTHZ_HTTP_OPTS` - параметр `opts_str` на входе, это строка вида `uri_regexp:post_arg=str1${token_attr}str2 ,
uri_regexp:get_arg!=${token_attr} , uri_regexp:@header~regexp_expr, uri_regexp : post_arg !=^ ${token_attr}`, где:

- `uri_regexp` - регулярное выражение применяемое к URI запроса (параметр `uri`), которое определяет необходимость
  применения (проверяться может не чистый URI, а например `method+uri`, в зависимости от того что будет в параметре uri)
  данного правила авторизации (наличие совпадения);
- `post_arg` - аргумент из тела POST запроса;
- `get_arg` - аргумент из query URI (для не POST запросов);
- `header` - имя http заголовка из запроса;
- `token_attr` - имя атрибута в id-токене (может быть путь к атрибуту);
- `regexp_expr` - регулярное выражение, по которому нужно будет проверить данные из запроса (применяется с
  операндами "~" или "!~").

Возможные операнды сравнения:

- `=` равенство строк;
- `!=` неравенство строк;
- `=^` равенство строк без учета регистра;
- `!=^` неравенство строк без учета регистра;
- `~` есть совпадения по regex;
- `!~` нет совпадений по regex.

### Параметры подключения к авторизации `API OPA`

- `AUTHZ_OPA_URL` - пример "https://10.x.x.10:8443/ssp-portal/rm/api/v1/check", url для вызова `API` OPA
- `AUTHZ_OPA_SSL_VERIFY` - пример False, проверять сертификат на endpoint `authz_spas_url`

Для включения авторизации в настройках ответвления (в applyJctRequestFilter) задаем `common/rds-authz-opa.location.conf`

### Использование аутентификации direct-auth

Аутентификации direct-auth, это аутентификация по данным из заголовка запроса (обычно по заголовку `Authorization`), и
предназначена для использования из утилит, которые не поддерживают аутентификацию по OIDC (например maven), но умеют
`HTTP Authentication` (например `Basic Authentication`, подробнее см. RFC7235).

При этом типе аутентификации не используется сессия IAM Proxy, токены IAM Proxy получает за один back-запрос к
провайдеру (пользователь аутентифицируется на провайдере по данным из заголовка `Authorization`). Полученные токены
сохраняются в локальном кеше на IAM Proxy (с ключом основанном на данным из заголовка `Authorization`), и
переиспользуются при всех последующих вызовах, а при необходимости обновляются.

Диаграмму аутентификации используя direct-auth смотрите в
[Диаграммы последовательностей](../architecture/sequence-diagram.md).

Для включения аутентификации direct-auth на всех ответвлениях, необходимо задать в env-параметрах контейнера
`OIDC_DIRECT_AUTH_ENABLED` в `True`. Если надо точечно включить аутентификации direct-auth, то на необходимых
ответвлениях надо добавить в `applyJctRequestFilter` опцию `common/rds-use-direct-auth.location.conf`.

Через параметр контейнера `OIDC_DIRECT_AUTH_HEADER` можно настроить, из какого HTTP заголовка запроса брать
аутентификационные данные пользователя. Обычно этот параметр задавать не нужно, большинство "толстых-клиентов"
используют HTTP заголовок `Authorization`, который используется по умолчанию.

В случае ожидания "толстым-клиентом" предварительного HTTP заголовка `WWW-Authenticate` от IAM Proxy, его можно включить
через параметр контейнера `OIDC_DIRECT_AUTH_CHALLENGE_TEXT`, задав в нем `challenge`
(пример для "Basic Authentication": `Basic realm="PlatformAuth Users`). HTTP заголовок `WWW-Authenticate` будет
добавляться в ответ при отсутствии в запросе HTTP заголовка `Authorization` или при ошибке аутентификации direct-auth (
например если не удалось получить токены от провайдера). Если будет задан `OIDC_DIRECT_AUTH_CHALLENGE_TEXT`, то на
ответвлениях где включен direct-auth стандартную аутентификацию по OIDC использовать не получится.

### Параметры RDS Client

- `RDS_SERVER_URLS` - указывается список RDS-серверов через ";", используется клиентская
  failover-балансировка. "https://mycompany-auth-svc-idp1-dev2.ca.mycompany.ru:7443/rds-for-proxy/active-conf-json;
  http://mycompany-auth-svc-idp1-dev2.ca.mycompany.ru:7080/rds-for-proxy/active-conf-json";
- `READ_DATA_FROM_RDS_TIME_IN_SEC` - частота опроса RDS-серверов на наличие изменений;
- `RDS_KEYSTORE` - из этой базы берется клиентский сертификат для mTLS, и доверенные ЦС;
- `RDS_CLIENT_KEYALIAS` - клиентский сертификат для mTLS;
- `RDS_CLIENT_KEYSTOREPASSWORD` - пароль к RDS_KEYSTORE;
- `RDS_START_CONF` - первоначальная конфигурация ответвлений на прокси в формате JSON (при задании данного параметра
  каталог `/opt/iamproxy/conf/jct/` будет очищен).

Пример:

```json
{
  "junctions": [
    {
      "junctionPoint": "/my-portal",
      "indexUrl": "/my-portal/start-page",
      "junctionName": "Мой портал",
      "description": "Расширенное описание подсистемы [Мой портал]",
      "https": true,
      "sslCommonName": "my.apps.company.ru",
      "transparent": true,
      "applyJctRequestFilter": "ssl-sni-on, set-header-host-to-backend",
      "serverAddresses": [
        "my.apps.company.ru:443"
      ]
    },
    {
      "junctionPoint": "/jct-snoop",
      "indexUrl": "/snoop/snoop/xxxxxx",
      "junctionName": "Тестовые ответвления/Диагностика (snoop непрозрачный)",
      "description": "Сервис диагностики (snoop непрозрачный)",
      "https": true,
      "sslCommonName": "*",
      "transparent": false,
      "applyJctRequestFilter": "set-header-host-to-backend",
      "serverAddresses": [
        "snoop:8443"
      ]
    },
    {
      "junctionPoint": "/jct-snoop-admin",
      "indexUrl": "/snoop/snoop/admin",
      "junctionName": "Тестовые ответвления/Диагностика с авторизацией по роли (snoop непрозрачный)",
      "description": "Сервис диагностики с авторизацией по роли (snoop непрозрачный)",
      "https": true,
      "sslCommonName": "*",
      "transparent": false,
      "authorizeByRoleTemplate": "my_snoop_*",
      "serverAddresses": [
        "snoop:8443"
      ]
    },
    {
      "junctionPoint": "/snoop",
      "indexUrl": "/snoop/snoop/yyyyyy/",
      "junctionName": "Диагностика (snoop прозрачный)",
      "description": "Сервис диагностики (snoop прозрачный)",
      "https": false,
      "transparent": true,
      "serverAddresses": [
        "127.0.0.1:10080"
      ]
    }
  ]
}
```

> Примечание:
> Endpoint по `<RDS_SERVER_URLS>/active` отдает json в таком же виде/формате, как указано выше в примере,
> и при необходимости это дает возможность реализовать свой сервис по формированию junctions динамически,
> и менять маршруты проксирования на IAM Proxy в реальном времени.

Описание атрибутов в `junctions` смотрите в
разделе [Заполнение раздела с описанием параметров ответвлений](installation.md).

### Авторизация на прокси по JWT-токену из заголовка (aka OAuth2 Authorization on resource)

- `OIDC_NEED_AUTH` - [true|false] Задание необходимости прохождения аутентификации по OIDC;
- `AUTHZ_BY_OAUTH_JWT_ENABLED` - [true|false] Проверять токен из запроса, в случае отсутствия или ошибок проверки будет
  403;
- `AUTHZ_BY_OAUTH_JWT_PUBLIC_KEY` - Открытый ключ или сертификат для проверки подписи токена (в PEM, включая head/foot);

## Использование контейнера с корневой системой в ReadOnly

Необходимо по пути `$TMP_PATH` (`/tmp/platformauth-proxy`) смонтировать файловую систему доступную на запись (например с
типом `EmptyDir`). При старте контейнера в этом каталоге будут размещены все файлы и каталоги, в которые необходимо
делать запись.

Для переключения корневой файловой системы контейнера в ReadOnly задается в Deployment (`kind: Deployment`)
атрибут `spec.template.spec.containers[0].securityContext.readOnlyRootFilesystem: true`

## Использование проксирования по корневому контексту

При необходимости проксировать все запросы начиная с корня, необходимо добавить ответвление с пустым `junctionPoint`, и
с `transparent` равным `true`.

Пример:

```json
{
  "junctions": [
    {
      "junctionPoint": "",
      "indexUrl": "/start-page",
      "junctionName": "Мой портал",
      "https": true,
      "sslCommonName": "my-app-ui.apps.company.ru",
      "transparent": true,
      "applyJctRequestFilter": "ssl-sni-on, set-header-host-to-backend",
      "serverAddresses": [
        "my-app-ui.apps.company.ru:443"
      ]
    }
  ]
}
```

Все остальные endpoint IAM Proxy будут так-же, как и ранее, доступны по тем же URI.

Стартовая техническая страница IAM Proxy будет доступна по URI `https://<host:port>/proxy/`
, `https://<host:port>/proxy/`.

Версию IAM Proxy можно узнать по URI `https://<host:port>/proxy/version`.

## Использование активного healthcheck

В случае необходимости вывода нерабочих узлов ответвлений из балансировки по результату HTTP вызова, реализована
функциональность периодического вызова серверов бэкенд и по результатам изменение их статуса в пуле балансировки IAM
Proxy (активный healthcheck).

Для использования функциональности активного healthcheck на IAM Proxy, его необходимо включить задав в параметре
`PROXY_HEALTHCHECK_ENABLE` значении `true`. Далее, для каждого ответвления активный healthcheck включается и
настраивается индивидуально через параметры ответвления (параметры приведены в
разделе [Заполнение раздела с описанием параметров ответвлений (junctions)](installation.md)).

Пример включения активного healthcheck по конкретному ответвлению:

```json
{
  "junctions": [
    {
      "junctionPoint": "",
      "indexUrl": "/start-page",
      "junctionName": "Мой портал",
      "https": true,
      "sslCommonName": "my-app-ui.apps.company.ru",
      "transparent": true,
      "healthcheck": {
        "enabled": true,
        "path": "/actuator/healthcheck",
        "interval": "30000"
      },
      "applyJctRequestFilter": "ssl-sni-on, set-header-host-to-backend",
      "serverAddresses": [
        "my-app-ui.apps.company.ru:443"
      ]
    }
  ]
}
```

## Передача секретов через файлы вместо переменных окружения

Для чтения содержимого параметра с секретом из файла необходимо в нужном параметре окружения задать символ `@` или путь
до файла в виде `@/path/file`. В первом случае, файл будет использоваться с именем переменной из каталога `/secrets`(
например `/secrets/PROXY_SESSION_SECRET`).

Допустимые имена параметров для передачи значений из файлов:

- `PROXY_SESSION_SECRET`;
- `PROXY_OIDC_CLIENT_ID`;
- `PROXY_OIDC_CLIENT_SECRET`;
- `OIDC_CLIENT_RSA_PRIVATE_KEY`;
- `AUTHZ_SPAS_SECRET`;
- `AUTHZ_BY_OAUTH_JWT_PUBLIC_KEY`;
- `RDS_CLIENT_KEYSTOREPASSWORD`.

Примечание: Пробелы и переводы строк по концам будут отброшены после чтения из файла.

Для обеспечения данной логики в OSE/k8s можно создать секрет (kind: Secret), добавить в него все необходимые параметры с
секретами, а потом его смонтировать в каталог `/secrets`. Пример:

```yaml
kind: Secret
apiVersion: v1
metadata:
  name: iamproxy-secrets
stringData:
  RDS_CLIENT_KEYSTOREPASSWORD: "myPA***ord_"
  TEST_VALUE: "myTest_"
  PROXY_SESSION_SECRET: "eEphMlo4VWxLN1owZVYwSmZQbzFLO...."
  OIDC_CLIENT_RSA_PRIVATE_KEY: |
    -----BEGIN PRIVATE KEY-----
    MIIEvgIBADANBgkqhkiG9w0BAQEFAASCBKgwggSkAgEAAoIBAQCOfXFnv2XWPEpX...
    -----END PRIVATE KEY-----
type: Opaque
```

```yaml
kind: Deployment
apiVersion: apps/v1
spec:
  selector:
    matchLabels:
      app: iamproxy
  template:
    spec:
      volumes:
        - name: volume-secrets
          secret:
            secretName: iamproxy-secrets
            defaultMode: 277
      #........
      containers:
        - name: iamproxy-test
          volumeMounts:
            - name: volume-secrets
              mountPath: /secrets
          env:
            - name: PROXY_SESSION_SECRET
              value: '@'
            - name: OIDC_CLIENT_RSA_PRIVATE_KEY
              value: '@'
            - name: RDS_CLIENT_KEYSTOREPASSWORD
              value: '@'
  #........
```

В результате при старте контейнера получим 4 файла в каталоге `/secrets` :

- `/secrets/RDS_CLIENT_KEYSTOREPASSWORD`;
- `/secrets/TEST_VALUE`;
- `/secrets/PROXY_SESSION_SECRET`;
- `/secrets/OIDC_CLIENT_RSA_PRIVATE_KEY`.

Значения из этих файлов будут переданы в соответствующие параметры контейнера `IAM Proxy`.

## Использование альтернативного client-id при работе с провайдером

Для организации аутентификации на отдельных ответвлениях по отдельному flow аутентификации (например для использования
дополнительных факторов аутентификации, для более привилегированных ресурсов), есть возможность использования
альтернативного client-id на нужных ответвлениях. Альтернативный client-id задается в параметре
`PROXY_OIDC_CLIENT_ID_ALT`, а секрет к нему в `PROXY_OIDC_CLIENT_SECRET_ALT` (или при получении секретов из
Vault/SecMan, в ключах `PROXY_OIDC_CLIENT_ID_ALT` и `PROXY_OIDC_CLIENT_SECRET_ALT`). На ответвлениях где требуется
использовать альтернативный client-id надо добавить в `applyJctRequestFilter` опцию
`common/rds-use-client-alt.location.conf`.

> Для альтернативного client-id возможно указать отдельно только client-secret. Клиентский сертификат и другие параметры
> обращения к провайдеру будут использоваться такие же, как и для основного client-id.
>
> Для перенаправления с корня на стартовую страницу приложения можно использовать параметр
> `PROXY_REDIRECT_FROM_ROOT_TO_URL`, в котором задается относительный url на контекст с отключенной аутентификацией.
>
> Для смены client-id в сессии, и для перехода из приложения по url c реаутентификацией можно использовать
> параметр `needReauthentification` (подробнее смотрите выше раздел "Принудительный запуск аутентификации заново").
> Таким образом можно например организовать переход из UI Пользователя в UI Администратора по ссылке/кнопке.

Диаграмму сценария аутентификации используя OIDC Code Flow с использованием альтернативного провайдера смотрите в
[Диаграммы последовательностей](../architecture/sequence-diagram.md).

При получении запроса на IAM Proxy идентифицируется соответствующий ему location, для location определяется
`client_id` + `client_secret` (через опции ответвления), далее при обработке используется выбранный `client_id`. При
аутентификации используется `client_id` ответвления, а при принятии решения по существующей аутентификации на IAM Proxy
сравнивается `client_id` ответвления c полями ID token `azp` (при его наличии) и `aud`. При отсутствии совпадения с
`azp+aud` будет отказ по HTTP запросу (если в `aud` массив, то проверка совпадения с одним из элементов).

При переходе на ответвление с типом клиента, отличным от того, по которому прошли аутентификацию, будет стандартное
сообщение об ошибке аутентификации (с кодом 401 на API запросах, или с кодом 406 на UI запросах).

> Разделение на уровне мониторинга и журналирования событий отсутствует. При необходимости определения разделения,
> определение доступно по url, то есть использовать разделение на уровне ответвлений.

## Использование альтернативного OIDC провайдера

На IAM Proxy есть возможность при недоступности основного OpenID Connect провайдера, в автоматическом режиме временно
переключаться на аутентификацию в альтернативном провайдере. При этом текущая аутентификация у пользователей не пропадет 
моментально, и будет действовать пока действительны токены (выданные ранее провайдером).     

При не пустой опции `OIDC_ALT_IDP_DISCOVERY_URL` включается функциональность использования альтернативного OpenID
Connect провайдера (который используется в случае недоступности основного провайдера). При этом IAM Proxy будет 
отправлять периодически запрос по HTTP URL до метаданных основного провайдера, чтобы проверить его состояние/доступность.
В случае недоступности основного провайдера, аутентификация начинает временно выполняться на альтернативном провайдере,
до восстановления работоспособности основного.

Диаграмму сценария аутентификации используя OIDC Code Flow с использованием альтернативного провайдера смотрите в
[Диаграммы последовательностей](../architecture/sequence-diagram.md).

В случае необходимости, при подключении к API альтернативного провайдера можно использовать клиентский сертификат
(клиентский сертификат один, используемый как для альтернативного провайдера, так и для основного, так и для вызова
бэкенд по ответвлениям).

> Обязательным условием работы проверки доступности провайдера, является включение параметра `PROXY_HEALTHCHECK_ENABLE`.
>
> Важно! Без включения этого параметра, проверка доступности провайдера работать не будет.

> Количество неуспешных запросов для определения провайдера неработоспособным, задается в параметре
> `OIDC_IDP_HEALTHCHECK_FALL`.
>
> Подробное описание параметров, а также параметры настройки для VM, приведены в разделе
> [Соответствие имен параметров для разных сред и инструментов установки](params.md).

Альтернативный провайдер имеет настройки, аналогичные настройкам основного провайдера. Описание всех доступных
параметров для настройки данной функциональности приведено в разделе
[Соответствие имен параметров для разных сред и инструментов установки](params.md).

Если IAM Proxy переключился на альтернативный провайдер, то новые аутентификации перенаправляются на него, а запросы с
просроченной аутентификацией и требующие обращения к основному провайдеру для обновления сессии IAM Proxy, приводят к
полной реаутентификации пользователя на альтернативном провайдере.

> При переключении на альтернативный провайдер, и обратно, выполняется запись об этом в лог.
>
> Пример:
> ```
> 2025/03/17 16:51:15 [warn] 451#451: *593 [lua] healthcheck.lua:211: peer_fail(): peer auth.mycompany.ru:443 is turned down after 2 failure(s), context: ngx.timer
> ```
> Тут пример перехода основного провайдера в недоступное состояние, что приводит к использованию альтернативного провайдера.

> Состояние доступности основного провайдера, и статистику использования конкретного провайдера можно получить через
> метрики IAM Proxy.
>
> Пример:
> ```
> curl 127.0.0.1:10080/metrics/
> pstream_state{upstream="oidc_idp",upstream_server="",is_backup="false"} 1
> ```
> Тут метрика с `upstream="oidc_idp"` со значением `1` означает, что основной провайдер доступен. В случае его
> недоступности будет `0`.

После восстановления доступности основного провайдера, происходит переключение IAM Proxy на основной провайдер, что
спустя какое-то время (не более времени жизни access-токена) приведет к реаутентификации пользователя на основном
провайдере (при условии, что пользователь ранее аутентифицировался на альтернативном провайдере).

### Параметры подключения к альтернативному OIDC провайдеру

- `OIDC_ALT_IDP_DISCOVERY_URL` - Опциональный. Задание URL метаданных OIDC альтернативного IDP;
- `OIDC_ALT_IDP_SCOPE` - Опциональный. Дополнительные scope OIDC при аутентификации на альтернативном IDP;
- `OIDC_ALT_IDP_LOGOUT_URI` - Опциональный. URI logout на альтернативном IDP;
- `PROXY_OIDC_ALT_IDP_CLIENT_ID` - `CLIENT_ID` используемый на прокси при аутентификации на альтернативном IDP;
- `PROXY_OIDC_ALT_IDP_CLIENT_SECRET` - Секрет к `CLIENT_ID` при аутентификации на альтернативном IDP;
- `OIDC_IDP_HEALTHCHECK_URI` - Опциональный. URL проверки доступности IDP;
- `OIDC_IDP_HEALTHCHECK_INTERVAL` - Опциональный. Частота вызова URL проверки доступности IDP в миллисекундах;
- `OIDC_IDP_HEALTHCHECK_TIMEOUT` - Опциональный. Timeout подключения при проверке доступности IDP в миллисекундах (по
  умолчанию 2000);
- `OIDC_IDP_HEALTHCHECK_FALL` - Опциональный. Количество неуспешных вызовов подряд к URL проверки доступности IDP (по
  умолчанию 2);
- `OIDC_IDP_HEALTHCHECK_RISE` - Опциональный. Количество успешных вызовов подряд к URL проверки доступности IDP (по
  умолчанию 2);
- `OIDC_IDP_HEALTHCHECK_VALID_STATUS` - Опциональный. Проверка доступности IDP (по умолчанию 200,204,302).

> Примечание:
>
> Полный перечень параметров подключения к альтернативному OIDC провайдеру на VM приведен в разделе
> [Соответствие имен параметров для разных сред и инструментов установки](params.md).

## Использование черного-списка для HTTP заголовка `User-Agent`

Для ограничения доступа к сервису по HTTP заголовку `User-Agent`, необходимо включить использование черного-списка,
задав параметр `PROXY_BLACKLIST_ENABLE` в `True`.

По умолчанию на IAM Proxy уже имеется список наиболее распространенных сканеров и роботов, который можно найти по пути
`/opt/iamproxy/conf/blacklist/blacklist-useragent.txt`. Если имеется свой уникальный черный-список строк, то его нужно
разместить по пути `/iamproxy/conf/blacklist/blacklist-useragent.txt` (используя монтирование config-map в файл), и
указать путь к этому файлу задав его в параметре `PROXY_BLACKLIST_USERAGENT_FILE`.

Так же можно задать отдельные правила проверки HTTP заголовка `User-Agent` на принадлежность к черному или белому
списку. Файлы с правилами белого списка размещаются по пути
`/iamproxy/conf/custom.d/whitelist-useragent*.map-rules.conf`, с правилами черного списка
`/iamproxy/conf/custom.d/blacklist-useragent*.map-rules.conf`. Правило в результате возвращающее 0, будет означать
принадлежность к белому списку, а которое возвращает 1, будет означать принадлежность к черному списку (белые списки
имеют приоритет над черными, и в итоге правила включаются в секцию `map`
по проверке HTTP заголовка `User-Agent`, подробнее
в [описании секции map](https://nginx.org/ru/docs/http/ngx_http_map_module.html#map)).

Пример файла с правилами `/iamproxy/conf/custom.d/whitelist-useragent-my-agents.map-rules.conf`:

```
# Мобильное приложение АС
"~^myApp/[0-9.,]+"  0;
```

Пример файла с правилами `/iamproxy/conf/custom.d/blacklist-useragent-ddos.map-rules.conf`:

```
# DDOS агент
"~*^Ripper/.*"  1;
```

> **В случае установки на VM**
>
> Для примеров выше, в случае с VM, файлы можно доставить до серверов при установке, разместив их в профиле установки
> по путям:
> - `ansible/inventories/StandX/files/proxy/conf/blacklist/blacklist-useragent.txt`
> - `ansible/inventories/StandX/files/proxy/conf/custom.d/whitelist-useragent-my-agents.map-rules.conf`
> - `ansible/inventories/StandX/files/proxy/conf/custom.d/blacklist-useragent-ddos.map-rules.conf`

## Использование gRPC в ответвлениях

Для функционирования gRPC необходимо активировать поддержку протокола HTTP/2 с помощью параметра `PROXY_HTTP2_ENABLE`.

### Рекомендации по конфигурации IAM Proxy при использовании gRPC

| Параметр                              | Описание                                                                                                   | Рекомендуемые диапазон значений | Критерии выбора                                                                                                                                                                                                                                                                               |
|:--------------------------------------|:-----------------------------------------------------------------------------------------------------------|:--------------------------------|:----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `PROXY_KEEPALIVE_BACKEND_CONNECTIONS` | По умолчанию отключен. Максимальное число неактивных постоянных соединений до бэкенд.                      | 32 – 256                        | Значение ниже 32 приводит к частым повторным TLS-рукопожатиям между прокси и upstream-сервисом. Значение значительно выше 256 увеличивает объем потребления оперативной памяти и число открытых файловых дескрипторов и не приводит к значимому улучшению.                                    |
| `PROXY_KEEPALIVE_BACKEND_TIMEOUT`     | По умолчанию `60s`. Тайм-аут неактивного постоянного соединения до бэкенд.                                 | 60 – 300 секунд                 | Тайм-аут должен быть не меньше интервала, с которым gRPC-клиенты посылают HTTP/2-PING. Иначе прокси закроет соединение в момент, когда клиент ожидает, что соединение еще активно.                                                                                                            |
| `PROXY_KEEPALIVE_BACKEND_REQUESTS`    | По умолчанию `1000`. Максимальное количество запросов через одно TCP-соединение к бэкенд.                  | 1000 – 10000                    | Значения в этом диапазоне позволяют провести несколько полных заполнений лимита одновременных потоков HTTP/2 через одно соединение и при этом регулярно освобождать накопленное состояние. Установка значения 0 (бесконечность) повышает риск роста потребления памяти при длительной работе. |
| `PROXY_HTTP2_MAX_CONCURRENT_STREAMS`  | По умолчанию `128`. Максимальное количество параллельных потоков (streams) в одном HTTP/2-соединении.      | 128 — 1024                      | Значение 128 достаточно для большинства рабочих нагрузок, когда на одного клиента приходится около 100 параллельных вызовов. Повышение параметра выше 1024 увеличивает нагрузку на таблицы потоков и внутренние структуры данных SynGX, что может привести к росту задержек.                  |

> **ВАЖНО**
>
> `PROXY_KEEPALIVE_BACKEND_REQUESTS ÷ PROXY_HTTP2_MAX_CONCURRENT_STREAMS ≥ 4` - обеспечивает не менее четырех
> полных заполнений лимита одновременных потоков HTTP/2 до закрытия соединения и снижает вероятность разрывов
> под нагрузкой и позволяет регулярно освобождать состояние.

**Пример настройки ответвлений с gRPC**:

```json
{
  "junctions": [
    {
      "junctionName": "Сервис gRPC (grpc.my-app.App-Service)",
      "description": "Пример проксирования gRPCs трафика с аутентификацией по имени сервиса",
      "protocol": "grpc",
      "transparent": true,
      "applyJctRequestFilter": "common/rds-proxy-buffering-off.location.conf",
      "junctionPoint": "/grpc.my-app.App-Service",
      "serverAddresses": [
        "grpc.mycompany.ru:9443"
      ]
    },
    {
      "junctionName": "Сервис gRPC-Web",
      "description": "Пример проксирования gRPC-Web трафика с аутентификацией с базовым URI",
      "junctionPoint": "/grpc-web",
      "serverAddresses": [
        "grpc-web.mycompany.ru:9443"
      ]
    },
    {
      "junctionName": "Сервис gRPC без аутентификации (plaintext)",
      "description": "Пример проксирования gRPC трафика без TLS с базовым URI",
      "protocol": "grpc",
      "transparent": true,
      "applyJctRequestFilter": "common/oidc-unauth-access.location.conf, common/rds-proxy-buffering-off.location.conf",
      "junctionPoint": "/grpc",
      "https": false,
      "serverAddresses": [
        "grpc.mycompany.ru:9080"
      ]
    },
    {
      "junctionName": "gRPC Сервер без аутентификации",
      "description": "Пример проксирования gRPCs трафика с базовым URI",
      "protocol": "grpc",
      "transparent": true,
      "applyJctRequestFilter": "common/oidc-unauth-access.location.conf, common/rds-proxy-buffering-off.location.conf",
      "junctionPoint": "/grpcs",
      "serverAddresses": [
        "grpc.mycompany.ru:9443"
      ]
    }
  ]
}
```

> Настройка конфигурации одинакова для всех вариантов развертывания (как на VM, так и в среде контейнеризации).
> Для настройки конфигурации в k8s с использованием PV DOT CDJE необходимо заполнить параметры в файлах
> `<репозиторий подсистемы>/<директория конфигурации>/conf/config/parameters/auth.*.conf` и
> `<репозиторий подсистемы>/<директория конфигурации>/conf/custom_property.conf.yml`.
>
> Параметры настройки на VM, задаются в файле параметров `ansible/inventories/StandX/group_vars/all/main.yml`
> или в `ansible/inventories/StandX/group_vars/proxy.yml`.
>
> Полный перечень и описание параметров настройки IAM Proxy приведен
> в разделе [Соответствие имен параметров для разных сред и инструментов установки](params.md).

### Требования к gRPC-Native клиенту при работе через IAM Proxy с аутентификацией

На клиенте необходимо поддержать минимум один из режимов работы:

- Session Cookie (аналог использования `Cookie` в браузерах)
  - отправлять сессионные `Cookie` во всех gRPC запросах
    (отправлять в метаданных `Cookie: PLATFORM_SESSION=<…>; PLATFORM_SESSION_2=<…>; ...`). По умолчанию сессионные
    cookie, это все cookie имена которых начинаются с "PLATFORM_SESSION";
  - обновлять значение в сессионных `Cookie` при получении `Set-Cookie` в метаданных ответа gRPC.
- Direct-Auth (аналог использования `HTTP Authentication` в браузерах)
  - в каждый gRPC запрос добавлять в метаданных поле `Authorization: <scheme> <credentials>`
    (поддерживаемые `<scheme>` и `<credentials>` определяются на стороне IDP-провайдера);
  - после прохождения OIDC повторять исходный RPC с тем же `Authorization` (прокси возьмет токен из кеша).

В клиентской реализации необходимо предусмотреть, что может произойти разрыв gRPC соединения, и по необходимости делать
переподключение на длительных соединениях. Так же на длительных соединениях рекомендуется принудительно делать
авто-переподключение спустя время, равное времени жизни access-токена.

### Поведение при отсутствии сессии/токена

- **Прокси не возвращает `grpc-status`, а только HTTP-статус и заголовки.**
- клиент использует стандартный маппинг HTTP->gRPC своей библиотеки (напр., `401 -> UNAUTHENTICATED`);
- дополнительно к стандартному маппингу:
  - Любой **`HTTP 30x`** трактовать как **`gRPC status 16 (UNAUTHENTICATED)`**. Если в `30x` или `401` присутствует
    **`Location: <url>`**:
    1. перейти по `Location` (браузер/WebView) и пройти OIDC-аутентификацию;
    2. ровно один раз повторить исходный RPC (в режиме Direct-Auth или Session Cookie).
  - Если `Location` отсутствует — завершить вызов ошибкой **`UNAUTHENTICATED`**.
  - При повторном `HTTP 401` после повтора — вернуть ошибку приложению (`UNAUTHENTICATED`).

Диаграмму последовательности процесса аутентификации gRPC-Native клиента (используя direct-auth) смотрите в
[Диаграммы последовательностей](../architecture/sequence-diagram.md).

## Использование IAM Proxy в разных кейсах

Примеры и различные кейсы с примерами использования можно найти в
разделе [Использование IAM Proxy](proxy-deploy-docker-cases.md).

Более полное описание, состав и примеры задания параметров смотрите в разделах
[Соответствие имен параметров для разных сред и инструментов установки](../installation-guide/params.md) и
[Установка](../installation-guide/installation.md).


