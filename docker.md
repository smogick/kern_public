# Запуск kern в Docker

## Получение образа

```bash
docker pull ghcr.io/smogick/kern:latest

# Конкретная версия
docker pull ghcr.io/smogick/kern:v0.6.0
```

---

## Базовый запуск

Тест-план передаётся через volume. Конфиг читается внутри контейнера по пути `/plan.json`:

```bash
docker run --rm \
  -v $(pwd)/plan.json:/plan.json \
  ghcr.io/smogick/kern:latest --config /plan.json
```

---

## Сохранение отчёта

### На локальный диск

```bash
docker run --rm \
  -v $(pwd)/plan.json:/plan.json \
  -v $(pwd)/results:/results \
  ghcr.io/smogick/kern:latest --config /plan.json --output /results/
```

Отчёт появится в `./results/report-<timestamp>.json`.

### В S3

AWS-креденшлы передаются через переменные окружения:

```bash
docker run --rm \
  -v $(pwd)/plan.json:/plan.json \
  -e AWS_ACCESS_KEY_ID=... \
  -e AWS_SECRET_ACCESS_KEY=... \
  -e AWS_REGION=us-east-1 \
  ghcr.io/smogick/kern:latest --config /plan.json --output s3://my-bucket/kern/runs/
```

---

## Prometheus

По умолчанию `/metrics` доступен на порту `9090`. Пробросьте порт, если нужен доступ снаружи:

```bash
docker run --rm \
  -v $(pwd)/plan.json:/plan.json \
  -p 9090:9090 \
  ghcr.io/smogick/kern:latest --config /plan.json
```

Или смените порт:

```bash
docker run --rm \
  -v $(pwd)/plan.json:/plan.json \
  -p 8080:8080 \
  ghcr.io/smogick/kern:latest --config /plan.json --metrics-addr :8080
```

Если метрики не нужны:

```bash
ghcr.io/smogick/kern:latest --config /plan.json --prometheus=false
```

---

## docker-compose

Удобно для локального запуска с Prometheus + Grafana:

```yaml
services:
  kern:
    image: ghcr.io/smogick/kern:latest
    volumes:
      - ./plan.json:/plan.json
      - ./results:/results
    command: >
      --config /plan.json
      --output /results/
      --timeline
      --timeline-csv /results/timeline.csv
    ports:
      - "9090:9090"

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9091:9090"
```

`prometheus.yml`:

```yaml
scrape_configs:
  - job_name: kern
    static_configs:
      - targets: ["kern:9090"]
```

Запуск:

```bash
docker compose up
```

---

## Kubernetes (масштабирование по VU)

При запуске нескольких подов используйте `--vus`, чтобы разделить нагрузку.
Например, план на 500 VU + 4 пода → каждый держит 125 VU:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kern
spec:
  replicas: 4
  template:
    spec:
      containers:
        - name: kern
          image: ghcr.io/smogick/kern:latest
          args:
            - --config=/plan.json
            - --vus=125
            - --output=s3://my-bucket/kern/runs/
          volumeMounts:
            - name: plan
              mountPath: /plan.json
              subPath: plan.json
          env:
            - name: AWS_REGION
              value: us-east-1
      volumes:
        - name: plan
          configMap:
            name: kern-plan
```

`--vus` пропорционально масштабирует целевое количество VU из плана.
Итоговые отчёты каждого пода пишутся в S3 с уникальным именем файла.

---

## Полезные флаги при запуске в контейнере

| Флаг | Для чего |
|------|----------|
| `--cpu-monitor` | Предупреждать, если kern сам становится бутылочным горлышком (Linux, читает `/proc/stat`) |
| `--log-vus` | Видеть процесс набора VU в логах контейнера |
| `--request-timeout 0` | Отключить таймаут — поведение как у Locust |
| `--report=false --prometheus=false` | Минимальный оверхед, чистый генератор нагрузки |
