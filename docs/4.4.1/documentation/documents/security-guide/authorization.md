# Авторизация

Авторизация на IAM Proxy может осуществляться следующими способами (описаны в разделе
[Описание IAM Proxy при запуске в контейнере](../../installation-guide/md/proxy-deploy-docker-description.md)):

- по ролям из токена;
- по клиентскому сертификату;
- по валидности JWT-токена из запроса;
- по данным из API внешнего сервиса (API AZGT или API OPA).

> Примечание:
> В случае использования для авторизации данных из компонента OCA, информация о сценариях авторизации, о развертывании
> ролевой модели описаны в документации на компонент ОСА (AUTZ) продукта Platform V IAM.

Авторизация между IAM Proxy на VM и другими сервисами обеспечивается при помощи цифровых сертификатов (mTLS v1.3/v1.2).

Авторизация между IAM Proxy в среде контейнеризации и другими сервисами обеспечивается при помощи цифровых
сертификатов (mTLS v1.3/v1.2). При использовании Platform V Synapse Service Mesh (SSM) авторизация так-же обеспечивается
средствами Platform V Synapse Service Mesh.

Для функционального администрирования IAM Proxy предполагается наличие роли функционального администратора IAM
`platformauth_admin`. Данная роль назначается для УЗ функционального администратора в Platform V IAM KeyCloak.SE, и
используется при настройке ответвлений, проксирование по которым необходимо разрешить функциональному администратору
IAM.

При развертывании IAM Proxy на VM необходимы учетные записи:

| Учетная запись                                                              | Описание                                                                                                                                                                                                                                            |
|-----------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| параметр запуска DevOps `SSH_USER` (задается в Job Jenkins)                 | Привилегированная УЗ DevOps с правами SUDO, под которой будет производиться подготовка серверов к развертыванию (УЗ используется только при запуске ansible-playbook `platformauth-system-prepare-playbook.yml` для подключения по SSH к серверам). |
| параметр профиля DevOps `deployer_user` (задается в ansible inventory)      | УЗ DevOps для установки продукта (УЗ создается автоматически в `platformauth-system-prepare-playbook.yml`, и используется при запуске ansible-playbook `platformauth-deploy-playbook.yml` для подключения по SSH к серверам).                       |
| параметр профиля DevOps `nexus_url_username` (задается в ansible inventory) | УЗ DevOps для загрузки дистрибутива из Nexus (опциональная и требуется когда дистрибутив хранится в Nexus).                                                                                                                                         |

## Ролевая модель в операционной системе для IAM Proxy на VM

### Учетные записи

Для работы сервиса и его эксплуатации предполагается использование следующих УЗ и групп в ОС:

- УЗ для DevOps и автоматизации (привилегированная технологическая УЗ для работы скриптов подготовки серверов при
  установке продукта). Позволяет управлять состоянием сервера, устанавливать необходимые системные приложения (в
  частности rpm-пакеты), создавать ТУЗ для сервисов продукта, настраивать службы - systemd, logrotate, sudo. Не
  допускается использовать для ручных операций. Привилегии предоставляются через sudo;
- УЗ для DevOps и автоматизации (технологическая УЗ `iamproxy-deployer` для работы скриптов при установке и настройке
  продукта, имя ТУЗ можно использовать произвольное, УЗ создается автоматически). Позволяет управлять состоянием
  сервисов продукта, менять конфигурацию продукта, позволяет менять исполняемые файлы продукта. Не допускается
  использовать для ручных операций. Привилегии предоставляются через точечные sudo и через права доступа файловой
  системы. Является владельцем файлов конфигурации и исполняемых файлов;
- УЗ для сопровождения продукта администраторами (персональные УЗ администраторов, права на чтение которым
  предоставляются через включение персональных УЗ в группу `iamproxy`, права на запись конфигурации через включение
  персональных УЗ в группу `iamproxy-admins`). Позволяет управлять состоянием сервисов продукта, частично менять
  конфигурацию продукта, читать журналы (логи) продукта. Права предоставляются через точечные sudo и через права доступа
  файловой системы (с использованием setfacl). Не позволяет модифицировать исполняемые файлы;
- УЗ для работы основных сервисов продукта (технологическая УЗ `iamproxy` для работы сервисов systemd
  `iamproxy.service`, `iamproxy-helper.service`, `lb-iamproxy.service`, УЗ создается автоматически). Интерактивный и
  удаленный вход под этой УЗ должен быть запрещен. У этой УЗ есть права только на чтение для своих собственных
  конфигураций и собственных исполняемых файлов, и соответственно УЗ не может быть владельцем этих файлов. Разрешен
  доступ на изменение только к своим рабочим каталогам и временным данным, необходимым для функционирования сервиса;
- УЗ для работы сервиса продукта `rds-client.service` (технологическая УЗ `rds-client` для работы сервиса systemd, УЗ
  создается средствами DevOps продукта). Интерактивный и удаленный вход под этой УЗ должен быть запрещен. У этой УЗ есть
  права только на чтение для своих собственных конфигураций и собственных исполняемых файлов, и соответственно УЗ не
  может быть владельцем этих файлов. Разрешен доступ на изменение только к своим рабочим каталогам и временным данным,
  необходимым для функционирования сервиса;
- УЗ для работы сервиса продукта `rds-server.service` (технологическая УЗ `rds-server` для работы сервиса systemd, УЗ
  создается средствами DevOps продукта). Интерактивный и удаленный вход под этой УЗ должен быть запрещен;
- УЗ для работы сервиса продукта `fluent-bit.service` (технологическая УЗ `fluent-bit` для работы сервиса systemd, УЗ
  создается средствами DevOps продукта). Интерактивный и удаленный вход под этой УЗ должен быть запрещен. У этой УЗ есть
  права на чтение файлов журналов всех сервисов продукта;
- УЗ для работы сервиса `vault-agent.service` (технологическая УЗ `vault` для работы сервиса systemd, УЗ создается
  средствами DevOps продукта). Интерактивный и удаленный вход под этой УЗ должен быть запрещен. У этой УЗ есть права
  только на чтение для своих собственных конфигураций и собственных исполняемых файлов, и соответственно УЗ не может
  быть владельцем этих файлов. Разрешен доступ на изменение только к своим рабочим каталогам и временным данным,
  необходимым для функционирования сервиса;
- УЗ для изменения файлов конфигурации сервиса `vault-agent.service` (технологическая УЗ `vault-config` используемая в
  сервисах systemd потребляющих секреты из `vault-agent.service`, УЗ создается средствами DevOps продукта).
  Интерактивный и удаленный вход под этой УЗ должен быть запрещен. У этой УЗ есть права только на чтение конфигурации
  сервисов потребителей секретов, и на чтение/запись в файлы конфигурации `vault-agent.service`;
- УЗ для работы сервиса продукта `keycloak.service` (технологическая УЗ `keycloak` для работы сервиса systemd, УЗ
  создается средствами DevOps продукта). Интерактивный и удаленный вход под этой УЗ должен быть запрещен;
- группы для получения доступа только на чтение конфигурации продукта и его журналов, и доступ на управление состоянием
  сервисов продукта, это группы `iamproxy`, `keycloak`, `fluent-bit`, `vault`, `keycloak`, `rds-server` (
  группы предоставляют доступ к данным одноименных сервисов). Используются как при работе сервисов systemd, так и при
  сопровождении продукта администраторами. Технологические УЗ сервисов включаются в необходимые группы средствами DevOps
  продукта. Персональные УЗ администраторов включаются точечно в необходимые группы, при необходимости предоставления
  доступа;
- группа `vault` предназначена для получения доступа только на чтение конфигурации `vault-agent.service`, доступа к
  журналам и к полученным секретам. Используются при работе сервисов systemd. Технологические УЗ сервисов включаются в
  группу средствами DevOps продукта;
- группа `iamproxy-admins` предназначена для получения администраторами доступа на запись в файлы конфигурации сервисов
  продукта (даются права на запись только к тем файлам конфигурации, в которых есть изменяемые средствами DevOps
  параметры), и на чтение к файлам журналов `vault-agent.service`. Персональные УЗ администраторов включаются точечно в
  группу, при необходимости предоставления доступа;
- группа `iamproxy-secrets` предназначена для доступа на чтение файлов с секретами сервисов продукта `iamproxy.service`,
  `lb-iamproxy.service`, `rds-client.service`. Технологические УЗ сервисов (`iamproxy`, `rds-client`) включаются в
  группу средствами DevOps продукта. Группа не используется, если секреты доставляются через `vault-agent.service`;
- группа `fluent-bit-secrets` предназначена для доступа на чтение файлов с секретами сервиса продукта
  `fluent-bit.service`. Технологическая УЗ сервиса (`fluent-bit`) включаются в группу средствами DevOps продукта.

### Процессы

Основные процессы сервисов продукта запускаются как сервисы systemd под непривилегированными технологическими УЗ, не
имеющие возможности интерактивного и удаленного входа.

Сервисы systemd:

- `iamproxy.service`, основной сервис продукта (на базе SynGX), обеспечивающий обратное проксирование с аутентификацией.
  Запускается под УЗ `iamproxy` и группой `iamproxy`, с применением `Capabilities=CAP_NET_BIND_SERVICE` (позволяет
  процессу прослушивать порты ниже 1024, для запуска веб-сервера на стандартных HTTP и HTTPS портах, 80 и 443), с
  `LimitNOFILE=500000` для обработки большого количества сетевых подключений, `LimitNPROC=500000` для запуска большого
  количества дочерних рабочих процессов;
- `fluent-bit.service`, сервис (на базе Fluent Bit) обеспечивающий доставку событий аудита и операционных событий из
  сервисов `iamproxy.service`, `rds-client.service`, `rds-server.service`, `keycloak.service` до удаленных сервисов
  централизованного хранения событий. Запускается под УЗ `fluent-bit` и группой `fluent-bit`;
- `iamproxy-helper.service`, вспомогательный сервис для работы `iamproxy.service`, который обеспечивает функционал
  горячей перезагрузки конфигурации (`Hot Reload`) в случае изменения конфигурации на файловой системе. Запускается под
  УЗ `iamproxy` и группой `iamproxy`;
- `keycloak.service`, сервис компонента Keycloak.SE (`KCSE`), обеспечивающий аутентификацию пользователей по OIDC.
  Запускается под УЗ `keycloak` и группой `keycloak`, с применением `Capabilities=CAP_NET_BIND_SERVICE` (позволяет
  процессу прослушивать порты ниже 1024, для запуска веб-сервера на стандартных HTTP и HTTPS портах, 80 и 443), с
  `LimitNOFILE=500000` для обработки большого количества сетевых подключений;
- `lb-iamproxy.service`, сервис L4 балансировщика нагрузки (на базе SynGX) для web-служб предоставляемых сервисами
  `iamproxy.service` и `keycloak.service`. Запускается под УЗ `iamproxy`, с применением
  `Capabilities=CAP_NET_BIND_SERVICE` (позволяет процессу прослушивать порты ниже 1024, для запуска веб-сервера на
  стандартных HTTP и HTTPS портах, 80 и 443), с `LimitNOFILE=500000` для обработки большого количества сетевых
  подключений, `LimitNPROC=500000` для запуска большого количества дочерних рабочих процессов;
- `vault-agent.service`, сервис `Vault Agent` обеспечивающий доставку из Vault/SecMan секретов необходимых для работы
  сервиса `iamproxy.service`. Запускается под УЗ `vault`. Исполняемые файлы и сама служба systemd `vault-agent.service`
  должны быть предварительно установлены на сервер (процесс установки не входит в DevOps продукта);
- `rds-client.service`, вспомогательный сервис для работы `iamproxy.service`, который обеспечивает функционал изменения
  конфигурации продукта в run-time (по данным из HTTP API). Запускается под УЗ `rds-client` и группой `iamproxy`;
- `rds-server.service`, сервис `RDS Server` обеспечивающий централизованное управление конфигурацией продукта в
  run-time, а так-же обеспечивающий интеграцию с централизованными сервисами управления конфигурацией и управления
  доступностью контуров (сервис PACMAN и сервис Журналирования). Запускается под УЗ `rds-server` и группой `rds-server`.

Задачи Logrotate:

- задача `iamproxy`, выполнение ротации журналов сервиса `iamproxy.service` (`/opt/iamproxy/logs/*.log`) по размеру и по
  количеству хранимых файлов. Запускается каждые 10 минут под технологической УЗ `iamproxy`;
- задача `lb-iamproxy`, выполнение ротации журналов сервиса `lb-iamproxy.service` (`/opt/iamproxy/lb-logs/*.log`) по
  размеру и по количеству хранимых файлов. Запускается каждые 10 минут под технологической УЗ `iamproxy`;
- задача `vault-agent`, выполнение ротации журналов сервиса `lb-iamproxy.service` (`/opt/vault/log/wrapped-*.log`) по
  размеру и по количеству хранимых файлов. Запускается каждые 10 минут под технологической УЗ `vault`.

### Файловая система на VM c IAM Proxy

```shell
sudo id deployer iamproxy rds-client fluent-bit vault vault-config
```

```
uid=1011(deployer) gid=1011(deployer) groups=1011(deployer),1013(iamproxy),1014(iamproxy-secrets),1015(fluent-bit),1016(
fluent-bit-secrets),20100(vault)
uid=1012(iamproxy) gid=1013(iamproxy) groups=1013(iamproxy),1014(iamproxy-secrets),20100(vault)
uid=1013(rds-client) gid=1013(iamproxy) groups=1013(iamproxy),1014(iamproxy-secrets),20100(vault)
uid=1014(fluent-bit) gid=1015(fluent-bit) groups=1015(fluent-bit),1013(iamproxy),1016(fluent-bit-secrets),20100(vault)
uid=20100(vault) gid=20100(vault) groups=20100(vault)
uid=20101(vault-config) gid=20101(vault-config) groups=20101(vault-config),1013(iamproxy),1015(fluent-bit),20100(vault)
```

> **Примечание**
>
> Цифровые uid и gid могут отличаться от приведенных выше, и будут уникальны на каждом сервере.

> **Примечание**
>
> В выводе прав из `getfacl` ниже, в `USER` указывается владелец файла и его права, в `GROUP` группа на файле и права
> группы.
>
> В правах символы в верхнем регистре ("W", "X", "R") означают недействующие права, и которые не предоставляются из-за
> ограничений наложенных маской.
>
> Для папок может присутствовать вторая колонка с правами, означающая default-права назначаемые новым файлам при их
> создании в этой папке.

```shell
find /opt/iamproxy /opt/fluent-bit /etc/fluent-bit /var/log/fluent-bit /opt/vault -print0 | sort -z | xargs -0 getfacl -tp
```

```
# file: /etc/fluent-bit
USER   deployer         rwx  rwx
GROUP  fluent-bit       r-x  r-x
group  iamproxy-admins  r-x  rw-
mask                    r-x  rwx
other                   ---  ---

# file: /etc/fluent-bit/fluent-bit.conf
USER   deployer         rw-     
GROUP  fluent-bit       r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /etc/fluent-bit/iamproxy-audit-metamodel.json
USER   deployer         rw-     
GROUP  iamproxy         r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /etc/fluent-bit/input_metamodel.conf
USER   deployer         rw-     
GROUP  fluent-bit       r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /etc/fluent-bit/input_proxy.conf
USER   deployer         rw-     
GROUP  fluent-bit       r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /etc/fluent-bit/input_rds_client.conf
USER   deployer         rw-     
GROUP  fluent-bit       r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /etc/fluent-bit/output_audit_kafka.conf
USER   deployer         rw-     
GROUP  fluent-bit       r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /etc/fluent-bit/output_logger_kafka.conf
USER   deployer         rw-     
GROUP  fluent-bit       r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /etc/fluent-bit/parsers.conf
USER   deployer         rw-     
GROUP  fluent-bit       r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /etc/fluent-bit/plugins.conf
USER   deployer         rw-     
GROUP  fluent-bit       r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /etc/fluent-bit/send-audit-metamodel.conf
USER   deployer         rw-     
GROUP  fluent-bit       r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /etc/fluent-bit/send-audit-metamodel.sh
USER   deployer         rw-     
GROUP  fluent-bit       r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /etc/fluent-bit/ssl
USER   deployer         rwx  rwx
GROUP  fluent-bit       r-x  r-x
group  iamproxy-admins  r-x  r--
mask                    r-x  r-x
other                   ---  ---

# file: /etc/fluent-bit/ssl/logger_cacerts_chains.pem
USER   deployer         rw-     
GROUP  fluent-bit       r--     
group  iamproxy-admins  r--     
mask                    r--     
other                   ---     

# file: /etc/fluent-bit/ssl/logger_ca_solution.mycompany.crt.pem
USER   deployer         rw-     
GROUP  fluent-bit       r--     
group  iamproxy-admins  r--     
mask                    r--     
other                   ---     

# file: /etc/fluent-bit/ssl/logger_cert.crt.pem
USER   deployer         rw-     
GROUP  fluent-bit       r--     
group  iamproxy-admins  r--     
mask                    r--     
other                   ---     

# file: /etc/fluent-bit/ssl/logger_cert.key.pem
USER   deployer            rw-     
GROUP  fluent-bit-secrets  r--     
mask                       r--     
other                      ---     

# file: /etc/fluent-bit/ssl/logger_new_cert.crt.pem
USER   deployer         rw-     
GROUP  fluent-bit       r--     
group  iamproxy-admins  r--     
mask                    r--     
other                   ---     

# file: /etc/fluent-bit/ssl/logger_new_cert.key.pem
USER   deployer            rw-     
GROUP  fluent-bit-secrets  r--     
mask                       r--     
other                      ---     

# file: /opt/fluent-bit
USER   root      rwx     
GROUP  root      r-x     
other            r-x     

# file: /opt/fluent-bit/bin
USER   root      rwx     
GROUP  root      r-x     
other            r-x     

# file: /opt/fluent-bit/bin/fluent-bit
USER   root      rwx     
GROUP  root      r-x     
other            r-x     

# file: /opt/iamproxy
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/conf
USER   deployer    rwx     
user   rds-client  rwx     
GROUP  iamproxy    r-x     
mask               rwx     
other              ---     

# file: /opt/iamproxy/conf/blacklist
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/conf/blacklist/blacklist.http.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/blacklist/blacklist.server.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/blacklist/blacklist-useragent.map-rules.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/blacklist/blacklist-useragent.txt
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/blacklist/whitelist-useragent.map-rules.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common
USER   deployer    rwx     
user   rds-client  rwx     
GROUP  iamproxy    r-x     
mask               rwx     
other              ---     

# file: /opt/iamproxy/conf/common/access-log-format.http.conf
USER   deployer         rw-     
GROUP  iamproxy         r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /opt/iamproxy/conf/common/access-log.http.conf
USER   deployer         rw-     
GROUP  iamproxy         r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /opt/iamproxy/conf/common/cache-control.http.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/iamproxy-audit-metamodel.json
USER   deployer         rw-     
GROUP  iamproxy         r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /opt/iamproxy/conf/common/index-root.server.conf
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/conf/common/jct.upstream.conf
USER   deployer         rw-     
GROUP  iamproxy         r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /opt/iamproxy/conf/common/large-headers.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/limits.http.conf
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/conf/common/limits.server.conf
USER   deployer         rw-     
GROUP  iamproxy         r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /opt/iamproxy/conf/common/log-trace.http.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/log-trace.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/lua.http.conf
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/conf/common/mobile-auth.server.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/not-transparent-jct.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/oidc-auth.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/oidc-auth-secrets.server.conf
USER   vault     rw-     
GROUP  vault     r--     
other            ---     

# file: /opt/iamproxy/conf/common/oidc-auth.server.conf
USER   deployer         rw-     
GROUP  iamproxy         r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /opt/iamproxy/conf/common/oidc-unauth-access.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/prometheus-metrics.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-add-header-efs-jct-rest.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-add-header-zone-offline.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-add-header-zone-standin.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-auth-in-esia.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-authz-opa.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-block-all-keycloak-admin-api.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-block-for-custom-keycloak-admin-api.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-cache-control-no-store.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-cache-control-off.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-disable-sso.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-disable-support-isam-headers.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-disable-use-clientcert.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-enable-cors.location.conf
USER   deployer         rw-     
GROUP  iamproxy         r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /opt/iamproxy/conf/common/rds-extra-large-upload.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-healthcheck-active.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-http-heathcheck.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-http-retry-without-idempotent.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-is-api-req.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-large-upload.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-limit-dry-run.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-limit-req-off.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-log-trace.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-log-trace-to-dir.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-mtls-front.location.conf
USER   deployer         rw-     
GROUP  iamproxy         r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /opt/iamproxy/conf/common/rds-no-need-oidc-auth.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-no-need-oidc-auth-when-token.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-opts-cloudera.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-proxy-buffering-off.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-public-access-disable.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-public-access.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-remove-domain-session-cookie.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-rquid-response-headers.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-set-header-auth-token.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-set-header-host-to-backend.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-ssl-sni-on.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-ssl-sni-on.server.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-unauth-access.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-use-aio.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-use-authz-by-url-and-method.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-use-authz-by-url-bridge.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-use-authz-by-url.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-use-authz-oauth-jwt.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-use-client-alt.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-use-client-from-auth.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/rds-use-direct-auth.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/redirect-http-to-https.http.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/req-echo-snoop.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/set-authz-by-role-admin.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/set-headers-jct.server.conf
USER   deployer         rw-     
GROUP  iamproxy         r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /opt/iamproxy/conf/common/set-headers-response.server.conf
USER   deployer         rw-     
GROUP  iamproxy         r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /opt/iamproxy/conf/common/ssl.server.conf
USER   deployer         rw-     
GROUP  iamproxy         r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /opt/iamproxy/conf/common/status-healthcheck.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/unauth-healthcheck-nginx.server.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/unauth-metrics-nginx.server.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/unauth-reload-nginx.server.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/unauth-status-nginx.server.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/common/variables.map.conf
USER   deployer         rw-     
GROUP  iamproxy         r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /opt/iamproxy/conf/common/writer-to-file.location.conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/custom.d
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/conf/custom.d/README.md
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/html_custom
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/conf/html_custom/README.md
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/iamproxy.conf
USER   deployer         rw-     
GROUP  iamproxy         r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /opt/iamproxy/conf/iamproxy.conf.rds
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/conf/jct
USER   deployer         rwx  rw-
user   rds-client       rwx     
GROUP  iamproxy         r-x  r--
group  iamproxy-admins  rwx  rw-
mask                    rwx  rw-
other                   ---  ---

# file: /opt/iamproxy/conf/jct/jct.name-server.server.jinja2
USER   deployer         rw-     
GROUP  iamproxy         r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /opt/iamproxy/conf/jct/jct.name-upstream.upstream.jinja2
USER   deployer         rw-     
GROUP  iamproxy         r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /opt/iamproxy/conf/jct/jct-snoop.server.conf
USER   rds-client       rw-     
GROUP  iamproxy         r--     
group  iamproxy-admins  rW-     
mask                    r--     
other                   ---     

# file: /opt/iamproxy/conf/jct/jct-snoop.upstream.conf
USER   rds-client       rw-     
GROUP  iamproxy         r--     
group  iamproxy-admins  rW-     
mask                    r--     
other                   ---     

# file: /opt/iamproxy/conf/jct/port443.server.conf
USER   rds-client       rw-     
GROUP  iamproxy         r--     
group  iamproxy-admins  rW-     
mask                    r--     
other                   ---     

# file: /opt/iamproxy/conf/jct/port443.upstream.conf
USER   rds-client       rw-     
GROUP  iamproxy         r--     
group  iamproxy-admins  rW-     
mask                    r--     
other                   ---     

# file: /opt/iamproxy/conf/jct/port8443.server.conf
USER   rds-client       rw-     
GROUP  iamproxy         r--     
group  iamproxy-admins  rW-     
mask                    r--     
other                   ---     

# file: /opt/iamproxy/conf/jct/port8443.upstream.conf
USER   rds-client       rw-     
GROUP  iamproxy         r--     
group  iamproxy-admins  rW-     
mask                    r--     
other                   ---     

# file: /opt/iamproxy/conf/jct/rds.server.conf
USER   rds-client       rw-     
GROUP  iamproxy         r--     
group  iamproxy-admins  rW-     
mask                    r--     
other                   ---     

# file: /opt/iamproxy/conf/jct/rds-tls.server.conf
USER   rds-client       rw-     
GROUP  iamproxy         r--     
group  iamproxy-admins  rW-     
mask                    r--     
other                   ---     

# file: /opt/iamproxy/conf/jct/rds-tls.upstream.conf
USER   rds-client       rw-     
GROUP  iamproxy         r--     
group  iamproxy-admins  rW-     
mask                    r--     
other                   ---     

# file: /opt/iamproxy/conf/jct/rds.upstream.conf
USER   rds-client       rw-     
GROUP  iamproxy         r--     
group  iamproxy-admins  rW-     
mask                    r--     
other                   ---     

# file: /opt/iamproxy/conf/jct/readme.txt
USER   deployer         rw-     
GROUP  iamproxy         r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /opt/iamproxy/conf/lua-lib
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/conf/lua-lib/readme.txt
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/lua-lib/resty
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/conf/lua-lib/resty/http.lua
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/lua-lib/resty/openidc.lua
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/lua-lib/resty/upstream
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/conf/lua-lib/resty/upstream/healthcheck.lua
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/lua-lib/sbt
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/conf/lua-lib/sbt/authz_http.lua
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/lua-lib/sbt/authz_opa.lua
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/lua-lib/sbt/helper.lua
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/lua-lib/sbt/oidc.lua
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/lua-lib/sbt/qp.lua
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/lua-lib/sbt/spas.lua
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/mime.types
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/ssl
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/conf/ssl/readme.txt
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/conf/ssl/server.crt.pem
USER   vault     rw-     
GROUP  vault     r--     
other            ---     

# file: /opt/iamproxy/conf/ssl/server.key.pem
USER   vault     rw-     
GROUP  vault     r--     
other            ---     

# file: /opt/iamproxy/conf/ssl/trusted_chain.crt.pem
USER   vault     rw-     
GROUP  vault     r--     
other            ---     

# file: /opt/iamproxy/conf/tools
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/conf/tools/cert
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/conf/tools/cert/cert-der-to-pem.sh
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/conf/tools/cert/cert-load.sh
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/conf/tools/cert/trusted-chain-update.sh
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/conf/tools/iamproxy-restart.sh
USER   deployer         rwx     
GROUP  iamproxy         r--     
group  iamproxy-admins  rw-     
mask                    rwx     
other                   ---     

# file: /opt/iamproxy/conf/tools/monitor-on-modify.conf
USER   deployer         rw-     
GROUP  iamproxy         r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /opt/iamproxy/conf/tools/monitor-on-modify.sh
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/conf/tools/patch-file.sh
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/conf/tools/reload.sh
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/conf/tools/wait-exist-files.sh
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/conf/vault-conf
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/conf/vault-conf/vault-templates-iamproxy.hcl
USER   deployer         rw-     
GROUP  iamproxy         r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /opt/iamproxy/conf/wait-exist-files.list
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/html
USER   deployer    rwx     
user   rds-client  rwx     
GROUP  iamproxy    r-x     
mask               rwx     
other              ---     

# file: /opt/iamproxy/html/403.html
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/html/404.html
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/html/509.html
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/html/50x.html
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/html/590.html
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/html/bg.png
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/html/favicon.ico
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/html/images
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/html/images/background-min.png
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/html/index-DFantoD8.js
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/html/index.html
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/html/index-TLdnxKl7.js
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/html/info.css
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/html/info.html
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/html/info.json
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/html/jwt-parse.html
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/html/openid-connect-auth
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/html/openid-connect-auth/403.html
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/html/openid-connect-auth/50x.html
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/html/openid-connect-auth/logoutSuccessful.html
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/html/rds-client-nginx-test.txt
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/html/version.txt
USER   deployer         rw-     
GROUP  iamproxy         r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /opt/iamproxy/logs
USER   iamproxy  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/logs/access.log
USER   iamproxy  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/logs/error.log
USER   iamproxy  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/logs/error.log-2025-09-13_00-00.gz
USER   iamproxy  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/logs/error.log-2025-09-14_00-00.gz
USER   iamproxy  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/logs/error.log-2025-09-15_00-00.gz
USER   iamproxy  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/logs/json-access.log
USER   iamproxy  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/logs/json-audit-access.log
USER   iamproxy  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/lua-lib-base
USER   root      rwx     
GROUP  root      r-x     
other            r-x     

# file: /opt/iamproxy/lua-lib-site
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/lua-lib-site/readme.txt
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/lua-lib-site/resty
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/lua-lib-site/resty/hmac.lua
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/lua-lib-site/resty/jwt.lua
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/lua-lib-site/resty/jwt-validators.lua
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/lua-lib-site/resty/openidc.lua
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/lua-lib-site/resty/session
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/lua-lib-site/resty/session/ciphers
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/lua-lib-site/resty/session/ciphers/aes.lua
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/lua-lib-site/resty/session/ciphers/none.lua
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/lua-lib-site/resty/session/encoders
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/lua-lib-site/resty/session/encoders/base16.lua
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/lua-lib-site/resty/session/encoders/base64.lua
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/lua-lib-site/resty/session/encoders/hex.lua
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/lua-lib-site/resty/session/hmac
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/lua-lib-site/resty/session/hmac/sha1.lua
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/lua-lib-site/resty/session/identifiers
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/lua-lib-site/resty/session/identifiers/random.lua
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/lua-lib-site/resty/session.lua
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/lua-lib-site/resty/session/serializers
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/lua-lib-site/resty/session/serializers/json.lua
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/lua-lib-site/resty/session/storage
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/lua-lib-site/resty/session/storage/cookie.lua
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/lua-lib-site/resty/session/storage/memcached.lua
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/lua-lib-site/resty/session/storage/memcache.lua
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/lua-lib-site/resty/session/storage/redis.lua
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/lua-lib-site/resty/session/storage/shm.lua
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/lua-lib-site/resty/session/strategies
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/lua-lib-site/resty/session/strategies/default.lua
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/lua-lib-site/resty/session/strategies/regenerate.lua
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/modules
USER   root      rwx     
GROUP  root      r-x     
other            r-x     

# file: /opt/iamproxy/rds-client
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/rds-client/cache
USER   rds-client  rwx     
GROUP  iamproxy    r-x     
other              ---     

# file: /opt/iamproxy/rds-client/cache/context.log
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/rds-client/cache/last-conf.json
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/rds-client/cache/reload-nginx.sh
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/rds-client/logs
USER   rds-client  rwx     
GROUP  iamproxy    r-x     
other              ---     

# file: /opt/iamproxy/rds-client/logs/json_application_2025-09-10_0.log
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/rds-client/logs/json_application_2025-09-11_0.log
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/rds-client/logs/json_application_2025-09-11_1.log
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/rds-client/logs/json_application_2025-09-12_0.log
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/rds-client/logs/json_application_2025-09-13_0.log
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/rds-client/logs/json_application_2025-09-14_0.log
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/rds-client/logs/json_application_2025-09-15_0.log
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/rds-client/logs/log-2025-09-09.log.zip
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/rds-client/logs/log-2025-09-10.log.zip
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/rds-client/logs/log-2025-09-11.log.zip
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/rds-client/logs/log-2025-09-12.log.zip
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/rds-client/logs/log-2025-09-13.log.zip
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/rds-client/logs/log-2025-09-14.log.zip
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/rds-client/logs/log-2025-09-15.log
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/rds-client/rds-client-conf.properties
USER   deployer         rw-     
GROUP  iamproxy         r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /opt/iamproxy/rds-client/rds-client.jar
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/rds-client/rds-client-start.conf
USER   deployer         rw-     
GROUP  iamproxy         r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /opt/iamproxy/rds-client/rds-client-start.sh
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/rds-client/rds-keystore.p12
USER   vault     rw-     
GROUP  vault     r--     
other            ---     

# file: /opt/iamproxy/rds-client/readme.txt
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/rds-client/results
USER   rds-client  rwx     
GROUP  iamproxy    r-x     
other              ---     

# file: /opt/iamproxy/rds-client/results/conf
USER   rds-client  rwx     
GROUP  iamproxy    r-x     
other              ---     

# file: /opt/iamproxy/rds-client/results/conf/common
USER   rds-client  rwx     
GROUP  iamproxy    r-x     
other              ---     

# file: /opt/iamproxy/rds-client/results/conf/common/index-root.server.conf
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/rds-client/results/conf/common/limits.http.conf
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/rds-client/results/conf/common/lua.http.conf
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/rds-client/results/html
USER   rds-client  rwx     
GROUP  iamproxy    r-x     
other              ---     

# file: /opt/iamproxy/rds-client/results/html/info.html
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/rds-client/results/html/info.json
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/rds-client/results/jct-snoop.server.conf
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/rds-client/results/jct-snoop.upstream.conf
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/rds-client/results/port443.server.conf
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/rds-client/results/port443.upstream.conf
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/rds-client/results/port8443.server.conf
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/rds-client/results/port8443.upstream.conf
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/rds-client/results/rds.server.conf
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/rds-client/results/rds-tls.server.conf
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/rds-client/results/rds-tls.upstream.conf
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/rds-client/results/rds.upstream.conf
USER   rds-client  rw-     
GROUP  iamproxy    r--     
other              ---     

# file: /opt/iamproxy/rds-client/templates
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/rds-client/templates/index_root_server_conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/rds-client/templates/info_html
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/rds-client/templates/info_json
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/rds-client/templates/jct.name-server.server.jinja2
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/rds-client/templates/jct.name-upstream.upstream.jinja2
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/rds-client/templates/limits_http_conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/rds-client/templates/lua_http_conf
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/rds-client/templates/name-server
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/rds-client/templates/name-upstream
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/rds-client/templates/README.md
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/rds-client/templates/template.index_root_server_conf.jinja2
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/rds-client/templates/template.info_html.jinja2
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/rds-client/templates/template.info_json.jinja2
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/rds-client/templates/template.limits_http_conf.jinja2
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/rds-client/templates/template.lua_http_conf.jinja2
USER   deployer  rw-     
GROUP  iamproxy  r--     
other            ---     

# file: /opt/iamproxy/regid.2021-12.ru.sbertech_auth-4.7.1.swidtag
USER   deployer  rw-     
GROUP  deployer  r--     
other            ---     

# file: /opt/iamproxy/sbin
USER   deployer  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/sbin/nginx
USER   root      rwx     
GROUP  root      r-x     
other            r-x     

# file: /opt/iamproxy/temp
USER   iamproxy  rwx     
GROUP  iamproxy  r-x     
other            ---     

# file: /opt/iamproxy/temp/client_body_temp
USER   iamproxy  rwx     
GROUP  iamproxy  ---     
other            ---     

# file: /opt/iamproxy/temp/fastcgi_temp
USER   iamproxy  rwx     
GROUP  iamproxy  ---     
other            ---     

# file: /opt/iamproxy/temp/proxy_temp
USER   iamproxy  rwx     
GROUP  iamproxy  ---     
other            ---     

# file: /opt/iamproxy/temp/scgi_temp
USER   iamproxy  rwx     
GROUP  iamproxy  ---     
other            ---     

# file: /opt/iamproxy/temp/uwsgi_temp
USER   iamproxy  rwx     
GROUP  iamproxy  ---     
other            ---     

# file: /opt/vault
USER   deployer         rwx     
GROUP  vault            r-x     
group  iamproxy-admins  r-x     
mask                    r-x     
other                   ---     

# file: /opt/vault/bin
USER   deployer  rwx     
GROUP  vault     r-x     
other            ---     

# file: /opt/vault/bin/vault
USER   root      rwx     
GROUP  root      r-x     
other            r-x     

# file: /opt/vault/.cache
USER   vault     rwx     
GROUP  vault     r-x     
other            ---     

# file: /opt/vault/.cache/snowflake
USER   vault     rwx     
GROUP  vault     --x     
other            --x     

# file: /opt/vault/.cache/snowflake/ocsp_response_cache.json
USER   vault     rwx     
GROUP  vault     --x     
other            --x     

# file: /opt/vault/conf
USER   deployer         rwx  rwx
user   vault-config     rwx  rw-
GROUP  vault            r-x  r--
group  iamproxy-admins  r-x     
mask                    rwx  rw-
other                   ---  ---

# file: /opt/vault/conf/agent.json
USER   deployer         rw-     
user   vault-config     rw-     
GROUP  vault            r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /opt/vault/log
USER   vault            rwx  rwx
GROUP  vault            r-x  r--
group  iamproxy-admins  r-x  r--
mask                    r-x  r--
other                   ---  ---

# file: /opt/vault/log/wrapped-token-secretid.log
USER   vault            rw-     
GROUP  vault            r--     
group  iamproxy-admins  r--     
mask                    r--     
other                   r--     

# file: /opt/vault/refresh-secret-id.conf
USER   deployer         rw-     
GROUP  vault            r--     
group  iamproxy-admins  rw-     
mask                    rw-     
other                   ---     

# file: /opt/vault/refresh-secret-id.sh
USER   deployer  rwx     
GROUP  vault     r-x     
other            ---     

# file: /opt/vault/secrets
USER   vault     rwx     
GROUP  vault     r-x     
other            ---     

# file: /opt/vault/secrets/iamproxy
USER   vault     rwx     
GROUP  vault     r-x     
other            ---     

# file: /opt/vault/secrets/iamproxy/oidc-auth-secrets.server.conf
USER   vault     rw-     
GROUP  vault     r--     
other            ---     

# file: /opt/vault/secrets/iamproxy/proxy-server.crt.pem
USER   vault     rw-     
GROUP  vault     r--     
other            ---     

# file: /opt/vault/secrets/iamproxy/proxy-server.key.pem
USER   vault     rw-     
GROUP  vault     r--     
other            ---     

# file: /opt/vault/secrets/iamproxy/rds-keystore.p12
USER   vault     rw-     
GROUP  vault     r--     
other            ---     

# file: /opt/vault/secrets/iamproxy/trusted_chain.crt.pem
USER   vault     rw-     
GROUP  vault     r--     
other            ---     

# file: /opt/vault/secrets/secretid
USER   vault     rw-     
GROUP  vault     ---     
other            ---     

# file: /opt/vault/secrets/token.sink
USER   vault     rw-     
GROUP  vault     r--     
other            ---     

# file: /opt/vault/tokens
USER   deployer         rwx  rwx
user   vault                 rw-
GROUP  vault            rwx  ---
group  iamproxy-admins  r-x  -w-
mask                    rwx  rw-
other                   ---  ---

# file: /opt/vault/tokens/roleid
USER   deployer         rw-     
user   vault            rw-     
GROUP  vault            r--     
group  iamproxy-admins  -w-     
mask                    rw-     
other                   ---     

# file: /opt/vault/tokens/wrapped-token-secretid
USER   vault            rw-     
user   vault            rw-     
GROUP  vault            ---     
group  iamproxy-admins  -w-     
mask                    rw-     
other                   ---     

# file: /opt/vault/tokens/wrapped-token-secretid.exp
USER   vault            rw-     
user   vault            rw-     
GROUP  vault            ---     
group  iamproxy-admins  -w-     
mask                    rw-     
other                   ---     

# file: /var/log/fluent-bit
USER   fluent-bit       rwx  rwx
GROUP  fluent-bit       r-x  r--
group  iamproxy-admins  r-x  r--
mask                    r-x  r--
other                   ---  ---

# file: /var/log/fluent-bit/proxy-access.db
USER   fluent-bit       rw-     
GROUP  fluent-bit       r--     
group  iamproxy-admins  r--     
mask                    r--     
other                   ---     

# file: /var/log/fluent-bit/proxy-access.db-shm
USER   fluent-bit       rw-     
GROUP  fluent-bit       r--     
group  iamproxy-admins  r--     
mask                    r--     
other                   ---     

# file: /var/log/fluent-bit/proxy-access.db-wal
USER   fluent-bit       rw-     
GROUP  fluent-bit       r--     
group  iamproxy-admins  r--     
mask                    r--     
other                   ---     

# file: /var/log/fluent-bit/proxy-audit-access.db
USER   fluent-bit       rw-     
GROUP  fluent-bit       r--     
group  iamproxy-admins  r--     
mask                    r--     
other                   ---     

# file: /var/log/fluent-bit/proxy-audit-access.db-shm
USER   fluent-bit       rw-     
GROUP  fluent-bit       r--     
group  iamproxy-admins  r--     
mask                    r--     
other                   ---     

# file: /var/log/fluent-bit/proxy-audit-access.db-wal
USER   fluent-bit       rw-     
GROUP  fluent-bit       r--     
group  iamproxy-admins  r--     
mask                    r--     
other                   ---     

# file: /var/log/fluent-bit/proxy-error.db
USER   fluent-bit       rw-     
GROUP  fluent-bit       r--     
group  iamproxy-admins  r--     
mask                    r--     
other                   ---     

# file: /var/log/fluent-bit/proxy-error.db-shm
USER   fluent-bit       rw-     
GROUP  fluent-bit       r--     
group  iamproxy-admins  r--     
mask                    r--     
other                   ---     

# file: /var/log/fluent-bit/proxy-error.db-wal
USER   fluent-bit       rw-     
GROUP  fluent-bit       r--     
group  iamproxy-admins  r--     
mask                    r--     
other                   ---     

# file: /var/log/fluent-bit/proxy-trace.db
USER   fluent-bit       rw-     
GROUP  fluent-bit       r--     
group  iamproxy-admins  r--     
mask                    r--     
other                   ---     

# file: /var/log/fluent-bit/proxy-trace.db-shm
USER   fluent-bit       rw-     
GROUP  fluent-bit       r--     
group  iamproxy-admins  r--     
mask                    r--     
other                   ---     

# file: /var/log/fluent-bit/proxy-trace.db-wal
USER   fluent-bit       rw-     
GROUP  fluent-bit       r--     
group  iamproxy-admins  r--     
mask                    r--     
other                   ---     

# file: /var/log/fluent-bit/rds-client.db
USER   fluent-bit       rw-     
GROUP  fluent-bit       r--     
group  iamproxy-admins  r--     
mask                    r--     
other                   ---     

# file: /var/log/fluent-bit/rds-client.db-shm
USER   fluent-bit       rw-     
GROUP  fluent-bit       r--     
group  iamproxy-admins  r--     
mask                    r--     
other                   ---     

# file: /var/log/fluent-bit/rds-client.db-wal
USER   fluent-bit       rw-     
GROUP  fluent-bit       r--     
group  iamproxy-admins  r--     
mask                    r--     
other                   ---     
```

> Состав папок и имена файлов на реальном стенде могут отличаться от указанного здесь, в зависимости от параметров
> установки продукта.

### Права предоставляемые через SUDO на VM c IAM Proxy

В зависимости от параметров установки продукта средствами DevOps устанавливаются необходимые файлы в каталоге
`/etc/sudoers.d/` для предоставления требуемых полномочий УЗ и группам.

Файл `/etc/sudoers.d/ansible-deployers`:

```
User_Alias DEPLOYERS = deployer
Runas_Alias DEPLOY_SVC_USERS = keycloak,iamproxy,rds-client,rds-server,fluent-bit,vault
DEPLOYERS ALL=(DEPLOY_SVC_USERS) NOPASSWD:ALL
DEPLOYERS ALL=(root) NOEXEC: NOPASSWD: /bin/chown -R deployer /opt/iamproxy/conf /opt/iamproxy/html,/bin/chown -R deployer /opt/rds-server,/bin/chown -R deployer /etc/fluent-bit,/bin/chown -R deployer /opt/vault,/bin/chown -R deployer /opt/keycloak
Defaults:deployer !requiretty
```

Файл `/etc/sudoers.d/iamproxy-user` (присутствует при наличии службы `iamproxy.service`):

```
deployer,iamproxy,%iamproxy ALL=(root) NOEXEC: NOPASSWD: /bin/systemctl daemon-reload,/bin/systemctl status iamproxy,/bin/systemctl is-enabled iamproxy,/bin/systemctl stop iamproxy,/bin/systemctl stop --no-block iamproxy,/bin/systemctl start iamproxy,/bin/systemctl restart iamproxy,/bin/systemctl reload iamproxy,/bin/systemctl enable iamproxy,/bin/systemctl disable iamproxy,/bin/systemctl stop iamproxy-helper,/bin/systemctl start iamproxy-helper,/bin/systemctl restart iamproxy-helper,/bin/systemctl reload iamproxy-helper,/bin/systemctl enable iamproxy-helper,/bin/systemctl disable iamproxy-helper
Defaults:iamproxy !requiretty
```

Файл `/etc/sudoers.d/rds-client-user` (присутствует при наличии службы `rds-client.service`):

```
deployer,rds-client,%iamproxy ALL=(root) NOEXEC: NOPASSWD: /bin/systemctl status rds-client,/bin/systemctl stop rds-client,/bin/systemctl start rds-client,/bin/systemctl restart rds-client,/bin/systemctl reload rds-client,/bin/systemctl daemon-reload,/bin/systemctl enable rds-client,/bin/systemctl is-enabled rds-client,/bin/systemctl disable rds-client
rds-client ALL=(iamproxy) NOPASSWD:ALL
Defaults:rds-client !requiretty
```

Файл `/etc/sudoers.d/fluent-bit-user` (присутствует при наличии службы `fluent-bit.service`):

```
deployer,fluent-bit,%fluent-bit ALL=(root) NOEXEC: NOPASSWD: /bin/systemctl status fluent-bit,/bin/systemctl stop fluent-bit,/bin/systemctl start fluent-bit,/bin/systemctl restart fluent-bit,/bin/systemctl reload fluent-bit,/bin/systemctl daemon-reload,/bin/systemctl enable fluent-bit,/bin/systemctl disable fluent-bit
Defaults:fluent-bit !requiretty
```

Файл `/etc/sudoers.d/vault-agent-user` (присутствует при наличии службы `vault-agent.service`):

```
deployer,vault,%vault ALL=(root) NOEXEC: NOPASSWD: /bin/systemctl daemon-reload,/bin/systemctl status vault-agent,/bin/systemctl stop vault-agent,/bin/systemctl start vault-agent,/bin/systemctl restart vault-agent,/bin/systemctl restart --no-block vault-agent,/bin/systemctl reload vault-agent,/bin/systemctl enable vault-agent,/bin/systemctl disable vault-agent,/bin/systemctl status /opt/vault/secrets,/bin/systemctl stop /opt/vault/secrets,/bin/systemctl start /opt/vault/secrets,/bin/systemctl restart /opt/vault/secrets
Defaults:vault !requiretty
```
