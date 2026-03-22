# Практическое занятие №4
## Настройка Prometheus + Grafana для метрик. Интеграция с приложением

**ФИО:** Бакланова Е.С.
**Группа:** ЭФМО-01-25

## Цели работы

- Научиться собирать и визуализировать метрики сервиса: трафик, ошибки, задержки, активные запросы.

## Теория



### Основная идея и минимальный набор метрик

Файл .proto выступает в роли контракта между клиентом и сервером. Он описывает:

1. Сервисы (Services) - какие RPC-методы доступны.
2. Сообщения (Messages) - структуры запросов и ответов.
Любое изменение в API должно начинаться с изменения этого файла.

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



3. Получить несколько раз список задач

```bash
  curl -X GET http://localhost:8082/v1/tasks \
  -H "Authorization: Bearer demo-token-test_kottia:kate"
```



4. Несколько раз запрашиваем с ошибкой




5. Запрашиваем по id

```bash
  curl -X GET http://localhost:8082/v1/tasks/{id} \
  -H "Authorization: Bearer demo-token-test_kottia:kate"
```

<img width="900" height="620" alt="image" src="https://github.com/user-attachments/assets/98edb3b8-dff7-4ff7-8a9c-6152d74844ae" />


6. Получаем метрики


```bash
  curl -X GET http://localhost:8082/v1/metrics \
  -H "Authorization: Bearer demo-token-test_kottia:kate"
```

7. Проверка prometheus

http://localhost:9090



### Контрольные вопросы

1.	Что такое .proto и почему он считается контрактом?

файл, в котором описываются сервисы и структуры данных для gRPC. Он считается контрактом, потому что и клиент, и сервер строго следуют этому описанию

2. Что такое deadline в gRPC и чем он полезен?

максимальное время ожидания ответа от сервера. Полезен тем, что, если сервер упал или тормозит — по истечении времени приходит ошибка

3. Почему "exactly-once" не даётся просто так даже в RPC?

Из-за ненадёжности сети подтверждение о выполнении может потеряться, и клиент повторит запрос

4. Как обеспечивать совместимость при расширении .proto?

- Не менять числовые теги у существующих полей
- Добавлять новые поля с новыми тегами
- Не удалять старые поля без reserved
- Старые клиенты не видят новые поля 
