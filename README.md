# Практическое занятие №4
## Настройка Prometheus + Grafana для метрик. Интеграция с приложением

**ФИО:** Бакланова Е.С.
**Группа:** ЭФМО-01-25

## Цели работы

- Научиться собирать и визуализировать метрики сервиса: трафик, ошибки, задержки, активные запросы.

### Содержание проекта

**Auth service** (порт 8081 +  gRPC порт 50051)
- Аутентификация пользователей
- Выдача токенов
- Проверка токенов через gRPC

**Tasks service** (порт 8082)
- CRUD для задач (TODO-список)
- Перед каждой операцией проверяет токен, отправляя gRPC-запрос в Auth service

### Описание метрик и labels

Метрики

 | Метрика | Тип | Описание |
 |-------|----------|----------|
 | http_requests_total | Counter | Общее количество HTTP-запросов к сервису Tasks | 
 | http_request_duration_seconds | Histogram | Длительность выполнения HTTP-запросов (в секундах) | 
 | http_request_duration_seconds | Gauge | Текущее количество одновременно обрабатываемых запросов | 

Labels

 | Label | Значение | Пример | Метрики | 
 |-------|----------|----------|----------|----------|
 | method | HTTP-метод запроса | GET, POST, PATCH, DELETE | http_requests_total, http_request_duration_seconds |
 | route | Нормализованный путь |/v1/tasks, /v1/tasks/{id}, /metrics | http_requests_total, http_request_duration_seconds |
 | status | HTTP-статус отве | 200, 201, 401, 404, 503 | http_requests_total |


### Структура

<img width="349" height="852" alt="image" src="https://github.com/user-attachments/assets/f3cbd9e7-b9b0-4576-843c-3c97d6e964b1" />


### Инструкция по запуску

1.  Запуск Auth

  - cd services/auth
  - $env:AUTH_PORT="8081"
  - go run ./cmd/auth
  
2. Запуск Tasks

  - cd services/tasks
  - $env:TASKS_PORT="8082"
  - $env:AUTH_GRPC_ADDR="localhost:50051"
  - go run ./cmd/tasks

3. Запуск мониторинга
  - cd deploy/monitoring
  - docker-compose up -d


### Описание docker-compose и prometheus.yml

*docker-compose.yml*

```bash
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest          # официальный образ Prometheus
    container_name: prometheus             # имя контейнера
    ports:
      - "9090:9090"                        # проброс порта для доступа к веб-интерфейсу
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml  # монтируем конфиг
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'   # указываем путь к конфигу

  grafana:
    image: grafana/grafana:latest          # официальный образ Grafana
    container_name: grafana                # имя контейнера
    ports:
      - "3000:3000"                        # проброс порта для доступа к веб-интерфейсу
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin   # пароль админа
```

*prometheus.yml*

```bash
global:
  scrape_interval: 5s      # интервал сбора метрик каждые 5 секунд
  evaluation_interval: 5s  # интервал вычисления правил

scrape_configs:
  - job_name: 'tasks'                      # имя задания 
    static_configs:
      - targets: ['host.docker.internal:8082']   # адрес Tasks сервиса
```

### Тестирование

1. Получить токен для дальнейшей работе

```bash
curl -i -X POST http://localhost:8081/v1/auth/login \
  -H "Content-Type: application/json" \
  -H "X-Request-ID: test_kottia" \
  -d "{\"username\":\"kate\",\"password\":\"secret\"}"
```

2. Создаем нескольо раз таски

```bash
  curl -i -X POST http://localhost:8082/v1/tasks \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer demo-token-test_kottia:kate" \
  -d '{"title":"","description":"","due_date":"2026-03-21"}'
```

<img width="900" height="529" alt="image" src="https://github.com/user-attachments/assets/a5ff7142-ab04-4cb2-b9de-01707ad274e5" />

3. Получить несколько раз список задач

```bash
  curl -X GET http://localhost:8082/v1/tasks \
  -H "Authorization: Bearer demo-token-test_kottia:kate"
```

<img width="900" height="739" alt="image" src="https://github.com/user-attachments/assets/46f30c34-1e3b-45f5-bc4a-101faa29bbd0" />

4. Несколько раз запрашиваем с ошибкой

<img width="900" height="515" alt="image" src="https://github.com/user-attachments/assets/086bc46e-60d5-4e95-bda3-0e041a068c89" />

5. Запрашиваем по id

```bash
  curl -X GET http://localhost:8082/v1/tasks/{id} \
  -H "Authorization: Bearer demo-token-test_kottia:kate"
```

<img width="900" height="538" alt="image" src="https://github.com/user-attachments/assets/b33aaf93-2b96-4954-815c-22748c4bc79b" />

6. Получаем метрики

```bash
  curl -X GET http://localhost:8082/v1/metrics \
  -H "Authorization: Bearer demo-token-test_kottia:kate"
```

<img width="900" height="620" alt="image" src="https://github.com/user-attachments/assets/8a5b111f-0271-4f63-8d49-763283b04b66" />

<img width="900" height="111" alt="image" src="https://github.com/user-attachments/assets/8e8d3c97-fb81-42e2-a998-af4838fa306b" />


7. Проверка prometheus

http://localhost:9090

<img width="900" height="175" alt="image" src="https://github.com/user-attachments/assets/1814f436-3290-48b9-97a7-72b36b6432fc" />

Проверием запросы: 

7.1. RPS (запросы в секунду)

rate(http_requests_total[1m])

<img width="900" height="231" alt="image" src="https://github.com/user-attachments/assets/d3b9b1f2-5ded-48c7-ab56-5ce4eb92f207" />

7.2. Ошибки 4xx

rate(http_requests_total{status=~"4.."}[1m])

<img width="900" height="192" alt="image" src="https://github.com/user-attachments/assets/6f101b92-513d-4c27-b531-7400f250954c" />

7.3. Ошибки 5xx

rate(http_requests_total{status=~"5.."}[1m])

<img width="900" height="196" alt="image" src="https://github.com/user-attachments/assets/1ba39a33-e1cf-4b71-84d3-39de25acdd38" />

Ошибок не было

7.4. Latency p95

histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[1m])) by (le, method, route))

<img width="900" height="210" alt="image" src="https://github.com/user-attachments/assets/f8278523-6020-440d-a283-d0cea9fb6cea" />

7.5. Текущее количество активных запросов

http_in_flight_requests

<img width="900" height="196" alt="image" src="https://github.com/user-attachments/assets/16d15795-784f-49f3-be22-d87d756d4184" />

7.6. посмотреть все метрики

{__name__=~".+"}
 
<img width="900" height="390" alt="image" src="https://github.com/user-attachments/assets/1f3812a1-63fb-4771-a233-82af27c39d05" />

8. Настраиваем дашборд в графане

 http://localhost:3000

<img width="900" height="429" alt="image" src="https://github.com/user-attachments/assets/3503fc0f-65a8-4441-a411-3df44c49677e" />

<img width="900" height="428" alt="image" src="https://github.com/user-attachments/assets/f6162449-2a08-4745-83cc-d12793ecdee5" />




### Контрольные вопросы

1.	Чем метрики отличаются от логов и зачем нужны оба подхода?

Метрики — агрегированные числа (графики, алерты). Логи — детальные события (дебаг)

2.	Чем Counter отличается от Gauge?

Counter — только увеличивается (запросы, ошибки). Gauge — может расти и падать (активные соединения, память)

5.	Почему latency нужно измерять histogram, а не просто средним значением?

Среднее скрывает выбросы. Гистограмма позволяет считать перцентили (p95, p99), которые показывают реальную задержку для "длинного хвоста"

7.	Что такое labels и почему опасна высокая кардинальность?

Labels — теги для разделения метрик (method, status). Высокая кардинальность (например, user_id) создаёт миллионы временных рядов, что влияет на производительность

9.	Зачем нужны p95/p99 и почему среднее может “врать”?

Если 99% запросов быстрые, а 1% — очень медленные, среднее будет плохим, хотя большинство пользователей довольны. P95 показывает задержку для 95% пользователей
