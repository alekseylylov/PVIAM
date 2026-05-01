# Настройка интеграции с Platform V Audit SE (AUDT)/Platform V Monitor (COTE)

## Подключение клиентского модуля Аудит (AUDT)/Единый коллектор телеметрии (COTE) с подключением по REST

> Подключение компонента Единый коллектор телеметрии (COTE) осуществляется так же, как и с компонентом Аудит (AUDT).

Для интеграции с Platform V Audit SE (AUDT)/Platform V Monitor (COTE) по REST API необходимо настроить клиентский модуль прокси на стороне Audit (далее -
прокси-audit).

Пример настройки application-*.yml:

```yaml
audit:
  rest:
    enabled: true        # включение интеграции по REST API (в этом случае будет игнорироваться клиентский модуль Audit, использующий Kafka)
    module-name: my-rds-server-name      # необязательный параметр, имя модуля в событиях, отправляемых в API AUDT, значение по умолчанию: rds-server
    metamodel-location: my-audit.metamodel.json # необязательный параметр, положение json-файла с метамоделью прокси-audit, значение по умолчанию: classpath:audit.metamodel.json
    proxy-url: https://ext-http.audit2-http-proxy.apps.mycompany.ru   # URL-адрес до прокси-audit
    truststore:
      location: ssl/trust-store.p12   # путь до хранилища доверия, в котором хранятся SSL-сертификаты (создается на целевых серверах из файлов trusted_*.cer/trusted_*.crt.pem)
      password: notapassword123456ahFFHgiu52073250097KF@@95287   # пароль, используемый для доступа к хранилищу доверия
    keystore:
      location: server.keystore.p12   # путь к хранилищу ключей, в котором хранится клиентский сертификат/ключ
      password: notapassword123456ahFFHgiu52073250097KF   # пароль, используемый для доступа к хранилищу ключей
    key-password: notapassword123456ahFFHgiu52073250097KF # пароль, используемый для доступа к ключу в хранилище ключей
```

Заполнение параметров `truststore`, `keystore` и `key-password` является необязательным, если обращения в модуль
прокси-audit будут проходить по HTTP.

### Изменение имени модуля в событиях, отправляемых в API Аудит (AUDT)/Единый коллектор телеметрии (COTE)

Для возможности однозначной идентификации инсталляции IAM Proxy, из которой было получено событие в Аудит (AUDT),
задайте в параметре имени модуля идентификатор инсталляции (например название АС, использующей IAM Proxy).

Для изменения имени модуля (по умолчанию имя `RDS-server`) необходимо:

- если используется файл application-*.yml для настройки приложения, то задать в application-*.yml имя модуля в
  параметре `audit.rest.module-name`;
- если используются переменные окружения для настройки RDS Server (при этом по факту будет использован
  application-PROM.yml встроенный в jar), то задать в переменной окружения `rds.audit.rest.moduleName` имя модуля (
  подробнее в разделе [Описание настроек приложения RDS Server](rds-server-configuration.md)).

> Имя модуля из выше указанных параметров имеет приоритет, над именем модуля указанном в json-файле метамодели.

Размещение метамодели и описание атрибутов событий, передаваемых в Аудит (AUDT), приведено в разделе
[События передаваемые из IAM Proxy в Аудит (AUDT)](../security-guide/audit.md)

## Подключение клиентской библиотеки Аудит (AUDT)/Единый коллектор телеметрии (COTE) с подключением по Kafka

> Подключение компонента Единый коллектор телеметрии (COTE) осуществляется так же, как и с компонентом Аудит (AUDT).

Для активации интеграции с Platform V Audit SE (AUDT), необходимо задать параметры audit.kafka.

Пример настройки application-*.yml:

```yaml
audit:
  kafka:
    enabled: true
    main:
      server: 10.xx.xx.10:9092
    fallback:
      server: 10.xx.xx.11:9092
    enabled.mock: false
    security.protocol: PLAINTEXT
```

В случае использования SSL при подключении к kafka необходимо заполнить парамеры audit.kafka.ssl:

````yaml
audit:
  kafka:
    security.protocol: SSL
    ssl:
      protocol: TLSv1.3
      key.password: keyPass
      keystore:
        location: /keystore/location
        password: keystorePass
      truststore:
        location: /truststore/location
        password: truststorePass
````

Для возможности работы приложения при недоступности Аудит (AUDT), необходимо задать параметр:

```yaml
audit:
  runWithoutAudit: false
```

> Интеграция с Platform V Audit SE (AUDT) одновременно и по kafka и REST API не поддерживается!
> Нужно выбрать только одну - `audit.kafka.enabled=true`, либо `audit.rest.enabled=true`.
> Рекомендуется использовать интеграцию с Platform V Audit SE (AUDT).

> Для настройки id/name модуля в событии, отправляемом в Аудит (AUDT), нужно изменить параметр spring.
> application.name. Этот параметр задает "Название АС" и "Модуль" события.

RDS направляет только одно событие связанное с изменением зоны: "CHANGING_ZONE". Пример события из Kafka

```
{"v":4,
  "metamodelHash":"rds-server:13f83bc345b02f1073a1870fe613c9fc",
  "name":"CHANGING_ZONE",
  "uuid":"83c9c6b2-076b-468a-8f3c-55da929196a1",
  "module":"rds-server",
  "user":"UserForRDSForProxy39c4a61a-4a68-44c3-8a6d-e1dee102e8e2",
  "userFullName":"",
  "ticket":"TicketForRDSForProxy85249112-ee61-4f0b-b6f9-c201984261b4",
  "timestamp":1636448825316,
  "nodeId":"node1",
  "sourceSystem":"rds-server-system",
  "chainUUID":null,
  "truncatedBy":null,
  "operationUUID":"a003fc45-0294-4ad5-87a9-e9feff33b253",
  "parentOperationUUID":"a003fc45-0294-4ad5-87a9-e9feff33b253",
  "isSuccess":true,
  "params":[
    {"name":"zone.current-name","oldValue":null,"value":"default"},
    {"name":"zone.new-name","oldValue":null,"value":"offline"},
    {"name":"ip-address","oldValue":null,"value":"127.0.0.1"}
    ],
  "rootOperationUUID":"a003fc45-0294-4ad5-87a9-e9feff33b253",
  "eventType":"USER_EVENT"
 }
```