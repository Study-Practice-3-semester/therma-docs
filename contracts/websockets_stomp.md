# Контракты WebSockets (STOMP)

Архитектура реактивного обмена данными построена на протоколе STOMP поверх WebSockets.
Бэкенд выступает в роли Message Broker.

## 1. Подключение
* **Endpoint URL:** `ws://localhost:8080/ws-endpoint`
* **Библиотека для Frontend:** `@stomp/stompjs`

---

## 2. Топик: Поток телеметрии (Real-time данные)
Этот топик используется для отрисовки Live-графиков. Симулятор пушит сюда новую точку каждую секунду для каждого работающего каупера.

* **Канал подписки (Subscribe):** `/topic/telemetry`
* **Формат Payload (JSON):**
  Точно соответствует схеме `TelemetryPoint` из OpenAPI.
  ```json
  {
    "cowperId": "STOVE_1",
    "recordedAt": "2026-11-11T10:05:00Z",
    "phase": "HEATING",
    "domeTemp": 1335.5,
    "hotBlastTemp": 0,
    "flueGasTemp": 250.0,
    "bfGasFlow": 45000.0,
    "natGasFlow": 1500.0,
    "combustionAirFlow": 55000.0
  }
  ```
## 3. Топик: Инциденты и Рекомендации (Alerts)
Этот топик используется для мгновенного показа всплывающих окон (Toast/Notifications) оператору, когда экспертная система выявляет отклонение.

- Канал подписки (Subscribe): /topic/recommendations

- Формат Payload (JSON):
    Соответствует базовой структуре Recommendation, но отправляется только при генерации (статус всегда PENDING).
```json
{
  "id": 1042,
  "cowperId": "STOVE_1",
  "createdAt": "2026-11-11T10:05:02Z",
  "ruleCode": "ERR_DOME_OVERHEAT",
  "message": "КРИТИЧЕСКАЯ ТЕМПЕРАТУРА КУПОЛА (1355 °C). Немедленно снизьте расход газа!",
  "priority": "CRITICAL",
  "status": "PENDING",
  "expectedEffect": {
    "tempDelta": -15.5,
    "gasDelta": -500.0
  },
  "reasoning": [
    "domeTemp > 1350.0 (Текущее: 1355.0)"
  ]
}
```
## 4. Топик: Статус системы (System Status)
Этот топик используется для мгновенного оповещения фронтенда о переходе экспертного ядра в безопасный режим (например, при обрыве связи с симулятором или пропусках в данных).
* **Канал подписки (Subscribe):** `/topic/system-status`
* **Формат Payload (JSON):**
  ```json
  {
    "status": "INSUFFICIENT_DATA",
    "timestamp": "2026-11-11T10:06:00Z",
    "message": "Отсутствует телеметрия более 10 секунд. Выдача рекомендаций приостановлена."
  }
  ```
## 5. Жизненный цикл для Frontend
1. При монтировании дашборда:
    - Делаем REST-запрос `GET /api/v1/telemetry` (например, за последние 30 минут), чтобы заполнить график Recharts.
    - Подключаемся к WebSocket `/ws-endpoint`.
    - Подписываемся на `/topic/telemetry` и добавляем каждую новую точку в state массива графика.
2. Действия оператора (Ответ на алерт):
    - Клиент НЕ отправляет данные через WebSockets.
    - Нажатие кнопки "Применить" вызывает синхронный REST POST запрос на `/api/v1/recommendations/{id}/apply`.