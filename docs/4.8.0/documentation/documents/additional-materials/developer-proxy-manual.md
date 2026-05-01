# Описание прокси сервера с аутентификацией (IAM Proxy)

## Описание

IAM Proxy, это прокси сервер с поддержкой аутентификации и базовой авторизации (реализован на базе SynGX).

> В основе SynGX используется Nginx с поддержкой Lua.
> Подробнее в документации на [Platform V SynGX](doc://SNX).

### Назначение компонента

- осуществление проксирования трафика от Пользователя/Браузера к бизнес-приложению на платформе;
- при необходимости аутентификации, выполнение роли OIDC RP, перенаправление на прохождение аутентификации в OIDC OP;
- обеспечение балансировки между серверами бэкенд.

### Поддерживаемая функциональность

- поддержка непрозрачных ответвлений (с добавлением дополнительного корневого контекста в URL);
- stateless-сессия;
- аутентификация по OIDC (функциональность RP). Поддерживаемые методы аутентификации: "client_secret_basic", ".
  private_key_jwt";
- авторизация по роли из ID токена OIDC;
- стартовая страница со списком проксируемых контекстов;
- версия конфигурации для работы в режиме L4 балансировщика;
- передача SNI в бэкенд;
- ограничение запросов по заголовкам User-Agent из черного списка.

## Endpoints

### Служебные

- информация о версии сервиса
  > [unauth]GET /proxy/version
  > [unauth]GET /proxy/version.txt (deprecated)

- состояние сервиса (отображение доступности и статистики прокси)
  > [unauth on 127.0.0.1:10080]GET /status/
  >
  > Можно использовать в readness/liveness/startup пробах в k8s.
  >
  > После `/status/` допускается указать любой дополнительный путь (например `/status/readinessProbe`).

- статус для startupProbe
  > [unauth on 127.0.0.1:10080]GET /status/startupProbe
  >
  > По этому endpoint будет выдаваться статус состояния запуска сервиса, код 200 (отдельный endpoint по CN 3.0).

- статус активного healthcheck
  > [unauth on 127.0.0.1:10080]GET /status/healthcheck
  > [unauth on 127.0.0.1:10080]GET /status-healthcheck/ (deprecated)
  >
  > По этому endpoint будет выдаваться статус состояния активного healthcheck, по тем ответвлениям, где healthcheck
  > включен. Имена в статусе соответствуют именам групп серверов в SynGX (специальное имя `oidc_idp` соответствует
  > статусу доступности провайдера аутентификации OIDC).

- статус для readinessProbe
  > [unauth on 127.0.0.1:10080]GET /status/readinessProbe
  >
  > По этому endpoint будет выдаваться состояние готовности принимать запросы, код 200 готов, код 503 не готов.

- статус для livenessProbe
  > [unauth on 127.0.0.1:10080]GET /status/livenessProbe
  >
  > По этому endpoint будет выдаваться статус работоспособности сервиса, код 200 (отдельный endpoint по CN 3.0).

- тестовый endpoint, отображающий заголовки и тело полученного HTTP-запроса
  > [unauth on 127.0.0.1:10080]GET /snoop/

- выполнить reload - перечитать конфигурацию с файловой системы, с плавным перезапуском воркер-процессов
  > [unauth on 127.0.0.1:10080]GET /reload/

- тестовый endpoint, получение текущих OIDC токенов
  > GET /proxy/jwt/[ ?refresh=1 | ?id=1 | ?access=1]
  >
  > **Присутствует только на стендах Dev и ИФТ. На ПРОМ должен быть выключен!**

- выполнить обновление OIDC токенов
  > GET /openid-connect-auth/refresh

- запустить реаутентификацию
  > GET /openid-connect-auth/reauth?redirect_uri=/my-app/cash/move/all?to=40666810100000000000

- универсальный health-эндпоинт (настраиваемый путь)
> [unauth]GET /health  
>
> Упрощенный endpoint для проверки жизнеспособности сервиса. Возвращает `HTTP 200 OK`, если IAM Proxy запущен и работает.

Путь до endpoint может быть изменен через параметр конфигурации `PROXY_HEALTH_URI_PATH`.  
По умолчанию значение — `/health`.

**Пример переопределения пути**:  
Если задано:  
`PROXY_HEALTH_URI_PATH=/check/live`  

Тогда альтернативный health-эндпоинт будет доступен по:  
> [unauth]GET /check/live

Ответ возвращается в формате JSON, структура:
```json
{
  "status": "ok",
  "service": "IAM Proxy",
  "timestamp": "2025-04-05T10:00:00Z"
}
```

Этот endpoint предназначен для использования внешними системами мониторинга и балансировщиками нагрузки.

### Метрики

- статистика IAM Proxy в формате Prometheus
  > [unauth on 127.0.0.1:10080]GET /metrics/
  > [unauth on 127.0.0.1:10080]GET /metrics (deprecated)

- статистика SynGX в формате Prometheus
  > [unauth on 127.0.0.1:10080]GET /status/prometheus

- статистика SynGX по использованию TLS-соединений и по сертификатам (в форматах Prometheus)
  > [unauth on 127.0.0.1:10080]GET /status/certificates_prometheus

- статистика SynGX по запросам в разрезе виртуальных серверов (в формате html,json,Prometheus)
  > [unauth on 127.0.0.1:10080]GET /status/vts
  > [unauth on 127.0.0.1:10080]GET /status/vts/format/json
  > [unauth on 127.0.0.1:10080]GET /status/vts/format/prometheus

### Мобильные приложения

- стандартная аутентификация с дополнительным scope `offline_access`
  > [unauth]GET /mobile/auth/

- получить текущий токен offline-token, отдаем если в токене имеется scope `offline_access`
  > GET /mobile/get-token/

- возобновить сессию по offline-token, а так же обновить токен ЕСИА
  ```
  [unauth]POST /mobile/restore-session/
  token=xxx
  ```

- завершить текущую сессию, при этом будет отозван offline-token из текущей сессии
  > GET /openid-connect-auth/logout-token/

## Тестовые примеры

### Стандартная аутентификация с дополнительным скоупом offline_access

[unauth]GET /mobile/auth/

```shell
curl -kisL https://platform-dev3.mycompany.ru/mobile/auth/ -c  my-cookie-jar |grep -Po '(?<=action=").*(?=")'
loginurl=$(curl -kis https://platform-dev3.mycompany.ru/mobile/auth/ -c my-cookie-jar -L|grep -Po '(?<=action=").*(?=")')
#urldecode
loginurl="${loginurl//&amp;/&}"
curl -kisL -c  my-cookie-jar -b  my-cookie-jar $loginurl -d username=admin -d 'password=Q!w2e3r4'
curl -ks https://platform-dev3.mycompany.ru/mobile/auth/ -c my-cookie-jar -b my-cookie-jar |jq
```

### Получить текущий токен offline-token, отдаем если в токене имеется скоуп offline_access

GET /mobile/get-token/

```shell
curl -ks https://platform-dev3.mycompany.ru/mobile/get-token/ -c my-cookie-jar -b  my-cookie-jar |jq
token=$(curl -ks https://platform-dev3.mycompany.ru/mobile/get-token/ -c my-cookie-jar -b my-cookie-jar |jq -r ".token")
```

### Возобновить сессию по offline-token, а так же обновить токен ЕСИА

```
[unauth]POST /mobile/restore-session/
token=xxx
```

```shell
#curl -kis https://platform-dev3.mycompany.ru/mobile/restore-session/ -c my-cookie-jar --data-binary @token-devb.txt
curl -kis https://platform-dev3.mycompany.ru/mobile/restore-session/ -c my-cookie-jar --data "token=$token"
```

### Завершить текущую сессию, при этом будет отозван offline-token

```shell
curl -kis https://platform-dev3.mycompany.ru/openid-connect-auth/logout-token/ -b my-cookie-jar
```
