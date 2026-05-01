# Удаление

Для удаления сервиса с сервера на VM, рекомендуется использовать скрипт удаления из дистрибутива
`package/system-prepare/system-prepare.zip/system-prepare/unprep-all.sh`, который запускается с правами root.

В рамках выполнения скрипта, удалятся следующие артефакты/сервисы:

- балансировщик нагрузки;
- Fluent Bit;
- RDS Client;
- компонент IAM Proxy;
- компонент Keycloak.SE.

> Удаляет на исполняемом экземпляре, при наличии артефакта/сервиса в нем.

## Сценарий удаления 1

**Стратегия удаления**:  
Ручное удаление всех компонентов IAM Proxy и связанных сервисов через скрипт `unprep-all.sh`.

### Шаги удаления

1. **Запуск скрипта удаления**:
    - перейдите в директорию дистрибутива и распакуйте архив со скриптами подготовки:
      ```shell
      cd package/system-prepare
      unzip system-prepare.zip -d /tmp && cd /tmp/system-prepare
      cd /tmp/system-prepare
      ```  
    - запустите скрипт с правами root, и задав в env `DEPLOYER_USER` имя УЗ DevOps которая использовалась для деплоя
      продукта:
      ```shell
      chmod a+x *.sh
      export DEPLOYER_USER=deployer
      sudo ./unprep-all.sh
      ```  

2. **Проверка удаления**:
    - убедитесь, что следующие компоненты удалены:
        - балансировщик нагрузки;
        - Fluent Bit;
        - RDS Client;
        - IAM Proxy;
        - Keycloak.SE.

3. **Очистка остаточных файлов**:
    - проверьте что служебные УЗ и группы были действительно удалены (iamproxy-deployer, iamproxy, vault, ...).

## Сценарий удаления 2

**Стратегия удаления**:  
Автоматизация удаления через Ansible Playbook с использованием скрипта `unprep-all.sh`.

### Шаги удаления

1. **Подготовка Playbook**:
    - перейдите в директорию дистрибутива и распакуйте архив со скриптами подготовки:
      ```shell
      cd package/system-prepare
      unzip system-prepare.zip -d /tmp && cd /tmp/system-prepare
      ```  
    - создайте файл `remove-iamproxy.yml` с содержимым (задав в env `DEPLOYER_USER` имя УЗ DevOps которая использовалась
      для деплоя продукта):
      ```yaml
      - name: Remove IAM Proxy components
        hosts: all
        become: yes
        environment:
          DEPLOYER_USER: deployer
        tasks:
          - name: Execute unprep-all.sh
            script: /tmp/system-prepare/unprep-all.sh
      ```  
    - создайте файл `inventory/hosts` с содержимым:
      ```
      [servers]
      server1.mycompany.ru
      server2.mycompany.ru
      ```  

2. **Запуск Playbook**:
    - выполните команду:
      ```shell
      ansible-playbook -i inventory/hosts remove-iamproxy.yml
      ```  
    - Playbook скопирует на сервера и запустит `unprep-all.sh`, который остановит службы и удалит остаточные файлы
      продукта.

3. **Проверка результата**:
    - используйте команду `systemctl list-units --type=service | grep iamproxy` для проверки отсутствия активных служб;
    - убедитесь, что директории `/opt/iamproxy/` и `/var/log/fluent-bit/` удалены.

> **Примечание**:
> - Скрипт `unprep-all.sh` удаляет только те компоненты, которые были установлены на данном сервере.
> - Для корректного удаления необходимо запускать скрипт с правами root.

## Сценарий удаления 3: Удаление базы данных IAM

### Шаги удаления

1. Остановите базу данных: `systemctl stop postgresql`.
2. Удалите данные `rm -rf /var/lib/postgresql/data/iam_db`.
3. Удалите пользователя и роль:

```postgresql
DROP DATABASE iam_db;
DROP USER iam_user;
```
