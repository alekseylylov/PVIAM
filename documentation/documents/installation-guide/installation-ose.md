# Установка в среде контейнеризации

## Содержание

- [Сборка базового образа контейнера для IAM Proxy](proxy-deploy-docker-base-image.md)
- [Сборка образа контейнера IAM Proxy](proxy-deploy-docker-build.md)
- [Описание IAM Proxy при запуске в контейнере](proxy-deploy-docker-description.md)
- [Кейсы использования IAM Proxy в контейнере](proxy-deploy-docker-cases.md)
- [Установка IAM Proxy в OpenShift/k8s при помощи инструментов Platform V DevOps Tools](proxy-deploy-ose-cd.md)
- [Интеграция с Vault в OpenShift](vault-ose.md)

## Цель выполнения

> Данный шаг рекомендуется выполнять при необходимости автоматизировать развертывание IAM Proxy в OpenShift/k8s с использованием инструментов Platform V DevOps Tools. Выполнение шага позволяет интегрировать процесс деплоя в существующие CI/CD-конвейеры, обеспечивает контроль версий, мониторинг состояния и быстрое восстановление. Шаг обязателен, если требуется централизованное управление инфраструктурой через DevOps-платформу.

## Последовательность действий

1. **Настроить Git-репозиторий**:
   - создать директорию `iam-proxy-deploy` в репозитории проекта;
   - добавить файлы конфигурации Helm Chart (`Chart.yaml`, `values.yaml`) и манифесты Kubernetes (Deployment, Service, Ingress);
   - пример структуры:
     ```
     iam-proxy-deploy/
     ├── Chart.yaml
     ├── values.yaml
     └── templates/
         ├── deployment.yaml
         ├── service.yaml
         └── ingress.yaml
     ```

2. **Интегрировать с DevOps Tools**:
   - в конфигурационном файле CI/CD (например, `pipeline.yaml`) добавить этап сборки и развертывания:
     ```yaml
     stages:
       - build
       - deploy
       
     deploy:
       stage: deploy
       script:
         - helm package iam-proxy-deploy/
         - helm upgrade --install iam-proxy ./iam-proxy-0.1.0.tgz -f iam-proxy-values-prod.yaml
       environment:
         name: production
         url: https://iam-proxy.mycompany.ru
     ```

3. **Настроить параметры безопасности**:
   - указать секреты (токены, сертификаты) в `values.yaml` через ссылки на Kubernetes Secrets;
   - пример:
     ```yaml
     secrets:
       vaultToken: 
         secretKeyRef:
           name: vault-credentials
           key: token
     ```

4. **Проверить политики RBAC**:
   - создать Role и RoleBinding для сервиса IAM Proxy в namespace:
     ```yaml
     apiVersion: rbac.authorization.k8s.io/v1
     kind: Role
     metadata:
       name: iam-proxy-role
     rules:
       - apiGroups: [""]
         resources: ["pods"]
         verbs: ["get", "list"]
     ---
     apiVersion: rbac.authorization.k8s.io/v1
     kind: RoleBinding
     metadata:
       name: iam-proxy-binding
     subjects:
       - kind: ServiceAccount
         name: iam-proxy-sa
     roleRef:
       kind: Role
       name: iam-proxy-role
       apiGroup: rbac.authorization.k8s.io
     ```

5. **Активировать мониторинг и логирование**:
   - добавить аннотации Prometheus в Deployment:
     ```yaml
     annotations:
       prometheus.io/scrape: "true"
       prometheus.io/port: "8080"
     ```

## Проверка результата

1. **Статус развертывания**:
   - проверить готовность подов:
     ```shell
     kubectl get pods -n <namespace>
     ```
   - убедиться, что статус всех подов `Ready`.

2. **Функциональность сервиса**:
   - выполнить тестовый запрос к IAM Proxy:
     ```shell
     curl -k https://iam-proxy.mycompany.ru/healthz
     ```
   - ожидаемый ответ: `{"status":"ok"}`.

3. **Логи и метрики**:
   - проверить логи пода на наличие ошибок:
     ```shell
     kubectl logs <pod-name> -n <namespace>
     ```
   - убедиться, что метрики доступны через Prometheus:
     ```shell
     curl http://prometheus.mycompany.ru/api/v1/query?query=up{job="iam-proxy"}
     ```

4. **Интеграция с DevOps**:
   - запустить pipeline вручную и проверить успешное завершение этапа `deploy`;
   - выполните обновление при помощи PV DOT CDJE, выбрав при установке плейбуки `MIGRATION_FP_CONF`, `OPENSHIFT_DEPLOY`, `OPENSHIFT_INGRESS_EGRESS_DEPLOY`;
   - проверьте доступность приложения curl https://iam-proxy.mycompany.ru;
   - убедитесь, что новые поды прошли проверку готовности (`Ready`).
