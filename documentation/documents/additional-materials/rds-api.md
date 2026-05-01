# Описание API для реализации динамической настройки IAM Proxy

Для обеспечения на IAM Proxy оперативного изменения поведения проксирования в сторону бэкенд, есть возможность
предоставлять для IAM Proxy данные о бэкенд через API (API RDS) в реальном времени.

> URL до API указывается в параметрах `rds_server_urls`/`RDS_SERVER_URLS` при установке/запуске IAM Proxy.
> При этом в фоновом режиме будет запущено приложение RDS Client,
> которое будет опрашивать указанный URL с заданной периодичностью,
> и при обнаружении изменения данных (по сравнению с ранее полученными из API)
> будут обновлены конфигурационные файлы IAM Proxy и запушен reload конфигурации.

> Как частный случай реализации API RDS, это RDS Server, который предоставляет следующее API для этих целей:
>
> | Метод / Endpoint                     | Content-Type ответа | Описание                             |
> |--------------------------------------|---------------------|--------------------------------------|
> | GET /rds-for-proxy/active-conf-json  | application/json    | Активная конфигурация в формате JSON |

Для того чтобы `RDS Client` корректно обработал опции для ответвлений, необходимо чтобы json-объект, возвращаемый по
API, соответствовал JSON-схеме (актуальная схема находится в дистрибутиве, в файле
`package/bh/rdsserver/config/rds-server-active-configuration.schema.json`):

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "rds-active-configuration.schema.json",
  "title": "Объект активной зоны с опциями ответвлений",
  "type": "object",
  "properties": {
    "name": {
      "type": "string",
      "description": "Имя активной зоны по которой передаются ответвления"
    },
    "junctions": {
      "type": "array",
      "description": "Список ответвлений из активной зоны",
      "minItems": 0,
      "uniqueItems": true,
      "items": {
        "type": "object",
        "properties": {
          "junctionName": {
            "type": "string",
            "description": "Понятное описание для отображения ответвления сервиса на технической странице IAM Proxy. На стартовой странице IAM Proxy возможно объединить сервисы в раскрывающиеся группы, путем использования `/`"
          },
          "description": {
            "type": "string",
            "description": "Расширенное описание проксируемой подсистемы/сервиса на технической странице IAM Proxy"
          },
          "limitRequests": {
            "type": "number",
            "minimum": 0,
            "maximum": 3200,
            "description": "Опциональный. По умолчанию 0. Лимит в количестве запросов в секунду (rps), действующий по конкретному ответвлению, в случае превышения лимита запросы будут приостанавливаться на время необходимое для соблюдения этого лимита (скорости). Лимит применяется в рамках каждого сервера IAM Proxy автономно. При установке параметра `proxy_limit_req_jct_is_per_session` в `true` лимит будет действовать в рамках каждой отдельной сессии пользователя, а не в общем для всего сервера. Значение '0' не устанавливает дополнительных ограничений по ответвлению"
          },
          "limitRequestsZone": {
            "type": "string",
            "pattern": "[\\w-]*",
            "description": "Опциональный. По умолчанию ''. Имя зоны действия лимита `limitRequests`. Можно установить одно имя зоны по нескольким ответвлениям, чтобы организовать общий лимит по пулу ответвлений (по этим ответвлениям параметр `limitRequests` также должен устанавливаться одинаковым)"
          },
          "https": {
            "type": "boolean",
            "description": "Использование SSL на серверах бэкенд (необязателен,`default True`)"
          },
          "indexUrl": {
            "type": "string",
            "description": "URL относительно корня на бэкенд, по которому осуществляется основной вход в UI подсистемы. Пустое значение или `-` позволяет не показывать ссылку/ответвление на стартовой странице IAM Proxy. Данный параметр используется исключительно для формирования ссылки на стартовой странице IAM Proxy, и не оказывает влияния на функциональность проксирования"
          },
          "sslCommonName": {
            "type": "string",
            "description": "Шаблон/значение имени из CN сертификата бэкенд-серверов, используется при соединении с бэкенд по HTTPS. Значение \"*\" - отключает проверку SSL. (необязателен, `default .mycompany.ru`)"
          },
          "transparent": {
            "type": "boolean",
            "description": "\"Прозрачность\" url (необязателен, `default False`). `True` - при проксировании запросы будут проходить без изменения URL. URL введенный в адресной строке браузера будет совпадать с URL который придет в HTTP-запросе на бэкенд (на сервер приложения). `False` - значение из `junctionPoint` будет вырезано из URL запросов, и вставлено в URL-ы контента ответов"
          },
          "junctionPoint": {
            "type": "string",
            "pattern": "(/[\\S]{0,100}[^/])|($^)",
            "description": "Корневой контекст запросов, по которому будет определяться принадлежность запроса к конкретной подсистеме/бэкенд, и в какой бэкенд будет проксироваться запрос"
          },
          "serverAddresses": {
            "type": "array",
            "description": "Список серверов (сервер:порт:опции), на которые будут проксироваться/балансироваться запросы для данного контекста `junctionPoint`. Каждый end-point должен быть описан строкой следующего вида: 'адрес узла'[:'порт'[:'опция 1'][:'опция 2']...]. Где: 'адрес узла' – IP-адрес узла кластера или его DNS-имя; 'порт' – порт по которому происходит перенаправление (по умолчанию 443). Пример описания адреса узла: '10.x.x.01:9443' или 'mycompany.ru:4444:max_fails=10:fail_timeout=30s'",
            "items": {
              "pattern": "^[a-zA-Z\\d._-]+(:\\d{0,5}(:[a-z].*)?)?$",
              "type": "string"
            }
          },
          "applyJctRequestFilter": {
            "type": "string",
            "pattern": "[, a-z_\\d-./]*",
            "description": "Список дополнительных опций или конфигурационных файлов, применяемые к данному `junctionPoint`, которые необходимо применить к данному контексту. Варианты значений опций можно посмотреть в разделе настройки через компонент PACMAN (CFGA) продукта Platform V Configuration (CFG)"
          },
          "authorizeByRoleTemplate": {
            "type": "string",
            "description": "Шаблон (регулярное выражение) авторизации ролей пользователя. Пул ролей, по которым осуществляется доступ к junction формируется из ID токена (правила задаются настройками IAM Proxy). По умолчанию пул ролей формируется из двух списков, из списка ролей Realm Keycloak (атрибуты `id_token.realm_access.roles` или `id_token.roles` или `id_token.groups`), и списка ролей используемого Client Keycloak (атрибут `id_token.resource_access[используемый в IAM Proxy client-id].roles`)"
          }
        },
        "required": [
          "junctionPoint"
        ],
        "additionalProperties": false
      }
    }
  },
  "required": [
    "name",
    "junctions"
  ],
  "additionalProperties": false
}
```

Пример корректного объекта с опциями по ответвлениям:

```json
{
  "name": "default",
  "junctions": [
    {
      "junctionName": "Audit",
      "description": "AC Audit",
      "limitRequests": 20,
      "limitRequestsZone": "audit",
      "https": false,
      "indexUrl": "/business-audit/",
      "sslCommonName": "*",
      "transparent": true,
      "junctionPoint": "/business-audit",
      "serverAddresses": [
        "10.x.x.10:8080"
      ]
    },
    {
      "junctionName": "RDS Server",
      "description": "Route Discovery Server",
      "limitRequests": 20,
      "limitRequestsZone": "rdsServer",
      "https": false,
      "indexUrl": "/rds-for-proxy/",
      "sslCommonName": "*",
      "transparent": true,
      "authorizeByRoleTemplate": "ROLE_ADMIN",
      "junctionPoint": "/rds-for-proxy",
      "serverAddresses": [
        "node1.rds-server.my.company.ru:9443"
      ]
    }
  ]
}
```
