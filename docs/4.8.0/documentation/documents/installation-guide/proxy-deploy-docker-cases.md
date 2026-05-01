# Кейсы использования IAM Proxy в контейнере

## Примечание

> Все компоненты IAM Proxy поддерживает работу контейнера когда корневая ФС в ReadOnly.

## Использование получения секретов из SecMan при старте контейнера IAM Proxy

Секреты доставляются до pod агентом `SecMan` (`vault-agent`), который может быть запущен как init-контейнер, так и как
sidecar. Необходимость установки в pom и запуска агента определяется тегами (`spec.template.metadata.labels`
,`spec.template.metadata.annotations`) в Deployment (`kind: Deployment`).

### Доставка файлов сертификатов для IAM Proxy

Сертификаты размещаются в каталоге /certs (ключи, и доверенные УЦ). Детали по именам файлов в
разделе [Каталоги и файлы для настройки](proxy-deploy-docker-description.md).

### Пример фрагмента Deployment

```yaml
kind: Deployment
apiVersion: apps/v1
spec:
  selector:
    matchLabels:
      deployment: iamproxy-test
  template:
    metadata:
      labels:

        deployment: iamproxy-test
        secman-injector: enabled

      annotations:
        #vault.hashicorp.com/log-level: debug
        vault.hashicorp.com/role: role-ga-secman-iam-proxy
        #vault.hashicorp.com/namespace: mynamespace # указывается если используются отдельные пространства ролей в secman
        vault.hashicorp.com/agent-inject: 'true'
        vault.hashicorp.com/agent-init-first: 'true'
        vault.hashicorp.com/agent-pre-populate-only: 'true'
        vault.hashicorp.com/secret-volume-path: /secrets

        # записываем все значения в файлы /secrets/* из секрета DEV/AUTH/SC/KV/OSE.my-auth-dev-01
        vault.hashicorp.com/agent-inject-secret-builder: 'true'
        vault.hashicorp.com/agent-inject-command-builder: sh -c "cd /secrets ; source ./builder"
        vault.hashicorp.com/agent-inject-template-builder: |
          {{- with secret "DEV/AUTH/SC/KV/OSE.my-auth-dev-01" -}}
          umask 277
          pwd; ls -Ral
            {{ range $k, $v := .Data }}
              {{ $filename := ($k | replaceAll ".base64" "") }}
          echo '*** Create file {{ $filename }}'
              {{- if eq $k $filename }}
          cat > '{{ $filename }}' <<EOF
              {{- else }}
          base64 --decode > '{{ $filename }}' <<EOF
              {{- end }}
          {{ $v }}
          EOF
            {{- end }}
          {{- end }}
          echo $(date)>>builder_finish.info
          ls -Ral

        # записываем все значения в файлы /certs/* из секрета DEV/AUTH/SC/KV/OSE.my-auth-dev-01.certs
        # бинарные файлы в секрете кодируются base64 и к ним добавляется расширение ".base64"
        vault.hashicorp.com/agent-inject-secret-builder-certs: 'true'
        vault.hashicorp.com/secret-volume-path-builder-certs: /certs
        vault.hashicorp.com/agent-inject-command-builder-certs: sh -c "cd /certs ; source ./builder-certs"
        vault.hashicorp.com/agent-inject-template-builder-certs: |
          {{- with secret "DEV/AUTH/SC/KV/OSE.my-auth-dev-01.certs" -}}
          umask 277
          pwd; ls -Ral
            {{ range $k, $v := .Data }}
              {{ $filename := ($k | replaceAll ".base64" "") }}
          echo '*** Create file {{ $filename }}'
              {{- if eq $k $filename }}
          cat > '{{ $filename }}' <<EOF
              {{- else }}
          base64 --decode > '{{ $filename }}' <<EOF
              {{- end }}
          {{ $v }}
          EOF
            {{- end }}
          {{- end }}
          echo $(date)>>builder_finish.info
          ls -Ral
```

Этот пример подключит vault-agent как init-контейнер (`vault-agent-init`), который получит секреты, разложит их по
файлам и завершит свою работу. После успешного завершения init-контейнера будет запушен IAM Proxy, который возьмет все
необходимые секреты из созданных агентом файлов (из каталогов `/certs`, `/secrets`).

### Особенности при использовании Istio/Synapse в namespace

При использовании istio-proxy в поде появляется проблема обращения vault-agent к хранилищу секретов по HTTP, как к
внешнему сервису. Если нет требования обязательно использовать egress для этих обращений, то достаточно в Deployment
добавить порт и/или ip в исключения для istio-proxy:

- `spec.template.metadata.annotations.traffic.sidecar.istio.io/excludeOutboundPorts: '443'`;
- `spec.template.metadata.annotations.traffic.sidecar.istio.io/excludeOutboundIPRanges: '10.x.x.0/24'`.

Если же необходимо обязательно запросы до хранилища секретов выполнять через egress, тогда vault-agent как
init-контейнер использовать не получится, потому-что на init-фазе еще не будет запущен sidecar istio-proxy,
обеспечивающий перенаправление трафика в egress. Vault-agent необходимо запускать как sidecar, дождаться получения всех
секретов, и потом стартовать IAM Proxy (и/или другие sidecar, секреты которых необходимо получить от vault-agent). Для
этого в выше указанной конфигурации Deployment необходимо изменить/установить параметры (
в `spec.template.metadata.annotations`):

```yaml
        sidecar.istio.io/inject: 'true'
        #vault.hashicorp.com/namespace: mynamespace # указывается если используются отдельные пространства ролей в secman
        vault.hashicorp.com/agent-inject: 'true'
        vault.hashicorp.com/agent-init-first: 'false'
        vault.hashicorp.com/agent-pre-populate: 'false'
        vault.hashicorp.com/agent-pre-populate-only: 'false'
```

Для ожидания получения всех секретов установить опцию `WAIT_EXIST_FILES` контейнера IAM Proxy:

- `WAIT_EXIST_FILES=/certs/builder_finish.info /secrets/builder_finish.info`.

Пример фрагмента Deployment с Istio

```yaml
kind: Deployment
apiVersion: apps/v1
spec:
  selector:
    matchLabels:
      deployment: iamproxy-test
  template:
    metadata:
      labels:
        deployment: iamproxy-test
        secman-injector: enabled

      annotations:
        sidecar.istio.io/inject: 'true'

        #vault.hashicorp.com/log-level: debug
        vault.hashicorp.com/role: role-ga-secman-iam-proxy
        #vault.hashicorp.com/namespace: mynamespace # указывается если используются отдельные пространства ролей в secman
        vault.hashicorp.com/agent-inject: 'true'
        vault.hashicorp.com/agent-init-first: 'false'
        vault.hashicorp.com/agent-pre-populate: 'false'
        vault.hashicorp.com/agent-pre-populate-only: 'false'
        vault.hashicorp.com/secret-volume-path: /secrets

        # записываем все значения в файлы /secrets/* из секрета DEV/AUTH/SC/KV/OSE.my-auth-dev-01
        vault.hashicorp.com/agent-inject-secret-builder: 'true'
        vault.hashicorp.com/agent-inject-command-builder: sh -c "cd /secrets ; source ./builder"
        vault.hashicorp.com/agent-inject-template-builder: |
          {{- with secret "DEV/AUTH/SC/KV/OSE.my-auth-dev-01" -}}
          umask 277
          pwd; ls -Ral
            {{ range $k, $v := .Data }}
              {{ $filename := ($k | replaceAll ".base64" "") }}
          echo '*** Create file {{ $filename }}'
              {{- if eq $k $filename }}
          cat > '{{ $filename }}' <<EOF
              {{- else }}
          base64 --decode > '{{ $filename }}' <<EOF
              {{- end }}
          {{ $v }}
          EOF
            {{- end }}
          {{- end }}
          echo $(date)>>builder_finish.info
          ls -Ral

        # записываем все значения в файлы /certs/* из секрета DEV/AUTH/SC/KV/OSE.my-auth-dev-01.certs
        # бинарные файлы в секрете кодируются base64 и к ним добавляется расширение ".base64"
        vault.hashicorp.com/agent-inject-secret-builder-certs: 'true'
        vault.hashicorp.com/secret-volume-path-builder-certs: /certs
        vault.hashicorp.com/agent-inject-command-builder-certs: sh -c "cd /certs ; source ./builder-certs"
        vault.hashicorp.com/agent-inject-template-builder-certs: |
          {{- with secret "DEV/AUTH/SC/KV/OSE.my-auth-dev-01.certs" -}}
          umask 277
          pwd; ls -Ral
            {{ range $k, $v := .Data }}
              {{ $filename := ($k | replaceAll ".base64" "") }}
          echo '*** Create file {{ $filename }}'
              {{- if eq $k $filename }}
          cat > '{{ $filename }}' <<EOF
              {{- else }}
          base64 --decode > '{{ $filename }}' <<EOF
              {{- end }}
          {{ $v }}
          EOF
            {{- end }}
          {{- end }}
          echo $(date)>>builder_finish.info
          ls -Ral

    spec:
      containers:
        - name: iamproxy-test
          env:
            - name: WAIT_EXIST_FILES
              value: '/certs/builder_finish.info /secrets/builder_finish.info'
```

Пример настройки startupProbe/readinessProbe/livenessProbe в манифестах pod IAM Proxy.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: iamproxy
# ...
spec:
  selector:
    matchLabels:
      deployment: iamproxy-test
  template:
    metadata:
      labels:
        deployment: iamproxy-test
    spec:
      containers:
        - name: iamproxy
          ports:
            - containerPort: 10080
              name: http-status
              protocol: TCP
          # ...
          startupProbe:
            initialDelaySeconds: 3
            periodSeconds: 5
            failureThreshold: 3
            httpGet:
              path: /status/startupProbe
              port: http-status
              scheme: HTTP
            successThreshold: 1
            timeoutSeconds: 1
          readinessProbe:
            initialDelaySeconds: 3
            periodSeconds: 5
            failureThreshold: 1
            httpGet:
              path: /status/readinessProbe
              port: http-status
              scheme: HTTP
            successThreshold: 1
            timeoutSeconds: 1
          livenessProbe:
            initialDelaySeconds: 3
            periodSeconds: 30
            failureThreshold: 3
            httpGet:
              path: /status/livenessProbe
              port: http-status
              scheme: HTTP
            successThreshold: 1
            timeoutSeconds: 1
# ...
```

## Передача журналов работы с помощью fluent-bit

Принцип передачи журналов заключается в том, что приложение пишет логи в файлы в определенном формате (в основном в
JSON), а sidecar fluent-bit поднятый на одном pod с приложением эти файлы периодически сканирует и читает, отправляя
результат парсинга в настроенные для отправки каналы (например в kafka или http-api).

Для доставки конфигурации до fluent-bit нужно создать ConfigMap, внутри которой разместить содержимое файлов
fluent-bit.conf и parsers.conf. Ниже из данной ConfigMap будут монтироваться файлы в папку с конфигурацией fluent-bit
контейнера sidecar.

Пример файла для реализации [ose-iamproxy-fluentbit.yaml](resources/ose-iamproxy-fluentbit.yaml).

Чтобы создать тестовое приложение в текущем проекте OSE, можно поместить выше указанный yaml-текст в
iamproxy-test-fluent-bit.yaml и выполнить команды:

```shell
oc login
oc create -f iamproxy-test-fluent-bit.yaml
```

Чтобы удалить примеры созданные выше, выполнить команды:

```shell
oc delete deployment,cm,secret,service,dc --selector app=iamproxy-test-fluent-bit
```

Примечания:

- актуальные input-конфигурации и парсеры fluent-bit под IAM Proxy (и другие компоненты) можно найти в дистрибутиве IAM
  Proxy по пути `fluent-bit/config/`;
- конфигурация fluent-bit очень чувствительна к отступам, и как отступ используется 4 пробела, на это нужно обязательно
  обращать внимание при изменении конфигурации;
- все conf-файлы конфигурации fluent-bit должны быть в одном каталоге (!), там же где основной conf-файл (который
  указывается при запуске fluent-bit), т.к. директива `@INCLUDE` не поддерживает указание произвольного пути к файлам;
- для приема логов вместо файлов можно использовать протокол syslog, и передавать события по локальному сетевому
  соединению, тогда будет исключено исчерпание выделенной контейнеру памяти под размещение логов. Пример:
  ```
    [INPUT]
        Name     syslog
        Tag      logger.proxy.access.syslog
        Listen   127.0.0.1
        Port     15140
        Mode     udp
        Buffer_Chunk_Size 256k
        Buffer_Max_Size 2MB
        Parser syslog-rfc3164
  ```

### Пример IAM Proxy + Hashicorp Vault + Istio + Fluent-bit

В примере ниже используется два хранилища секретов (тип KV) размещенные в Hashicorp Vault :

1. По пути `DEV/AUTH/SC/KV/OSE.my-auth-dev-01`

- PROXY_OIDC_CLIENT_ID
- PROXY_OIDC_CLIENT_SECRET
- PROXY_SESSION_SECRET

2. По пути `DEV/AUTH/SC/KV/OSE.my-auth-dev-01.certs`

- kafka-cacerts.pem
- kafka-cert.pem
- kafka-key.pem
- server.crt.pem
- server.key.pem
- trusted_ca_egress.crt.pem
- trusted_ca_intermediate.crt.pem
- trusted_ca_root.crt.pem

Доступ к этим секретам производится по роли `role-ga-secman-iam-proxy`.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: volume-conf-fluent-bit-iamproxy-sidecar
  labels:
    app: iamproxy-test-fluent-bit
data:
  fluent-bit.conf: |-
    [SERVICE]
        Flush        1
        Daemon       Off
        Parsers_File parsers.conf
        HTTP_Server  On
        HTTP_Listen  0.0.0.0
        HTTP_PORT    ${fluentbit_monitoring_port_number}

    @INCLUDE input_*.conf

    # Добавляем общие атрибуты для любых событий журналов
    # ${env-name} в секции [FILTER] подставляются значениями из переменных среды
    [FILTER]
        Name    modify
        Match   logger.*
        Add     deploymentUnit AUTH
        Add     distribVersion {{ deploy_platformauth_version }}
        Add     subsystem {{ stend_abbr|default('auth',True) }}-iamproxy
        Add     hostName ${HOSTNAME}

    # Добавляем общие атрибуты для любых событий аудита
    [FILTER]
    Name  modify
    Match audit.*
    # атрибуты ниже не являются частью метамодели аудита
    Add   nodeId ${HOSTNAME}

    [OUTPUT]
        Name stdout
        Match *
        Format json_lines
        json_date_format iso8601

    @INCLUDE output_*.conf

  input_proxy.conf: |-
    [INPUT]
        Name              tail
        Tag               proxy.access
        Path              /var/log/logsshare/iamproxy-json-access*.log
        Mem_Buf_Limit     32MB
        Skip_Long_Lines   On
        Refresh_Interval  5
        Rotate_Wait       2
        Read_from_Head    Off
        DB                /var/log/fluent-bit/proxy-access.db
        Parser            json_timestamp_unix

    [INPUT]
        Name              tail
        Tag               proxy.error
        Path              /var/log/logsshare/iamproxy-error*.log
        Mem_Buf_Limit     32MB
        Skip_Long_Lines   On
        Refresh_Interval  5
        Rotate_Wait       2
        Read_from_Head    Off
        DB                /var/log/fluent-bit/proxy-error.db
        Multiline         On
        Parser_Firstline  nginx_error

    [INPUT]
        Name              tail
        Tag               proxy.trace
        Path              /var/log/logsshare/iamproxy-trace-*.log
        Mem_Buf_Limit     32MB
        Skip_Long_Lines   On
        Refresh_Interval  5
        Rotate_Wait       2
        Read_from_Head    Off
        DB                /var/log/fluent-bit/proxy-trace.db
        Parser            json_timestamp_unix

    [FILTER]
        Name   modify
        Match  proxy.*
        Add    moduleId IAM.Proxy
        Add    moduleVersion ${platformauth.proxy.version}
    [FILTER]
        Name   modify
        Match  proxy.access
        Add    type access
    [FILTER]
        Name   modify
        Match  proxy.error
        Add    type error
    [FILTER]
        Name   modify
        Match  proxy.trace
        Add    type trace

  input_rds_client.conf: |-
    [INPUT]
        Name             tail
        Tag              logger.rds_client
        Path             {{ logger_options.rds_client_log_dir|default(proxy_path_target~'/rds-client/logs') ~ '/json*.log' }}
                Buffer_Chunk_Size 256k
        Buffer_Max_Size 2MB
        Mem_Buf_Limit    32MB
        Skip_Long_Lines  Off
        Refresh_Interval 5
        Rotate_Wait      2
        Read_from_Head   On
        DB               /var/log/fluent-bit/rds-client.db
        Parser           json_timestamp_iso8601

    [FILTER]
        Name    modify
        Match   logger.rds_client
        Add     moduleId IAM.RDS-Client
        Add     moduleVersion ${platformauth.rds-client.version}

  output_kafka.conf: |-
    [OUTPUT]
        Name kafka
        Match file.tail
        Brokers ${kafka.bootstrap.servers}
        Topics  ${topic_name}
        rdkafka.security.protocol SSL
        rdkafka.ssl.key.location         /certs/kafka-key.pem
        rdkafka.ssl.certificate.location /certs/kafka-cert.pem
        rdkafka.ssl.ca.location          /certs/kafka-cacerts.pem
        rdkafka.sticky.partitioning.linger.ms 4000
        rdkafka.queue.buffering.max.kbytes 5120

  parsers.conf: |-
    [PARSER]
        Name        json_timestamp_unix
        Format      json
        Time_Key    timestamp
        # время с начала эпохи в секундах
        Time_Format %s.%L

    [PARSER]
        Name        json_timestamp_iso8601
        Format      json
        Time_Key    timestamp
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z

    [PARSER]
        Name        nginx_error
        Format      regex
        Regex       /^(?<timestamp>[\d\/]{10} [\d\:]{8}) \[(?<level>[a-zA-Z]+)\] (?<proc>\d+)#(?<subproc>\d+)\:( \*(?<thread>\d+))? (?<message>.*)/
        Time_Key    timestamp
        Time_Format %Y/%m/%d %H:%M:%S

    # ---- optional, and not used is now ----
    [PARSER]
        Name        nginx
        Format      regex
        Regex       ^(?<remote>[^ ]*) (?<host>[^ ]*) (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^\"]*?)(?: +\S*)?)?" (?<code>[^ ]*) (?<size>[^ ]*)(?: "(?<referer>[^\"]*)" "(?<agent>[^\"]*)")
        Time_Key    timestamp
        Time_Format %d/%b/%Y:%H:%M:%S %z

    [PARSER]
        Name loggermod_head
        Format regex
        Regex ^(?<time>[^ ]* {1,2}[^ ]*) \[(?<threadName>[^ ]*)\] \[(?<levelStr>[^ ]*)\] \((?<loggerName>[^ ]*)\) \[(?<callerClass>[^:]*)\:\:(?<callerMethod>[^:]*)\:(?<callerLine>[^ ]*)\](\s+mdc:)(?<mdc>[^|]*)(\|\s+)(?<message>.*)
        Time_Key time
        Time_Format %Y-%m-%d %H:%M:%S,%L
        Time_Keep On

    [PARSER]
        Name loggermod
        Format regex
        Regex (?m-ix)^(?<time>[^ ]* {1,2}[^ ]*) \[(?<threadName>[^ ]*)\] \[(?<levelStr>[^ ]*)\] \((?<loggerName>[^ ]*)\) \[(?<callerClass>[^:]*)\:\:(?<callerMethod>[^:]*)\:(?<callerLine>[^ ]*)\](\s+mdc:)(?<mdc>[^|]*)(\|\s+)(?<message>.*)
        Time_Key time
        Time_Format %Y-%m-%d %H:%M:%S,%L
        Time_Keep On

    [PARSER]
        Name        fluentbit-rfc5424
        Format      regex
        Regex       ^\<(?<pri>[0-9]{1,5})\>1 (?<time>[^ ]+) (?<host>[^ ]+) (?<ident>[^ ]+) (?<pid>[-0-9]+) (?<msgid>[^ ]+) (?<extradata>(\[(.*?)\]|-)) (?<message>.+)$
        Time_Key    time
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z
        Time_Keep   On

    [PARSER]
        Name        fluentbit-rfc3164
        Format      regex
        Regex       /^\<(?<pri>[0-9]+)\>(?<time>[^ ]* {1,2}[^ ]* [^ ]*) (?<host>[^ ]*) (?<ident>[a-zA-Z0-9_\/\.\-]*)(?:\[(?<pid>[0-9]+)\])?(?:[^\:]*\:)? *(?<message>.*)$/
        Time_Key    time
        Time_Format %b %d %H:%M:%S
        Time_Keep   On

    [PARSER]
        Name        json_serverEventDatetime
        Format      json
        Time_Key    serverEventDatetime
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: conf-fluent-bit-iamproxy-sidecar
  labels:
    app: iamproxy-test-fluent-bit
data:
  fluentbit_monitoring_port_number: '8081'
  stend_abbr: "dev"
  deploy_platformauth_version: "D-01.01.01"
  platformauth.proxy.version: "4.3.1"
  platformauth.rds-client.version: "4.2.0"
  kafka.bootstrap.servers: "10.x.x.1"
  topic_name: "iamproxy"

---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: iamproxy-test5-alt2
  labels:
    app: iamproxy-test-fluent-bit
spec:
  selector:
    matchLabels:
      app: iamproxy-test-fluent-bit
  template:
    metadata:
      labels:
        app: iamproxy-test-fluent-bit
        secman-injector: enabled
      annotations:
        sidecar.istio.io/inject: 'true'
        vault.hashicorp.com/role: role-ga-secman-iam-proxy
        #vault.hashicorp.com/namespace: mynamespace # указывается если используются отдельные пространства ролей в secman
        vault.hashicorp.com/agent-inject: 'true'
        vault.hashicorp.com/agent-init-first: 'false'
        vault.hashicorp.com/agent-pre-populate: 'false'
        vault.hashicorp.com/agent-pre-populate-only: 'false'
        vault.hashicorp.com/secret-volume-path: /secrets

        # записываем все значения в файлы /secrets/* из секрета DEV/AUTH/SC/KV/OSE.my-auth-dev-01
        vault.hashicorp.com/agent-inject-secret-builder: 'true'
        vault.hashicorp.com/agent-inject-command-builder: sh -c "cd /secrets ; source ./builder"
        vault.hashicorp.com/agent-inject-template-builder: |
          {{- with secret "DEV/AUTH/SC/KV/OSE.my-auth-dev-01" -}}
          umask 277
          pwd; ls -Ral
            {{ range $k, $v := .Data }}
              {{ $filename := ($k | replaceAll ".base64" "") }}
          echo '*** Create file {{ $filename }}'
              {{- if eq $k $filename }}
          cat > '{{ $filename }}' <<EOF
              {{- else }}
          base64 --decode > '{{ $filename }}' <<EOF
              {{- end }}
          {{ $v }}
          EOF
            {{- end }}
          {{- end }}
          echo $(date)>>builder_finish.info
          ls -Ral

        # записываем все значения в файлы /certs/* из секрета DEV/AUTH/SC/KV/OSE.my-auth-dev-01.certs
        # бинарные файлы в секрете кодируются base64 и к ним добавляется расширение ".base64"
        vault.hashicorp.com/agent-inject-secret-builder-certs: 'true'
        vault.hashicorp.com/secret-volume-path-builder-certs: /certs
        vault.hashicorp.com/agent-inject-command-builder-certs: sh -c "cd /certs ; source ./builder-certs"
        vault.hashicorp.com/agent-inject-template-builder-certs: |
          {{- with secret "DEV/AUTH/SC/KV/OSE.my-auth-dev-01.certs" -}}
          umask 277
          pwd; ls -Ral
            {{ range $k, $v := .Data }}
              {{ $filename := ($k | replaceAll ".base64" "") }}
          echo '*** Create file {{ $filename }}'
              {{- if eq $k $filename }}
          cat > '{{ $filename }}' <<EOF
              {{- else }}
          base64 --decode > '{{ $filename }}' <<EOF
              {{- end }}
          {{ $v }}
          EOF
            {{- end }}
          {{- end }}
          echo $(date)>>builder_finish.info
          ls -Ral

    spec:
      volumes:
        - name: tmp
          emptyDir:
            medium: Memory
        - name: volume-logsshare
          emptyDir: { }
        - name: volume-fluent-bit-log
          emptyDir: { }
        - name: volume-conf-fluent-bit-iamproxy-sidecar
          configMap:
            name: volume-conf-fluent-bit-iamproxy-sidecar

      containers:
        - resources:
            limits:
              cpu: '1'
              memory: 512Mi
            requests:
              cpu: 100m
              memory: 256Mi
          name: iamproxy-test5-alt2
          env:
            - name: DEBUG
              value: '1'
            - name: KEYCLOAK_FRONT_DNS_NAME
              value: platformauth-devb.mycompany.ru
            - name: KEYCLOAK_HTTPS_PORT_ON_FRONTEND
              value: '443'
            - name: OIDC_SSL_VERIFY
              value: 'False'
            - name: PROXY_DNSNAME
              value: iamproxy.mycompany.ru
            - name: PROXY_OIDC_CLIENT_ID
              value: '@'
            - name: PROXY_OIDC_CLIENT_SECRET
              value: '@'
            - name: PROXY_SESSION_SECRET
              value: '@'
            - name: STEND_TYPE
              value: dev
            - name: PROXY_LOG_TO_FLUENT_BIT_ENABLE
              value: 'True'
            - name: WAIT_EXIST_FILES
              value: '/certs/builder_finish.info /secrets/builder_finish.info'
          ports:
            - containerPort: 10080
              protocol: TCP
            - containerPort: 8080
              protocol: TCP
            - containerPort: 8443
              protocol: TCP
            - containerPort: 8444
              protocol: TCP
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: tmp
              mountPath: /tmp/platformauth-proxy
            - mountPath: /var/log/logsshare
              name: volume-logsshare
          image: >-
            mycompany.sw.sbc.space/mycompany_dev/ci90000019_iamproxy_dev/platformauth-proxy@sha256:5b107927113b6d784f1c5d4d6978404d9d39f8698a428da26883bb426251e74c
          securityContext:
            readOnlyRootFilesystem: true
            runAsNonRoot: true

        - name: fluent-bit-sidecar
          command:
            - sh
            - -c
            - |
              filename=/certs/builder-certs
              echo -n "Waiting to exist $filename ... "
              until [ -e "$filename" ] ; do
                 echo -n "."
                 sleep 1
              done
              #/bin/sh/docker-entrypoint.sh
              echo -e "\n*** Starting Fluent-bit ***"
              exec /opt/td-agent-bit/bin/td-agent-bit -c /tmp/fluent-bit/fluent-bit.conf

          image: >-
            mycompany.sw.sbc.space/mycompany_dev/ci90000016_kintsugi_dev/external/fluent-bit@sha256:e38955b3495fe6a40b0506326c1d8458e2f03ad67747406af75f3719678b6036
          resources:
            limits:
              cpu: 500m
              memory: 256Mi
          env:
            - name: podname
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
          envFrom:
            - configMapRef:
                name: conf-fluent-bit-iamproxy-sidecar
          volumeMounts:
            - mountPath: '/tmp/fluent-bit'
              name: volume-conf-fluent-bit-iamproxy-sidecar
            - mountPath: '/var/log/fluent-bit'
              name: volume-fluent-bit-log
            - mountPath: '/var/log/logsshare'
              name: volume-logsshare
          securityContext:
            readOnlyRootFilesystem: true
            runAsNonRoot: true
```

## Полный пример конфигурации включая настройку Istio/Mesh

Ingress и Egress в примере используются общие из NS с Control Plane (SMCP), развернутые в OSE оператором SMCP.

Значения, которые потребуется заменить на свои при адаптации конфигурации под стенд:

- `iamproxy-with-fluent-bit` - имя сервиса;
- `ingress-auth.apps.stands-vdc01.solution.mycompany` - NS в котором размещены Ingress, Egress;
- `platformauth-devb.mycompany.ru` - Open ID Connect Provider (IDP), обеспечивающий аутентификацию по OIDC;
- `iamproxy.mycompany.ru` - FQDN по которому будет доступен сервис для клиентов/браузеров;
- `mycompany.sw.sbc.space/mycompany_dev/ci90000019_iamproxy_dev/platformauth-proxy@sha256:00aefaf3f52cdb6fa6e65feb91a76a6fb15ed58dfddc72c1c8e8e439c48c485c` -
путь до образа IAM Proxy;
- `mycompany.sw.sbc.space/mycompany_dev/ci90000016_kintsugi_dev/external/fluent-bit@sha256:e38955b3495fe6a40b0506326c1d8458e2f03ad67747406af75f3719678b6036` -
путь до образа Fluent-bit от сервиса Журналирования.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: istio-egressgateway # определяем egress для трафика из нашего NS, и то, какие запросы он будет уметь маршрутизировать через себя
spec:
  selector:
    istio: egressgateway # label по которому выбираются pods egress для применения данной конфигурации на них
  servers:
    - hosts:
        - platformauth-devb.mycompany.ru # здесь и ниже это IDP (Keycloak) где делаем аутентификацию, который вне mesh
      port:
        name: https-port-for-tls
        number: 443 # порт должен присутствовать в сервисе egress, если его там нет то надо добавить !!!
        protocol: HTTPS
      tls:
        mode: SIMPLE
        privateKey: /etc/istio/egressgateway-certs/tls.key # сертификаты от УЦ на вход в egress, размещаем сами отдельно на pod-ах egress
        serverCertificate: /etc/istio/egressgateway-certs/tls.crt
    - hosts:
        - secman-mycompany.solution.mycompany # здесь и ниже это наш Hashicorp Vault , где получаем секреты, который вне mesh
      port:
        name: https-port-for-tls-8443
        number: 8443
        protocol: HTTPS
      tls:
        mode: SIMPLE
        privateKey: /etc/istio/egressgateway-certs/tls.key
        serverCertificate: /etc/istio/egressgateway-certs/tls.crt
---
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: istio-ingressgateway # определяем ингресс для трафика входящего в наш NS, и то, какие запросы он будет уметь маршрутизировать через себя
spec:
  selector:
    istio: ingressgateway
  servers:
    - hosts:
        - iamproxy.mycompany.ru # хост используемый в клиенте/браузере для доступа к сервисам, регистрируем в DNS на IPs кластера или как алиас dns apps.stands-vdc01.solution.mycompany
        - ingress-auth.apps.stands-vdc01.solution.mycompany
      port:
        name: https
        number: 443 # порт используемый в клиенте/браузере для доступа к сервисам
        protocol: HTTPS
      tls:
        mode: SIMPLE
        privateKey: /etc/istio/ingressgateway-certs/tls.key # сертификаты от УЦ на вход в ингресс, они будут видны из браузера (в SAN не забываем добавить хост)
        serverCertificate: /etc/istio/ingressgateway-certs/tls.crt
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: direct-idp-through-egress-gateway # тут описана маршрутизация до IDP
spec:
  exportTo:
    - .
    - tribe-sc-auth-istio-dev-01 # здесь и ниже это NS в котором развернут SMCP (все компоненты mesh)
  gateways:
    - mesh
    - istio-egressgateway
  hosts:
    - platformauth-devb.mycompany.ru
  http:
    - match: # перенаправление с егресса в наружу mesh на IDP
        - gateways:
            - istio-egressgateway
          port: 443
      route:
        - destination:
            host: platformauth-devb.mycompany.ru
            port:
              number: 443
          weight: 100
  tls:
    - match: # перенаправление на егресс
        - gateways:
            - mesh
          port: 443
          sniHosts:
            - platformauth-devb.mycompany.ru
      route:
        - destination:
            host: istio-egressgateway.tribe-sc-auth-istio-dev-01.svc.cluster.local
            port:
              number: 443
            subset: idp-subset
    - match: # перенаправление для passthrough с егресса в наружу mesh в IDP (этот блок не используется, и можно удалить)
        - gateways:
            - istio-egressgateway
          port: 443
          sniHosts:
            - platformauth-devb.mycompany.ru
      route:
        - destination:
            host: platformauth-devb.mycompany.ru
            port:
              number: 443
          weight: 100
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: direct-secman-through-egress-gateway
spec:
  exportTo:
    - .
    - tribe-sc-auth-istio-dev-01
  gateways:
    - mesh
    - istio-egressgateway
  hosts:
    - secman-mycompany.solution.mycompany
  http:
    - match: # перенаправление с егресса в наружу mesh на SecMan
        - gateways:
            - istio-egressgateway
          port: 8443
      route:
        - destination:
            host: secman-mycompany.solution.mycompany
            port:
              number: 8443
          weight: 100
  tls:
    - match: # перенаправление на егресс
        - gateways:
            - mesh
          port: 8443
          sniHosts:
            - secman-mycompany.solution.mycompany
      route:
        - destination:
            host: istio-egressgateway.tribe-sc-auth-istio-dev-01.svc.cluster.local
            port:
              number: 8443
            subset: secman-subset
    - match: # перенаправление для passthrough с egress в наружу mesh в Hashicorp Vault  (этот блок не используется, и можно удалить)
        - gateways:
            - istio-egressgateway
          port: 8443
          sniHosts:
            - secman-mycompany.solution.mycompany
      route:
        - destination:
            host: secman-mycompany.solution.mycompany
            port:
              number: 8443
          weight: 100
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: from-ingress-to-iamproxy
spec:
  exportTo:
    - .
    - tribe-sc-auth-istio-dev-01
  gateways:
    - istio-ingressgateway
  hosts:
    - 'iamproxy.mycompany.ru' # указываем фронтовое FQDN имя iamproxy, по которому обращается клиент/браузер
  http:
    - match: # перенаправляем все с ингресс на http iamproxy
        - gateways:
            - istio-ingressgateway
      route:
        - destination:
            host: iamproxy-with-fluent-bit # имя сервиса iamproxy внутри текущего NS
            port:
              number: 8080 # подключаться по порту с http к iamproxy (трафик внутри NS шифруется средствами istio, см. DR iamproxy-with-fluent-bit-istio-mtls)
          weight: 100
---
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: auth-idp # регистрация IDP, как внешнего для mesh сервиса
spec:
  exportTo:
    - tribe-sc-auth-istio-dev-01
    - .
  hosts:
    - platformauth-devb.mycompany.ru
  ports:
    - name: https
      number: 443
      protocol: HTTPS
  resolution: DNS
---
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: secman # регистрация Hashicorp Vault , как внешнего для mesh сервиса
spec:
  exportTo:
    - tribe-sc-auth-istio-dev-01
    - .
  hosts:
    - secman-mycompany.solution.mycompany
  ports:
    - name: https
      number: 8443
      protocol: HTTPS
  resolution: DNS
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: egressgateway-for-auth-idp # отключение создания TLS на sidecar istio-proxy для запросов в сторону егресса (TLS делает сам iamproxy)
spec:
  exportTo:
    - .
  host: istio-egressgateway.tribe-sc-auth-istio-dev-01.svc.cluster.local
  subsets:
    - name: idp-subset
      trafficPolicy:
        tls:
          mode: DISABLE
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: egressgateway-for-secman # отключение создания TLS на sidecar istio-proxy для запросов в сторону egress (TLS делает сам Hashicorp Vault )
spec:
  exportTo:
    - .
  host: istio-egressgateway.tribe-sc-auth-istio-dev-01.svc.cluster.local
  subsets:
    - name: secman-subset
      trafficPolicy:
        tls:
          mode: DISABLE
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: iamproxy-with-fluent-bit-istio-mtls # накладываем требование по mTLS, и его организацию, весь трафик в сторону iamproxy будет с mTLS на серт-ах istio
spec:
  exportTo:
    - .
    - tribe-sc-auth-istio-dev-01
  host: iamproxy-with-fluent-bit
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: originate-tls-for-auth-idp # накладываем требование по TLS, весь трафик (в нашем случае с егресс) в сторону IDP должен быть с TLS
spec:
  exportTo:
    - tribe-sc-auth-istio-dev-01
  host: platformauth-devb.mycompany.ru
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
    portLevelSettings:
      - port:
          number: 443
        tls:
          mode: SIMPLE
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: originate-tls-for-secman # накладываем требование по mTLS, весь трафик (в нашем случае с egress) в сторону Hashicorp Vault  должен быть с mTLS
spec:
  exportTo:
    - tribe-sc-auth-istio-dev-01
  host: secman-mycompany.solution.mycompany
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
    tls:
      caCertificates: /etc/istio/egressgateway-ca-certs/ca-secman.crt # доверенные сертификаты УЦ Hashicorp Vault , заносим на pods egress их отдельно
      clientCertificate: /etc/istio/egressgateway-certs/ext-mtls.crt # клиентский сертификат от УЦ для подключения к Hashicorp Vault , заносим на pods egress их отдельно
      mode: MUTUAL
      privateKey: /etc/istio/egressgateway-certs/ext-mtls.key
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: iamproxy-with-fluent-bit
  name: iamproxy-test
spec:
  selector:
    matchLabels:
      app: iamproxy-with-fluent-bit
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      annotations:
        sidecar.istio.io/inject: "true"
        vault.hashicorp.com/agent-extra-secret: egress-ca-certs
        vault.hashicorp.com/agent-init-first: "false"
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/agent-inject-command-builder: sh -c "cd /secrets ; source ./builder"
        vault.hashicorp.com/agent-inject-command-builder-certs: sh -c "cd /certs ; source ./builder-certs"
        vault.hashicorp.com/agent-inject-secret-builder: "true"
        vault.hashicorp.com/agent-inject-secret-builder-certs: "true"
        vault.hashicorp.com/agent-inject-template-builder: |
          {{- with secret "DEV/AUTH/SC/KV/OSE.my-auth-dev-01" -}}
          umask 277
          pwd; ls -Ral
            {{ range $k, $v := .Data }}
              {{ $filename := ($k | replaceAll ".base64" "") }}
          echo '*** Create file {{ $filename }}'
              {{- if eq $k $filename }}
          cat > '{{ $filename }}' <<EOF
              {{- else }}
          base64 --decode > '{{ $filename }}' <<EOF
              {{- end }}
          {{ $v }}
          EOF
            {{- end }}
          {{- end }}
          echo $(date)>>builder_finish.info
          ls -Ral
        vault.hashicorp.com/agent-inject-template-builder-certs: |
          {{- with secret "DEV/AUTH/SC/KV/OSE.my-auth-dev-01.certs" -}}
          umask 277
          pwd; ls -Ral
            {{ range $k, $v := .Data }}
              {{ $filename := ($k | replaceAll ".base64" "") }}
          echo '*** Create file {{ $filename }}'
              {{- if eq $k $filename }}
          cat > '{{ $filename }}' <<EOF
              {{- else }}
          base64 --decode > '{{ $filename }}' <<EOF
              {{- end }}
          {{ $v }}
          EOF
            {{- end }}
          {{- end }}
          echo $(date)>>builder_finish.info
          ls -Ral
        vault.hashicorp.com/agent-pre-populate: "false"
        vault.hashicorp.com/agent-pre-populate-only: "false"
        vault.hashicorp.com/ca-cert: /vault/custom/cacert
        vault.hashicorp.com/log-level: info
        vault.hashicorp.com/role: role-ga-secman-iam-proxy
        vault.hashicorp.com/secret-volume-path: /secrets
        vault.hashicorp.com/secret-volume-path-builder-certs: /certs
        vault.hashicorp.com/tls-skip-verify: "true" # установить в false на НЕ тестовых стендах, при этом сертификат на egress должен в SAN содержать FQDN сервера Hashicorp Vault
      labels:
        app: iamproxy-with-fluent-bit
        secman-injector: enabled
    spec:
      containers:
        - env:
            - name: DEBUG # удалить параметр на НЕ тестовых стендах
              value: "1"
            - name: KEYCLOAK_FRONT_DNS_NAME
              value: platformauth-devb.mycompany.ru
            - name: KEYCLOAK_HTTPS_PORT_ON_FRONTEND
              value: "443"
            - name: OIDC_SSL_VERIFY # установить в True на НЕ тестовых стендах, при этом сертификат на egress должен в SAN содержать FQDN IDP (в этом примере - platformauth-devb.mycompany.ru)
              value: "False"
            - name: PROXY_DNSNAME
              value: iamproxy.mycompany.ru
            - name: PROXY_OIDC_CLIENT_ID
              value: '@'
            - name: PROXY_OIDC_CLIENT_SECRET
              value: '@'
            - name: PROXY_SESSION_SECRET
              value: '@'
            - name: STEND_TYPE
              value: dev
            - name: PROXY_LOG_TO_FLUENT_BIT_ENABLE
              value: "True"
            - name: WAIT_EXIST_FILES
              value: /certs/builder_finish.info /secrets/builder_finish.info
          image: mycompany.sw.sbc.space/mycompany_dev/ci90000019_iamproxy_dev/platformauth-proxy@sha256:00aefaf3f52cdb6fa6e65feb91a76a6fb15ed58dfddc72c1c8e8e439c48c485c
          imagePullPolicy: IfNotPresent
          livenessProbe:
            failureThreshold: 2
            httpGet:
              path: /status/livenessProbe
              port: http-status
              scheme: HTTP
            initialDelaySeconds: 3
            periodSeconds: 30
            successThreshold: 1
            timeoutSeconds: 1
          name: iamproxy-test
          ports:
            - containerPort: 10080
              name: http-status
              protocol: TCP
            - containerPort: 8080
              protocol: TCP
            - containerPort: 8443
              protocol: TCP
            - containerPort: 8444
              protocol: TCP
          startupProbe:
            failureThreshold: 1000
            httpGet:
              path: /status/startupProbe
              port: http-status
              scheme: HTTP
            initialDelaySeconds: 3
            periodSeconds: 5
            successThreshold: 1
            timeoutSeconds: 1
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /status/readinessProbe
              port: http-status
              scheme: HTTP
            initialDelaySeconds: 3
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          resources:
            limits:
              cpu: "1"
              memory: 512Mi
            requests:
              cpu: 100m
              memory: 256Mi
          securityContext:
            readOnlyRootFilesystem: true
            runAsNonRoot: true
          volumeMounts:
            - mountPath: /tmp/platformauth-proxy
              name: tmp
            - mountPath: /var/log/logsshare
              name: volume-logsshare
            - mountPath: /var/run/secrets/ca-certs
              name: egress-ca-certs
              readOnly: true
            - mountPath: /iamproxy/conf/custom.d
              name: vloume-configs
        - command:
            - sh
            - -c
            - |
              filename=/certs/builder-certs
              echo -n "Waiting to exist $filename ... "
              until [ -e "$filename" ] ; do
                 echo -n "."
                 sleep 1
              done
              #/bin/sh/docker-entrypoint.sh
              echo -e "\n*** Starting Fluent-bit ***"
              exec /opt/td-agent-bit/bin/td-agent-bit -c /tmp/fluent-bit/fluent-bit.conf
          env:
            - name: podname
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.name
          envFrom:
            - configMapRef:
                name: conf-fluent-bit-iamproxy-sidecar
          image: mycompany.sw.sbc.space/mycompany_dev/ci90000016_kintsugi_dev/external/fluent-bit@sha256:e38955b3495fe6a40b0506326c1d8458e2f03ad67747406af75f3719678b6036
          imagePullPolicy: IfNotPresent
          name: fluent-bit-sidecar
          resources:
            limits:
              cpu: 200m
              memory: 256Mi
          securityContext:
            readOnlyRootFilesystem: true
            runAsNonRoot: true
          volumeMounts:
            - mountPath: /tmp/fluent-bit
              name: volume-conf-fluent-bit-iamproxy-sidecar
            - mountPath: /var/log/fluent-bit
              name: volume-fluent-bit-log
            - mountPath: /var/log/logsshare
              name: volume-logsshare
            - mountPath: /var/run/secrets/ca-certs
              name: egress-ca-certs
              readOnly: true
      volumes:
        - emptyDir:
            medium: Memory
          name: tmp
        - configMap:
            defaultMode: 420
            name: iam-proxy-http-on
          name: vloume-configs
        - emptyDir: { }
          name: volume-logsshare
        - emptyDir: { }
          name: volume-fluent-bit-log
        - configMap:
            defaultMode: 420
            name: volume-conf-fluent-bit-iamproxy-sidecar
          name: volume-conf-fluent-bit-iamproxy-sidecar
        - name: egress-ca-certs
          secret:
            defaultMode: 420
            secretName: egress-ca-certs
---
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    app: iamproxy-with-fluent-bit
  name: conf-fluent-bit-iamproxy-sidecar
data:
  # тут указываются версии компонент, и другие параметры необходимые для отправки в сервис журналирования
  deploy_platformauth_version: "D-01.01.01" # версия поставки/дистрибутива iamproxy
  fluentbit_monitoring_port_number: "8081" # порт на котором публиковать метрики fluentbit
  platformauth.proxy.version: "4.3.1" # версия компонента iamproxy
  platformauth.rds-client.version: "4.2.0" # версия компонента rds-client
  stend_abbr: "dev" # тип стенда
  kafka.bootstrap.servers: "10.x.x.1:9092,service-kafka-logger:9092" # сервера кафка, куда отправлять события (при внешнем сервисе нужно будет указать имя:port ведущие на egress)
  topic_name: "iamproxy" # топик кафка, куда отправлять события
---
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    app: iamproxy-with-fluent-bit
  name: volume-conf-fluent-bit-iamproxy-sidecar
data:
  fluent-bit.conf: |-
    [SERVICE]
        Flush        1
        Daemon       Off
        Parsers_File parsers.conf
        HTTP_Server  On
        HTTP_Listen  0.0.0.0
        HTTP_PORT    ${fluentbit_monitoring_port_number}

    @INCLUDE input_*.conf

    # Добавляем общие атрибуты для любых событий журналов
    # ${env-name} в секции [FILTER] подставляются значениями из переменных среды
    [FILTER]
        Name    modify
        Match   logger.*
        Add     deploymentUnit AUTH
        Add     distribVersion {{ deploy_platformauth_version }}
        Add     subsystem {{ stend_abbr|default('auth',True) }}-iamproxy
        Add     hostName ${HOSTNAME}

    # Добавляем общие атрибуты для любых событий аудита
    [FILTER]
    Name  modify
    Match audit.*
    # атрибуты ниже не являются частью метамодели аудита
    Add   nodeId ${HOSTNAME}

    [OUTPUT]
        Name stdout
        Match *
        Format json_lines
        json_date_format iso8601

    @INCLUDE output_*.conf
  input_proxy.conf: |-
    [INPUT]
        Name              tail
        Tag               proxy.access
        Path              /var/log/logsshare/iamproxy-json-access*.log
        Mem_Buf_Limit     32MB
        Skip_Long_Lines   On
        Refresh_Interval  5
        Rotate_Wait       2
        Read_from_Head    Off
        DB                /var/log/fluent-bit/proxy-access.db
        Parser            json_timestamp_unix

    [INPUT]
        Name              tail
        Tag               proxy.error
        Path              /var/log/logsshare/iamproxy-error*.log
        Mem_Buf_Limit     32MB
        Skip_Long_Lines   On
        Refresh_Interval  5
        Rotate_Wait       2
        Read_from_Head    Off
        DB                /var/log/fluent-bit/proxy-error.db
        Multiline         On
        Parser_Firstline  nginx_error

    [INPUT]
        Name              tail
        Tag               proxy.trace
        Path              /var/log/logsshare/iamproxy-trace-*.log
        Mem_Buf_Limit     32MB
        Skip_Long_Lines   On
        Refresh_Interval  5
        Rotate_Wait       2
        Read_from_Head    Off
        DB                /var/log/fluent-bit/proxy-trace.db
        Parser            json_timestamp_unix

    [FILTER]
        Name   modify
        Match  proxy.*
        Add    moduleId IAM.Proxy
        Add    moduleVersion ${platformauth.proxy.version}
    [FILTER]
        Name   modify
        Match  proxy.access
        Add    type access
    [FILTER]
        Name   modify
        Match  proxy.error
        Add    type error
    [FILTER]
        Name   modify
        Match  proxy.trace
        Add    type trace
  input_rds_client.conf: |-
    [INPUT]
        Name             tail
        Tag              logger.rds_client
        Path             {{ logger_options.rds_client_log_dir|default(proxy_path_target~'/rds-client/logs') ~ '/json*.log' }}
        Buffer_Chunk_Size 256k
        Buffer_Max_Size 2MB
        Mem_Buf_Limit    32MB
        Skip_Long_Lines  Off
        Refresh_Interval 5
        Rotate_Wait      2
        Read_from_Head   On
        DB               /var/log/fluent-bit/rds-client.db
        Parser           json_timestamp_iso8601

    [FILTER]
        Name    modify
        Match   logger.rds_client
        Add     moduleId IAM.RDS-Client
        Add     moduleVersion ${platformauth.rds-client.version}
  output_kafka.conf: |-
    [OUTPUT]
        Name kafka
        Match file.tail
        Brokers ${kafka.bootstrap.servers}
        Topics  ${topic_name}
        rdkafka.security.protocol SSL
        rdkafka.ssl.key.location         /certs/kafka-key.pem
        rdkafka.ssl.certificate.location /certs/kafka-cert.pem
        rdkafka.ssl.ca.location          /certs/kafka-cacerts.pem
        rdkafka.sticky.partitioning.linger.ms 4000
        rdkafka.queue.buffering.max.kbytes 5120
  parsers.conf: |-
    [PARSER]
        Name        json_timestamp_unix
        Format      json
        Time_Key    timestamp
        # время с начала эпохи в секундах
        Time_Format %s.%L

    [PARSER]
        Name        json_timestamp_iso8601
        Format      json
        Time_Key    timestamp
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z

    [PARSER]
        Name        nginx_error
        Format      regex
        Regex       /^(?<timestamp>[\d\/]{10} [\d\:]{8}) \[(?<level>[a-zA-Z]+)\] (?<proc>\d+)#(?<subproc>\d+)\:( \*(?<thread>\d+))? (?<message>.*)/
        Time_Key    timestamp
        Time_Format %Y/%m/%d %H:%M:%S

    # ---- optional, and not used is now ----
    [PARSER]
        Name        nginx
        Format      regex
        Regex       ^(?<remote>[^ ]*) (?<host>[^ ]*) (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^\"]*?)(?: +\S*)?)?" (?<code>[^ ]*) (?<size>[^ ]*)(?: "(?<referer>[^\"]*)" "(?<agent>[^\"]*)")
        Time_Key    timestamp
        Time_Format %d/%b/%Y:%H:%M:%S %z

    [PARSER]
        Name loggermod_head
        Format regex
        Regex ^(?<time>[^ ]* {1,2}[^ ]*) \[(?<threadName>[^ ]*)\] \[(?<levelStr>[^ ]*)\] \((?<loggerName>[^ ]*)\) \[(?<callerClass>[^:]*)\:\:(?<callerMethod>[^:]*)\:(?<callerLine>[^ ]*)\](\s+mdc:)(?<mdc>[^|]*)(\|\s+)(?<message>.*)
        Time_Key time
        Time_Format %Y-%m-%d %H:%M:%S,%L
        Time_Keep On

    [PARSER]
        Name loggermod
        Format regex
        Regex (?m-ix)^(?<time>[^ ]* {1,2}[^ ]*) \[(?<threadName>[^ ]*)\] \[(?<levelStr>[^ ]*)\] \((?<loggerName>[^ ]*)\) \[(?<callerClass>[^:]*)\:\:(?<callerMethod>[^:]*)\:(?<callerLine>[^ ]*)\](\s+mdc:)(?<mdc>[^|]*)(\|\s+)(?<message>.*)
        Time_Key time
        Time_Format %Y-%m-%d %H:%M:%S,%L
        Time_Keep On

    [PARSER]
        Name        fluentbit-rfc5424
        Format      regex
        Regex       ^\<(?<pri>[0-9]{1,5})\>1 (?<time>[^ ]+) (?<host>[^ ]+) (?<ident>[^ ]+) (?<pid>[-0-9]+) (?<msgid>[^ ]+) (?<extradata>(\[(.*?)\]|-)) (?<message>.+)$
        Time_Key    time
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z
        Time_Keep   On

    [PARSER]
        Name        fluentbit-rfc3164
        Format      regex
        Regex       /^\<(?<pri>[0-9]+)\>(?<time>[^ ]* {1,2}[^ ]* [^ ]*) (?<host>[^ ]*) (?<ident>[a-zA-Z0-9_\/\.\-]*)(?:\[(?<pid>[0-9]+)\])?(?:[^\:]*\:)? *(?<message>.*)$/
        Time_Key    time
        Time_Format %b %d %H:%M:%S
        Time_Keep   On

    [PARSER]
        Name        json_serverEventDatetime
        Format      json
        Time_Key    serverEventDatetime
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z

---
apiVersion: v1
kind: Service
metadata:
  name: iamproxy-with-fluent-bit
spec:
  ports:
    - name: https
      port: 443
      protocol: TCP
      targetPort: "8443"
    - name: http
      port: 8080
      protocol: TCP
      targetPort: "8080"
  selector:
    app: iamproxy-with-fluent-bit
  type: ClusterIP
---
apiVersion: v1
kind: Secret
metadata:
  name: egress-ca-certs
type: Opaque
data:
  cacert: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURMVENDQWhXZ0F3SUJBZ0lKQU4yM0MzUXdDN29ETUEwR0NTcUdTSWIzRFFFQkN3VUFNQzB4RlRBVEJnTlYKQkFvTURHVjRZVzF3YkdVZ1NXNWpMakVVTUJJR0ExVUVBd3dMWlhoaGJYQnNaUzVqYjIwd0hoY05Nakl4TURJNApNVE16TWpVNVdoY05Nak14TURJNE1UTXpNalU1V2pBdE1SVXdFd1lEVlFRS0RBeGxlR0Z0Y0d4bElFbHVZeTR4CkZEQVNCZ05WQkFNTUMyVjRZVzF3YkdVdVkyOXRNSUlCSWpBTkJna3Foa2lHOXcwQkFRRUZBQU9DQVE4QU1JSUIKQ2dLQ0FRRUF2dmJoQnRLQWVjY0d3Qk9CTXJOajR6TlRtU2Jkd3dXNFdnRWEwcjN0QUs4VXBNcWpGVkRGWnNkTgpKYXBiM0F0elVJTllRanJSMEorLzZramtpRjB6ZTBsd2FlKzZUdzdiaGFXZzVJR0g3NmpxNnY4SFVrL0grcmtVCkNSRjZaajlabmxHN2FMYlJtKzh2R3RpeDFvVzhIL0Fja2J1UlJKdDZuNDFPbFhGYnBkMzlqSjRiVDhWbzd2aGMKcVdtYy9xcDV6OUg3OWp1RnQ5QkNEVkVJeXRBdEk3UUMwZ1lPSFY5aENFNVlLcEMxd2c0SlZ2RWxZRWUzdDFuOApBMU90dm9LRjc0MlFTYjlkWVhKNDRZODlRNFY0OEp3U1IrQ2MxemQ3V1dnKy92MUJRT0hlUVlLRENuMDB3OVE4Cm8wd2tQM2QvQnkxZU1OcVNGVnBJWkRZMFNqdEdZd0lEQVFBQm8xQXdUakFkQmdOVkhRNEVGZ1FVUGV0aWpEbjkKT2orY3FKSkVWZjI3QXU4RmdJb3dId1lEVlIwakJCZ3dGb0FVUGV0aWpEbjlPaitjcUpKRVZmMjdBdThGZ0lvdwpEQVlEVlIwVEJBVXdBd0VCL3pBTkJna3Foa2lHOXcwQkFRc0ZBQU9DQVFFQVJmdTB3TWhabUZJSE15bUNBOVJmCmNLYnVPY0taeEVRVmFFSDIranNsZW5SUHZNUW9PSFdLQXNobG1obkNyOHlWYmZEM1BZeVdhTG9ZaGluMjFqTEoKamFtNnZKM0FWK29Zcm9XYnExdmtFeEQxSnJWZ1dlc0tFOGNBUG1RQXlYRVZGcFNTeXo5U3lxc3VyUnZLcFdzdgp2aFRTa3QrRWE1QWNaNnpZd1drNkxJSlhxL2RRckdvUDRxa2ROMUwzWEhoZzdTUXZXMm4rZWFzd3lEejMvTXFpCnpoTzZSUXhNdkRGZXhXUzY1NWJQdHBRSTN1UEZnUGhsTmxtejVTWjRSRSt2V0YycmVtNlNLNGEvZ25iU0Rwc3AKVHBBcmFFUllMNm44bTI4UUVsWFZhQjJialdubS90WjFVajlGSUN4YUozRHVOd2RFbmFKU0d3czVGRm5oakRPYgpCUT09Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K

```

## Использование запуска IAM Proxy с изменением файлов конфигурации в отдельном контейнере (в init/sidecar контейнере)

Для обеспечения более высокого уровня безопасности в IAM Proxy, есть возможность в основном контейнере IAM Proxy
ограничить доступ только на чтение к файлам конфигурации nginx, а инициализацию и обновление файлов конфигурации
выполнять в отдельном контейнере, запущенном рядом (в k8s это может быть как init, так и sidecar контейнер). Таким
образом в основном контейнере, обеспечивающем обработку пользовательского трафика, доступ на запись останется только в
каталоги обеспечивающие кеширование проксируемого контента, а защита от модификации файлов конфигурации будет обеспечена
средствами файловой системы (она будет в режиме read-only).

Для раздельного запуска контейнера необходимо использовать переменную окружения `CONTAINER_EXEC_MODE`. Со значением
`CONTAINER_EXEC_MODE=only_conf` запускаем первый контейнер в режиме инициализации и обновления файлов конфигурации. А со
значением `CONTAINER_EXEC_MODE=only_proxy` запускам второй основной контейнер в режиме работы прокси на готовой
конфигурации. Таким образом первый контейнер создает все необходимые файлы на примонтированном разделе файловой системы
в режиме read-write, а второй контейнер читает файлы конфигурации с этого же раздела файловой системы, только
примонтированного уже в режиме read-only. Если оба контейнера запускаются параллельно, то второй контейнер автоматически
будет дожидаться первого контейнер и окончания работы по инициализации файлов конфигурации.

В случае, когда после инициализации конфигурации часть файлов еще не существуют (например когда файлы сертификатов
доставляются отдельно с использованием vault-agent), можно пропустить этап тестирования конфигурации nginx в
init/sidecar контейнере, используя переменную окружения `NGINX_CONF_SKIP_TEST=true`. При этом так же необходимо задать
явно использование в контейнере еще фактически несуществующих файлов, это делается через задание имен файлов в
переменных окружения: `PROXY_CERT_FILE`, `PROXY_KEY_FILE`, `PROXY_TLS_TRUST_FILE`, `PROXY_MTLS_CERT_FILE`,
`PROXY_MTLS_KEY_FILE`, `PROXY_SECRETS_FILE` (при непустом значении этих переменных считается, что конкретный файл
используется в конфигурации и должен присутствовать в момент запуска nginx).

При запуске sidecar контейнера IAM Proxy в k8s может быть полезно задание переменной окружения
`CONTAINER_EXEC_NOT_EXIT=true`, которая позволяет не завершать контейнер sidecar после создания конфигурации, чтобы это
не приводило к неожиданному рестарту пода, в каких то не стандартных ситуациях.

### Пример использование раздельного запуска в docker

В примере:

- Предварительно удаляются существующие контейнеры (с именами которые мы используем в тесте), чтобы небыло конфликта.
- Создается локальная изолированная сеть `mynet`, чтобы в контейнерах можно было делать запросы друг к другу, это нужно
  для работы Hot Reload (применение новых настроек и сертификатов без рестарта контейнера), для которого так же задаются
  переменные окружения `HOT_RELOAD_PATHS` и `NGINX_RELOAD_URL`.
- Запускается два контейнера, с корневой файловой системой в режиме read-only. Оба контейнера подключаются к локальному
  каталогу `./tmpfs.shared`, один с возможностью записи в него, другой в режиме read-only.
- Делаем локальный запрос к созданному контейнеру, чтобы проверить, что поднялся порт для обработки HTTP запросов.

После успешного запуска останутся работать оба контейнера. Контейнер `my-iamproxy` будет обрабатывать HTTP запросы, а
контейнер `my-iamproxy-conf` будет мониторить изменение конфигурации и при изменении файлов сертификатов или
конфигурации (если их подключить к контейнеру `my-iamproxy-conf`) они будут автоматически применены в контейнере
`my-iamproxy`.

```shell
IMAGE="iamproxy:latest"

CONTAINER_NAME="my-iamproxy"
TMPFS_SHARED_DIR="./tmpfs.shared"

# запустить в фоновом режиме
DOCKER_OPTS_PROXY+=" --detach"
DOCKER_OPTS_PROXY+=" -e CONTAINER_EXEC_MODE=only_proxy"
# запустить с корневой системой в read-only
DOCKER_OPTS_PROXY+=" --read-only"
DOCKER_OPTS_PROXY+=" --tmpfs /tmp/platformauth-proxy.cache:noexec,size=10m,uid=10001"
DOCKER_OPTS_PROXY+=" --mount type=bind,src=$TMPFS_SHARED_DIR,dst=/tmp/platformauth-proxy,ro"
#DOCKER_OPTS_PROXY+=" -e DEBUG=1"
#DOCKER_OPTS_PROXY+=" -e DEBUG=wait"

# запустить в фоновом режиме
DOCKER_OPTS_CONFIG+=" --detach"
DOCKER_OPTS_CONFIG+=" -e CONTAINER_EXEC_MODE=only_conf"
DOCKER_OPTS_CONFIG+=" -e STEND_TYPE=dev"
DOCKER_OPTS_CONFIG+=" -e KEYCLOAK_FRONT_DNS_NAME=auth.mycompany.ru"
DOCKER_OPTS_CONFIG+=" -e KEYCLOAK_HTTPS_PORT_ON_FRONTEND=443"
DOCKER_OPTS_CONFIG+=" -e PROXY_OIDC_CLIENT_ID=test_client"
DOCKER_OPTS_CONFIG+=" -e PROXY_OIDC_CLIENT_SECRET=A5R2bxxxxxxxxxxxxxxxKi3cXE0v"
DOCKER_OPTS_CONFIG+=" -e OIDC_SSL_VERIFY=false"
# запустить с корневой системой в read-only
DOCKER_OPTS_CONFIG+=" --read-only"
DOCKER_OPTS_CONFIG+=" --tmpfs /tmp/platformauth-proxy.cache:noexec,size=10m,uid=10001"
DOCKER_OPTS_CONFIG+=" --mount type=bind,src=$TMPFS_SHARED_DIR,dst=/tmp/platformauth-proxy"
DOCKER_OPTS_CONFIG+=" -e HOT_RELOAD_PATHS=/opt/iamproxy/**/*.conf"
DOCKER_OPTS_CONFIG+=" -e NGINX_RELOAD_URL=http://${CONTAINER_NAME}:10080/reload"
#DOCKER_OPTS_CONFIG+=" -e DEBUG=1"
#DOCKER_OPTS_CONFIG+=" -e DEBUG=wait"

docker rm -f ${CONTAINER_NAME}-conf # удаляем старый контейнер если его запускали ранее
docker rm -f ${CONTAINER_NAME} # удаляем старый контейнер если его запускали ранее
rm -fr "$TMPFS_SHARED_DIR"
mkdir -p "$TMPFS_SHARED_DIR"

docker network create mynet

# запуск контейнера обеспечивающего конфигурацию
# задание MSYS_NO_PATHCONV позволяет корректно запустить контейнер из терминала MINGW на Windows
MSYS_NO_PATHCONV=1 \
docker run --name ${CONTAINER_NAME}-conf --network mynet -e CONTAINER_EXEC_MODE=only_conf $DOCKER_OPTS "$IMAGE"

# запуск основного контейнера
# задание MSYS_NO_PATHCONV позволяет корректно запустить контейнер из терминала MINGW на Windows
MSYS_NO_PATHCONV=1 \
docker run --name ${CONTAINER_NAME} --network mynet -p 8443:8443 -p 8080:8080 -p 10080:10080 $DOCKER_OPTS_PROXY "$IMAGE"

# проверяем что поднялся основной контейнер IAM Proxy
echo "Ждем 15 сек. для завершения запуска контейнера ..."
sleep 15
curl -v http://127.0.0.1:8080/proxy/version \
  --connect-timeout 2 --retry-all-errors --retry 4 --retry-delay 2 --retry-max-time 30
```

### Пример использование раздельного запуска в k8s с sidecar контейнером

IAM Proxy запускать как sidecar контейнер есть смысл, если требуется обновлять конфигурацию или секреты в реальном
времени (без рестарта pod). Это режим позволяет обеспечить автоматическое и частое обновление сертификатов и других
секретов, а так же обеспечить динамическое конфигурирование ответвлений (используя RDS API) при их большом количестве
или при наличии потребности динамического переключения на разные контура/backend.

В примере контейнеры `iamproxy` и `iamproxy-conf-sidecar` будут запущены почти одновременно. Контейнер `iamproxy` будет
ожидать, пока не будет закончена инициализация конфигурации в `iamproxy-conf-sidecar`, и только после этого продолжит
выполнение и будет запущен nginx и подняты порты для приема HTTP запросов. Контейнер `iamproxy-conf-sidecar` дождется
появления файлов с сертификатами и секретами, потом сформирует конфигурацию для `iamproxy` (создаст файлы в каталоге
`/tmp/platformauth-proxy` и по завершении создаст файл-флаг `/tmp/platformauth-proxy/init-end.flag`, означающий, что
конфигурация готова), и перейдет в режим мониторинга изменений на файловой системе (файлы указанные в
`HOT_RELOAD_PATHS`).

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: auth-iamproxy
  namespace: namespace-auth-ift-2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: auth-iamproxy
  template:
    metadata:
      labels:
        app: auth-iamproxy
    spec:
      volumes:
        - name: tmp
          emptyDir:
            medium: Memory
            sizeLimit: 20Mi
        - name: cache
          emptyDir:
            medium: Memory
            sizeLimit: 100Mi
        - name: cache-conf
          emptyDir:
            medium: Memory
            sizeLimit: 2Mi
      securityContext:
        fsGroup: 12000

      containers:
        - name: iamproxy
          image: registry.mycompany.ru:8999/mycompany_docker/ci90000019_iamproxy/iamproxy:latest
          resources:
            limits:
              cpu: 400m
              memory: 500Mi
            requests:
              cpu: 60m
              memory: 200Mi
          readinessProbe:
            httpGet:
              path: /status/readinessProbe
              port: http-status
              scheme: HTTP
            initialDelaySeconds: 3
            timeoutSeconds: 1
            periodSeconds: 30
            successThreshold: 1
            failureThreshold: 3
          env:
            - name: CONTAINER_EXEC_MODE
              value: only_proxy
          securityContext:
            runAsUser: 10001
            runAsGroup: 12000
            runAsNonRoot: true
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
          ports:
            - name: http-status
              containerPort: 10080
              protocol: TCP
            - name: http
              containerPort: 8080
              protocol: TCP
            - name: https
              containerPort: 8443
              protocol: TCP
          volumeMounts:
            - name: tmp
              mountPath: /tmp/platformauth-proxy
              readOnly: true
            - name: cache
              mountPath: /tmp/platformauth-proxy.cache

        - name: iamproxy-conf-sidecar
          image: registry.mycompany.ru:8999/mycompany_docker/ci90000019_iamproxy/iamproxy:latest
          resources:
            limits:
              cpu: 400m
              memory: 200Mi
            requests:
              cpu: 60m
              memory: 100Mi
          env:
            - name: CONTAINER_EXEC_NOT_EXIT
              value: 'true'
            - name: WAIT_EXIST_FILES
              value: "/certs/trusted_chain.crt.pem /certs/server.key.pem /certs/server.crt.pem /secrets/oidc-auth-secrets.server.conf"
            - name: CONTAINER_EXEC_MODE
              value: only_conf
            - name: HOT_RELOAD_PATHS
              value: '/secrets/* conf/ssl/* conf/custom.d/* conf/blacklist/* '
          envFrom:
            - configMapRef:
                name: auth-cm-iamproxy-opts
          securityContext:
            runAsUser: 10001
            runAsGroup: 12000
            runAsNonRoot: true
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
          volumeMounts:
            - name: tmp
              mountPath: /tmp/platformauth-proxy
            - name: cache-conf
              mountPath: /tmp/platformauth-proxy.cache
```

> Примечание
>
> В манифесте не показано монтирование файлов `/certs/trusted_chain.crt.pem`, `/certs/server.key.pem`,
> `/certs/server.crt.pem`, `/secrets/oidc-auth-secrets.server.conf`, которые будет ожидать контейнер
> `iamproxy-conf-sidecar`, и эти файлы необходимо подключить самостоятельно или убрать их
> из параметра `WAIT_EXIST_FILES` (например подключить можно через монтирование `ConfigMap`, или используя контейнер с
> `vault-agent`).
>
> `ConfigMap` с именем `auth-cm-iamproxy-opts` следует создать самостоятельно,
> в котором указать необходимые вам параметры IAM Proxy
> (подробнее смотрите [Соответствие имен параметров для разных сред и инструментов установки](params.md)).

### Пример использование раздельного запуска в k8s с init контейнером

IAM Proxy запускать как init контейнер есть смысл, если не требуется обновлять конфигурацию или секреты в реальном
времени (без рестарта pod). Это режим позволяет немного сэкономить ресурсы в k8s, особенно если запущенных экземпляров
IAM Proxy планируется много.

В примере сначала будет запушен `iamproxy-conf-init`, и не дожидаясь появления файлов с сертификатами и секретами,
начнется формирование конфигурацию для `iamproxy` (в каталоге `/tmp/platformauth-proxy`), и после этого контейнер
завершит свою работу. Далее будет запущен контейнер `iamproxy`, он дождется появления файлов с сертификатами и
секретами, и после этого продолжит выполнение и будет запущен nginx и подняты порты для приема HTTP запросов.

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: auth-iamproxy
  namespace: namespace-auth-ift-2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: auth-iamproxy
  template:
    metadata:
      labels:
        app: auth-iamproxy
    spec:
      volumes:
        - name: tmp
          emptyDir:
            medium: Memory
            sizeLimit: 20Mi
        - name: cache
          emptyDir:
            medium: Memory
            sizeLimit: 100Mi
        - name: cache-conf
          emptyDir:
            medium: Memory
            sizeLimit: 2Mi
      securityContext:
        fsGroup: 12000

      containers:
        - name: iamproxy
          image: registry.mycompany.ru:8999/mycompany_docker/ci90000019_iamproxy/iamproxy:latest
          resources:
            limits:
              cpu: 400m
              memory: 500Mi
            requests:
              cpu: 60m
              memory: 200Mi
          readinessProbe:
            httpGet:
              path: /status/readinessProbe
              port: http-status
              scheme: HTTP
            initialDelaySeconds: 3
            timeoutSeconds: 1
            periodSeconds: 30
            successThreshold: 1
            failureThreshold: 3
          env:
            - name: CONTAINER_EXEC_MODE
              value: only_proxy
            - name: WAIT_EXIST_FILES
              value: "/certs/trusted_chain.crt.pem /certs/server.key.pem /certs/server.crt.pem /secrets/oidc-auth-secrets.server.conf"
          securityContext:
            runAsUser: 10001
            runAsGroup: 12000
            runAsNonRoot: true
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
          ports:
            - name: http-status
              containerPort: 10080
              protocol: TCP
            - name: http
              containerPort: 8080
              protocol: TCP
            - name: https
              containerPort: 8443
              protocol: TCP
          volumeMounts:
            - name: tmp
              mountPath: /tmp/platformauth-proxy
              readOnly: true
            - name: cache
              mountPath: /tmp/platformauth-proxy.cache

        - name: iamproxy-conf-init
          image: registry.mycompany.ru:8999/mycompany_docker/ci90000019_iamproxy/iamproxy:latest
          resources:
            limits:
              cpu: 400m
              memory: 200Mi
            requests:
              cpu: 60m
              memory: 100Mi
          env:
            - name: CONTAINER_EXEC_MODE
              value: only_conf
            # отключаем проверку корректности конфигурации, потому-что на init этапе может не быть секретов и невозможно проверить конфигурацию без них
            - name: NGINX_CONF_SKIP_TEST
              value: "true"
            # Включаем явное использование сертификата сервера (чтобы при реальном отсутствии файлов, в конфигурации их все-равно использовали)
            - name: PROXY_CERT_FILE
              value: server.crt.pem
            - name: PROXY_KEY_FILE
              value: server.key.pem
            - name: PROXY_TLS_TRUST_FILE
              value: trusted_chain.crt.pem
            # Включаем явное использование секретов из файла в /secrets/oidc-auth-secrets.server.conf (чтобы при реальном отсутствии файлов, в конфигурации их все-равно использовали)
            - name: PROXY_SECRETS_FILE
              value: "/secrets/oidc-auth-secrets.server.conf"
          envFrom:
            - configMapRef:
                name: auth-cm-iamproxy-opts
          securityContext:
            runAsUser: 10001
            runAsGroup: 12000
            runAsNonRoot: true
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
          volumeMounts:
            - name: tmp
              mountPath: /tmp/platformauth-proxy
            - name: cache-conf
              mountPath: /tmp/platformauth-proxy.cache
```

> Примечание
>
> В манифесте не показано монтирование файлов `/certs/trusted_chain.crt.pem`, `/certs/server.key.pem`,
> `/certs/server.crt.pem`, `/secrets/oidc-auth-secrets.server.conf`, которые будет ожидать контейнер
> `iamproxy-conf-sidecar`, и эти файлы необходимо подключить самостоятельно или убрать их
> из параметра `WAIT_EXIST_FILES` (например подключить можно через монтирование `ConfigMap`, или используя контейнер с
> `vault-agent`).
>
> `ConfigMap` с именем `auth-cm-iamproxy-opts` следует создать самостоятельно,
> в котором указать необходимые вам параметры IAM Proxy
> (подробнее смотрите [Соответствие имен параметров для разных сред и инструментов установки](params.md)).  
