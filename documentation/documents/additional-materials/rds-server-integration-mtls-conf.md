# Настройка mTLS

Для включения работы только с двух-сторонней аутентификацией по сертификатам (mTLS), необходимо задать параметры
приложения:

```yaml
server:
  port: 8443
  ssl:
    enabled: true
    client-auth: need
    key-store: C:\work\keystore.jks
    key-password: changeit
    key-store-password: changeit
    trust-store: C:\work\truststore.jks
    trust-store-password: changeit
```

Для поддержки обоих протоколов http и https необходима настройка вида:

```yaml
server:
  port: 8443
  http:
    port: 8080
    enable: true
  ssl:
    enabled: true
    client-auth: need
    key-store: C:\work\keystore.jks
    key-password: changeit
    key-store-password: changeit
    trust-store: C:\work\truststore.jks
    trust-store-password: changeit
```

> По умолчанию в профиле PROM используется https-8443