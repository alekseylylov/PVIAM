# Откат

Раздел описывает процедуру отката на предыдущую версию для нескольких вариантов инсталляций:

- IAM Proxy + KeyCloak.SE;
- IAM Proxy.

Если требуется откатить только IAM Proxy, то шаги по дампу нужно пропустить.

> Возможно осуществление отката на любую из поддерживаемых предыдущих версий (LTS-версии)
> При откате на ранние версии, функциональности добавленные в поздних версиях будут отсутствовать

## Предусловия для отката IAM Proxy в составе с KeyCloak.SE

Перед началом отката должны быть выполнены следующие действия:

1. Подготовлен архив в папке (создается автоматически развертыванием в
   `/home/nginx/backup_nginx_<ver>_<date install>.tgz`).
2. Подготовлен дамп базы данных или схемы базы данных.

Если дамп базы данных отсутствует, необходимо:

1. Удостовериться в наличии доступа по SSH к кластеру СУБД.
2. Удостовериться в доступности утилиты `psql` при подключении к кластеру СУБД.
3. Используя утилиту `pg_dump` снять дамп базы данных.
4. Снять дамп базы данных KeyCloak.SE:
   Пример:

```
pg_dump -U postgres -Fcbvf "/db/my_db_keycloak.dump" db_keycloak
```

где:

`-U postgres` - пользователь, который подключиться к БД db_keycloak;

`/db/my_db_keycloak.dump` - расположение файла в который будет выгружен дамп;

`db_keycloak` - наименование базы данных;

Снять дамп схемы базы данных KeyCloak.SE:

Пример:

```
pg_dump -U postgres -Ftar --schema=schema_name  db_keycloak > /db/my_schema_name_db_keycloak.tar
```

где:

`-U postgres` - пользователь, который подключиться к БД db_keycloak

`--schema=schema_name` - наименование схемы, дамп которой планируется снять.

`db_keycloak` - наименование базы данных в которой хранится схема.

`/db/my_schema_name_db_keycloak.tar` - расположение файла в который будет выгружен дамп. Важно не указывать временную
директорию. Если сохранить дамп в `/tmp/`, то после отката схема не сохранится.

Наименование базы данных и схем можно посмотреть в профиле развертывания `keycloak.yml` или в `standalone-ha.xml`.

### Шаги отката

### Вариант 1. Откат через развертывание дистрибутива предыдущей версии

Для отката с помощью развертывания предыдущей версии дистрибутива:

1. Подключитесь к кластеру Pangolin.
   * Дамп базы данных.
   
     Восстановиться из ранее созданного дампа базы данных `/db/my_db_keycloak.dump`. Для этого нужно воспользоваться утилитой
     pg_restore:
     
     Пример:
     
     ```
     pg_restore -U postgres -d db_keycloak -v "/db/my_db_keycloak.dump"
     ```
     
     где:
     
     `-U postgres` - пользователь, который подключиться к откатываемой БД db_keycloak
     
     `"/db/my_db_keycloak.dump"` - расположение файла дампа который будет использован для восстановления БД
   
   * Дамп схемы базы данных.
   
     Восстановиться из ранее созданного дампа схемы `/db/my_schema_name_db_keycloak.tar`. Для этого нужно воспользоваться
     утилитой pg_restore:
     
     Пример:
     
     ```
     pg_restore -Ft -U postgres -d db_keycloak  /db/my_schema_name_db_keycloak.tar
     ```
     
     где:
     
     `-U postgres` - пользователь, который подключиться к откатываемой БД `db_keycloak`
     
     `/db/my_schema_name_db_keycloak.tar` - расположение схемы которая будет использована для восстановления
   
2. При помощи Jenkins job развернуть релиз на который откатываем, например `D-04.003.00_7_4.2.0`.

### Вариант 2. Откат через ручное восстановление файлов из резервной копии

1. Подключиться к кластеру Pangolin

   * Дамп базы данных
   
     Восстановиться из ранее созданного дампа базы данных `/db/my_db_keycloak.dump`. Для этого нужно воспользоваться утилитой
     pg_restore:
     
     Пример:
     
     ```
     pg_restore -U postgres -d db_keycloak -v "/db/my_db_keycloak.dump"
     ```
     
     где:
     
     `-U postgres` - пользователь, который подключиться к откатываемой БД db_keycloak
     
     `"/db/my_db_keycloak.dump"` - расположение файла дампа который будет использован для восстановления БД
     
   * Дамп схемы базы данных
   
     Восстановиться из ранее созданного дампа схемы `/db/my_schema_name_db_keycloak.tar`. Для этого нужно воспользоваться
     утилитой pg_restore:
     
     Пример:
     
     ```
     pg_restore -Ft -U postgres -d db_keycloak  /db/my_schema_name_db_keycloak.tar
     ```
     
     где:
     
     `-U postgres` - пользователь, который подключиться к откатываемой БД db_keycloak
     
     `/db/my_schema_name_db_keycloak.tar` - расположение схемы которая будет использована для восстановления
     
2. В директории пользователя сервиса:

   - удалить каталоги `/opt/iamproxy/conf`, `/opt/iamproxy/html`
     , `/opt/iamproxy/logs`, `/opt/iamproxy/site`;
   - перейти в папку `/home/nginx`, она содержит файлы-копий backup_nginx_{ver}_{date install}.tgz, автоматически
     создаваемые развертыванием в разрезе версия+дата;
   - выбрать файл с резервной копией (/home/nginx/backup_nginx_{ver}_{date install}.tgz), и восстановить из архива
     удаленные выше каталоги (или восстановить все файлы сервиса под root, командой `tar -xf my.tgz -C /`).
   
   > **Важно!**\
   > Не стоит перезаписывать бинарные файлы при восстановлении из резервной копии, в противном случае
   > могут возникнуть проблемы с правами.
   > Если все же необходимо перезаписать исполняемые файлы, то на RHEL надо будет восстановить capabilities на них
   > (по аналогии того, как делают скрипты подготовки), например:\
   > `setcap CAP_NET_BIND_SERVICE=+eip /opt/iamproxy/sbin/nginx`
   
   `Справочно: развертывание создает резервную копию 1 раз в день.`
   
## Проблемы и решения

### Откат завершился неудачно по причине отсутствия новой функциональности в версии на который производим откат

```
fatal: [proxy_1]: FAILED! => {"changed": true, "cmd": "sudo -n -u root /bin/systemctl enable nginx\nsudo -n -u root /bin/systemctl
restart iamproxy || (echo \"Error restart proxy!\" ; tail /opt/iamproxy/logs/nginx.log ; exit 1 )\n",
"delta": "0:00:00.192197", "end": "2022-12-28 11:02:04.769195", "msg": "non-zero return code", "rc": 1,
"start": "2022-12-28 11:02:04.576998", "stderr": "Job for iamproxy.service failed because the control process exited
with error code. See \"systemctl status iamproxy.service\" and \"journalctl -xe\" for details.", "stderr_lines":
 ["Job for iamproxy.service failed because the control process exited with error code. See \"systemctl status iamproxy.service\"
 and \"journalctl -xe\" for details."], "stdout": "Error restart proxy!", "stdout_lines": ["Error restart proxy!"]}
```

#### Решение

- убедитесь в том, что конфигурационные настройки соответствуют версии релиза на который производится откат;
- перезапустите сборку;
- перезапустите `systemd nginx IAM Proxy unit`.
