# Варианты и сценарии использования

## Варианты использования

```puml
@startuml

left to right direction
skinparam backgroundColor #F9F9FF
skinparam ActorBackgroundColor Beige
actor Пользователь
actor "Не аутентифицированный пользователь" as AnonUser
actor "Аутентифицированный пользователь" as AuthUser
actor Администратор
actor [Провайдер аутентификации] as IDP
actor [Ресурсная система \n (бэкенд)] as Бэкенд

Администратор -|> AuthUser
AnonUser -|> Пользователь
AuthUser -|> Пользователь

rectangle "IAM Proxy" {
   usecase "Получить публичный ресурс" as doAnonReq
   usecase "Получить ресурс \n требующий аутентификации" as doAuthReq

   AnonUser --> (Пройти аутентификацию)
   (Пройти аутентификацию) ---> IDP

   Пользователь --> doAnonReq
   AuthUser --> doAuthReq
   doAnonReq --|> (Получить ресурс)
   doAuthReq --|> (Получить ресурс)
   (Получить ресурс) --> Бэкенд
 
   AuthUser --> (Выйти из системы)
   (Выйти из системы) --> IDP
}

rectangle "Система с UI управления ответвлениями" {
   (Управлять ответвлениями) -|> doAuthReq
   Администратор --> (Управлять ответвлениями)
   (Управлять ответвлениями) <|-- (Создать ответвление)
   (Управлять ответвлениями) <|-- (Изменить ответвление)
   (Управлять ответвлениями) <|-- (Удалить ответвление)
}

@enduml
```

## Сценарии использования

IAM Proxy реализует следующие сценарии:

- [Пройти аутентификацию](use-case-authentification.md)
- [Получить ресурс требующий аутентификации](use-case-accessing-resource.md)
- [Получить публичный ресурс](use-case-accessing-public-resource.md)
- [Управление ответвлениями](use-case-junctions-management.md)
- [Выход из системы](use-case-user-logout.md)

