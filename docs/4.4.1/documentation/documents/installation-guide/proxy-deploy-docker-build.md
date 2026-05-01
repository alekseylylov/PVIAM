# Сборка образов контейнеров компонента IAM Proxy

## Ограничения

При сборке образов компонентов IAM Proxy рекомендуется использовать "чистые" базовые образы ОС (без предустановленного
дополнительного ПО). Рекомендуется использовать следующие образы, как базовые при сборке:

- `rhel8`;
- `ubi8`;
- `alt-sp10`;
- `sberlinux8`.

В базовых образах должна быть возможность использовать пакетный менеджер `yum` или `apt`, и возможность ими в
инфраструктуре сборки установить пакеты (и все их зависимости):

- `unzip`;
- `gettext`;
- `python3`;
- `openssl`;
- `java-1.8.0-openjdk-headless`;
- `java-11-openjdk-headless`;
- `ivykis`
- `glibc-langpack-en` (только для rhel);
- `glibc-langpack-ru` (только для rhel);
- `libmaxminddb libgd3` (только для altlinux);
- `python3-module-setuptools` (только для altlinux);
- `libnsl.so.1`, `libivykis.so.0`, `libssl10`, `libnet2` (только для altlinux);

> Примечание:
> Для RHEL8 и SberLinux8 все необходимые пакеты и зависимости можно получить,
> подключив репозитории: baseos, appstream, EPEL.

В базовых образах должна быть возможность использовать пакетный менеджер `pip`, и возможность им в инфраструктуре сборки
установить пакеты (и все их зависимости): `j2cli[yaml]`

## Сборка образов при помощи Platform V DevOps Tools

Сборку образов и их выгрузку в репозиторий хранения docker-образов можно выполнить при объединении дистрибутивов
утилитой `Platform V DevOps Tools Merger`. Для корректной сборки необходимо в конфигурацию `Merger` добавить
сопоставление имен базовых образов из Dockerfile дистрибутива, с базовыми образами доступными в инфраструктуре, где
производится сборка. При этом используемые базовые образы должны позволять устанавливать пакеты пакетным
менеджером (`yum`, `apt`, `pip`), и содержать внутри:

- либо уже установленные требуемые для IAM Proxy пакеты;
- либо настроенные репозитории пакетов, доступных в инфраструктуре сборки (смотрите раздел
  [Сборка базового образа контейнера для IAM Proxy](../installation-guide/proxy-deploy-docker-base-image.md));
- либо задать в параметрах `Merger` настройки пакетных менеджеров на репозитории пакетов, доступных в инфраструктуре
  сборки (смотрите подраздел "Настройки пакетных менеджеров в параметрах Merger").

> Dockerfile в дистрибутиве AUTH расположен по пути:
> - IAM Proxy: `package/docker/iamproxy/Dockerfile`;
> - RDS Server: `package/docker/rdsserver/Dockerfile`.

### Настройка соответствия базовых образов

В файле настройки Merger (`merger.yml`) добавить/заменить соответствия в параметре `base_image_mapping`:

```yaml
docker:
  registry: # координаты реджистри для сборки и загрузки докер-образов
    url: https://registry.mycompany.ru
    creds: tuz_cd_auth
  # путь для загрузки докер-образов
  registry_path: platform-v/iam
  # маппинг базовых образов
  base_image_mapping:
    - from: ".*openjdk-17.*"
      to: registry.mycompany.ru/base/redhat/ubi/openjdk-17:latest
    - from: .*ubi8:.*
      to: registry.mycompany.ru/base/redhat/ubi/ubi8:latest
```

> Примечание:
> Более подробное описание настройки Merger смотрите в документации продукта `Platform V DevOps Tools`.

### Настройки пакетных менеджеров в параметрах Merger

В настройках Merger необходимо настроить расширения, которые будут формировать файлы конфигурации пакетных менеджеров,
для этого файле `merger.yml` добавить:

```yaml
extensions:
  - stage: 'Rebuild_docker' # наименование шага, для которого включается точка расширения
    phase: 'before' # этап шага, на котором включается точка расширения
    cmd:
      - "ls -la $WORKSPACE/solution/auth-bin-*/package/bh/*/packages"
      # добавляем репо с пакетами RHEL8
      - |
        for dir in $WORKSPACE/solution/auth-bin-*/package/bh/*/packages ; do
          echo -e '
        [RHEL8-baseos]
        name=RHEL8-baseos
        baseurl=http://registry.mycompany.ru/trusted_repo/rhel/el8/8.4-rhel-8-for-x86_64-baseos-eus-rpms/
        gpgcheck=0
        enabled=1

        [RHEL8-appstream]
        name=RHEL8-appstream
        baseurl=http://registry.mycompany.ru/trusted_repo/rhel/el8/8.4-rhel-8-for-x86_64-appstream-eus-rpms/
        gpgcheck=0
        enabled=1

        [RHEL8-EPEL]
        name=RHEL8-EPEL
        baseurl=http://registry.mycompany.ru/trusted_repo/rhel/el8/rhel-extras/EPEL8/
        gpgcheck=0
        enabled=1
        '>$dir/mirror_rhel8_in_dmz.repo
        done

      # добавляем репо с пакетами питона
      - |
        for dir in $WORKSPACE/solution/auth-bin-*/package/bh/*/packages ; do
          echo -e '
        [global]
        index_url=https://token:xxxxxxxxxxxxxxxxxxxxxxxxxx@osc.mycompany.ru/osc/repo/pypi/simple
        trusted_host=osc.mycompany.ru
        default_timeout=200
        '>$dir/pip.conf
        done
```

> Примечания
>
> Выше указан пример для настройки yum в RHEL8.
>
> При использовании в базовом образе apt, необходимо в примере
> вместо `mirror_rhel8_in_dmz.repo` задать `mirror_altlinux_in_dmz.list`
> (`altlinux` заменяется на ID ОС из базового образа).

## Сборка образов стандартными средствами (без использования инструментов Platform V DevOps Tools)

Для сборки образа можно использовать скрипт `makeDockerImage.sh` входящий в состав
дистрибутива (`proxy/platformauth-proxy.zip/Docker/makeDockerImage.sh`). Данный скрипт по сути запускает по очереди:

- `docker build ...`;
- `docker tag ...`;
- `docker push ...`.

1. Распаковать архив `proxy/platformauth-proxy.zip` с компонентом прокси в отдельный каталог, на сервере где есть
   установленный `docker` и есть доступ к репозиториям используемым пакетным менеджером в базовом образе (yum, apt, ...)
   , и к репозиториям пакетов Python (pip simple)
2. Перейти в каталог Docker, в том месте куда распаковали ранее архив, и запустить `makeDockerImage.sh` с указанием в
   параметрах репозитория для выгрузки образа, и параметров для сборки (для `docker build`)

### Параметры сборки

```shell
./makeDockerImage.sh [docker_registry_for_push] [params_for_docker_build ...]
```

При запуске сборки необходимо будет первым параметром задать репозиторий для выгрузки, а далее указать параметры
для `docker build` (с ключом `--build-arg`):

- `BASE_IMAGE` - базовый образ, на котором будет собран IAM Proxy (поддерживаются RHEL8, UBI8, Альт 10 СП) (по
  умолчанию используется `ubi8`);
- `MIRROR_REPO_FILE` - ссылка на url, который по http отдает конфиг-файл yum/apt для настроек получения пакетов (по
  умолчанию используются репозитории из каталога сборки `packages/*.repo`);
- `MIRROR_PIP` - ссылка на репозиторий пакетов для Python (pip-simple, по умолчанию используются репозиторий из каталога
  сборки `packages/pip.conf`).

Примечание: При получении данных о репозитории из URL в `MIRROR_REPO_FILE` или из каталога `packages`, все репозитории
полученные из базового образа будут предварительно отключены. Если не задан `MIRROR_REPO_FILE` и нет файлов в
каталоге `packages`, то репозитории будут использованы из базового образа.

### Примеры сборки на разных базовых образах

Пример под RHEL8:

```shell
./makeDockerImage.sh registry.mycompany.ru/platform-v/iam \
  --build-arg BASE_IMAGE=registry.mycompany.ru/base/redhat/rhel8:latest \
  --build-arg MIRROR_REPO_FILE=http://mirror.mycompany.ru/rhel/conf/mirror_rhel8.repo \
  --build-arg MIRROR_PIP=https://token:XXXXXXXXXXXXXXXXXX@osc.mycompany.ru/repo/pypi/simple
```

Пример под ОС RHEL UBI8:

```shell
cat > ../packages/mirror_rhel8.repo <<EOF
[RHEL8-baseos]
name=RHEL8-baseos
baseurl=http://registry.mycompany.ru/repo/rhel8/rhel-8-for-x86_64-baseos-rpms/
gpgcheck=0
enabled=1

[RHEL8-appstream]
name=RHEL8-appstream
baseurl=http://registry.mycompany.ru/repo/rhel8/rhel-8-for-x86_64-appstream-rpms/
gpgcheck=0
enabled=1

[RHEL8-EPEL]
name=RHEL8-EPEL
baseurl=http://registry.mycompany.ru/trusted_repo/rhel/el8/rhel-extras/EPEL8/
gpgcheck=0
enabled=1
EOF

./makeDockerImage.sh registry.mycompany.ru/platform-v/iam \
  --build-arg BASE_IMAGE=registry.mycompany.ru/registry_redhat_io/ubi8/ubi8:latest \
  --build-arg MIRROR_PIP=https://token:4f9b1b78ad43xxxx7887@develop.mycompany.ru/osc/repo/pypi/simple
```

Пример под Альт 10 СП (где alt10.mirror.mycompany.ru - пример хоста репозитория пакетов Альт 10 СП):

```shell
cat > ../packages/mirror_altlinux10.list <<EOF
rpm [cert8] http://alt10.mirror.mycompany.ru/ c9f2/branch/x86_64 classic
rpm [cert8] http://alt10.mirror.mycompany.ru/ c9f2/branch/x86_64-i586 classic
rpm [cert8] http://alt10.mirror.mycompany.ru/ c9f2/branch/noarch classic
EOF

./makeDockerImage.sh registry.mycompany.ru/platform-v/iam \
  --build-arg BASE_IMAGE=mycompany.ru/mycompany-base/gostech/alt-sp10/clean:v0.5.15 \
  --build-arg MIRROR_PIP=https://token:XXXXXXXXXXXXXXX@osc.mycompany.ru/osc/repo/pypi/simple
```

Пример под SberLinux:

```shell
cat > ../packages/mirror_sberlinux8.repo <<EOF
[SberLinux8-baseos]
name=SberLinux8-baseos
baseurl=http://registry.mycompany.ru/repo/sberlinux8/sberlinux-8-for-x86_64-baseos-rpms/
gpgcheck=0
enabled=1

[SberLinux8-appstream]
name=SberLinux8-appstream
baseurl=http://registry.mycompany.ru/repo/sberlinux8/sberlinux-8-for-x86_64-appstream-rpms/
gpgcheck=0
enabled=1

[SberLinux8-EPEL]
name=SberLinux8-EPEL
baseurl=http://registry.mycompany.ru/repo/rhel-extras/EPEL8/
gpgcheck=0
enabled=1
EOF

./makeDockerImage.sh registry.mycompany.ru/platform-v/iam \
  --build-arg BASE_IMAGE=registry.mycompany.ru/repo/sberlinux8/sberlinux8:latest \
  --build-arg MIRROR_PIP=https://token:XXXXXXXXXXXXXXXXXXX@osc.mycompany.ru/osc/repo/pypi/simple
```