# Настройка получения конфигурации

Раздел описывает источники и варианты получения конфигурации для RDS Server.

> Примечание
>
> Полученная в RDS Server и преобразованная информация о параметрах публикуется на http API и используется на IAM Proxy.
>
> Вариант администрирования с помощью PACMAN (CFGA) является устаревшим (deprecated).
> Для администрирования с использованием PACMAN (CFGA) или yaml-файла в формате PACMAN, требуется наличие RDS Server на
> стенде. Поддержку данной интеграции и RDS Server планируется прекратить с релиза Platform V IAM 5.0.0.

Конфигурационные данные можно получить из:

- модулей 3-го поколения компонента PACMAN (CFGA);
- компонента PACMAN (CFGA) по REST API;
- yaml-файла, полученный из URI или файла.

Подробнее о параметрах RDS Client смотрите
раздел [Параметры RDS Client](../installation-guide/proxy-deploy-docker-description.md)

## Интеграция с использованием модулей 3-го поколения компонента PACMAN (CFGA)

Параметры задаются в файле параметров RDS Server (application-*.yml) или в переменных окружения.

Пример настройки:

```yaml
configStore:
  nodeId: node1
  config-store-disable: false
  jndi-store-disabled: true
  read-from-third-generation: true
  config-store-jdbc-login: login
  config-store-jdbc-pas: P@ssw0rd
  config-store-jdbc-url: jdbc:postgresql://101.x.x.256:5432/db
  config-store-crypto-pas: P@ssw0rd896543964359824569872456
```

> **Внимание!** Приложению нужны права на запись в каталог из systemProperty `rds.tmp.dir`

### Интеграция с компонентом PACMAN (CFGA) по REST API

Для возможности считывания конфигурации из компонента PACMAN (CFGA) по REST API, необходимо добавить источник в список
источников и соответствующим образом его настроить.

В `sourceParam` необходимо внести корректный URL до REST API компонента PACMAN (CFGA). В нем должны содержаться данные о
группе, узле и модуле, разделенные запятой.

В случае если группа или какие-то критерии перекрытия пустые, необходимо так и писать. Например, в
`https://pacman.mycompany.ru//configuration/v2/configurator-yaml/rn/DEFAULT/configs/,platformv-platform-proxy,` -
значение группы и модуля пустые (`.../<группа пустая>,<узел platformv-platform-proxy>,<модуль пустой>`).

Для получения конфигурации должен быть заполнен хотя бы один критерий перекрытия, иначе получить конфигурацию по REST
API будет невозможно.

Данные из компонента PACMAN (CFGA) по REST API можно получить только с корректным клиентским сертификатом. Для этого
необходимо настроить SSL (указать данные для truststore и keystore).

Пример настройки:

```yaml
configSources:
  sources:
    - id: pacmanREST
      sourceParam: https://pacman.mycompany.ru/configuration/v2/configurator-yaml/rn/DEFAULT/configs/,platformv-platform-proxy
  ssl:
    enabled: true
    key-store: ssl/rds-server-keystore.p12
    key-store-type: PKCS12
    #    key-alias: 1                     # если необходимо взять конкретный ключ, можно указать его алиас тут
    key-password: 123456
    key-store-password: 123456
    trust-store: ssl/trust-store.p12
    trust-store-type: PKCS12
    trust-store-password: 123456
  priorityList:
    - pacmanREST
```

`sources[n].sourceParam` - параметр источника.

Поддерживаются:

- путь до файла (полный / относительный);
- URI ссылка на yaml-файл;
- URI ссылка на API компонента PACMAN (CFGA).

`ssl.trust-store` - путь к хранилищу ключей, в котором хранится клиентский сертификат / ключ от сервиса компонента
PACMAN (CFGA). В случае использования одного хранилища сертификатов необходимо добавить в rds-server-keystore.p12
клиентский сертификат от сервиса компонента PACMAN (CFGA).

`ssl.key-store` - путь до хранилища доверия, в котором хранятся SSL-сертификаты (пример, ssl/trust-store.p12 - создается
на целевых серверах из файлов trusted_*.cer/trusted_*.crt.pem).

### yaml-файл, полученный из URI или файла

Ридер конфигурации из yaml совмещает в себе как чтение из файла / URI yaml-формата конфигурации, так и ридер источника
конфигурации - компонент PACMAN (CFGA).

Для возможности считывания конфигураций из файлов / URI необходимо выполнить настройку ридера.

При формировании `priorityList` нужно учитывать следующее:

- первый источник перекрывается вторым в списке, второй - третьим и так далее;
- в списке есть зарезервированный идентификатор - `PACMAN`. Открывается в режиме чтения конфигурации из компонента
  PACMAN (CFGA) при помощи клиентского модуля.

В целях обратной совместимости, при отсутствии настроек yaml-ридера в application.yaml будет подгружен только один
источник - компонент PACMAN (CFGA).

Пример настройки:

```yaml
configSources:
  sources: # Список источников конфигураций
    - id: myfile                      # Уникальный идентификатор источника. Должен совпадать с ID из списка priorityList
      sourceParam: zoneConfig.yml     # Параметр источника. Поддерживаются: путь до файла (полный/относительный), URI ссылка
    - id: someUrl
      sourceParam: http://127.0.0.1:8887/zoneConfig.yml
  priorityList: # Порядок в этом списке важен! По мере прохода по списку будет выполнено перекрытие совпадающих джанкшенов с последующим источником. Уникальные - будут объедены в общий список.
    - PACMAN     # Уникальный идентификатор источника PACMAN.
    - myfile     # Произвольный идентификатор, соответствующий объявленному выше источнику
    - someUrl    # Произвольный идентификатор, соответствующий объявленному выше источнику
```

## Настройка источников конфигурации при развертывании в OpenShift/k8s

В случае развертывания RDS Server в OpenShift, необходимо задать соответствующие параметры профиля установки.

Для подключения к PACMAN (CFGA) используется параметр `rdsserver.k8s.rds.configSources.pacman.rest-url`.

Для использования конфигурации из yaml-файла его содержимое задается как текст в параметре `rsd_conf` в файле
`custom_property.conf.yml`.

Пример:

```yaml
rds_conf: |
  ZoneConfig[Core]:
    JunctionConfig[Snoop]:
      junctionName: "Заглушки / Test healthcheck"
      junctionPoint: /port443
      indexUrl: /port443/
      sslCommonName: '.dev.mycompany.ru'
      https: "false"
      transparent: "True"
      limitRequests: 10
      applyJctRequestFilter: "common/rds-enable-cors.location.conf"
      serverAddresses:
      - vm-dev-auth-nginx-253.vdc01.mycompany.ru
    Zone[offline]:
      Junction[Snoop]:
        serverAddresses:
        - vm-knt-auth-wf-222.vdc01.mycompany.ru:25000
      Junction[Audit]:
        serverAddresses:
        - ufs-security-ift2.mycompany.ru:25080
    Zone[standin]:
      Junction[Audit]:
        serverAddresses:
        - ufs-security-ift2.mycompany.ru:25141
      Junction[Snoop]:
        serverAddresses:
        - vm-knt-auth-wf-222.vdc01.mycompany.ru:25061
    zoneNameStandin: standin
    zoneNameOffline: offline
  checkConfigurationFrequency: 10
  checkStandInFrequency: 10
  httpClientMaxPoolSize: 1
  manualMode: "false"
  transportRequestTimeout: 9
  usePlatformSemaphore: "true"
```