# Первый запуск

Предположим, у нас есть todo-сервис на `http://localhost:8000`.

```
POST /auth/login          → возвращает JWT в заголовке Authorization
GET  /todos               → список задач
POST /todos               → создать задачу, возвращает {"id": 42, ...}
DELETE /todos/:id         → удалить задачу
```

---

## Установка

Скачайте бинарь со [страницы Releases](https://github.com/smogick/kern_public/releases/latest) и сделайте его исполняемым:

```bash
chmod +x kern-linux-amd64
mv kern-linux-amd64 /usr/local/bin/kern
```

---

## Сценарий 1: полный путь пользователя

Каждый виртуальный пользователь на каждой итерации:
1. Авторизуется и получает JWT-токен
2. Читает список задач
3. Создаёт задачу со случайным названием
4. Удаляет только что созданную задачу

Создайте файл `todo-journey.json`:

```json
{
  "stages": [
    {"duration": "30s", "target_vus": 10, "spawn_rate": 5},
    {"duration": "2m",  "target_vus": 10},
    {"duration": "30s", "target_vus": 0,  "spawn_rate": 5}
  ],
  "steps": [
    {
      "name": "login",
      "request": {
        "method": "POST",
        "url": "http://localhost:8000/auth/login",
        "headers": {"Content-Type": "application/json"},
        "body": {"username": "loadtest@example.com", "password": "secret"}
      },
      "extract": [
        {"var": "token", "from": "header", "key": "Authorization"}
      ]
    },
    {
      "name": "list_todos",
      "request": {
        "method": "GET",
        "url": "http://localhost:8000/todos",
        "headers": {"Authorization": "Bearer {{token}}"}
      }
    },
    {
      "name": "create_todo",
      "request": {
        "method": "POST",
        "url": "http://localhost:8000/todos",
        "headers": {
          "Authorization": "Bearer {{token}}",
          "Content-Type": "application/json"
        },
        "body": {
          "title": "Task {{rnd_string(8)}}",
          "priority": "{{rnd_int(1,5)}}"
        }
      },
      "extract": [
        {"var": "todo_id", "from": "body", "path": "$.id"}
      ]
    },
    {
      "name": "delete_todo",
      "request": {
        "method": "DELETE",
        "url": "http://localhost:8000/todos/{{todo_id}}",
        "headers": {"Authorization": "Bearer {{token}}"}
      }
    }
  ]
}
```

Запуск:

```bash
kern --config todo-journey.json --output ./results/
```

Что произойдёт:
- 30 секунд: рост до 10 VU со скоростью 5 VU/с (достигнет за 2 с, остаток держит 10 VU)
- 2 минуты: 10 VU непрерывно прогоняют 4 шага
- 30 секунд: снижение до 0 VU со скоростью 5 VU/с
- В папке `./results/` появится файл `kern-YYYYMMDD-HHMMSS.json` с отчётом

Метрики в реальном времени (пока тест идёт):

```bash
curl http://localhost:9090/metrics
```

---

## Сценарий 2: нагрузка на один эндпоинт с токеном авторизации

Если не нужен сценарий входа — токен уже известен заранее — достаточно одного шага.

Создайте файл `todo-read.json`:

```json
{
  "stages": [
    {"duration": "30s", "target_vus": 50, "spawn_rate": 10},
    {"duration": "5m",  "target_vus": 50},
    {"duration": "30s", "target_vus": 0,  "spawn_rate": 10}
  ],
  "steps": [
    {
      "name": "list_todos",
      "request": {
        "method": "GET",
        "url": "http://localhost:8000/todos",
        "headers": {
          "Authorization": "Bearer eyJhbGciOiJIUzI1NiJ9.your.token"
        }
      }
    }
  ]
}
```

Запуск:

```bash
kern --config todo-read.json
```

Если нужно быстро проверить максимальную пропускную способность с другим числом VU, не меняя файл плана:

```bash
kern --config todo-read.json --vus 100
```

Флаг `--vus 100` масштабирует все `target_vus` в плане пропорционально:
план рассчитан на 50 VU → при `--vus 100` все значения удваиваются.

---

## Куда смотреть результаты

### В реальном времени — Prometheus

Пока тест идёт, метрики доступны на `http://localhost:9090/metrics`.
Можно подключить Grafana или смотреть напрямую:

```bash
# RPS по шагам
curl -s http://localhost:9090/metrics | grep kern_http_requests_total

# Активные VU
curl -s http://localhost:9090/metrics | grep kern_vus_active
```

### По завершении — JSON-отчёт

```bash
kern --config todo-journey.json --output ./results/
cat ./results/kern-*.json
```

```json
{
  "duration_seconds": 180,
  "total_requests": 54321,
  "total_errors": 2,
  "steps": {
    "login":       {"rps": 150.3, "latency": {"p99_ms": 45.1}},
    "list_todos":  {"rps": 150.3, "latency": {"p99_ms": 12.4}},
    "create_todo": {"rps": 150.3, "latency": {"p99_ms": 38.7}},
    "delete_todo": {"rps": 150.3, "latency": {"p99_ms": 22.0}}
  }
}
```

Отчёт также можно отправить сразу в S3:

```bash
kern --config todo-journey.json --output s3://my-bucket/kern/results/
```
