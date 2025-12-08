# Observability - Руководство для Senior Backend интервью

> 💡 **Как думать об observability на интервью:**
> "Observability — это способность понять внутреннее состояние системы по внешним сигналам. Три столпа: logs, metrics, traces. Без этого production — чёрный ящик."

---

## 1. Три столпа Observability

### 🎯 Что спрашивают
> "Что такое observability и зачем нужна?"

### Logs, Metrics, Traces

```
┌─────────────────────────────────────────────────────┐
│                   OBSERVABILITY                      │
├─────────────────┬─────────────────┬─────────────────┤
│      LOGS       │     METRICS     │     TRACES      │
│                 │                 │                 │
│ What happened   │ System health   │ Request flow    │
│                 │                 │                 │
│ • Events        │ • Counters      │ • Spans         │
│ • Errors        │ • Gauges        │ • Context       │
│ • Debug info    │ • Histograms    │ • Latency       │
│                 │                 │                 │
│ ELK, Loki       │ Prometheus      │ Jaeger, Zipkin  │
└─────────────────┴─────────────────┴─────────────────┘
```

### Когда что использовать
| Вопрос | Инструмент |
|--------|------------|
| "Что сломалось?" | Logs |
| "Насколько загружена система?" | Metrics |
| "Почему этот запрос медленный?" | Traces |

### 📝 Фраза для интервью
> "Logs для debugging конкретных событий. Metrics для понимания здоровья системы и алертинга. Traces для анализа latency и dependencies между сервисами. Нужны все три для полной картины."

---

## 2. Structured Logging

### 🎯 Что спрашивают
> "Как правильно логировать в production?"

### ❌ Плохой лог
```
[2024-01-15] ERROR: Something went wrong
[2024-01-15] INFO: User logged in
```

### ✅ Структурированный лог
```json
{
    "timestamp": "2024-01-15T10:30:00Z",
    "level": "error",
    "service": "payment-service",
    "trace_id": "abc123",
    "user_id": 456,
    "event": "payment_failed",
    "error": "insufficient_funds",
    "amount": 100.00,
    "currency": "USD"
}
```

### Python structlog
```python
import structlog

structlog.configure(
    processors=[
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.add_log_level,
        structlog.processors.JSONRenderer()
    ]
)

logger = structlog.get_logger()

logger.info("payment_processed", 
    user_id=123, 
    amount=100.00,
    currency="USD"
)
```

### Correlation ID для трейсинга
```python
from contextvars import ContextVar

request_id: ContextVar[str] = ContextVar('request_id')

@app.middleware("http")
async def add_request_id(request, call_next):
    req_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))
    request_id.set(req_id)
    response = await call_next(request)
    response.headers["X-Request-ID"] = req_id
    return response

# Все логи автоматически включают request_id
logger.info("processing", request_id=request_id.get())
```

### 📝 Фраза для интервью
> "Structured logging в JSON для парсинга и поиска. Correlation ID для связи логов одного запроса. Log levels правильно: DEBUG для разработки, INFO для важных событий, ERROR для проблем."

---

## 3. Metrics с Prometheus

### 🎯 Что спрашивают
> "Какие метрики вы отслеживаете?"

### Типы метрик

| Тип | Описание | Пример |
|-----|----------|--------|
| **Counter** | Только растёт | Количество запросов |
| **Gauge** | Может расти/падать | Текущие соединения |
| **Histogram** | Распределение значений | Latency запросов |
| **Summary** | Квантили | p99 latency |

### Python prometheus-client
```python
from prometheus_client import Counter, Histogram, Gauge, start_http_server

# Counter
REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

# Histogram
REQUEST_LATENCY = Histogram(
    'http_request_duration_seconds',
    'HTTP request latency',
    ['method', 'endpoint'],
    buckets=[0.01, 0.05, 0.1, 0.5, 1.0, 5.0]
)

# Gauge
ACTIVE_CONNECTIONS = Gauge(
    'active_connections',
    'Number of active connections'
)

@app.middleware("http")
async def metrics_middleware(request, call_next):
    start = time.time()
    response = await call_next(request)
    
    REQUEST_COUNT.labels(
        method=request.method,
        endpoint=request.url.path,
        status=response.status_code
    ).inc()
    
    REQUEST_LATENCY.labels(
        method=request.method,
        endpoint=request.url.path
    ).observe(time.time() - start)
    
    return response

# Экспорт метрик
start_http_server(8000)  # /metrics endpoint
```

### RED Method (для сервисов)
- **R**ate — запросов в секунду
- **E**rrors — процент ошибок
- **D**uration — latency (p50, p95, p99)

### USE Method (для ресурсов)
- **U**tilization — % использования
- **S**aturation — очереди
- **E**rrors — ошибки

### 📝 Фраза для интервью
> "Prometheus для сбора метрик, Grafana для визуализации. RED method для сервисов: Rate, Errors, Duration. USE для ресурсов: Utilization, Saturation, Errors. Histogram для latency с кастомными buckets."

---

## 4. Distributed Tracing

### 🎯 Что спрашивают
> "Как отлаживать запрос через несколько сервисов?"

### Проблема
```
User → API Gateway → User Service → Order Service → Payment Service
                                                          ↓
                                              Где задержка?
```

### Решение: Distributed Tracing
```
Trace ID: abc-123
├── Span: API Gateway (50ms)
│   ├── Span: User Service (20ms)
│   └── Span: Order Service (200ms)
│       └── Span: Payment Service (150ms)  ← Bottleneck!
```

### OpenTelemetry (стандарт)
```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.exporter.jaeger.thrift import JaegerExporter

# Setup
trace.set_tracer_provider(TracerProvider())
jaeger_exporter = JaegerExporter(
    agent_host_name="localhost",
    agent_port=6831,
)
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(jaeger_exporter)
)

tracer = trace.get_tracer(__name__)

# Использование
@app.get("/users/{user_id}")
async def get_user(user_id: int):
    with tracer.start_as_current_span("get_user") as span:
        span.set_attribute("user.id", user_id)
        
        with tracer.start_as_current_span("db_query"):
            user = await db.get_user(user_id)
        
        with tracer.start_as_current_span("cache_update"):
            await cache.set(f"user:{user_id}", user)
        
        return user
```

### Context Propagation
```python
# Передаём trace context между сервисами
headers = {
    "traceparent": f"00-{trace_id}-{span_id}-01"
}
response = await httpx.get(url, headers=headers)
```

### 📝 Фраза для интервью
> "Distributed tracing через OpenTelemetry. Trace состоит из spans. Trace ID передаётся между сервисами в headers. Jaeger/Zipkin для визуализации. Находим bottlenecks, понимаем dependencies."

---

## 5. Alerting

### 🎯 Что спрашивают
> "Как настроить алертинг?"

### Принципы хорошего алертинга

| Принцип | Описание |
|---------|----------|
| **Actionable** | Алерт требует действия |
| **Urgent** | Требует немедленной реакции |
| **Unique** | Не дублирует другие алерты |
| **Understandable** | Понятно что делать |

### SLI / SLO / SLA
```
SLI (Service Level Indicator):
  - Request latency p99 < 200ms
  - Error rate < 0.1%
  - Availability 99.9%

SLO (Service Level Objective):
  - "99.9% запросов за месяц < 200ms"

SLA (Service Level Agreement):
  - SLO + последствия (деньги, credits)
```

### Error Budget
```
SLO: 99.9% availability
Monthly minutes: 43,200
Error budget: 43,200 × 0.1% = 43.2 minutes/month

Если израсходовали error budget → stop deployments, focus on reliability
```

### Prometheus Alerting Rules
```yaml
groups:
- name: api-alerts
  rules:
  - alert: HighErrorRate
    expr: |
      sum(rate(http_requests_total{status=~"5.."}[5m]))
      /
      sum(rate(http_requests_total[5m])) > 0.01
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "High error rate ({{ $value | humanizePercentage }})"
      runbook: "https://wiki.example.com/runbooks/high-error-rate"
  
  - alert: HighLatency
    expr: |
      histogram_quantile(0.99, 
        rate(http_request_duration_seconds_bucket[5m])
      ) > 0.5
    for: 5m
    labels:
      severity: warning
```

### 📝 Фраза для интервью
> "SLI — метрика, SLO — цель, SLA — обязательство с последствиями. Error budget — сколько можем сломать оставаясь в SLO. Алерты должны быть actionable, на SLO violations, не на симптомы."

---

## 6. Dashboards

### 🎯 Что спрашивают
> "Какие дашборды должны быть?"

### Типы дашбордов

#### 1. Overview Dashboard
```
┌─────────────┬─────────────┬─────────────┐
│ Requests/s  │ Error Rate  │ P99 Latency │
│    1,234    │    0.1%     │   150ms     │
├─────────────┴─────────────┴─────────────┤
│        Request Rate (24h graph)         │
├─────────────────────────────────────────┤
│        Error Rate by Service            │
├─────────────────────────────────────────┤
│        Top 5 Slow Endpoints             │
└─────────────────────────────────────────┘
```

#### 2. Service Dashboard
```
Service: payment-service
┌─────────────────────────────────────────┐
│ Dependencies Health                      │
│ [✓] Database  [✓] Redis  [!] Stripe    │
├─────────────────────────────────────────┤
│ Endpoints Performance                    │
│ POST /payments - 200ms p99              │
│ GET /payments/{id} - 50ms p99           │
├─────────────────────────────────────────┤
│ Resource Usage                           │
│ CPU: 45% | Memory: 2.1GB | Pods: 5     │
└─────────────────────────────────────────┘
```

### 📝 Фраза для интервью
> "Overview dashboard для быстрого понимания здоровья. Service dashboards для deep dive. Four Golden Signals: latency, traffic, errors, saturation. Runbook links на каждом алерте."

---

## 🎤 Частые вопросы Observability

### "Как дебажить медленный запрос в production?"
> "Смотрю trace для этого request_id. Нахожу span с наибольшей latency. Проверяю логи этого span'а. Смотрю метрики сервиса в это время (CPU, memory, DB connections)."

### "Сколько логов хранить?"
> "Зависит от требований. Hot storage (Elasticsearch): 7-30 дней для поиска. Cold storage (S3): месяцы-годы для compliance. Sampling для high-volume сервисов."

### "Как не утонуть в алертах?"
> "Alert fatigue — реальная проблема. Алерты только на SLO violations. Группировка связанных алертов. Runbook для каждого алерта. Регулярный review и удаление шумных алертов."

### "OpenTelemetry vs Jaeger vs Zipkin?"
> "OpenTelemetry — vendor-neutral стандарт для instrumentation. Jaeger/Zipkin — backends для хранения и визуализации traces. Рекомендую OpenTelemetry SDK + любой backend."

### "Как мониторить Kubernetes?"
> "kube-state-metrics для состояния объектов. metrics-server для resource usage. Prometheus Operator для автоматического discovery. Grafana dashboards из kubernetes-mixin."
