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

### OpenTelemetry — теперь безальтернативный стандарт, а не "один из вариантов"

Важно чётко ответить на интервью, если спросят "почему именно OpenTelemetry, а не Jaeger SDK/vendor-specific агент напрямую": к 2025-2026 OpenTelemetry (OTel) — это не просто библиотека для трейсинга, а **единый vendor-neutral стандарт сразу для всех трёх столпов** (traces, metrics, **и logs** — logs API/SDK дозрел и вышел из experimental статуса позже двух остальных, поэтому в старых материалах OTel иногда описывают только как tracing-инструмент). Практически все крупные observability-вендоры (Datadog, Grafana, New Relic, Honeycomb и др.) либо нативно принимают OTLP (OpenTelemetry Protocol), либо дают свой collector-экспортер — это и есть смысл стандарта: инструментируешь код один раз, backend меняешь без переинструментирования.

```
                    Приложение (OTel SDK)
                            │
                 traces + metrics + logs
                    (единый OTLP-протокол)
                            │
                            ▼
                  ┌───────────────────┐
                  │  OTel Collector    │  ← receivers / processors / exporters
                  │  (опционально, но  │     (батчинг, sampling, PII-редактирование,
                  │   типично в проде) │      маршрутизация в 1+ backend)
                  └─────────┬──────────┘
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
          Jaeger      Prometheus     Datadog/Grafana/
        (traces)      (metrics)       любой OTLP-backend
```

Что стоит знать на senior-уровне:
- **OTel Collector** — отдельный, часто пропускаемый на собеседовании кусок: это не просто "прокси", а место для батчинга, sampling (в т.ч. tail-based — решение "оставить/выбросить" trace принимается уже после того, как он весь собран, что позволяет гарантированно оставлять все trace с ошибками), редактирования PII перед отправкой во внешний backend, и fan-out в несколько бэкендов одновременно без изменения кода приложения.
- **Auto-instrumentation** — для многих языков/фреймворков не нужно вручную оборачивать каждый вызов в span: агент/библиотека инструментирует популярные фреймворки (HTTP-клиенты, ORM, очереди) автоматически, ручная (manual) инструментация остаётся для бизнес-специфичных spans.
- **Semantic conventions** — OTel стандартизировал имена атрибутов (`http.method`, `db.system`, `service.name` и т.д.), поэтому дашборды и алерты переносимы между сервисами и даже между командами без придумывания своих соглашений заново.
- Использовать проприетарный vendor-agent напрямую (а не через OTel) сейчас оправдано в основном как **исключение**: узкая специфичная фича вендора, которой ещё нет в OTel, или legacy-система, куда дорого добавлять новую инструментацию — но это осознанный trade-off с vendor lock-in, а не дефолт.

### 📝 Фраза для интервью
> "Distributed tracing через OpenTelemetry — trace состоит из spans, trace ID передаётся между сервисами в headers (`traceparent`). Но OpenTelemetry сейчас — это не только tracing: единый vendor-neutral стандарт для traces, metrics и logs через общий OTLP-протокол, с Collector'ом посередине для батчинга, sampling и маршрутизации в любой backend без переинструментирования кода. Я использую OTel SDK по умолчанию и обращаюсь к проприетарному vendor-agent только как к осознанному исключению — например, ради фичи, которой ещё нет в стандарте."

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

### Multi-Window Multi-Burn-Rate алертинг (Google SRE workbook — must-know для senior)

Наивный алерт "error rate > X% за 5 минут" на практике плохо балансирует между двумя проблемами: слишком чувствительный — будит на кратковременных всплесках, которые не угрожают месячному SLO; слишком грубый — узнаёте о серьёзной проблеме, когда error budget уже сожжён. Google SRE workbook решает это через **burn rate** — во сколько раз быстрее обычного тратится error budget:

```
Burn rate = 1  → тратим error budget ровно по плану (100% budget за весь period)
Burn rate = 14.4 → при таком темпе сожжём весь budget за 2 часа

Комбинация из "быстрого, но короткого" и "медленного, но длинного" окна
ловит и внезапные инциденты, и медленную деградацию:

  Fast burn:  burn rate > 14.4 за 1h  AND  burn rate > 14.4 за 5m   → page (немедленно)
  Slow burn:  burn rate > 6    за 6h  AND  burn rate > 6    за 30m  → ticket (не срочно)
```

- Двойное окно (короткое + длинное с одинаковым порогом) — защита от алерта на кратковременный шум: реальный инцидент должен подтвердиться на обоих окнах.
- Это прямая эволюция простого "alert if error_rate > threshold" из примера выше — тот же Prometheus-стек, просто выражение считает burn rate относительно SLO/error budget, а не абсолютный процент ошибок.
- На интервью это хороший сигнал зрелости: наивный порог "5% ошибок = алерт" работает для демо, а multi-window burn rate — то, что реально используют команды, которые уже обожглись на alert fatigue или на пропущенном медленно нарастающем инциденте.

### 📝 Фраза для интервью
> "SLI — метрика, SLO — цель, SLA — обязательство с последствиями. Error budget — сколько можем сломать оставаясь в SLO. Алерты должны быть actionable, на SLO violations, не на симптомы. На практике не использую наивный 'error rate > X% за 5 минут' — настраиваю multi-window multi-burn-rate алертинг по методологии Google SRE: комбинация короткого и длинного окна на одном пороге burn rate ловит и резкие инциденты (page немедленно), и медленную деградацию error budget (тикет, не срочно), при этом не будит на кратковременном шуме."

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

## 7. eBPF-based Observability: "zero-instrumentation"

### 🎯 Что спрашивают
> "Как получить трейсы/метрики для сервиса, который вы не можете (или не успели) заинструментировать вручную?"

### Простое объяснение
Всё, что было в разделах 2-4, — это **ручная (или auto-) инструментация**: код (или подключённый агент) сам генерирует логи/метрики/spans изнутри процесса. **eBPF** (extended Berkeley Packet Filter) — технология ядра Linux, позволяющая безопасно выполнять сендбоксированный код прямо в кернеле, перехватывая системные вызовы, сетевые пакеты и события планировщика — без единой строчки кода в самом приложении и без его перезапуска.

```
Традиционная (SDK) инструментация:      eBPF-инструментация:
┌─────────────────┐                     ┌─────────────────┐
│   Приложение     │                     │   Приложение     │
│  + OTel SDK код  │  ← нужно менять     │  (без изменений) │  ← ничего не трогаем
│  внутри процесса │     код/redeploy    │                  │
└─────────────────┘                     └────────┬─────────┘
                                                  │ syscalls, network,
                                                  │ scheduler events
                                         ┌────────▼─────────┐
                                         │  eBPF-программа   │
                                         │  в ядре Linux      │
                                         └────────┬─────────┘
                                                  │
                                          Trace/metrics наружу
                                          (Hubble, Beyla, Pixie...)
```

### Инструменты, о которых стоит знать
- **Cilium Hubble** — сетевая observability поверх Cilium (eBPF-based CNI для Kubernetes): видно, какой под с каким подом/сервисом реально общается на уровне L3-L7, без service mesh sidecar'ов.
- **Grafana Beyla** — авто-инструментация HTTP/gRPC-трафика на уровне eBPF, экспортирует прямо в OpenTelemetry формат (traces + metrics) без единой строчки кода в приложении и без пересборки образа.
- **Pixie-style инструменты** — авто-собираемые метрики/трейсы/даже полные request/response payloads для сервисов в Kubernetes-кластере "из коробки", в основном для debug-сценариев (не всегда для постоянного sampling из-за overhead).

### Trade-off, который важно проговорить
| | Manual/SDK инструментация (OTel SDK) | eBPF ("zero-instrumentation") |
|---|----------------------------------------|-------------------------------|
| **Что видно** | Всё, что явно заинструментировано + бизнес-контекст (custom attributes) | В основном сетевой/syscall уровень — HTTP/gRPC запросы, латентность, коды ответов; бизнес-логика "внутри" функции не видна |
| **Усилия на внедрение** | Нужен код, redeploy, поддержка версий SDK | Не требует изменений в приложении — разворачивается на уровне кластера/хоста |
| **Legacy/сторонний код** | Сложно/невозможно без доступа к исходникам | Работает одинаково для любого сервиса, включая legacy и чужие бинарники |
| **Глубина** | Можно добавить произвольный custom span/атрибут под конкретную бизнес-задачу | Ограничено тем, что видно на уровне ядра/сети |

На практике это не "или/или" — на senior-интервью хороший ответ: eBPF закрывает "быстро получить базовую видимость по всему кластеру, включая legacy-сервисы, без единой правки кода", а SDK-инструментация остаётся нужна там, где важен бизнес-контекст внутри trace (какой именно use case, какой tenant, какая бизнес-ошибка) — они дополняют друг друга, а не взаимоисключают.

### 📝 Фраза для интервью
> "eBPF даёт 'zero-instrumentation' observability — программа в ядре Linux перехватывает syscalls и сетевые события без единой строчки кода в приложении и без redeploy. Инструменты вроде Cilium Hubble (сетевая видимость в Kubernetes), Grafana Beyla (авто-трейсинг HTTP/gRPC в OTel-формат) или Pixie-подобные решения дают мгновенную базовую видимость по всему кластеру, включая legacy-сервисы, куда SDK не добавить. Но это не замена ручной инструментации целиком — eBPF видит сетевой/syscall уровень, а не бизнес-контекст внутри запроса; в проде обычно комбинирую оба подхода."

---

## 8. LLM/AI Observability

### 🎯 Что спрашивают
> "Как мониторить систему, где часть запроса — это вызов LLM?"

### Простое объяснение
Раз в архитектуре backend-систем всё чаще есть шаг с вызовом LLM (RAG, агент, просто генерация текста — см. файл про Prompt Engineering), классические три столпа (logs/metrics/traces) не отвечают на специфичные для LLM вопросы: сколько это стоило, была ли галлюцинация, деградировало ли качество ответа после смены промпта/модели. Возникла отдельная специализированная observability-практика поверх обычной.

### Что именно трейсят, чего нет в "обычном" APM

| Метрика/сигнал | Зачем |
|-----------------|-------|
| **Token usage** (input/output/cached) | Основа для cost-attribution — обычный APM не знает про токены |
| **Cost per request/per user/per feature** | LLM-вызовы часто самая дорогая часть запроса — нужна granular атрибуция, не общий облачный счёт постфактум |
| **Time to first token (TTFT) + inter-token latency** | Для streaming-ответов обычная "latency запроса" не отражает perceived UX |
| **Prompt/response content** (с учётом PII) | Нужно для debugging и для eval — обычные логи часто не хранят/не структурируют это отдельно |
| **Eval/quality-метрики** (relevance, groundedness, LLM-as-judge score) | Ответ может быть "быстрым и дешёвым", но плохим по существу — это не увидеть по latency/error rate |
| **Retrieval-метрики для RAG** (что именно retrieved, релевантность чанков) | Плохой ответ часто из-за плохого retrieval, а не самой генерации — нужна трассировка этого шага отдельно |

### Инструменты этой ниши
**Langfuse**, **LangSmith**, **Helicone** и похожие — специализированные платформы поверх (обычно) OpenTelemetry-совместимой модели трейсинга, заточенные конкретно под LLM-вызовы: они рисуют "trace" вызова агента как дерево промптов/tool calls/retrieval-шагов с токенами и стоимостью на каждом узле, а не просто HTTP span.

```python
# Иллюстративная идея: LLM-вызов оборачивается как обычный span,
# но с LLM-специфичными атрибутами вместо/вместе с http.*
with tracer.start_as_current_span("llm_call") as span:
    response = llm.generate(prompt)

    span.set_attribute("llm.model", model_name)
    span.set_attribute("llm.usage.input_tokens", response.usage.input_tokens)
    span.set_attribute("llm.usage.output_tokens", response.usage.output_tokens)
    span.set_attribute("llm.cost_usd", estimate_cost(response.usage))
    span.set_attribute("llm.latency_ttft_ms", response.ttft_ms)
```

Хорошая новость для интервью: OpenTelemetry уже вводит **semantic conventions для GenAI** (стандартные имена атрибутов для LLM-вызовов), так что эта ниша постепенно сходится к тому же единому стандарту, что и остальная observability, а не остаётся набором несовместимых проприетарных SDK.

### 📝 Фраза для интервью
> "Там, где в request path есть LLM-вызов, обычных трёх столпов недостаточно — добавляю LLM-специфичный слой: токены и cost per request/user, TTFT и inter-token latency для streaming, и eval/quality-метрики (LLM-as-judge, groundedness), потому что плохой ответ не обязательно медленный или с ошибкой — он может быть просто бесполезным. Для этого использую специализированные инструменты вроде Langfuse/LangSmith/Helicone, которые трейсят цепочку промпт → retrieval → tool calls как дерево с токенами и стоимостью на каждом узле. Это дополняет, а не заменяет обычный OpenTelemetry-трейсинг — сейчас туда же добавляются semantic conventions для GenAI, так что оба слоя постепенно сходятся к единому стандарту."

---

## 9. Cost Observability / FinOps

### 🎯 Что спрашивают
> "Как вы следите за тем, сколько стоит эксплуатация системы, и это чья зона ответственности?"

### Простое объяснение
Исторически "сколько мы тратим на облако" — вопрос отдельной финансовой/закупочной команды, разбирающей счёт постфактум, раз в месяц. К 2025-2026 стандартная практика — включить cost-сигналы в тот же observability-стек, где живут latency/error rate/saturation: **FinOps** — не отдельная дисциплина в стороне, а ещё один класс сигналов рядом с обычными метриками, с той же логикой "увидеть аномалию быстро, а не в конце месяца".

### Почему это стало частью observability, а не только finance-процессом

```
Раньше:                                  Сейчас:
Cloud bill → раз в месяц →               Cost per service/request/tenant →
  финансовая команда замечает              real-time dashboard рядом
  всплеск → расследование задним           с latency/error rate →
  числом, через недели                     инженер видит аномалию
                                            сразу, как и любой другой инцидент
```

- **Cost per request/per tenant/per feature** — та же granular атрибуция, что и для LLM-observability (раздел 8), только шире: сколько стоит один API-запрос, включая compute, storage, egress, сторонние API.
- **Unit economics в дашборде рядом с RED-метриками** — не "весь облачный счёт за месяц", а "$ за 1000 запросов этого эндпоинта" или "$ за активного пользователя" — метрика, которую можно алертить так же, как error rate.
- **Аномалия по cost = такой же incident-триггер**, как аномалия по latency: неожиданный рост стоимости часто и есть первый видимый симптом бага (утечка ресурсов, retry storm, забытый неоптимизированный запрос, LLM-вызов без кэширования/лимитов).
- **Tagging/attribution инфраструктуры** (по сервису, команде, окружению) — предпосылка для того, чтобы cost-observability вообще была возможна на грануларном уровне, а не только "весь аккаунт целиком".

### 📝 Фраза для интервью
> "FinOps я не рассматриваю как отдельную от observability функцию — cost per request/tenant/feature должен быть в том же дашборде, что latency и error rate, и алертиться так же, как SLO-нарушение. Причина простая: неожиданный рост стоимости часто первый видимый симптом реального бага — retry storm, утечки ресурсов, неоптимизированного LLM-вызова без кэширования — и я хочу узнать об этом в момент аномалии, а не постфактум из счёта за облако в конце месяца."

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

### "Почему сейчас все советуют именно OpenTelemetry, а не vendor SDK напрямую?"
> "OpenTelemetry — единый vendor-neutral стандарт сразу для traces, metrics и logs через общий протокол (OTLP): инструментируешь код один раз, backend (Jaeger, Prometheus, Datadog, Grafana, что угодно) меняешь без переинструментирования. Vendor-specific агент напрямую сейчас оправдан как осознанное исключение — под фичу, которой нет в OTel — а не как дефолтный выбор."

### "Что такое eBPF-based observability и зачем она, если уже есть OpenTelemetry SDK?"
> "eBPF даёт 'zero-instrumentation' — программа в ядре Linux перехватывает syscalls и сетевые события без единой строчки кода в приложении. Инструменты вроде Cilium Hubble или Grafana Beyla дают мгновенную базовую видимость по всему кластеру, включая legacy-сервисы, куда SDK физически не добавить. Это не замена SDK-инструментации — eBPF видит сетевой/syscall уровень, а не бизнес-контекст внутри запроса; используются вместе."

### "Что такое multi-window multi-burn-rate алертинг?"
> "Наивный порог 'error rate > X% за 5 минут' либо слишком шумный, либо пропускает медленную деградацию. Multi-burn-rate считает, во сколько раз быстрее обычного тратится error budget, и комбинирует короткое и длинное окно на одном пороге: быстрый и высокий burn rate на обоих окнах — page немедленно, медленный и умеренный — тикет не срочно. Так ловятся и резкие инциденты, и медленно нарастающие проблемы, без лишнего alert fatigue."

### "Как мониторить систему с LLM-вызовами в request path?"
> "Обычных RED-метрик недостаточно — добавляю LLM-специфичный слой: token usage и cost per request, time to first token для streaming, и eval/quality-метрики (LLM-as-judge, groundedness для RAG), потому что плохой ответ может быть быстрым и дешёвым, но просто бесполезным. Использую специализированные инструменты (Langfuse/LangSmith/Helicone), которые трейсят цепочку промпт → retrieval → tool calls с токенами и стоимостью на каждом шаге — подробнее об этом слое в материале про Prompt Engineering."

### "Разве FinOps — не отдельная зона ответственности финансовой команды?"
> "Раньше — да, разбор счёта постфактум раз в месяц. Сейчас cost per request/tenant/feature — такой же дашборд-сигнал, как latency и error rate, и алертится так же. Причина: неожиданный рост стоимости часто первый видимый симптом реального бага (retry storm, утечка ресурсов, неоптимизированный LLM-вызов), и я хочу узнать об этом в момент аномалии, а не в конце месяца из облачного счёта."

---

## 📚 Чек-лист для подготовки

### Базовый уровень
- [ ] Три столпа: Logs, Metrics, Traces — когда что использовать
- [ ] Structured logging и correlation/request ID
- [ ] Типы метрик Prometheus (Counter, Gauge, Histogram, Summary)
- [ ] RED method и USE method

### Средний уровень
- [ ] Distributed tracing: spans, trace context propagation
- [ ] OpenTelemetry как единый стандарт для traces+metrics+logs, OTel Collector
- [ ] SLI / SLO / SLA, error budget
- [ ] Принципы хорошего алертинга (actionable, understandable), runbooks

### Продвинутый уровень
- [ ] Multi-window multi-burn-rate алертинг (Google SRE workbook)
- [ ] eBPF-based observability (Cilium Hubble, Grafana Beyla) vs SDK-инструментация
- [ ] Semantic conventions OpenTelemetry (в т.ч. для GenAI)
- [ ] LLM/AI observability: token usage, cost per request, TTFT, eval-метрики (Langfuse/LangSmith/Helicone)
- [ ] Cost observability / FinOps как часть observability-стека, а не отдельный finance-процесс
- [ ] Golden Signals и построение overview/service дашбордов
