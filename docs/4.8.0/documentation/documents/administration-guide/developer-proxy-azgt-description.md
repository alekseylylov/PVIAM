# Функциональность авторизации на основе полномочий из компонента Авторизация (AZGT)

Авторизация по определенным частям URI запроса, с получением текущих полномочий (не ролей!) пользователя по API
Авторизации.

## Подключение к API Авторизации

Для настройки подключения к API Авторизации в IAM Proxy используются следующие переменные:

| Параметр профиля установки        | Описание                                                                                                              |
|:----------------------------------|:----------------------------------------------------------------------------------------------------------------------|
| authz_spas_url                    | URL для вызова API Авторизации                                                                                        |
| authz_spas_secret                 | Значение по умолчанию `123456`. Секрет для вызова API Авторизации                                                     |
| authz_spas_ticket_lifetime        | Значение по умолчанию `3600`. Частота обновления запроса Авторизации в секунду                                        |
| authz_spas_ticket_failed_lifetime | Значение по умолчанию `5`. Частота получения запроса Авторизации в секунду, если ранее попытка была неуспешной        |
| authz_spas_rights_lifetime        | Значение по умолчанию `60`. Частота обновления полномочий из Авторизации в секунду                                    |
| authz_spas_rights_failed_lifetime | Значение по умолчанию `5`. Частота обновления полномочий из Авторизации в секунду, если ранее попытка была неуспешной |
| authz_spas_ssl_verify             | Допустимые значения `1,0`. Значение по умолчанию `1`. Проверка сертификата на endpoint authz_spas_url                 |

Значения и примеры заполнения переменных приведены в разделах [Установка](../installation-guide/installation.md),
[Соответствие имен параметров для разных сред и инструментов установки](../installation-guide/params.md)

## Подключение в ответвлении (в location)

Включение функциональности через задание переменной use_authz_by_url в IAM Proxy, которая может принимать значения:

| Значение | Описание                                                                                                                                     |
|:---------|:---------------------------------------------------------------------------------------------------------------------------------------------|
| 0        | Функциональность выключена                                                                                                                   |
| 1        | moduleid идет в URI в конце jctroot, а permissionid следующим                                                                                |
| 2        | moduleid идет в URI сразу после jctroot, а permissionid следующим                                                                            |
| 3        | moduleid идет в URI в конце jctroot, а permissionid складывается из метода и не более 2-х уровней после jctroot (в итоге строка в camelcase) |

В нужном ответвлении в параметр `applyJctRequestFilter` добавить одну из опции перечисленных ниже, которая задаст
значение
`use_authz_by_url`:

- `common/rds-use-authz-by-url.location.conf` - задает use_authz_by_url=1;
- `common/rds-use-authz-by-url-bridge.location.conf` - задает use_authz_by_url=2;
- `common/rds-use-authz-by-url-and-method.location.conf` - задает use_authz_by_url=3.

## Примеры

### use_authz_by_url=1, rds-use-authz-by-url.location.conf

Если запрос будет `https://my.ru/myModule/readManagerInfo/xxx/yyyy` и `$jctroot='/myModule'`

Тогда в Авторизации нужно будет иметь полномочие `readManagerInfo` в модуле myModule.

> `readManagerInfo` - это абстрактный пример, а не реальное полномочие.

### use_authz_by_url=2, rds-use-authz-by-url-bridge.location.conf

```
location /mystand/ {
    set $jctroot "/mystand";
    set $server_jct '10.x.x.13';

    include common/rds-use-authz-by-url-bridge.location.conf;

    proxy_pass https://backend_jct_mystand;
}
```

Две последующих за $jctroot части URI запроса будут использоваться в качестве ModuleID и PermissionID при проверке
наличия полномочия в Авторизации.

Если запрос будет `https://my.ru/mystand/myModule/readManagerInfo/xxx/yyyy` и `$jctroot='/mystand'`

Тогда в Авторизации нужно будет иметь полномочие `readManagerInfo` в модуле myModule.

> `readManagerInfo` - это абстрактный пример, а не реальное полномочие.

### use_authz_by_url=3, rds-use-authz-by-url-and-method.location.conf

Если будет `POST` запрос к `https://my.ru/myModule/myApi/managerInfo/xxx/yyyy` и `$jctroot='/myModule'`.

Тогда в Авторизации нужно будет иметь полномочие `postMyApiManagerInfo` в модуле myModule.

## Особенности

Если Авторизация не ответил или вернул ошибку, то выдаем отказ в авторизации.

Включение опциональных конфигурационных файлов ("include common/*.conf") задается через профиль развертывания, в
переменной `main.yml/proxy_jct_list/applyJctRequestFilter` (несколько файлов-опций указывается через запятую).

