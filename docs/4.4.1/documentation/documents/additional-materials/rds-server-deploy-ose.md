# Установка RDS Server в OSE/k8s

## Параметры развертывания RDS Server в OSE/k8s

**Часть** параметров указанных ниже являются **не обязательными**, и имеют либо значения по умолчанию, либо опциональны
и при их отсутствии функциональность будет отключена.

### Параметры основной функциональности RDS Server

- `stand_type` - Тип стенда. Допустимы следующие значения: `DEV`, `PROM` (по умолчанию `PROM`);
- `DEBUG` - не пустое значение включает отладку на уровне bash-скриптов, значение `1` включает debug логирование на RDS
  Server, значение `wait` включает ожидание в 1 час после завершения процесса RDS Server;
- `WAIT_EXIST_FILES` - перечень файлов создание которые необходимо дождаться перед запуском RDS Server;
- `LOGDIR` - директория, в которой будут записываться лог-файлы.

В зависимости от типа стенда (переменная `stand_type`, можно использовать `DEV` или `PROM`) заполняются параметры
необходимого профиля ниже.

#### Настроить передачу значений в application-PROM.yml через переменные окружения

Возможные имена переменных окружения (после двоеточия значение по умолчанию):
Параметры подключения компонента PACMAN (CFGA):

- `rds.node:node1`;
- `rds.config-store-disable:false`;
- `rds.read-from-third-generation:true`;
- `rds.config-store-jdbc-login`;
- `rds.config-store-jdbc-pas`;
- `rds.config-store-jdbc-url`;
- `rds.config-store-crypto-pas`;
- `rds.platform-config-file:platform-config.properties`;
- `rds.configSources.ssl.enabled:true`;
- `rds.configSources.priorityList`;
- `rds.configSources.sources`.

> Параметр `rds.configSources.sources` требует особого форматирования.
> Детали описаны в разделе "[Настройка источников конфигурации при развертывании в OpenShift](rds-server-get-conf.md)".

- `rds.configSources.ssl.keyStore`;
- `rds.configSources.ssl.keyStoreType:PKCS12`;
- `rds.configSources.ssl.keyAlias`;
- `rds.configSources.ssl.keyPassword`;
- `rds.configSources.ssl.keyStorePassword`;
- `rds.configSources.ssl.trustStore`;
- `rds.configSources.ssl.trustStoreType:PKCS12`;
- `rds.configSources.ssl.trustStorePassword`.

Параметры подключения к функциональной подсистеме Аудит (AUDT) по Kafka:

- `rds.audit.kafka.enabled:false`;
- `rds.audit.kafka.main.server`;
- `rds.audit.kafka.fallback.server`;
- `rds.audit.kafka.enabled.mock:true`;
- `rds.audit.kafka.security.protocol:PLAINTEXT`;
- `rds.audit.kafka.ssl.protocol`;
- `rds.audit.kafka.ssl.key.password`;
- `rds.audit.kafka.ssl.keystore.location`;
- `rds.audit.kafka.ssl.keystore.password`;
- `rds.audit.kafka.ssl.truststore.location`;
- `rds.audit.kafka.ssl.truststore.password`.

Параметры подключения к функциональной подсистеме Аудит (AUDT) по REST:

- `rds.audit.rest.enabled:true`;
- `rds.audit.rest.moduleName:rds-server`;
- `rds.audit.rest.metamodelLocation:classpath:audit.metamodel.json`;
- `rds.audit.rest.proxyUrl`;
- `rds.audit.rest.truststore.location`;
- `rds.audit.rest.truststore.password`;
- `rds.audit.rest.keystore.location`;
- `rds.audit.rest.keystore.password`;
- `rds.audit.rest.keyPassword`.

Параметры подключения к функциональной подсистеме Прикладной журнал (APLJ):

- `rds.standin.rest.enabled:false`;
- `rds.standin.zone.id:TEST_ZONE`;
- `rds.standin.stub:false`;
- `rds.standin.bootstrap.servers:127.0.0.1`;
- `rds.standin.bootstrap.producerConfig`;
- `rds.standin.rest.url`;
- `rds.standin.rest.inStandIn`.

Параметры настройки mTLS:

- `rds.server.https.port:8443`;
- `rds.mtls.client-auth:need`;
- `rds.mtls.keystore.path:ssl/keystore.p12`;
- `rds.mtls.keystore.alias:server`;
- `rds.mtls.key.password`;
- `rds.mtls.keystore.password`;
- `rds.mtls.truststore.path:ssl/truststore.p12`;
- `rds.mtls.truststore.password`.

#### Настроить передачу значений в application-DEV.yml через переменные окружения

Возможные имена переменных окружения (после двоеточия значение по умолчанию):

- `rds.node:node1`;
- `rds.audit.kafka.enabled.mock:true`;
- `rds.audit.rest.enabled:false`;
- `rds.standin.zone.id:TEST_ZONE`;
- `rds.standin.bootstrap.servers:127.0.0.1`;
- `rds.server.https.port:8443`;
- `rds.server.http.port:8080`;
- `rds.mtls.client-auth:none`;
- `rds.mtls.keystore.path:ssl/keystore.p12`;
- `rds.mtls.keystore.alias:server`;
- `rds.mtls.key.password:changeit`;
- `rds.mtls.keystore.password:changeit`;
- `rds.mtls.truststore.path:ssl/truststore.p12`;
- `rds.mtls.truststore.password:changeit`.

### Чтение параметров из SecMan

Для использования секретов из SecMan в RDS Server необходимо добавить профиль `SECRET` в активные профили приложения
(это можно сделать через переменную окружения `stand_type`, пример:`stand_type=PROM,SECRET`).

Для чтения паролей из файлов, созданных SecMan, необходимо использовать символ `@`. Правило формирования значения
параметров следующие:

- если в value первым будет символ `@`, то значение будет прочитано из файла с именем после `@`
    - пример: `rds.mtls.keystore.password=@keystore.pass`. Чтение значения для `rds.mtls.keystore.password` будет
      выполнено из файла `keystore.pass`;
- если в value будет только символ `@`, то значение будет прочитано из файла с именем равным имени параметра
    - пример: `rds.mtls.keystore.password=@`. Чтение значения для `rds.mtls.keystore.password` будет выполнено из
      файла `rds.mtls.keystore.password`. В SecMan значения для параметров RDS Server хранятся в
      `DEV/AUTH/SC/KV/OSE.my-auth-dev-01.RDS`.

Для перекладки файлов `keystore/truststore` из SecMan в контейнер RDS Server необходимо:

- для текстовых файлов:
    - внести текст в соответствующую секцию в SecMan (например, `DEV/AUTH/SC/KV/OSE.my-auth-dev-01.RDS.certs`);
    - имя секрета будет равно имени сформированного файла в контейнере RDS Server.
- для бинарных файлов:
    - закодировать в `base64` и добавить к ним расширение `.base64` (пример: `secret.p12.base64`. По итогу в контейнер
      будет подложен файл `secret.p12`);
    - залить в SecMan полученный текст, в соответствующую секцию в SecMan (например,
      `DEV/AUTH/SC/KV/OSE.my-auth-dev-01.RDS.certs`); Все файлы секретов размещаются в папке `/opt/rds-server/ssl` в
      контейнере.

### Каталоги и файлы для настройки

```
/opt/rds-server/ssl/keystore.p12 - хранилище, в котором хранится клиентский сертификат/ключ
/opt/rds-server/ssl/truststore.p12 - хранилище, в котором хранятся сертификаты доверенных ЦС
/opt/rds-server/ssl/audit-keystore.jks - хранилище, в котором хранится клиентский сертификат/ключ для интеграции с
Platform V Audit SE (AUDT).
```

> Имена файлов могут использовать другие, и определяются настройками профиля развертывания.
