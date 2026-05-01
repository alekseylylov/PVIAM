# Описание настроек приложения RDS Server

## Конфигурация приложения через файлы параметров

Стандартный подход предусматривает использование профилей spring-приложения через `--spring.profiles.active=RDS_DEV`
(параметр запуска java).

Файлы профилей должны находиться в том же каталоге, что и jar-файл или в каталоге `./config`.

Если файл конфигурации приложения будет храниться не в одной директории с jar-файлом, и/или его имя отличается от
`application-*.yaml` или по каким-то причинам нужно использовать параметр `--spring.config.location`, то необходимо
указать полный путь до файла.

Для запуска RDS Server выполняем команду:

```shell
java -jar rds.jar --spring.config.location=file:/work/application-RDS_DEV.yml
```

## Конфигурация через переменные окружения (для Openshift / k8s)

> Данный способ настройки удобно использовать при запуске RDS Server в контейнере (в Openshift/k8s), задав все необходимые параметры в параметрах контейнера.

_Внимание! Указанные ниже примеры настроек в yaml относятся к конфигурации spring boot - application.yml_

В зависимости от типа стенда (переменная stand_type, можно использовать DEV или PROM) заполняются параметры необходимого
профиля ниже.

### Настроить значения в application-PROM.yml, через передачу переменных окружения

Возможные имена переменных окружения (после двоеточия значение по умолчанию)

- rds.node:node1
- rds.config-store-disable:false
- rds.read-from-third-generation:true
- rds.config-store-jdbc-login
- rds.config-store-jdbc-pas
- rds.config-store-jdbc-url
- rds.config-store-crypto-pas
- rds.platform-config-file:platform-config.properties
- rds.audit.kafka.enabled:false
- rds.audit.kafka.main.server
- rds.audit.kafka.fallback.server
- rds.audit.kafka.enabled.mock:true
- rds.audit.kafka.security.protocol:PLAINTEXT
- rds.audit.kafka.ssl.protocol
- rds.audit.kafka.ssl.key.password
- rds.audit.kafka.ssl.keystore.location
- rds.audit.kafka.ssl.keystore.password
- rds.audit.kafka.ssl.truststore.location
- rds.audit.kafka.ssl.truststore.password
- rds.audit.rest.enabled:true
- rds.audit.rest.moduleName:rds-server
- rds.audit.rest.metamodelLocation:classpath:audit.metamodel.json
- rds.audit.rest.proxyUrl
- rds.audit.rest.truststore.location
- rds.audit.rest.truststore.password
- rds.audit.rest.keystore.location
- rds.audit.rest.keystore.password
- rds.audit.rest.keyPassword
- rds.standin.zone.id:TEST_ZONE
- rds.standin.stub:false
- rds.standin.bootstrap.servers:127.0.0.1
- rds.standin.bootstrap.producerConfig
- rds.standin.rest.enabled:false
- rds.standin.rest.url
- rds.standin.rest.callbackUrl:IP
- rds.standin.rest.inStandIn:false
- rds.server.https.port:8443
- rds.mtls.client-auth:need
- rds.mtls.keystore.path:ssl/keystore.p12
- rds.mtls.keystore.alias:server
- rds.mtls.key.password
- rds.mtls.keystore.password
- rds.mtls.truststore.path:ssl/truststore.p12
- rds.mtls.truststore.password
- rds.configSources.ssl.enabled:true
- rds.configSources.priorityList
- rds.configSources.sources

> Параметр `rds.configSources.sources` требует особого форматирования.
> В разделе "[Настройка получения конфигурации](rds-server-get-conf.md)" описаны детали.

- rds.configSources.ssl.keyStore
- rds.configSources.ssl.keyStoreType:PKCS12
- rds.configSources.ssl.keyAlias
- rds.configSources.ssl.keyPassword
- rds.configSources.ssl.keyStorePassword
- rds.configSources.ssl.trustStore
- rds.configSources.ssl.trustStoreType:PKCS12
- rds.configSources.ssl.trustStorePassword

### Настроить значения в application-DEV.yml, через передачу переменных окружения

Возможные имена переменных окружения (после двоеточия значение по умолчанию)

- rds.node:node1
- rds.audit.kafka.enabled.mock:true
- rds.audit.rest.enabled:false
- rds.standin.zone.id:TEST_ZONE
- rds.standin.bootstrap.servers:127.0.0.1
- rds.server.https.port:8443
- rds.server.http.port:8080
- rds.mtls.client-auth:none
- rds.mtls.keystore.path:ssl/keystore.p12
- rds.mtls.keystore.alias:server
- rds.mtls.key.password:changeit
- rds.mtls.keystore.password:changeit
- rds.mtls.truststore.path:ssl/truststore.p12
- rds.mtls.truststore.password:changeit
