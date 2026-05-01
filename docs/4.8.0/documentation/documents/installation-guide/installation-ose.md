# Установка в среде контейнеризации

## Содержание

- [Сборка базового образа контейнера для IAM Proxy](proxy-deploy-docker-base-image.md)
- [Сборка образа контейнера IAM Proxy](proxy-deploy-docker-build.md)
- [Описание IAM Proxy при запуске в контейнере](proxy-deploy-docker-description.md)
- [Кейсы использования IAM Proxy в контейнере](proxy-deploy-docker-cases.md)
- [Установка IAM Proxy в OpenShift/k8s при помощи инструментов Platform V DevOps Tools](proxy-deploy-ose-cd.md)
- [Интеграция с Vault в OpenShift](vault-ose.md)

## Цель выполнения

Описание процесса установки IAM Proxy в среде контейнеризации OpenShift с использованием инструментов Platform V DevOps
Tools (CDJE), включая настройку Jenkins-задач, конфигурацию компонентов и проверку работоспособности развернутого
приложения.

## Последовательность действий

1. **Убедитесь в выполнении предусловий**:
    - установлен Platform V DevOps Tools;
    - настроены права доступа администратора к кластеру OpenShift;
    - при необходимости — подключена интеграция с Platform V Synapse Service Mesh и SecMan.

2. **Подготовьте конфигурацию CDJE**:
    - в репозитории `common` обновите следующие файлы:
        - `subsystems.json` — добавьте или измените секцию `IAM_PROXY` с указанием параметров артефакта конфигурации;
        - `multiClusters.json` — укажите параметры кластера OpenShift (API-адрес, проект, домен, учетные данные);
        - `environment.json` — настройте custom-секцию с параметрами развертывания, отключите ненужные плейбуки (
          например, связанные с WAS), активируйте режим работы с несколькими кластерами (
          `openshiftMultiClusters: "true"`).

3. **Настройте секреты и параметры безопасности**:
    - в директории `secrets` репозитория `common` создайте/обновите файлы `_passwords.conf` и `secret.yml`, зашифруйте
      их с помощью `ansible-vault` и `openssl`;
    - убедитесь, что в `environment.json` указаны корректные `credential ID` для доступа к секретам и OpenShift.

4. **Выполните миграцию конфигураций**:
    - запустите Jenkins job `Deploy` без параметров для миграции базовых конфигураций;
    - запустите job с опцией `MIGRATION_FP_CONF`, чтобы загрузить стендозависимые параметры IAM Proxy.

5. **Настройте параметры IAM Proxy**:
    - в репозитории подсистемы обновите файлы:
        - `auth.all.conf` — основные параметры интеграции (Service Mesh, SecMan, Audit, Logging и т.д.);
        - `auth.istio.all.conf` — параметры Istio, маршрутизации, сертификатов egress/ingress, Rate Limit;
        - `auth.iamproxy.conf` — настройки прокси, OIDC, лимитов, CORS, healthcheck, TLS и др.
    - при необходимости создайте файл `custom_property.conf.yml` для:
        - настройки ответвлений (`proxy_jct_list`);
        - добавления custom-конфигурационных файлов и статических ресурсов (`custom_files`);
        - настройки лимитов через SRLS.

6. **Выполните развертывание**:
    - запустите Jenkins job `Deploy` с параметром `OPENSHIFT_DEPLOY`;
    - дождитесь успешного завершения пайплайна.

7. **(Опционально) Настройте удаление ресурсов**:
    - добавьте в `environment.json` плейбуки `OPENSHIFT_PURGE_PROJECT` и `KUBERNETES_PURGE_PROJECT`;
    - перезагрузите конфигурацию job;
    - используйте соответствующую опцию в job для очистки namespace.

## Проверка результата

1. **Проверьте состояние подов**:
   ```bash
   oc get pods -n <project-name>
   ```

Убедитесь, что под `auth-proxy-*` находится в статусе `Running`.

2. **Проверьте доступность сервиса**:
    - откройте в браузере FQDN, указанный в `iamproxy.k8s.front.https.host`;
    - убедитесь, что выполняется перенаправление на IDP (например, Keycloak);

3. **Проверьте логи приложения**:
   ```bash
   oc logs <iam-proxy-pod-name> -c iamproxy -n <project-name>
   ```
   Убедитесь, что нет ошибок инициализации, подключения к IDP, SecMan, Vault и т.д.

4. **Проверьте интеграции**:
    - аутентификация через OIDC проходит успешно;
    - логи отправляются в Platform V Monitor (LOGA);
    - события аудита поступают в Platform V Audit SE;
    - метрики доступны в системе мониторинга (Prometheus/Grafana).

5. **Проверьте работу ответвлений**:
    - обратитесь к endpoint, указанным в `junctionPoint`;
    - убедитесь, что запросы проксируются на бэкенд и возвращаются с корректным содержимым.

6. **Проверьте работу security-функций**:
    - TLS между IAM Proxy и IDP активирован;
    - сертификаты egress/ingress загружены из SecMan/Vault;
    - при включенном SRLS — лимиты запросов соблюдаются.



