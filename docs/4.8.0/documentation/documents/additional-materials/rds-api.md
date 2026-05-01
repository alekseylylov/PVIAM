# Описание API для реализации динамической настройки IAM Proxy

Для обеспечения оперативного изменения поведения проксирования на стороне IAM Proxy возможно предоставление данных о
бэкендах через внешнее API (API RDS) в реальном времени.

> При обнаружении изменений в данных (по сравнению с предыдущей версией данных) RDS Client автоматически обновляет
> конфигурационные файлы IAM Proxy и инициирует перезагрузку конфигурации (`reload`).

Чтобы `RDS Client` корректно обработал данные, возвращаемые по API, данные должны быть в json-формате и соответствовать
json-схеме конфигурации `rds-api-configuration.schema.json` (актуальная схема доступна в дистрибутиве продукта по пути
`package/bh/iamproxy/config/rds-client/schemas/rds-api-configuration.schema.json`).

## Формат данных API

Пример json-схемы конфигурации (актуальную схему необходимо использовать из дистрибутива)

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "$id": "rds-api-configuration.schema.json",
  "title": "Объект активной зоны с опциями ответвлений",
  "type": "object",
  "properties": {
    "name": {
      "type": "string",
      "description": "Имя активной зоны, по которой передаются опции ответвлений"
    },
    "junctions": {
      "$ref": "#/definitions/Junctions",
      "description": "Список ответвлений из активной зоны"
    }
  },
  "required": [
    "junctions"
  ],
  "additionalProperties": false,
  "definitions": {
    "Junctions": {
      "type": "array",
      "description": "Список ответвлений (junctions) для проксирования запросов к внутренним сервисам",
      "minItems": 0,
      "uniqueItems": true,
      "items": {
        "$ref": "#/definitions/Junction"
      }
    },
    "Junction": {
      "type": "object",
      "description": "Настройки ответвления (настройка проксирования до конкретного base_url/backend)",
      "properties": {
        "junctionName": {
          "type": "string",
          "pattern": ".+",
          "description": "Понятное описание для отображения ответвления сервиса на технической странице IAM Proxy. На стартовой странице IAM Proxy возможно объединить сервисы в раскрывающиеся группы, путем использования `/`",
          "example": "Мое приложение"
        },
        "description": {
          "type": "string",
          "pattern": ".+",
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
          "pattern": "^[\\w-]*$",
          "description": "Опциональный. По умолчанию ''. Имя зоны действия лимита `limitRequests`. Можно установить одно имя зоны по нескольким ответвлениям, чтобы организовать общий лимит по пулу ответвлений (по этим ответвлениям параметр `limitRequests` также должен устанавливаться одинаковым)"
        },
        "https": {
          "type": "boolean",
          "default": true,
          "description": "Использование SSL на серверах backend (необязателен, `default True`)"
        },
        "indexUrl": {
          "type": "string",
          "pattern": "^(/\\S*$|-$|$)",
          "description": "URL относительно корня на backend, по которому осуществляется основной вход в UI подсистемы. Пустое значение или `-` позволяет не показывать ссылку/ответвление на стартовой странице IAM Proxy. Данный параметр используется исключительно для формирования ссылки на стартовой странице IAM Proxy, и не оказывает влияния на функциональность проксирования",
          "example": "/my-app/index.html"
        },
        "sslCommonName": {
          "type": "string",
          "default": ".ru",
          "pattern": "^[a-zA-Z\\d._*-]+$",
          "description": "Шаблон/значение fqdn для проверки SAN сертификата серверов backend, используется при соединении с backend по HTTPS. Значение `*` отключает проверку сертификатов. (необязателен, `default .mycompany.ru`)"
        },
        "transparent": {
          "type": "boolean",
          "default": false,
          "description": "\"Прозрачность\" url (необязателен, `default False`). `True` - при проксировании запросы будут проходить без изменения URL. URL введенный в адресной строке браузера будет совпадать с URL который придет в HTTP-запросе на backend (на сервер приложения). `False` - значение из `junctionPoint` будет вырезано из URL запросов, и вставлено в URL-ы контента ответов"
        },
        "junctionPoint": {
          "type": "string",
          "pattern": "^(/[\\S]{0,100}[^/])$|(^$)",
          "description": "Корневой контекст запросов, по которому будет определяться принадлежность запроса к конкретной подсистеме/backend, и в какой backend будет проксироваться запрос",
          "example": "/my-system"
        },
        "serverAddresses": {
          "type": "array",
          "description": "Список серверов (сервер[:порт[:опции]]), на которые будут проксироваться/балансироваться запросы для данного контекста `junctionPoint`",
          "items": {
            "pattern": "^[a-zA-Z\\d._-]+(:\\d{1,5})?(:[a-z].*)?$",
            "type": "string",
            "example": "server-1.my-company.ru:8443"
          }
        },
        "applyJctRequestFilter": {
          "type": "string",
          "pattern": "^[, a-z_\\d./-]+$",
          "description": "Список дополнительных опций или конфигурационных файлов, применяемые к данному `junctionPoint`, которые необходимо применить к данному контексту."
        },
        "authorizeByRoleTemplate": {
          "type": "string",
          "pattern": ".+",
          "description": "Задается шаблон в формате `lua string pattern` или `regex`. Чтобы задать `regex`, укажите префикс `regex:`. Перед сравнением в шаблон по краям добавляются '^' и '$', чтобы обеспечить полное совпадение шаблона с ролью. Пул ролей, по которым осуществляется сравнение, формируется из общих ролей в ID-токене (роли Realm, это атрибуты: `id_token.realm_access.roles` или `id_token.roles` или `id_token.groups`) и из ролей client в ID-токене (роли client, атрибут:`id_token.resource_access[текущий client].roles`)."
        },
        "protocol": {
          "type": "string",
          "enum": [
            "http",
            "grpc"
          ],
          "default": "http",
          "description": "Протокол для проксирования трафика"
        },
        "healthcheck": {
          "type": "object",
          "description": "Параметры Healthcheck до back-end",
          "properties": {
            "enabled": {
              "type": "boolean",
              "default": false,
              "description": "Использовать Healthcheck до backend"
            },
            "path": {
              "type": "string",
              "pattern": "^/\\S+$",
              "default": "/healthcheck",
              "description": "Путь в URL до Healthcheck на backend"
            },
            "interval": {
              "type": "number",
              "default": 10000,
              "minimum": 0,
              "description": "Частота вызова HC в миллисекундах"
            },
            "timeout": {
              "type": "number",
              "default": 20000,
              "minimum": 0,
              "maximum": 60000,
              "description": "Ожидание подключения при вызова HC в миллисекундах"
            },
            "fall": {
              "type": "number",
              "default": 2,
              "minimum": 0,
              "maximum": 100,
              "description": "Количество неуспешных вызовов, для вывода сервера из группы балансировки"
            },
            "rise": {
              "type": "number",
              "default": 2,
              "minimum": 0,
              "maximum": 100,
              "description": "Количество успешных вызовов, для возврата сервера в группу балансировки"
            },
            "validStatus": {
              "type": "string",
              "default": "200,204,302",
              "pattern": "^[1-5][0-9][0-9](,[1-5][0-9][0-9])*$",
              "description": "HTTP коды ответов, при которых вызов считается успешным. Пример: 200,204,301,302,403,401"
            },
            "requestBody": {
              "type": "string",
              "default": "GET %s HTTP/1.1\\r\\nHost: %s\\r\\n\\r\\n",
              "pattern": "^[A-Z]+ [^\\]* HTTP/1\\.[01]\\r\\n.*\\r\\n\\r\\n$",
              "description": "Содержимое raw-запроса к Healthcheck backend (включая первую строку запроса и заголовки, при этом в тексте первая %s будет заменена на URL, вторая %s будет заменена на хост). Пример: GET %s HTTP/1.1\\r\\nHost: %s\\r\\n\\r\\n"
            }
          }
        }
      },
      "required": [
        "junctionPoint",
        "junctionName"
      ],
      "additionalProperties": false
    }
  }
}
```

## Пример корректного ответа API

```json
{
  "name": "production",
  "junctions": [
    {
      "junctionName": "Бизнес-аудит",
      "description": "Система аудита бизнес-операций",
      "junctionPoint": "/business-audit",
      "serverAddresses": [
        "audit-svc.prod.local:8443"
      ],
      "https": true,
      "sslCommonName": "*.prod.mycompany.ru",
      "indexUrl": "/business-audit/",
      "transparent": true,
      "limitRequests": 50,
      "limitRequestsZone": "audit-group",
      "applyJctRequestFilter": "set-header-x-forwarded",
      "authorizeByRoleTemplate": "Audit_User",
      "healthcheck": {
        "enabled": true,
        "path": "/actuator/health",
        "interval": 15000,
        "timeout": 5000,
        "fall": 3,
        "rise": 2,
        "validStatus": "200,204"
      }
    },
    {
      "junctionName": "gRPC Сервис",
      "description": "Микросервис обработки данных",
      "junctionPoint": "/data-service",
      "serverAddresses": [
        "grpc-svc.prod.local:9090"
      ],
      "https": true,
      "protocol": "grpc",
      "authorizeByRoleTemplate": "regex:^DataService_Admin|User$",
      "healthcheck": {
        "enabled": true,
        "path": "/ready",
        "interval": 10000,
        "timeout": 3000,
        "fall": 2,
        "rise": 2,
        "validStatus": "200"
      }
    }
  ]
}
```