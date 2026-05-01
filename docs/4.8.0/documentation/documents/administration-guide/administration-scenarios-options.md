# Варианты администрирования

Администрирование IAM Proxy может выполняться различными способами в зависимости от метода установки и требований к динамичности изменений. Ниже приведены основные подходы:

- **через `inventory-ansible.yml`** — при использовании Ansible-установки. Изменения применяются при повторном запуске playbook;
- **через `auth.*.conf` и `custom_property.conf.yml`** — при установке через `DOT CDJE` в OpenShift/K8s. Требует переразвертывания;
- **через RDS API** — позволяет изменять конфигурацию в реальном времени без перезапуска.

Для выполнения административных задач выберите подходящий сценарий в зависимости от среды и требований к оперативности.

Ниже приведен **пример полного сценария** для демонстрации структуры.

## Сценарий: Динамическое удаление ответвления через RDS API

## Предусловия

1. IAM Proxy развернут и запущен.
2.  RDS API интегрирован с IAM Proxy и доступен по URL.
3. У пользователя есть:
   - доступ к RDS API (API-токен или OAuth2-креденшелс);
   - роль `admin` в системе управления конфигурацией.
4. Открыт HTTPS-доступ к эндпоинту: `https://rds-api.mycompany.ru/rds-for-proxy/active-conf-json`.

## Последовательность выполнения

### 1. Получить текущую конфигурацию
Выполните запрос к RDS API для получения актуальной конфигурации:

```shell
curl -k https://rds-api.mycompany.ru/rds-for-proxy/active-conf-json
```

Пример ответа:
```json
{
  "name": "production",
  "junctions": [
    {
      "junctionName": "Старый сервис",
      "junctionPoint": "/old-service",
      "serverAddresses": ["old-svc.mycompany.ru:8443"],
      "https": true,
      "indexUrl": "/old-service/",
      "authorizeByRoleTemplate": "AdminOnly"
    },
    {
      "junctionName": "Новый сервис",
      "junctionPoint": "/new-service",
      "serverAddresses": ["new-svc.mycompany.ru:8443"],
      "https": true
    }
  ]
}
```

### 2. Удалите ненужное ответвление
Из массива `junctions` удалите объект с `junctionPoint: "/old-service"`.

Обновленная конфигурация:
```json
{
  "name": "production",
  "junctions": [
    {
      "junctionName": "Новый сервис",
      "junctionPoint": "/new-service",
      "serverAddresses": ["new-svc.mycompany.ru:8443"],
      "https": true
    }
  ]
}
```

### 3. Опубликуйте обновленную конфигурацию
Загрузите измененный JSON обратно в RDS API :

```shell
curl -X PUT https://rds-api.mycompany.ru/rds-for-proxy/active-conf-json \
  -H "Authorization: Bearer <admin_token>" \
  -H "Content-Type: application/json" \
  -d @updated-config.json
```

> ⚠️ Примечание: RDS Client в IAM Proxy автоматически подтянет изменения при следующем опросе (интервал задается параметром `rds_polling_interval`).

### 4. Принудительная перезагрузка (опционально)
Если нужно применить изменения немедленно:

```shell
curl -X POST https://iam-proxy.mycompany.ru/reload/ -k
```

> Адрес `/reload/` доступен только с `127.0.0.1` по умолчанию.

## Результат

1. Ответвление `/old-service` удалено из конфигурации IAM Proxy.
2. Проверьте недоступность:
   ```shell
   curl -I https://iam-proxy.mycompany.ru/old-service/
   ```
   Ожидается: `HTTP 404 Not Found`.
3. Логи IAM Proxy содержат:
   ```
   [INFO] RDS config updated: removed junction '/old-service'
   ```
4. Метрики и мониторинг (например, Grafana) больше не показывают активности по этому `junctionPoint`.

## Описание механизмов безопасности

1. **Аутентификация в RDS API**:
   - доступ к RDS API должен быть защищен (OAuth2, JWT, API Key);
   - пример SynGX-конфигурации:
     ```nginx
     location /rds-for-proxy/ {
         auth_jwt "IAM";
         auth_jwt_key_file /etc/nginx/secrets/jwt-public.pem;
         proxy_pass http://rds-app:8080;
     }
     ```

2. **Шифрование трафика**:
   - все запросы к RDS API должны выполняться по HTTPS с валидным сертификатом.

3. **Ограничение прав**:
   - только администраторы могут модифицировать конфигурацию;
   - все изменения логируются в `audit.log` с указанием пользователя, времени и `revision`.

## Правила эксплуатации

1. **Формат данных**:
   - конфигурация должна соответствовать схеме `rds-api-configuration.schema.json`;
   - обязательные поля: `junctionPoint`, `junctionName`;
   - корректность можно проверить с помощью `jsonschema`:
     ```shell
     jsonschema -i updated-config.json rds-api-configuration.schema.json
     ```

2. **Источники данных**:
   - RDS API может получать конфигурацию из GitOps-репозитория или CI/CD.

3. **Особенности загрузки**:
   - максимальный размер конфигурации — 512 КБ;
   - рекомендуемая частота обновлений — не чаще 1 раза в 10 секунд.

4. **Обработка ошибок**:
   - при невалидной конфигурации IAM Proxy сохраняет предыдущую версию;
   - ошибка возвращается в виде:
     ```json
     { "error": "validation_failed", "details": "junctionPoint is required" }
     ```
