# Параметры для настройки RDS Server

## Введение

Раздел содержит описание конфигурации RDS Server, в части данных по ответвлениям для IAM Proxy. Конфигурация может быть
получена одновременно из разных источников (локальный yaml-файл, yaml по REST API, DB/PACMAN), которые задаются в
параметре `rds.configSources.sources`
(подробнее смотрите в разделе [Настройка получения конфигурации](../additional-materials/rds-server-get-conf.md)).

## Пример конфигурации

```
ZoneConfig[Core]:
  JunctionConfig[RDSServer]:
    junctionName: "RDS Server"
    junctionPoint: /rds-for-proxy
    indexUrl: /rds-for-proxy/index.html
    transparent: 'true'
    https: 'true'
    sslCommonName: '.rds-ift2.mycompany.ru'
    applyJctRequestFilter: ''
    serverAddresses:
    - node1.rds-ift2.mycompany.ru:9443
    - node2.rds-ift2.mycompany.ru:9443
  JunctionConfig[Snoop]:
    junctionName: Тестовые сервисы/Snoop (тестовое приложение)
    junctionPoint: /jct-snoop
    indexUrl: /snoop/snoop/snoop/
    sslCommonName: '*'
    https: "false"
    transparent: "false"
    serverAddresses:
    - 127.0.0.1:10080
  Zone[offline]:
    Junction[Snoop]:
      serverAddresses: []
    Junction[RDSServer]:
      serverAddresses: []
  Zone[standin]:
    Junction[Snoop]:
      serverAddresses:
      - 127.0.0.2:10080
    Junction[RDSServer]:
      serverAddresses:
      - node3.rds-ift2.mycompany.ru:9443
  zoneNameStandin: standin
  zoneNameOffline: offline
checkConfigurationFrequency: 10
checkStandInFrequency: 10
httpClientMaxPoolSize: 1
manualMode: "true"
transportRequestTimeout: 9
usePlatformSemaphore: "true"
```

## Общий пример заполнения параметров:

```
ZoneConfig[<tenant name>]:
  JunctionConfig[<junction name>]:
    junctionName: "<someFolder> / <someJunction>"
    junctionPoint: /<URL base path>
    indexUrl: /<URL path>/
    transparent: <Boolean>
    https: <Boolean>
    sslCommonName: '<FQDN or SAN or *>'
    applyJctRequestFilter: "<applying options, separated by comma>"
    serverAddresses:
      - <FQDN:port>
      - <IP:port>
      - …
  JunctionConfig[<junction name>]:
    …
  …
  Zone[<zone name>]:
    Junction[<name>]:
      serverAddresses:
        - <FQDN:port>
        - …
  Zone[<zone name>]:
    …
  …
  zoneNameStandin: <zone name>
  zoneNameOffline: <zone name>

checkConfigurationFrequency: <Int>
checkStandInFrequency: <Int>
httpClientMaxPoolSize: <Int>
manualMode: <Boolean>
transportRequestTimeout: <Int>
usePlatformSemaphore: <Boolean>
```

## Группа ZoneConfig

Группа `ZoneConfig[<tenant name>]` содержит основные параметры конфигурации IAM Proxy.
`<tenant name>` - это имя тенанта/региона в PACMAN, к которому подключен RDS Server. Значение тенанта может отличаться в
зависимости от стенда.

### Элемент ZoneConfig/JunctionConfig

Элемент `ZoneConfig/JunctionConfig[<junction name>]` содержит параметры ответвления (`junction`).
`<junction name>` - это имя элемента ответвления на RDS Server, по нему производится упорядочивание элементов на RDS
Server (сортировка по возрастанию кодов символов).

Список основных параметров (тут не все возможные параметры), и их краткое описание:

| Параметр                   | Значение по умолчанию | Описание                                                                                                         | Пример                                                   |
|----------------------------|-----------------------|------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------|
| junctionName  `***`        | Отсутствует           | Название ответвления                                                                                             | Система АС1/Консоль администрирования АС1                |
| description                | Отсутствует           | Описание ответвления                                                                                             | Страница предназначена для администрирования системы АС1 |
| junctionPoint `*` `***`    | Отсутствует           | Корневой контекст запросов, по которому будет определяться принадлежность запроса к конкретной подсистеме/бэкенд | /company                                                 |
| indexUrl  `***`            | Отсутствует           | URL относительно корня на бэкенд, по которому осуществляется основной вход в UI подсистемы на ответвлении        | /admin                                                   |
| transparent                | False                 | Параметр определяет необходимость изменения URL при проксировании                                                | True                                                     |
| https                      | True                  | Если True, то при подключении по HTTP к бэкенд использовать TLS                                                  | False                                                    |
| sslCommonName              | .mycompany.ru         | Шаблон/значение fqdn для проверки SAN сертификата серверов бэкенд                                                | .my-company2.ru                                          |
| serverAddresses[]          | Отсутствует           | Список серверов бэкенд, для осуществления проксирования в рамках данного ответвления                             | ["10.X.X.1:9443", "abs-4.mycompany.mycompany.ru:8443"]   |
| applyJctRequestFilter `**` | Отсутствует           | Применить опции для запросов по ответвлению (несколько опций указываются через через запятую)                    | set-header-host-to-backend , ssl-sni-on                  |

> Полный список параметров ответвления, и их детальное описание смотрите в
> разделе [Установка/Заполнение раздела с описанием параметров ответвлений (junctions)](../installation-guide/installation.md)

Примечания к таблице:

`*` - обязательный параметр.

`**` - Полный набор значений для `applyJctRequestFilter` приведен в разделе
[Установка](../installation-guide/installation.md).

`***` - Параметры `junctionName`, `indexUrl` и группировка ответвлений никак не влияют на сам процесс проксирования, и
используются для формирования ссылок на стартовой странице IAM Proxy (на `https://iamproxy.mycompany.ru/proxy/`).

### Элементы ZoneConfig/zoneNameStandin и ZoneConfig/zoneNameOffline

Параметры `ZoneConfig/zoneNameStandin` и `ZoneConfig/zoneNameOffline` задают, какая зона из конфигурации будет
использоваться при состоянии Standin, а какая при состоянии Offline.

| Параметр        | Значение по умолчанию | Описание                                                                                                                                                                                                                                     | Пример |
|:----------------|:----------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------|
| zoneNameStandin | standin               | Параметр предназначен для задания имени зоны, которая будет применена в состоянии Standin.                                                                                                                                                   | zone1  |
| zoneNameOffline | offline               | Параметр предназначен для задания имени зоны, которая будет применена в состоянии Offline. Состояние offline наступает, когда одновременно недоступны основная и StandIn зоны (например идет переключение на StandIn, или на основную зону). | zone2  |

Примечание:

Получение флагов состояния активности зоны (основной зоны и зоны Standin) производится на периодической из Прикладного
журнала.

### Группа ZoneConfig/Zone

Группа `ZoneConfig/Zone[<zone name>]` содержит параметры ответвлений, которые надо изменить при активности
зоны `<zone name>`.
`<zone name>` - имя элемента зоны на RDS Server.

| Параметр           | Значение по умолчанию | Описание                                                                           | Пример                                           |
|--------------------|-----------------------|------------------------------------------------------------------------------------|--------------------------------------------------|
| applyRequestFilter | Отсутствует           | При активности зоны применить указанные тут опции ответвлений ко всем ответвлениям | `common/rewrite-response-is-offline.server.conf` |

#### Группа ZoneConfig/Zone/Junction

Группа `ZoneConfig/Zone[<zone name>]/Junction[<junction name>]` предназначена для настройки конкретных ответвлений с
именами `<junction name>` при активности зоны `<zone name>`.

| Параметр          | Значение по умолчанию | Описание                                                              | Пример                                                 |
|-------------------|-----------------------|-----------------------------------------------------------------------|--------------------------------------------------------|
| serverAddresses[] | Отсутствует           | При активности зоны заменить список серверов бэкенд на указанный тут  | ["10.X.X.2:9443", "abs-5.mycompany.mycompany.ru:8443"] |

## Параметры в корневом контексте

Данная группа содержит параметры конфигурации работы RDS Server.

| Параметр                    | Значение по умолчанию | Описание                                                                                                                                                        | Пример |
|-----------------------------|-----------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|--------|
| checkConfigurationFrequency | 60                    | Частота проверки изменений в конфигурации (задается в секундах)                                                                                                 | 1      |
| checkStandInFrequency       | 10                    | Частота проверки флагов состояния контуров StandIn в ПЖ (задается в секундах)                                                                                   | 35     |
| manualMode                  | false                 | Включение в UI RDS Server режима ручного переключения на необходимый контур (true- разрешение смены контура вручную, false - запрет смены контура вручную в UI) | true   |
| usePlatformSemaphore        | false                 | Флаг использования платформенного семафора в ПЖ (true - использовать платформенный семафор, false - использовать прикладной семафор)                            | true   |

