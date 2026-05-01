# Настройка интеграции с Прикладным журналом (APLJ)

Прикладной журнал (APLJ) (далее - "Прикладной журнал") переключает сервисы на StandIn-контур и
отправляет актуальную конфигурацию RDS Server.

Для получения значения платформенного StandIn необходимо выполнить настройку.

## REST API

Необходимо заполнить параметры в секции standin.rest.

При установке RDS Server на VM для параметра `standin.rest.callbackUrl` (callback url для интеграции с ПЖ (APLJ) можно
использовать зарезервированные значения:

* `IP`   - Вычисляется IP хоста на котором установлен RDS Server. Контекст и порт вычисляется автоматически, подставлять
  в URL **не нужно**.
    * Пример использования: standin.rest.callbackUrl: IP
* `FQDN` - Вычисляется FQDN хоста на котором установлен RDS Server (при ошибке будет подставлен IP). Контекст и порт
  вычисляется автоматически, подставлять в URL **не требуется**.
    * Пример использования: standin.rest.callbackUrl: FQDN

При установке RDS Server на среду контейнеризации (например, OSE) для параметра `standin.rest.callbackUrl` необходимо
передать базовый URL (https://FQDN:PORT/CONTEXT) по которому можно будет вызвать API RDS Server из функциональной подсистемы Прикладной журнал (APLJ). Пример использования:
standin.rest.callbackUrl: https://ose-route.my.company.app/rds-for-proxy

Значение параметра `standin.rest.callbackUrl` по умолчанию = `IP`.

Пример настройки:

```yaml
standin:
  rest: # для получения значения платформенного StandIn по REST API
    enabled: true
    url: http://standin-arm-route.my.company.app/journal-arm/api/v1/standin/status/im   # URL для интеграции с ПЖ (APLJ)
    callbackUrl: https://my.company.app/rds-for-proxy   # URL для получения callback от ПЖ (APLJ)
    inStandIn: false   # определяет контур в котором находится RDS Server (true - StandIn, false - Normal)
  cloud:
    client:
      zoneId: TEST_ZONE
      stub: true       # при интеграции по REST **обязательно** нужно выставить stub = true
      sicl_refresh_state_period: 100
```

## Kafka

Параметры из секции standin.cloud используются для работы клиентского модуля "Прикладной журнал". В 4-ом поколении "
Прикладного журнала" осталась возможность получить значения *только* функционального StandIn.

```yaml
standin:
  cloud:
    client:
      zoneId: TEST_ZONE
      stub: false
      sicl_refresh_state_period: 100
    kafka:
      bootstrapServers: "10.x.x.11:9093"
      concurrency: 10
      groupId: group_1
      producer:
        "[some.ms]": 10
```