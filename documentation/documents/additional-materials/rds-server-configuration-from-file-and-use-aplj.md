# Подключение в RDS Server конфигурации PACMAN из yaml-файла, и настройка соединения с ПЖ

> Вариант администрирования с помощью PACMAN (CFGA) является устаревшим (deprecated).
> Для администрирования с использованием PACMAN (CFGA) или yaml-файла в формате PACMAN, требуется наличие RDS Server на
> стенде. Поддержку данной интеграции и RDS Server планируется прекратить с релиза Platform V IAM 5.0.0.

## Создать ConfigMap, с конфигурацией

Создать ConfigMap, с заданием в ней конфигурации, аналогичной той, которая задается в компоненте PACMAN (CFGA) (смотрите
пример ниже).

Пример ConfigMap:

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: rds-server-pacman-config
data:
  pacman-rdsserver-config.yml: |
    ZoneConfig[Core]:
      JunctionConfig[RDSServer]:
        junctionName: "RDS Server Group / RDS Server"
        junctionPoint: /rds-for-proxy
        indexUrl: /rds-for-proxy/
        sslCommonName: '*'
        transparent: "true"
        serverAddresses:
        - auth-svc-rdsserver:80
        - auth-svc-rdsserver:443
      JunctionConfig[Audit]:
        junctionName: Audit
        description: "Ответвление для Бизнес Аудит"
        junctionPoint: /business-audit
        indexUrl: /business-audit/
        sslCommonName: '*'
        https: "false"
        transparent: "true"
        serverAddresses:
        - 10.x.x.50:25041
        applyJctRequestFilter: "common/oidc-unauth-access.location.conf"
      JunctionConfig[Snoop]:
        junctionName: Snoop (тестовое приложение)
        junctionPoint: /jct-snoop
        indexUrl: /snoop/snoop/snoop/
        sslCommonName: '*'
        https: "false"
        transparent: "false"
        serverAddresses:
        - 127.0.0.1:10080
      JunctionConfig[SnoopTransparent]:
        junctionName: Snoop transparent (тестовое приложение нагрузка)
        junctionPoint: /jct-snoop-isam
        indexUrl: /snoop/snoop/snoop/
        sslCommonName: '*'
        https: "false"
        transparent: "false"
        serverAddresses:
        - 127.0.0.1:10080
      Zone[offline]:
        Junction[Snoop]:
          serverAddresses: []
      Zone[standin]:
        Junction[Snoop]:
          serverAddresses:
          - 127.0.0.2:10080
        Junction[Audit]:
          serverAddresses:
          - 10.x.x.50:8080
      zoneNameStandin: standin
      zoneNameOffline: offline
    checkConfigurationFrequency: 10
    checkStandInFrequency: 10
    httpClientMaxPoolSize: 1
    manualMode: "true"
    transportRequestTimeout: 9
    usePlatformSemaphore: "true"
```

Подробную информацию по каждому из параметров можно найти в
разделе: [Параметры для настройки RDS Server](../administration-guide/config-parameters.md)

## Внести изменения в Deployment RDS Server

Добавить/заменить указанные ниже параметры в текущий Deployment RDS Server:

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: auth-rdsserver
...
spec:
  template:
    spec:
      volumes:
        - name: rds-server-pacman-config
          configMap:
            name: rds-server-pacman-config
            defaultMode: 0600
      ...
      containers:
        - name: auth-rdsserver
          env:
            - name: rds.configSources.priorityList
              value: 'file'
            - name: rds.configSources.sources
              value: 'file@pacman-config/pacman-rdsserver-config.yml'
            - name: rds.configSources.ssl.enabled
              value: 'false'
            - name: rds.standin.rest.enabled
              value: 'true'
            - name: rds.standin.rest.url
              value: 'http://standin.my-aplj-service.mycompany.ru/journal-arm/api/v1/standin/status/im'
            - name: rds.standin.zone.id
              value: 'TEST_ZONE'
            - name: rds.standin.rest.inStandIn
              value: 'false'
            - name: rds.standin.stub
              value: 'true'
          volumeMounts:
            - name: rds-server-pacman-config
              mountPath: /opt/rds-server/pacman-config
              readOnly: true
...
```

Значения параметров выше (rds.standin.rest.url, rds.standin.zone.id) необходимо изменить, в соответствии с реальными
реквизитами стенда.

> При изменении ConfigMap с параметрами, изменения будут автоматически получены RDS Server, без перезапуска pod.