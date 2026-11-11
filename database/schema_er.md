# ER-диаграмма базы данных (Entity-Relationship)

В данном документе представлена логическая структура реляционной базы данных (PostgreSQL) для проекта «Интеллектуальная система рекомендаций по управлению воздухонагревателями». 

Схема разработана с учетом разделения на time-series данные (телеметрия), доменные сущности (кауперы) и бизнес-логику (отклонения, рекомендации и действия оператора).

```mermaid
erDiagram
    COWPER {
        bigint id PK
        varchar name
        varchar status "CHECK: HEATING, BLOWING, ISOLATED, RESERVE, MAINTENANCE"
        text description
    }

    TELEMETRY_LOG {
        bigint id PK
        bigint cowper_id FK
        timestamp_tz recorded_at
        varchar phase "CHECK: HEATING, BLOWING, CHANGING, ISOLATED"
        numeric hot_blast_temp "Температура дутья"
        numeric dome_temp "Температура купола"
        numeric flue_gas_temp "Температура дыма"
        numeric bf_gas_flow "Доменный газ"
        numeric nat_gas_flow "Природный газ"
        numeric combustion_air_flow "Воздух горения"
        boolean is_valid
    }

    DEVIATION_EVENT {
        bigint id PK
        bigint cowper_id FK
        timestamp_tz detected_at
        varchar parameter_name
        numeric actual_value
        numeric target_value_min
        numeric target_value_max
    }

    RECOMMENDATION {
        bigint id PK
        bigint deviation_id FK
        timestamp_tz created_at
        varchar rule_code "Напр: ERR_DOME_OVERHEAT"
        text message
        varchar priority "CHECK: INFO, WARNING, CRITICAL"
        varchar suggested_action
        numeric suggested_value_min
        numeric suggested_value_max
        numeric expected_temp_effect "Дельта температуры"
        numeric expected_gas_effect "Дельта газа"
        jsonb reasoning "Массив триггеров/факторов"
        varchar status "CHECK: PENDING, APPLIED, REJECTED, EXPIRED"
    }

    OPERATOR_ACTION {
        bigint id PK
        bigint recommendation_id FK
        bigint cowper_id FK
        timestamp_tz action_time
        varchar action_type
        numeric applied_value
    }

    SIMULATION_SCENARIO {
        bigint id PK
        varchar name
        jsonb config_json "Параметры шума и трендов"
    }

    COWPER ||--o{ TELEMETRY_LOG : "генерирует"
    COWPER ||--o{ DEVIATION_EVENT : "имеет"
    COWPER ||--o{ OPERATOR_ACTION : "получает"
    DEVIATION_EVENT ||--o| RECOMMENDATION : "является причиной"
    RECOMMENDATION ||--o| OPERATOR_ACTION : "закрывается через"
```
