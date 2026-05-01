# Типовые сценарии администрирования компонента IAM Proxy

Ниже приведены доступные сценарии:

- [Добавление нового ответвления](administration-scenarios-add-junction.md)
- [Изменение параметров ответвления](administration-scenarios-changes-junction.md)
- [Удаление ответвления](administration-scenarios-del-junction.md)
- [Как определить версию продукта](administration-scenarios-define-version.md)

> Параметры и дополнительные опции ответвлений (junction) соответствуют схеме `rds-api-configuration.schema.json`.  
> Актуальная структура описана в разделе [API для динамической настройки IAM Proxy](../additional-materials/rds-api.md).

Также описаны:
- контроль событий системного журнала — в разделе [События системного журнала](system-log.md);
- контроль событий мониторинга — в разделе [События мониторинга](monitoring.md).

## Пример: Добавление нового ответвления через RDS API (встроенная демонстрация структуры)

Для справки ниже приведен пример полного сценария с актуальной структурой конфигурации.

## Предусловия

1. У пользователя есть:
   - доступ к RDS API (например, через UI или CLI);
   - права администратора на изменение конфигурации;
   - токен для аутентификации в RDS API.
2. IAM Proxy настроен на работу с RDS: параметр `RDS_API_URLS` указан при запуске.
3. Бэкенд-сервис доступен по HTTPS, сертификат валиден, FQDN соответствует шаблону `sslCommonName`.

## Последовательность выполнения

### 1. Подготовьте новое ответвление в формате JSON

Создайте файл `new-junction.json` с конфигурацией, соответствующей схеме `rds-api-configuration.schema.json`:

```json
{
  "junctionName": "Внутренний API",
  "description": "Сервис внутренних бизнес-операций",
  "junctionPoint": "/api-internal",
  "serverAddresses": [
    "backend-internal.mycompany.ru:8443"
  ],
  "https": true,
  "sslCommonName": "*.mycompany.ru",
  "indexUrl": "/api-internal/",
  "transparent": false,
  "authorizeByRoleTemplate": "regex:^dev|qa$",
  "limitRequests": 100,
  "limitRequestsZone": "internal-api-group",
  "protocol": "http",
  "healthcheck": {
    "enabled": true,
    "path": "/actuator/health",
    "interval": 15000,
    "timeout": 5000,
    "fall": 3,
    "rise": 2,
    "validStatus": "200,204"
  }
}
```

> ⚠️ Обязательные поля: `junctionPoint`, `junctionName`.

### 2. Получите текущую конфигурацию из RDS API

```shell
curl -k https://rds-api.mycompany.ru/rds-for-proxy/active-conf-json > current-config.json
```

### 3. Добавьте новое ответвление в массив `junctions`

Объедините содержимое `current-config.json` и `new-junction.json`:

```json 
{
  "name": "production",
  "junctions": [
    // ... существующие junction'ы ...
    {
      "junctionName": "Внутренний API",
      "description": "Сервис внутренних бизнес-операций",
      "junctionPoint": "/api-internal",
      "serverAddresses": [
        "backend-internal.mycompany.ru:8443"
      ],
      "https": true,
      "sslCommonName": "*.mycompany.ru",
      "indexUrl": "/api-internal/",
      "transparent": false,
      "authorizeByRoleTemplate": "regex:^dev|qa$",
      "limitRequests": 100,
      "limitRequestsZone": "internal-api-group",
      "protocol": "http",
      "healthcheck": {
        "enabled": true,
        "path": "/actuator/health",
        "interval": 15000,
        "timeout": 5000,
        "fall": 3,
        "rise": 2,
        "validStatus": "200,204"
      }
    }
  ]
}
```

### 4. Опубликуйте обновленную конфигурацию

```shell
curl -X PUT https://rds-api.mycompany.ru/rds-for-proxy/active-conf-json \
  -H "Authorization: Bearer <admin_token>" \
  -H "Content-Type: application/json" \
  -d @updated-config.json
```

> RDS Client в IAM Proxy автоматически подтянет изменения в течение интервала `rds_polling_interval`.

### 5. Принудительная перезагрузка (опционально)

```shell
curl -X POST http://127.0.0.1:10080/reload/
```

> Эндпоинт `/reload/` доступен только локально.

## Результат

1. Ответвление `/api-internal` добавлено в IAM Proxy.
2. Проверьте доступность:
   ```shell
   curl -k https://iam-proxy.mycompany.ru/api-internal/
   ```
   Ожидается редирект на аутентификацию или ответ от бэкенда.
3. В логах IAM Proxy:
   ```
   [INFO] RDS config updated: added junction '/api-internal'
   ```
4. На стартовой странице IAM Proxy (`/`) появилась ссылка "Внутренний API".
5. Healthcheck активен — проверьте через:
   ```shell
   curl http://127.0.0.1:10080/status/healthcheck
   ```

## Описание механизмов безопасности

1. **Аутентификация**:
   - доступ к RDS API защищен OAuth2/JWT;
   - пользователь должен иметь роль `admin`.

2. **Шифрование**:
   - все запросы к RDS API — по HTTPS;
   - соединение с бэкендом — по TLS с проверкой сертификата (`sslCommonName`).

3. **Ограничение доступа**:
   - авторизация по шаблону ролей (`authorizeByRoleTemplate`);
   - можно дополнительно ограничить по IP через `applyJctRequestFilter`.

## Правила эксплуатации

1. **Формат данных**:
   - конфигурация должна строго соответствовать `rds-api-configuration.schema.json`;
   - валидацию можно выполнить локально:
     ```shell
     jsonschema -i updated-config.json rds-api-configuration.schema.json
     ```

2. **Источники данных**:
   - RDS API может получать конфигурацию из GitOps или CI/CD;
   - формат — `application/json`.

3. **Особенности**:
   - максимальный размер конфигурации — 512 КБ;
   - рекомендуемая частота обновлений — не чаще 1 раза в 10 секунд.

4. **Аудит**:
   - все изменения фиксируются в централизованном журнале (Platform V Audit SE);
   - события попадают в SIEM.
