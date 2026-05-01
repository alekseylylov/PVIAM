# Настройка интеграции с Объединенным мониторингом Unimon (MONA)

Для мониторинга состояния RDS Server в нем реализована отдача метрик в формате Prometheus по пути в
URL `/actuator/prometheus/`(пример URL: `http://localhost:9990/actuator/prometheus/`). Объединенный мониторинг Unimon (
MONA) необходимо настроить на периодическое получение метрик из RDS Server, и добавить на UI панель мониторинга
интересующие показатели.

Пример вывода метрик:

```
RDS_count_request_service_total{service="audit",status="failed",} 0.0
RDS_count_request_service_total{service="audit",status="success",} 1.0
RDS_count_request_service_total{service="configurator",status="failed",} 0.0
RDS_count_request_service_total{service="configurator",status="success",} 3.0
RDS_count_request_service_total{service="standIn",status="failed",} 0.0
RDS_count_request_service_total{service="standIn",status="success",} 2.0
RDS_count_request_total{request_type="active-zone",} 0.0
RDS_count_request_total{request_type="all",} 0.0
RDS_timed_3th_config_read_seconds_count{class="com.sbt.authentication.rds.cloud.component.config.third.PlatformClientAuditService",exception="none",method="init",} 2.0
RDS_timed_3th_config_read_seconds_max{class="com.sbt.authentication.rds.cloud.component.config.third.PlatformClientAuditService",exception="none",method="init",} 0.0036398
RDS_timed_3th_config_read_seconds_sum{class="com.sbt.authentication.rds.cloud.component.config.third.PlatformClientAuditService",exception="none",method="init",} 0.0061829
RDS_timed_standin_read_seconds_count{class="com.sbt.authentication.rds.cloud.service.StandInStateService",exception="none",method="updateStandInState",} 8.0
RDS_timed_standin_read_seconds_max{class="com.sbt.authentication.rds.cloud.service.StandInStateService",exception="none",method="updateStandInState",} 0.0032067
RDS_timed_standin_read_seconds_sum{class="com.sbt.authentication.rds.cloud.service.StandInStateService",exception="none",method="updateStandInState",} 0.0057997
```