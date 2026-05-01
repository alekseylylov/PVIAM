# Рекомендации по получению дистрибутива

В связи с особенностями сборки Docker image, описанными в документации компонент IAM Proxy [Сборка базового образа контейнера для IAM Proxy](proxy-deploy-docker-base-image.md) и
KeyCloak SE [Требования к базовому образу](docs://./documents/installation-guide/base_image_requirements.html) настройки переопределения базовых
образов также имеют особенности в использовании базового образа в качестве целевого.

Для успешной сборки контейнеров на стороне потребителя (то есть при подготовке дистрибутива с помощью Platform V
Devops Tools) необходимо сконфигурировать файл настроек `solution-merger` и сослаться на него при запуске
`solution-merger`.

Пример файла *merger.yml*:
```YAML
build_history:
  builds:
    days: 100                                                   # сколько дней хранить сборки
    num: 30                                                     # сколько хранить сборок

tooling:                                                        # какие конкретно утилиты использовать из инструментария jenkins
  jdk: openjdk-11.0.2_linux                                     # версия ждк, которую будет использовать мавен для скачивания и загрузки компонентов
  maven: Maven 3.6.3                                            # версия мавена, которую использовать для скачивания и загрузки архивов
  npm: v7.5.0-linux-x64                                         # версия нпм
  groovy: '3.0.7'                                               # версия груви, которой будет объединяться дистрибутив (из овнед и пати), версия groovy должна быть 3.x и выше

repositories:
  solution:
    download:                                                   # адрес репозитория и ID credential Jenkins для скачивания солюшна
      url: https://mycompany.domain/nexus-cd/repository/PROD
      creds: CRED_ID
    upload:                                                     # адрес репозитория и ID credential Jenkins для загрузки объединенного солюшна
      url: https://mycompany.domain/nexus-cd/repository/PROD
      creds: CRED_ID
      groupId: 'COMPANY.CMDBID000000_iam.iam_merge'             # groupId, с которым артефакт будет загружен в Нексус
      artifactId: 'iam'                                         # artifactId, с которым артефакт будет загружен в Нексус
  maven:
    download:                                                   # здесь настройка того, откуда мы будем брать плагины для maven
      url: https://mycompany.domain/nexus-ci/repository/maven-central-proxy/
      creds: CRED_ID
    upload:                                                     # адрес репозитория для загрузки мавен артефактов (сдк)
      url: https://mycompany.domain/nexus-ci/repository/mycompany_maven/
      creds: CRED_ID
  npm:                                                          # адрес репозитория для загрузки нпм артефактов (сдк)
    url: https://mycompany.domain/nexus-ci/repository/some-npm-repo
    creds: CRED_ID

docker:
  registry:                                                     # координаты реджистри для сборки и загрузки докер-образов
    url: https://registry.mycompany.domain
    creds: CRED_ID

  registry_path: somecontext/somepath/auth/merged               # путь для загрузки докер-образов

  base_image_mapping:                                           # маппинг базовых образов
    - from: .*openjdk-11-rhel8.*
      to: https://registry.mycompany.domain/somecontext/someregistripath/base/openjdk-11-rhel8:0.0.8
    - from: .*rhel7.*
      to: https://registry.mycompany.domain/somecontext/someregistripath/base/hel7-for-syslogng:0.0.8
    - from: .*alt-sp8.*
      to: https://registry.mycompany.domain/somecontext/someregistripath/base/ubi8-minimal-jdk11:1.0.0

  image_link_mapping:
    "mona/agent@sha256:[0-9a-f]{64}" : "mona/agent@sha256:cbd08dc4877d5b1b0cb81fd35b02c482737e12e1860ab6117962eef3075a6aa5"
    "supa/client@sha256:[0-9a-f]{64}" : "supa/client@sha256:7c42e54a4d3b3175b4912781fcfcfd0709d7f6bd3169c9f5a8b69ce7cfc75e3f"
    "otts/ott-sidecar:[0-9a-f]{64}" : "otts/ott-sidecar@sha256:ad17853e985d26ecb0bdc7efdbe7f789b3bcc03571b143cd5a5296f4286b50d5"

  additional_registries:
    - "registry.mycompany.domain"

# Расширение: подключение репозиториев при отсутствии в базовом образе
extensions:
  stage: 'Rebuild_docker'                                       # наименование шага, для которого включается точка расширения
  phase: 'before'                                               # этап шага, на котором включается точка расширения
  cmd: |
    # Проверка наличия директорий с пакетами
    if [ -d "$WORKSPACE/solution/auth-bin-/package/bh/packages" ]; then
      echo "=== Добавление YUM-репозиториев для RHEL8/SberLinux ==="
      for dir in $WORKSPACE/solution/auth-bin-/package/bh/packages/*; do
        if [ -d "$dir" ]; then
          echo "Настройка .repo файлов в $dir"
          cat > "$dir/sberlinux.repo" << 'EOF'
[BaseOS-bin]
name=SberLinux 8.9 - BaseOS bin
baseurl=http://some.repo.host/sberlinux/8.9.1/sberlinux-8-for-x86_64-baseos-rpms/
gpgcheck=0
enabled=1

[AppStream-bin]
name=SberLinux 8.9 - AppStream bin
baseurl=http://some.repo.host/sberlinux/8.9.1/sberlinux-8-for-x86_64-appstream-rpms/
gpgcheck=0
enabled=1

[rhel-8-for-x86_64-EPEL8-rpms]
name=rhel-8-for-x86_64-EPEL8-rpms
baseurl=http://some.repo.host/altlinux_trusted_repo/rhel/el8/rhel-extras/EPEL8
gpgcheck=0
enabled=1
EOF
        fi
      done

      echo "=== Добавление PIP-конфигурации для Python ==="
      for dir in $WORKSPACE/solution/auth-bin-/package/bh/packages/*; do
        if [ -d "$dir" ]; then
          echo "Настройка pip.conf в $dir"
          cat > "$dir/pip.conf" << 'EOF'
[global]
index_url=https://token:@100.x.x.100/osc/repo/pypi/simple
trusted_host=100.x.x.140
default_timeout=200
EOF
        fi
      done
    else
      echo "Предупреждение: не найдены пакетные директории для установки репозиториев."
      exit 1
    fi
```
