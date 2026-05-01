# Диаграммы последовательностей

Диаграммы ниже описывают взаимодействия возникающие при работе пользователя с ресурсами, используя IAM Proxy.

## Диаграмма основного сценария аутентификации используя OIDC Code Flow

Диаграмма описывает взаимодействия возникающие в сценариях:

- [Пройти аутентификацию](../about/use-case-authentification.md)
- [Получить ресурс требующий аутентификации](../about/use-case-accessing-resource.md)

```puml
@startuml

title Аутентификация на IAM Proxy используя OIDC Code Flow

!$firstURL  = "URL-1"

skinparam backgroundColor #F9F9FF
skinparam sequence {
    ArrowColor red
    LifeLineBorderColor black
    
    ActorBorderColor blue
    ActorBackgroundColor Beige
    ActorFontColor black
    ActorFontSize 16
}

autonumber

actor User as "Пользователь"
participant Browser as "Клиентское приложение \n (браузер, МП)"
collections Proxy as "RP \n Прокси сервер \n (IAM Proxy)"
participant IDP as "OP \n Провайдер аутентификации \n (IAM KCSE и т.п.)"
collections Backend as "Бэкенд ресурса \n (приложение АС)"

== Пройти аутентификацию ==
User -> Browser: Обращение к приложения по $firstURL
Browser -> Proxy: Обращение к ресурсу по $firstURL, \n требующему аутентификации
activate Proxy
note left of Proxy: Сессии IAM Proxy нет в запросе, \n или токены некорректны (просрочены и т.п.)
Proxy --> Browser: HTTP status 302. \n Перенаправление на провайдер аутентификации. \n Установка сессии IAM Proxy (установка cookie) 
deactivate Proxy
Browser -> IDP: Обращение за аутентификацией OIDC к провайдеру аутентификации
activate IDP
IDP --> Browser: Запрос данных для аутентификации \n (форма ввода данных, запрос сертификата и т.п.)
Browser -> User: Отображение формы \n для ввода данных
User --> Browser: Ввод данных для аутентификации
Browser -> IDP: Передача данных для аутентификации
IDP->IDP: Проверка данных аутентификации
IDP --> Browser: HTTP status 302. Перенаправление с кодом авторизации
deactivate IDP
Browser -> Proxy: Код авторизации в провайдере аутентификации
activate Proxy
Proxy -> IDP: Получение access, refresh и ID токенов \n по коду авторизации
activate IDP
IDP --> Proxy: Отправка access, refresh, ID токенов
deactivate IDP
Proxy --> Proxy: Запись токенов в сессию
Proxy --> Browser: HTTP status 302. \n Перенаправление на ресурс по $firstURL. \n Установка сессии IAM Proxy с токенами (установка cookie).
deactivate Proxy
== Получить ресурс требующий аутентификации ==
loop Выполнение запросов в рамках текущей сессии
  Browser -> Proxy: Обращение к ресурсу
  activate Proxy
  Proxy -> Proxy: Получение access, refresh и ID токенов из сессии
  alt Если срок access токена истек и есть действующий refresh токен, \n то выполнение обновления токенов
    Proxy -> IDP: Получение access, refresh и ID токенов по refresh токену
    activate IDP
    IDP --> Proxy: Отправка access, refresh, ID токенов
    deactivate IDP
    Proxy --> Proxy: Запись токенов в сессию
  else Если нет токенов, или срок access токена истек и нет действующего refresh токена
    break #Pink Если не получили токены
        Browser  <-- Proxy: Ошибка 401 или перенаправление на IDP
        ...
    end
  end
  Proxy -> Backend: Обращение к ресурсу + JWT-токен \n в HTTP заголовке Authorization (access или ID токен)
  Backend --> Proxy: HTTP ответ
  Proxy --> Browser: HTTP ответ. \n Если были изменения в сессии то установка \n сессии IAM Proxy (установка cookie).
  deactivate Proxy
end
Browser --> User: Отображение приложения по $firstURL
@enduml
```

## Диаграмма сценария получения публичного ресурса

Диаграмма описывает взаимодействия возникающие в сценариях:

- [Получить публичный ресурс](../about/use-case-accessing-resource.md)

```puml
@startuml

title Получение публичного ресурса через IAM Proxy

skinparam backgroundColor #F9F9FF
skinparam sequence {
    ArrowColor red
    LifeLineBorderColor black
    
    ActorBorderColor blue
    ActorBackgroundColor Beige
    ActorFontColor black
    ActorFontSize 16
}

autonumber

actor User as "Пользователь"
participant Browser as "Клиентское приложение \n (браузер, МП)"
collections Proxy as "Прокси сервер \n (IAM Proxy)"
collections Backend as "Бэкенд ресурса \n (приложение АС)"

== Получить публичный ресурс ==
User -> Browser: Обращение к ресурсу
Browser -> Proxy: Обращение к ресурсу
activate Proxy
Proxy -> Backend: Обращение к ресурсу
Backend --> Proxy: HTTP ответ
Proxy --> Browser: HTTP ответ
deactivate Proxy
Browser --> User: Отображение ресурса
@enduml
```

## Диаграмма сценария выхода из системы

Диаграмма описывает взаимодействия возникающие в сценариях:

- [Выход из системы](../about/use-case-accessing-resource.md)

```puml
@startuml

title Выход из системы использующей аутентификацию OIDC Code Flow через IAM Proxy  

skinparam backgroundColor #F9F9FF
skinparam sequence {
    ArrowColor red
    LifeLineBorderColor black
    
    ActorBorderColor blue
    ActorBackgroundColor Beige
    ActorFontColor black
    ActorFontSize 16
}

autonumber

actor User as "Пользователь"
participant Browser as "Клиентское приложение \n (браузер, МП)"
collections Proxy as "RP \n Прокси сервер \n (IAM Proxy)"
participant IDP as "OP \n Провайдер аутентификации \n (IAM KCSE и т.п.)"
collections Backend as "Бэкенд ресурса \n (приложение АС)"

== Выход из системы ==
User -> Browser: Завершение работы \n в UI приложения системы
Browser -> Proxy: Обращение к URL завершения сессии \n по пользователю в приложении системы
activate Proxy
Proxy -> Backend: Обращение к URL завершения сессии
activate Backend
Backend -> Backend: Приложение системы производит \n завершение сессии по пользователю
Backend --> Proxy: HTTP status 302. \n Перенаправляет на относительный URI завершения сессии на IAM Proxy
deactivate Backend
Proxy --> Browser: HTTP status 302
Browser -> Proxy: Вызов URI завершения сессии на IAM Proxy
Proxy -> Proxy: HTTP status 302. \n IAM Proxy очищает данные по сессии.
Proxy -> IDP: Отзыв токенов в провайдере аутентификации
activate IDP
IDP -> IDP: Провайдер аутентификации \n закрывает сессию по токенам
IDP --> Proxy: Результат отзыва токенов
deactivate IDP
Proxy --> Browser: HTTP status 302. \n Перенаправляет пользователя в провайдер аутентификации \n для выполнения выхода из сессии. \n Очистка сессии IAM Proxy (установка cookie) 
Browser -> IDP: Обращение к URL завершения сессии на провайдере
activate IDP
IDP -> IDP: Провайдер аутентификации \n закрывает текущую сессию для client_id
IDP --> Browser: HTTP status 302. \n Перенаправляет пользователя обратно на IAM Proxy.
deactivate IDP
Browser -> Proxy: Обращение к IAM Proxy
Proxy --> Browser: Страница успешного завершения выхода
deactivate Proxy
Browser --> User: Отображение успешного \n завершения выхода

@enduml
```

## Диаграмма сценария управления ответвлениями

Диаграмма описывает взаимодействия возникающие в сценариях:

- [Управление ответвлениями](../about/use-case-junctions-management.md)

```puml
@startuml

title Управления ответвлениями IAM Proxy  

skinparam backgroundColor #F9F9FF
skinparam sequence {
    ArrowColor red
    LifeLineBorderColor black
    
    ActorBorderColor blue
    ActorBackgroundColor Beige
    ActorFontColor black
    ActorFontSize 16
}

autonumber

actor User as "Администратор"
participant Browser as "Клиентское приложение \n (браузер, МП)"
collections Proxy as "Прокси сервер \n (IAM Proxy)"
collections Backend as "Бэкенд ресурса \n (приложение с UI управления ответвлениями)"

== Периодическое обновление конфигурации ответвлений ==
loop При старте IAM Proxy запускается периодическое получение информации по ответвлениям \n (запускается RDS Client, который вызывает API RDS и обновляет конфигурацию)
    Proxy -> Backend: HTTP вызов к URL API RDS с информацией по ответвлениям
    activate Proxy
    activate Backend
    Backend --> Proxy: json c информацией по ответвлениям
    deactivate Backend
    alt Если успешно получен ответ и информация отличается от ранее полученной
      Proxy -> Proxy: Формирование файлов конфигурации на основе \n полученной информации об ответвлениях 
      Proxy -> Proxy: Тестирование файлов конфигурации, \n и при успешном тесте применение новых файлов конфигурации \n (выполнение reload SynGX)
    end 
    deactivate Proxy
end
...
== Пройти аутентификацию ==
note right of User: Шаги аутентификации смотрите на диаграммах для сценариев аутентификации в этом разделе
...
== Управления ответвлениями ==
User -> Browser: В UI выполнение действия с ответвлением \n (создание/изменение/удаление)
Browser -> Proxy: Обращение к URL выполнения действия с ответвлением
activate Proxy
Proxy -> Backend: Обращение к URL + JWT токен
activate Backend
Backend -> Backend: Авторизация права на выполнение действия
Backend -> Backend: Выполнение действия с ответвлением \n и его сохранение
Backend --> Proxy: Результат операции
deactivate Backend
Proxy --> Browser: Результат операции
deactivate Proxy
Browser --> User: Отображение результата операции

@enduml
```

Подробнее про реализацию и использование API RDS смотрите в разделе
[Параметры RDS Client](../installation-guide/proxy-deploy-docker-description.md#параметры-rds-client).

## Диаграмма аутентификации используя direct-auth

Диаграмма описывает взаимодействия возникающие в сценариях:

- [Пройти аутентификацию](../about/use-case-authentification.md)
- [Получить ресурс требующий аутентификации](../about/use-case-accessing-resource.md)

```puml
@startuml

title Аутентификация на IAM Proxy используя direct-auth

skinparam backgroundColor #F9F9FF
skinparam sequence {
  ArrowColor red
  LifeLineBorderColor black

  ActorBorderColor blue
  ActorBackgroundColor Beige
  ActorFontColor black
  ActorFontSize 16
}

autonumber

actor User as "Пользователь"
participant Agent as "Клиентское приложение \n (браузер, mvn и другие 'толстые' клиенты)"
collections Proxy as "Прокси сервер \n (IAM Proxy)"
participant IDP as "Провайдер аутентификации \n (IAM KCSE и т.п.)"
collections Backend as "бэкенд ресурса \n (приложение)"

User -> Agent: Обращение к ресурсу
Agent->Proxy: Обращение к ресурсу
== Пройти аутентификацию ==
alt Если в запросе есть заголовок Authorization
  alt#Gold Получение токенов из локального кеша
    Proxy -> Proxy: Поиск токенов в локальном кеше по ключу из Authorization
  else #LightGreen Если не найдено токенов в кеше, то получение токенов от IDP
    Proxy -> IDP: Запрос на получение токенов + заголовок Authorization
    IDP -> IDP: Аутентификация пользователя по данным \n из заголовка Authorization \n (например по Basic Authentication)
    Proxy <-- IDP: Передача токенов: access, refresh, ID
    Proxy -> Proxy: Сохранение токенов в локальном кеше на срок жизни access токена
  end
  break #Pink Если не получили токены
      Agent  <-- Proxy: Ошибка 403
      ...
  end
else Если нет заголовка Authorization, то используем стандартную аутентификацию (OIDC Code Flow)
  Agent  <-- Proxy: Перенаправление на IDP
  ...
end
== Получить ресурс требующий аутентификации ==
Proxy -> Backend: Обращение к ресурсу + JWT-токен \n в HTTP заголовке Authorization (access или ID токен) 
Proxy <-- Backend: HTTP ответ
Agent <-- Proxy: HTTP ответ
User <-- Agent: Отображение ресурса

@enduml
```

Про описание и настройку этой функциональности смотрите в
разделе [Описание настройки IAM Proxy при запуске в контейнере](../installation-guide/proxy-deploy-docker-description.md).

## Диаграмма сценария аутентификации используя OIDC Code Flow с использованием альтернативного провайдера

Диаграмма описывает взаимодействия при использовании функциональности переключения на альтернативный провайдер, возникающие в
сценариях:

- [Пройти аутентификацию](../about/use-case-authentification.md)
- [Получить ресурс требующий аутентификации](../about/use-case-accessing-resource.md)

```puml
@startuml

title Аутентификация на IAM Proxy используя OIDC Code Flow с использованием альтернативного провайдера

!$firstURL  = "URL-1"

skinparam backgroundColor #F9F9FF
skinparam sequence {
    ArrowColor red
    LifeLineBorderColor black
    
    ActorBorderColor blue
    ActorBackgroundColor Beige
    ActorFontColor black
    ActorFontSize 16
}

autonumber

actor User as "Пользователь"
participant Browser as "Клиентское приложение \n (браузер, МП)"
collections Proxy as "RP \n Прокси сервер \n (IAM Proxy)"
participant IDP as "OP \n Провайдер аутентификации \n (IAM KCSE и т.п.)"
participant AltIDP as "OP \n Альтернативный \n провайдер аутентификации"
collections Backend as "Бэкенд ресурса \n (приложение АС)"

== Периодическая проверка доступности основного провайдера ==
loop При старте IAM Proxy запускается периодическая проверка доступности основного провайдера  
    Proxy -> IDP: HTTP вызов к URL с метаданным провайдера
    activate Proxy
    activate IDP
    IDP --> Proxy: Ответ от провайдера
    deactivate IDP
    alt Если успешно получен ответ (riseCount раз подряд)  
      Proxy --> Proxy: Снимаем признак недоступности основного провайдера \n (mainProviderDown=false)
    else Если была ошибка соединения/tls или HTTP код ответа отличен от 200 (fallCount раз подряд) 
      Proxy --> Proxy: Устанавливаем признак недоступности основного провайдера \n (mainProviderDown=true)
    end 
    deactivate Proxy
end
...
== Пройти аутентификацию ==
User -> Browser: Обращение к приложения по $firstURL
Browser -> Proxy: Обращение к ресурсу по $firstURL, \n требующему аутентификации
activate Proxy
note left of Proxy: Сессии IAM Proxy нет в запросе, \n или токены некорректны (просрочены и т.п.)
alt #LightGreen Если основной провайдер доступен (если mainProviderDown == false)
    Proxy --> Browser: HTTP status 302. \n Перенаправление на основной провайдер аутентификации. \n Установка сессии IAM Proxy (установка cookie).
    deactivate Proxy
    Browser -> IDP: Обращение за аутентификацией OIDC к провайдеру аутентификации
    activate IDP
    IDP --> Browser: Запрос данных для аутентификации \n (форма ввода данных, запрос сертификата и т.п.)
    Browser -> User: Отображение формы \n для ввода данных
    User --> Browser: Ввод данных для аутентификации
    Browser -> IDP: Передача данных для аутентификации
    IDP->IDP: Проверка данных аутентификации
    IDP --> Browser: HTTP status 302. Перенаправление с кодом авторизации.
    deactivate IDP
    Browser -> Proxy: Код авторизации в провайдере аутентификации
    activate Proxy
    Proxy -> IDP: Получение access, refresh и ID токенов \n по коду авторизации
    activate IDP
    IDP --> Proxy: Отправка access, refresh, ID токенов
    deactivate IDP
else #LightBlue Если основной провайдер недоступен (если mainProviderDown == true)
    Proxy --> Browser: HTTP status 302. \n Перенаправление на альт-й провайдер аутентификации. \n Установка сессии IAM Proxy (установка cookie).
    deactivate Proxy
    Browser -> AltIDP: Обращение за аутентификацией OIDC к провайдеру аутентификации
    activate AltIDP
    AltIDP --> Browser: Запрос данных для аутентификации \n (форма ввода данных, запрос сертификата и т.п.)
    Browser -> User: Отображение формы \n для ввода данных
    User --> Browser: Ввод данных для аутентификации
    Browser -> AltIDP: Передача данных для аутентификации
    AltIDP->AltIDP: Проверка данных аутентификации
    AltIDP --> Browser: HTTP status 302. Перенаправление с кодом авторизации.
    deactivate AltIDP
    Browser -> Proxy: Код авторизации в провайдере аутентификации
    activate Proxy
    Proxy -> AltIDP: Получение access, refresh и ID токенов \n по коду авторизации
    activate AltIDP
    AltIDP --> Proxy: Отправка access, refresh, ID токенов
    deactivate AltIDP
end
Proxy --> Proxy: Запись токенов в сессию
Proxy --> Browser: HTTP status 302. \n Перенаправление на ресурс по $firstURL. \n Установка сессии IAM Proxy с токенами (установка cookie).
deactivate Proxy
== Получить ресурс требующий аутентификации ==
loop Выполнение запросов в рамках текущей сессии
  Browser -> Proxy: Обращение к ресурсу
  activate Proxy
  Proxy -> Proxy: Получение access, refresh и ID токенов из сессии
  alt Если срок access токена истек и есть действующий refresh токен, то выполнение обновления токенов
    alt #LightGreen Если основной провайдер доступен (если mainProviderDown == false)
      Proxy -> IDP: Получение access, refresh и ID токенов по refresh токену
      activate IDP
      IDP --> Proxy: Отправка access, refresh, ID токенов
      deactivate IDP
      break #Pink Если не удалось обновить токены (в том числе по причине того, что refresh токен с альт-го провайдера)
          Browser <-- Proxy: Ошибка 401 или перенаправление на основной провайдер
          ...
      end
    else #LightBlue Если основной провайдер недоступен (если mainProviderDown == true)
      Proxy -> AltIDP: Получение access, refresh и ID токенов по refresh токену
      activate AltIDP
      AltIDP --> Proxy: Отправка access, refresh, ID токенов
      deactivate AltIDP
      break #Pink Если не удалось обновить токены (в том числе по причине того, что refresh токен с основного провайдера)
          Browser <-- Proxy: Ошибка 401 или перенаправление на альт-й провайдер
          ...
      end
    end
    Proxy --> Proxy: Запись токенов в сессию
  else Если нет токенов, или срок access токена истек и нет действующего refresh токена, \n или не прошла проверка текущего id токена на конкретном экземпляре IAM Proxy \n (например для проверки id токена нет ключа на доступном провайдере или его client_id в aud)
    break #Pink Если нет токенов, или срок access токена истек и нет действующего refresh токена
        Browser <-- Proxy: Ошибка 401 или перенаправление на основной или альт-й провайдер \n (в зависимости от статуса в mainProviderDown)
        ...
    end
  end
  Proxy -> Backend: Обращение к ресурсу + JWT-токен \n в HTTP заголовке Authorization (access или ID токен)
  Backend --> Proxy: HTTP ответ
  Proxy --> Browser: HTTP ответ. \n Если были изменения в сессии то установка \n сессии IAM Proxy (установка cookie).
  deactivate Proxy
end
Browser --> User: Отображение приложения по $firstURL
@enduml
```

Про описание и настройку этой функциональности  смотрите в
разделе [Описание настройки IAM Proxy при запуске в контейнере](../installation-guide/proxy-deploy-docker-description.md).

## Диаграмма сценария аутентификации используя OIDC Code Flow с использованием альтернативного client-id

Диаграмма описывает взаимодействия при аутентификации в UI Администратора на IAM Proxy используя альтернативный
client-id, возникающие в сценариях:

- [Пройти аутентификацию](../about/use-case-authentification.md)
- [Получить ресурс требующий аутентификации](../about/use-case-accessing-resource.md)

```puml
@startuml

title Аутентификация в UI Администратора на IAM Proxy используя альтернативный client-id

skinparam backgroundColor #F9F9FF
skinparam sequence {
    ArrowColor red
    LifeLineBorderColor black
    
    ActorBorderColor blue
    ActorBackgroundColor Beige
    ActorFontColor black
    ActorFontSize 16
}

autonumber

actor User as "Пользователь"
participant Agent as "Клиентское приложение \n (браузер)"
collections Proxy as "Прокси сервер \n (IAM Proxy)"
participant IDP as "Провайдер аутентификации \n (IAM KCSE и др.)"
collections Backend as "Бэкенд ресурса \n (приложение)"

User -> Agent: Обращение по корню
Agent -> Proxy: Обращение по корню (GET /)
Agent <-- Proxy: Перенаправление на стартовую страницу (/unauth/start-page)
Agent -> Proxy: Обращение по корню на стартовую страницу (GET /unauth/start-page)
Proxy -> Backend: Обращение по корню на стартовую страницу
Proxy <-- Backend: Ответ
Agent <-- Proxy: Ответ
User <-- Agent: Отображение ссылок на UI пользователя и UI Администратора

== Пройти аутентификацию ==
User -> Agent: Нажатие на ссылку UI Администратора
Agent->Proxy: Обращение к UI Администратора
activate Proxy
Proxy -> Proxy: URL относится к location, на котором стоит опция \n использовать альтернативный client-id
alt Если нет корректной сессии IAM Proxy
  Proxy --> Agent: Переадресация на провайдер аутентификации \n для аутентификации с указанием redirect_url
  deactivate Proxy
  Agent -> IDP: На страницу аутентификации провайдера аутентификации
  activate IDP
  IDP --> Agent: Форма входа
  Agent -> User: Отображение \n формы входа
  User --> Agent: Ввод данных \n для аутентификации
  Agent -> IDP: Передача данных \n для аутентификации
  IDP --> Agent: Переадресация с кодом аутентификации
  deactivate IDP
  Agent -> Proxy: Код аутентификации
  activate Proxy
  Proxy -> IDP: Запрос Access, Refresh и Id token OpenID Provider \n в обмен на код аутентификации
  activate IDP
  IDP --> Proxy: Access, Refresh token, ID token
  Proxy --> Proxy: проверка ID token (в том числе что client_id есть в атрибутах azp+aud ID token)
  Proxy --> Agent: Переадресация и обновление сессии IAM Proxy
else Если есть сессия, но созданная не с альтернативным client-id (client_id нет в атрибутах azp+aud ID token)
  Agent <-- Proxy: Ответ с кодом 406
  User <-- Agent: Отображение ошибки
end
== Получить ресурс требующий аутентификации ==
Agent -> Proxy: Обращение к UI Администратора
Proxy -> Backend: Обращение к UI Администратора + jwt-токен
Proxy <-- Backend: Ответ
Agent <-- Proxy: Ответ
deactivate Proxy
User <-- Agent: Отображение результата

@enduml
```

Про описание и настройку этой функциональности смотрите в
разделе [Описание настройки IAM Proxy при запуске в контейнере](../installation-guide/proxy-deploy-docker-description.md).

### Диаграмма последовательности процесса аутентификации gRPC-Native клиента (используя direct-auth)

Диаграмма описывает взаимодействия при аутентификации gRPC-Native клиента через IAM Proxy используя direct-auth,
возникающие в сценариях:

- [Пройти аутентификацию](../about/use-case-authentification.md)
- [Получить ресурс требующий аутентификации](../about/use-case-accessing-resource.md)

```puml
@startuml

title Аутентификация gRPC-Native клиента через IAM Proxy используя direct-auth

skinparam backgroundColor #F9F9FF
skinparam sequence {
    ArrowColor red
    LifeLineBorderColor black
    
    ActorBorderColor blue
    ActorBackgroundColor Beige
    ActorFontColor black
    ActorFontSize 16
}

autonumber

participant User as "Пользователь"
participant Client as "Клиентское приложение \n (gRPC-клиент)"
collections Proxy as "Прокси сервер \n (IAM Proxy)"
participant IDP as "Провайдер аутентификации \n (IAM KCSE)"
collections Backend as "Бэкенд ресурса \n (приложение АС)"

== Пройти аутентификацию ==
User -> Client: Обращение к ресурсу
activate Client
Client -> Proxy: Обращение к gRPC ресурсу,\n требующему аутентификации в Провайдере аутентификации \n (gRPC: HTTP/2 POST application/grpc) \n + (опционально) заголовок Authorization
activate Proxy

alt Если в запросе есть заголовок Authorization (режим direct-auth)
  alt Получение токенов из локального кеша
    Proxy -> Proxy: Поиск токенов в локальном кеше \n по hash от Authorization
  else Если не найдено токенов в кеше, то получение токенов от IDP
    Proxy -> IDP: Запрос на получение токенов + заголовок Authorization
    activate IDP
    IDP -> IDP: Аутентификация пользователя по данным \n из заголовка Authorization \n (например по Basic Authentication)
    Proxy <-- IDP: Передача токенов: access, refresh, ID
    deactivate IDP
    Proxy -> Proxy: Сохранение токенов в локальном кеше \n на срок жизни access-токена
  else Если не получили токены
    Client <-- Proxy: Ошибка аутентификации \n (HTTP/2 401, grpc-status: 16/UNAUTHENTICATED)
    Client --> User: Отображение ошибки
  end
else Если нет заголовка Authorization, то используем стандартную аутентификацию (OIDC code-flow) для gRPC-клиента
  alt Получение токенов из сессии в Cookie (Cookie: PLATFORM_SESSION*)
    Proxy -> Proxy: Расшифровка Cookie, \n проверка сроков действия сессии и токенов   
  else Если нет действующей сессии или токенов в Cookie 
    Proxy --> Client: gRPC: HTTP/2 302 (grpc-status: 16/UNAUTHENTICATED) \n + Location: redirect_url IdP
    deactivate Proxy
    Client -> IDP: Открытие URL провайдера аутентификации \n (через браузер или WebView)
    activate IDP
    IDP --> Client: Форма входа (страница IdP)
    Client -> User: Отображение формы входа
    User --> Client: Ввод данных для аутентификации
    Client -> IDP: Передача данных для аутентификации: POST
    IDP --> Client: HTTP 302. Переадресация с кодом авторизации
    deactivate IDP
    Client -> Proxy: Передача кода авторизации провайдера \n (auth redirect/callback)
    activate Proxy
    Proxy -> IDP: Запрос Access, Refresh и Id token OpenID Provider\n в обмен на код авторизации
    activate IDP
    IDP --> Proxy: Access, Refresh token, ID token
    deactivate IDP
    Proxy --> Client: Установка сессии iam proxy (Set-Cookie: PLATFORM_SESSION*)
    Client -> Proxy: Обращение к ресурсу \n (gRPC: HTTP/2 POST application/grpc) \n + Authorization (для direct-auth) \n или + cookie PLATFORM_SESSION* (для OIDC)
  end
end
== Получить ресурс требующий аутентификации ==
Proxy -> Backend: Обращение к ресурсу + jwt-токен
activate Backend
Backend -> Backend: Проверка jwt-токена, \n формирование ответа \n авторизованного по jwt-токену
Backend --> Proxy: Response
deactivate Backend
Proxy --> Client: Response (gRPC: HTTP/2 200, grpc-status: 0/SUCCESS)
deactivate Proxy
Client --> User: Отображение ресурса
deactivate Client
@enduml
```

Про описание и настройку этой функциональности смотрите в
разделе [Описание настройки IAM Proxy при запуске в контейнере](../installation-guide/proxy-deploy-docker-description.md).