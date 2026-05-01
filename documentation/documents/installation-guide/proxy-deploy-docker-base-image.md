# Сборка базового образа контейнера для IAM Proxy

## Назначение

Данное руководство предназначено для потребителей, которые будут собирать образы контейнеров IAM Proxy в ограниченной
инфраструктуре в отношении доступа к получению пакетов/зависимостей.

В связи с тем, что образы контейнеров компонента IAM Proxy в процессе сборки устанавливают пакеты (и их зависимости) для
работы компонентов IAM Proxy, появляется необходимость обеспечить из сборщика образов доступ к доверенным
репозиториям с пакетами для `yum`, `apt`, `python`, учитывая особенности инфраструктуры в которой происходит сборка.

> Dockerfile в дистрибутиве AUTH расположен по пути:
> - IAM Proxy: `package/docker/iamproxy/Dockerfile`;
> - RDS Server: `package/docker/rdsserver/Dockerfile`.

## Сборка

Для сборки своего базового образа, необходимо в используемых "чистых" базовых образах настроить подключение к доверенные
репозиториям, из которых можно будет загружать зависимости пакеты/зависимости в текущей инфраструктуре (в
инфраструктуре, где выполняется сборка образов IAM Proxy).

> Примечание
>
> Сборка и использование своего базового образа не является обязательной, и выполняется только по необходимости,
> исходя из требований текущей инфраструктуры.
> 
> Настроить подключение к репозиториям можно и без использования отдельного базового образа, создав необходимые файлы настроек yum и pip непосредственно при сборке образа компонентов.

Пример сборки базового образа для IAM Proxy, RDS Server на базе RHEL UBI8:

```Dockerfile
FROM registry.mycompany.ru/somecontext/redhat/ubi8:latest

RUN rm -f /etc/yum.repos.d/* && \
    echo -en  "\
[RHEL8-baseos]\n\
name=RHEL8-baseos\n\
baseurl=http://rhel-repo-mirror.mycompany.ru/repo/rhel8/rhel-8-for-x86_64-baseos-rpms/\n\
gpgcheck=0\n\
enabled=1\n\
\n\
[RHEL8-appstream]\n\
name=RHEL8-appstream\n\
baseurl=http://rhel-repo-mirror.mycompany.ru/repo/rhel8/rhel-8-for-x86_64-appstream-rpms/\n\
gpgcheck=0\n\
enabled=1\n\
\n\
[RHEL8-EPEL]\n\
name=RHEL8-EPEL\n\
baseurl=http://rhel-repo-mirror.mycompany.ru/repo/rhel8/EPEL8/\n\
gpgcheck=0\n\
enabled=1\n\
" > /etc/yum.repos.d/mirror_rhel8.repo && \
    mkdir ~/.pip && \
    echo -en  "\
[global]\n\
index_url=https://token:<some_tocken>@some.osc.domain/osc/repo/pypi/simple\n\
trusted_host=some.osc.domain\n\
default_timeout=200\
" >> ~/.pip/pip.conf
```

Для сборки базового образа по созданному выше примеру `Dockerfile`, на АРМ с установленным `docker` выполнить команды
ниже. При этом ссылки в примере Dockerfile заменить на ссылки из текущей инфраструктуры, а в командах
ниже `ubi8-mycompany` заменить на имя нового образа.

```
docker build -t ubi8-mycompany:latest .
docker image tag ubi8-mycompany registry.mycompany.ru/my-base-images/ubi8-mycompany:1.0
docker image push registry.mycompany.ru/my-base-images/ubi8-mycompany:1.0
```
