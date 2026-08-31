# System Design - Руководство для Senior Backend интервью

> 💡 **Как подходить к System Design на интервью:**
> "Сначала уточняю требования и constraints. Потом high-level дизайн. Затем deep dive в компоненты. Всегда обсуждаю trade-offs и bottlenecks."

---

## 1. Подход к System Design

### 🎯 Что спрашивают
> "Спроектируйте Twitter/URL Shortener/Instagram..."

### Framework для ответа (4 шага)

```
1. REQUIREMENTS (5 мин)
   - Функциональные: что система делает?
   - Нефункциональные: latency, throughput, availability?
   - Constraints: DAU, QPS, storage?

2. HIGH-LEVEL DESIGN (10 мин)
   - Основные компоненты
   - Data flow
   - API endpoints

3. DEEP DIVE (20 мин)
   - Database schema
   - Scaling каждого компонента
   - Caching strategy

4. BOTTLENECKS & TRADE-OFFS (5 мин)
   - Что может сломаться?
   - Как мониторить?
   - Альтернативные решения
```

### 📝 Фраза для интервью
> "Прежде чем проектировать, хочу уточнить requirements. Сколько пользователей? Какая нагрузка? Read-heavy или write-heavy? Какие SLA по latency?"

---

## 2. CAP Theorem

### 🎯 Что спрашивают
> "Объясните CAP теорему"

### Простое объяснение

```
       Consistency
          /\
         /  \
        /    \
       /  CA  \
      /________\
     /\        /\
    /  \  CP  /  \
   / AP \    /    \
  /______\  /______\
Availability  Partition
              Tolerance
```

| Буква | Значение | Пример |
|-------|----------|--------|
| **C** (Consistency) | Все видят одинаковые данные | После записи все читают новое значение |
| **A** (Availability) | Система всегда отвечает | Каждый запрос получает ответ |
| **P** (Partition Tolerance) | Работает при сетевых сбоях | Узлы могут терять связь |

### Выбор в реальности
- **CP** (Consistency + Partition): банковские системы → жертвуем доступностью
- **AP** (Availability + Partition): социальные сети → eventual consistency

### 📝 Фраза для интервью
> "CAP говорит, что при network partition нужно выбрать между consistency и availability. На практике выбираем partition tolerance (сеть ненадёжна), и решаем: CP для финансов, AP для социальных сетей с eventual consistency."

---

## 3. Consistent Hashing

### 🎯 Что спрашивают
> "Как распределить данные между серверами?"

### Проблема обычного хеширования
```python
server = hash(key) % N  # N серверов

# При добавлении сервера (N→N+1) — почти ВСЕ ключи перераспределяются!
```

### Решение: Consistent Hashing
```
        ┌─────────────────┐
        │    Hash Ring    │
        │                 │
   S1 ──┤    ●           │
        │     \          │
        │      K1        │
        │        \       │
   S2 ──┤         ●──K2  │
        │              \ │
        │               ●── S3
        │              /  │
        │            K3   │
        └─────────────────┘
        
При добавлении S4 — перемещается только часть ключей!
```

### Virtual Nodes
Проблема: неравномерное распределение.
Решение: каждый сервер = много точек на кольце.

### 📝 Фраза для интервью
> "Consistent hashing минимизирует перемещение данных при изменении числа серверов. Используется в distributed caches (Redis Cluster), databases (Cassandra), load balancers. Virtual nodes решают проблему неравномерности."

---

## 4. Load Balancing

### 🎯 Что спрашивают
> "Как распределить нагрузку между серверами?"

### Алгоритмы

| Алгоритм | Описание | Когда использовать |
|----------|----------|-------------------|
| **Round Robin** | По очереди | Одинаковые серверы |
| **Weighted RR** | С весами | Разная мощность |
| **Least Connections** | К наименее загруженному | Разное время обработки |
| **IP Hash** | По IP клиента | Sticky sessions |
| **Consistent Hash** | По ключу | Кеширование |

### Уровни Load Balancing
```
DNS (L7) → CDN → L4 Load Balancer → L7 Load Balancer → App Servers
                   (TCP/UDP)           (HTTP)
```

### Health Checks
```nginx
upstream backend {
    server backend1:8080;
    server backend2:8080;
    
    health_check interval=5s fails=3 passes=2;
}
```

### 📝 Фраза для интервью
> "L4 балансер работает на транспортном уровне (TCP), быстрее но не понимает HTTP. L7 понимает HTTP, может роутить по URL, headers. Для stateless приложений — Round Robin, для sticky sessions — IP Hash."

---

## 5. Caching Strategies

### 🎯 Что спрашивают
> "Как правильно кешировать?"

### Паттерны кеширования

#### Cache-Aside (Lazy Loading)
```python
def get_user(user_id):
    user = cache.get(f"user:{user_id}")
    if user is None:
        user = db.query(user_id)
        cache.set(f"user:{user_id}", user, ttl=3600)
    return user
```
✅ Простой, кешируем только нужное
❌ Cache miss = latency spike

#### Write-Through
```python
def update_user(user_id, data):
    db.update(user_id, data)
    cache.set(f"user:{user_id}", data)  # Сразу в кеш
```
✅ Данные всегда актуальны
❌ Write latency выше

#### Write-Behind (Write-Back)
```python
def update_user(user_id, data):
    cache.set(f"user:{user_id}", data)
    queue.push({"user_id": user_id, "data": data})  # Async write to DB
```
✅ Быстрые записи
❌ Риск потери данных

### Cache Invalidation
```
"There are only two hard things in Computer Science: 
cache invalidation and naming things."
```

**Стратегии:**
- TTL — автоматическое устаревание
- Event-based — при изменении в БД
- Version-based — версия в ключе

### 📝 Фраза для интервью
> "Cache-aside для read-heavy, write-through для write-heavy с консистентностью, write-behind для максимальной производительности записи. Инвалидация — через TTL или events. Главное — определить TTL под бизнес-требования."

---

## 6. Database Scaling

### 🎯 Что спрашивают
> "Как масштабировать базу данных?"

### Vertical vs Horizontal
```
Vertical: Больше CPU/RAM на одном сервере
          └── Ограничено, дорого

Horizontal: Больше серверов
           └── Сложнее, но масштабируется
```

### Read Replicas
```
           ┌─────────┐
           │ Primary │ ← Writes
           └────┬────┘
      ┌─────────┼─────────┐
      ▼         ▼         ▼
  ┌───────┐ ┌───────┐ ┌───────┐
  │Replica│ │Replica│ │Replica│ ← Reads
  └───────┘ └───────┘ └───────┘
```

### Sharding (Horizontal Partitioning)
```
Shard Key: user_id

user_id 1-1M      → Shard 1
user_id 1M-2M     → Shard 2
user_id 2M-3M     → Shard 3
```

**Проблемы sharding:**
- Cross-shard queries (JOIN между шардами)
- Rebalancing при добавлении шардов
- Distributed transactions

### 📝 Фраза для интервью
> "Сначала оптимизация (индексы, query), затем vertical scaling, потом read replicas. Sharding — крайняя мера, усложняет архитектуру. Выбор shard key критичен — по нему должны идти большинство запросов."

---

## 7. Message Queues & Event-Driven

### 🎯 Что спрашивают
> "Когда использовать очереди сообщений?"

### Use Cases
```
1. Async Processing
   User → API → Queue → Worker → Email sent
         ↓
       "OK" (instant)

2. Decoupling
   Order Service → Queue ← Payment Service
                        ← Inventory Service
                        ← Notification Service

3. Load Leveling
   Spike 10K RPS → Queue → Worker (100 RPS steady)
```

### RabbitMQ vs Kafka

| Aspect | RabbitMQ | Kafka |
|--------|----------|-------|
| Model | Message Queue | Event Log |
| Delivery | Push | Pull |
| Retention | Until consumed | Time-based (days) |
| Ordering | Per queue | Per partition |
| Use case | Task queues, RPC | Event streaming, replay |

### 📝 Фраза для интервью
> "Очереди для async processing, decoupling, load leveling. RabbitMQ — традиционный брокер, умное роутинг. Kafka — распределённый лог, для event streaming и когда нужен replay."

---

## 8. Rate Limiting

### 🎯 Что спрашивают
> "Спроектируйте rate limiter"

### Алгоритмы

#### Token Bucket
```
Bucket capacity: 10 tokens
Refill rate: 1 token/second

Request comes:
  - If tokens > 0: allow, tokens--
  - Else: reject (429)
```

#### Sliding Window
```python
def is_allowed(user_id, limit=100, window=60):
    now = time.time()
    key = f"rate:{user_id}"
    
    # Удаляем старые записи
    redis.zremrangebyscore(key, 0, now - window)
    
    # Проверяем count
    count = redis.zcard(key)
    if count < limit:
        redis.zadd(key, {str(now): now})
        redis.expire(key, window)
        return True
    return False
```

### Distributed Rate Limiting
```
┌─────────┐     ┌─────────┐
│ Server1 │────▶│  Redis  │◀──── Server2
└─────────┘     │ (shared │      └─────────┘
                │ counter)│
                └─────────┘
```

### 📝 Фраза для интервью
> "Token bucket для bursty traffic, sliding window для точного лимита. Distributed rate limiting через Redis (atomic INCR + EXPIRE). Важно: graceful degradation, 429 с Retry-After header."

---

## 9. URL Shortener Design

### 🎯 Классический вопрос
> "Спроектируйте URL shortener типа bit.ly"

### Requirements
- Shorten URL: `longurl.com/very/long/path` → `short.ly/abc123`
- Redirect: `short.ly/abc123` → 301 → original URL
- Scale: 100M URLs, 10K redirects/sec

### High-Level Design
```
┌──────────┐     ┌─────────────┐     ┌──────────┐
│  Client  │────▶│ API Gateway │────▶│   App    │
└──────────┘     └─────────────┘     └────┬─────┘
                                          │
                      ┌───────────────────┼───────────────────┐
                      ▼                   ▼                   ▼
                 ┌─────────┐         ┌─────────┐         ┌─────────┐
                 │  Redis  │         │ Postgres│         │   S3    │
                 │ (cache) │         │  (data) │         │ (stats) │
                 └─────────┘         └─────────┘         └─────────┘
```

### Key Generation
```python
# Вариант 1: Base62 encoding счётчика
def generate_short_url(counter):
    chars = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
    short = ""
    while counter > 0:
        short = chars[counter % 62] + short
        counter //= 62
    return short

# Вариант 2: Hash + collision handling
def generate_short_url(long_url):
    hash = md5(long_url)[:7]
    while db.exists(hash):
        hash = md5(long_url + random())[:7]
    return hash
```

### Database Schema
```sql
CREATE TABLE urls (
    short_code VARCHAR(7) PRIMARY KEY,
    original_url TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP,
    click_count INT DEFAULT 0
);
```

---

## 10. Microservices Patterns

### 🎯 Что спрашивают
> "Какие паттерны микросервисов вы знаете?"

### Circuit Breaker
```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failures = 0
        self.state = "CLOSED"
        self.last_failure = None
    
    def call(self, func):
        if self.state == "OPEN":
            if time.time() - self.last_failure > self.timeout:
                self.state = "HALF-OPEN"
            else:
                raise CircuitOpenError()
        
        try:
            result = func()
            self.failures = 0
            self.state = "CLOSED"
            return result
        except Exception:
            self.failures += 1
            self.last_failure = time.time()
            if self.failures >= self.failure_threshold:
                self.state = "OPEN"
            raise
```

### Saga Pattern (Distributed Transactions)
```
Order Saga:
1. Create Order (Order Service)
2. Reserve Inventory (Inventory Service)
3. Process Payment (Payment Service)
4. Ship Order (Shipping Service)

Compensation (rollback):
4. Cancel Shipment
3. Refund Payment
2. Release Inventory
1. Cancel Order
```

### API Gateway
```
                    ┌─────────────┐
   Client ─────────▶│ API Gateway │
                    └──────┬──────┘
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         ┌────────┐   ┌────────┐   ┌────────┐
         │ Users  │   │ Orders │   │Products│
         │Service │   │Service │   │Service │
         └────────┘   └────────┘   └────────┘
```

**Responsibilities:**
- Routing
- Authentication
- Rate limiting
- Request aggregation

### 📝 Фраза для интервью
> "Circuit Breaker предотвращает каскадные отказы. Saga для распределённых транзакций с compensation logic. API Gateway — единая точка входа с auth, rate limiting, routing."

---

## 11. Проектирование систем с LLM/AI-компонентом (RAG, AI-агенты)

### 🎯 Что спрашивают на интервью
> "Спроектируйте чат-бота поддержки на базе RAG" / "Спроектируйте backend для AI-ассистента для написания кода" / "Спроектируйте pipeline модерации контента с использованием LLM"

С 2025-2026 это отдельный, устоявшийся жанр System Design интервью для senior/staff уровня. Важно: интервьюер обычно не проверяет умение писать промпты — это отдельная дисциплина (см. **14_Prompt_Engineering.md**, разделы про RAG, Agents, Context Engineering, Evaluation, Prompt Security). Здесь проверяют **системное мышление**: throughput, latency budget, отказоустойчивость и cost — то же самое, что и для любой другой системы, но с непривычным набором failure modes, потому что вызов LLM — это не быстрый детерминированный DB-запрос, а медленный (секунды), дорогой и иногда нестабильный сетевой вызов к внешнему (или self-hosted) inference-сервису.

### High-Level Building Blocks

```
┌──────────┐   ┌───────────────┐   ┌──────────────┐   ┌───────────────┐
│  Client  │──▶│  API Gateway  │──▶│ Orchestrator │──▶│  Vector Store │
└──────────┘   │ (auth, rate   │   │   Service    │◀──│  (retrieval)  │
     ▲         │  limiting)    │   └──────┬───────┘   └───────────────┘
     │         └───────────────┘          │                   ▲
     │  SSE/streaming                     ▼                   │
     │                             ┌───────────────┐   ┌──────┴───────┐
     └─────────────────────────────│ LLM Inference │   │  Ingestion / │
                                    │ Layer         │   │  Chunking    │
                                    │ (+ fallback,  │   │  Pipeline    │
                                    │  timeout,     │   │ (batch/async)│
                                    │  cache)       │   └──────────────┘
                                    └───────────────┘
```

Компоненты и что о них нужно сказать на интервью:

| Компонент | Роль | На что обратить внимание в System Design |
|---|---|---|
| **Ingestion/chunking pipeline** | Превращает сырые документы в проиндексированные чанки | Асинхронный batch-процесс (не блокирует запись), инкрементальная переиндексация при обновлении источника, версионирование индекса |
| **Vector store** | Хранит embeddings, отдаёт top-k по similarity | Отдельный масштабируемый компонент (Pinecone/Weaviate/pgvector/OpenSearch) — read-heavy, требует своего SLA |
| **Retrieval + reranking** | Находит и сортирует релевантный контекст | Добавляет latency (ещё один сетевой хоп), обычно десятки-сотни мс — отдельная строка в latency budget |
| **LLM inference layer** | Генерирует ответ на основе контекста | Самый медленный и дорогой шаг (сотни мс — десятки секунд); нужен собственный слой отказоустойчивости (см. ниже) |
| **Response cache** | Кеширует LLM-ответы для повторяющихся/похожих запросов | Exact-match кеш дёшев; semantic cache (по similarity embedding-а запроса) сложнее, но покрывает больше повторов |
| **Streaming delivery** | Отдаёт токены клиенту по мере генерации | SSE/WebSocket вместо ожидания полного ответа — критично для perceived latency при генерации, занимающей секунды |
| **Cost control layer** | Ограничивает и предсказывает расходы | Per-user/per-tenant token budget, model routing, rate limiting отдельно от инфраструктурного rate limiting |
| **Evaluation/observability** | Отслеживает качество ответов в production | Логирование prompt+response, sampling для human review, LLM-as-judge метрики, drift/regression алерты (детали — файл 14, раздел Evaluation) |

### Latency Budget — думать иначе, чем для обычного API

Для обычного CRUD API p95 latency budget — это в основном DB query + network. Для LLM-системы latency почти целиком определяется inference-шагом:

```
Пример декомпозиции budget на p95 = 3s для RAG-ответа:
  Auth + routing              ~10ms
  Query embedding              ~20ms
  Vector search (top-k)        ~50ms
  Reranking                    ~80ms
  LLM inference (generation)   ~2500ms   ← доминирует бюджет
  Network/serialization        ~340ms
```

Вывод для интервью: оптимизировать retrieval-часть почти бессмысленно, если основной бюджет съедает генерация — приоритет оптимизации: streaming (perceived latency), выбор модели по сложности задачи (model routing), кеширование, и только потом — тюнинг retrieval pipeline.

### Failure Modes, специфичные для LLM-компонента

В отличие от обычного DB-вызова, вызов LLM может быть медленным, дорогим и нестабильным одновременно — систему нужно проектировать так, будто внешний API "иногда просто не отвечает":

| Failure mode | Защита |
|---|---|
| Timeout / зависший запрос | Строгий timeout + fallback (меньшая/быстрая модель, кэшированный generic-ответ, честное "попробуйте позже") |
| Provider rate limit / 429 | Exponential backoff, очередь, multi-provider fallback (если архитектура это позволяет) |
| Провайдер недоступен (outage) | Circuit breaker (см. раздел 10) вокруг LLM-вызова, деградация до retrieval-only ответа ("вот релевантные документы, ответ сгенерировать не удалось") |
| Дорогой/аномально длинный запрос | Лимит на длину входного контекста и max_output_tokens на уровне API-контракта |
| Некачественный/галлюцинированный ответ | Guardrails на выходе (валидация формата, grounding-check против retrieved-документов), human-in-the-loop для чувствительных сценариев |

### Cost Controls как часть дизайна, а не afterthought

- **Model routing**: простые/классификационные под-задачи — на маленькую дешёвую модель, сложную генерацию — на flagship (см. файл 14, раздел Production Best Practices).
- **Per-tenant/per-user token budget**: не только request-based rate limiting, но и лимит на токены за период — иначе один "тяжёлый" пользователь может съесть непропорционально много бюджета.
- **Кеширование** — самый простой и эффективный cost lever: повторяющиеся вопросы (FAQ-паттерн в support-чатботе) не должны каждый раз идти в LLM.
- **Batch/async путь** для non-realtime сценариев (например, ночная переиндексация, bulk-модерация) — дешевле синхронного inference.

Подробности про сами техники (chunking, hybrid search, agentic RAG, prompt caching, evaluation, защиту от prompt injection) — см. **14_Prompt_Engineering.md**; здесь важно на интервью показать, что вы embed-иваете LLM-вызов в систему так же аккуратно, как любой другой ненадёжный внешний dependency — с timeout, retry, circuit breaker, кэшем и explicit cost budget.

### 📝 Фраза для интервью
> "Для систем с LLM-компонентом я использую тот же system design framework, что и для любой системы, но отдельно закладываю: latency budget, где генерация LLM доминирует бюджет — поэтому в первую очередь streaming и model routing, а не оптимизация retrieval; отказоустойчивость вокруг LLM-вызова как вокруг любого медленного/нестабильного внешнего API — timeout, circuit breaker, fallback на меньшую модель или retrieval-only ответ; и explicit cost control — кэширование ответов, per-tenant token budget, model routing по сложности задачи, потому что здесь стоимость растёт с токенами, а не только с числом запросов. Сами техники RAG/prompting/агентов — это отдельный слой, который я разбираю в контексте prompt engineering, а не system design."

---

## 12. Change Data Capture (CDC)

### 🎯 Что спрашивают
> "Как синхронизировать данные из основной БД в поисковый индекс/кеш/другой сервис без риска рассинхронизации?"

### Проблема: dual-write

```
❌ Наивный подход (dual write):
   app.save_to_db(order)
   app.publish_to_kafka(order)   ← если это упадёт после успешного db.save —
                                    БД и Kafka/индекс/кеш разошлись
```

Два независимых write не атомарны — падение между ними (сеть, краш процесса, деплой) оставляет системы в противоречивом состоянии, и на масштабе это не редкий edge case, а вопрос времени.

### Решение: CDC (Debezium-style)

CDC читает **лог транзакций самой БД** (binlog в MySQL, WAL в PostgreSQL, oplog в MongoDB) — тот же механизм, которым БД реплицирует данные на реплики — и превращает каждое закоммиченное изменение в событие. Приложение продолжает делать только один write — в основную БД; событие гарантированно появится, потому что источник истины — сам transaction log, а не отдельный вызов из кода приложения.

```
┌──────────┐   commit   ┌──────────────┐  tail log   ┌───────────┐   events   ┌─────────────────┐
│   App    │───────────▶│  Primary DB  │────────────▶│ Debezium  │───────────▶│  Kafka topic     │
│          │  (1 write) │ (WAL/binlog) │  connector  │ Connector │            │  (order.events)  │
└──────────┘            └──────────────┘             └───────────┘            └────────┬─────────┘
                                                                                         │
                                          ┌──────────────────────────────────────────────┼──────────────┐
                                          ▼                          ▼                    ▼              ▼
                                   Search index update      Cache invalidation     Analytics/DWH   Другой сервис
```

### CDC vs Transactional Outbox

| Подход | Как работает | Trade-off |
|---|---|---|
| **Transactional Outbox** | Приложение в той же транзакции, что и основной write, пишет запись в таблицу `outbox`; отдельный процесс читает и публикует | Проще внедрить точечно, не требует доступа к WAL/прав на репликацию, но нужен отдельный relay-процесс |
| **CDC (Debezium)** | Читает транзакционный лог БД напрямую, без изменений в коде приложения | Не требует менять application-код и схему; требует инфраструктуры (Kafka Connect), доступа к логам БД, аккуратной schema evolution |

Оба решают одну и ту же проблему — исключить dual-write inconsistency. Outbox проще для одного сервиса, CDC масштабируется как общая платформенная возможность сразу для многих сервисов и таблиц.

### Типичные use cases
- Инвалидация/обновление кеша при изменении в БД (вместо TTL-only подхода из раздела 5).
- Синхронизация с поисковым индексом (Elasticsearch/OpenSearch) без дублирования write-логики.
- Event-driven интеграция между микросервисами без прямых синхронных вызовов между ними.
- Репликация в data lake/DWH для аналитики почти в реальном времени, без ночного batch ETL.

### Trade-offs
- Всё ещё eventual consistency — консьюмеры узнают об изменении с небольшой задержкой (обычно секунды).
- Schema evolution в исходной таблице должна быть аккуратной — иначе ломает downstream-консьюмеров.
- Дополнительный операционный компонент (Kafka Connect + Debezium), который сам нужно мониторить (replication lag — важная метрика).

### 📝 Фраза для интервью
> "Dual write — запись в БД и отдельно в очередь/индекс/кеш — не атомарна и на масштабе гарантированно рассинхронизируется. CDC (Debezium и подобные) решает это, читая транзакционный лог самой БД (binlog/WAL), вместо того чтобы просить приложение делать второй write руками — приложение как писало в одну БД, так и пишет, а событие в Kafka гарантированно появляется, потому что источник — сам commit. Использую для инвалидации кеша, синхронизации поискового индекса и event-driven интеграции между сервисами без прямых синхронных вызовов. Альтернатива для более локального случая — transactional outbox pattern, если не хочется поднимать инфраструктуру CDC ради одной интеграции."

---

## 13. Cell-based Architecture

### 🎯 Что спрашивают
> "Как ограничить blast radius инцидента, чтобы плохой деплой или перегрузка не роняли всех пользователей сразу?"

### Простое объяснение
Cell-based architecture — паттерн, при котором вся система разбивается на несколько независимых, идентично устроенных "ячеек" (cells), каждая из которых — полноценный, изолированный стек (compute + storage + кеш + очереди), обслуживающий свой ограниченный набор пользователей/тенантов. Это отличается от классического sharding: шардируется не только БД, а **весь стек целиком**, включая app-серверы.

```
                         ┌───────────────┐
        Requests ───────▶│  Cell Router  │  (по customer_id / tenant_id)
                         └───────┬───────┘
              ┌───────────────────┼───────────────────┐
              ▼                   ▼                   ▼
       ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
       │   Cell 1     │     │   Cell 2     │     │   Cell 3     │
       │ App+DB+Cache │     │ App+DB+Cache │     │ App+DB+Cache │
       │  + Queue     │     │  + Queue     │     │  + Queue     │
       │ customers    │     │ customers    │     │ customers    │
       │ 1-N          │     │ N+1-2N       │     │ 2N+1-3N      │
       └─────────────┘     └─────────────┘     └─────────────┘
    Плохой деплой/перегрузка/баг в Cell 2 не задевает Cell 1 и Cell 3
```

### Почему это не то же самое, что просто sharding БД
- Обычный database sharding изолирует только слой данных — баг в app-коде или перегрузка одного app-сервера всё равно может задеть всех пользователей, если app-слой общий.
- В cell-based архитектуре реплицируется **весь стек** на ячейку — деплой раскатывается по ячейкам поочерёдно (cell-by-cell rollout), баг в новой версии приложения проявится в одной ячейке, а не сразу везде.
- Router/control plane — тонкий, максимально простой и надёжный компонент (единственная по-настоящему общая часть системы), задача которого — только маршрутизация запроса к правильной ячейке.

### Trade-offs

| Плюсы | Минусы |
|---|---|
| Ограниченный blast radius — инцидент/баг/перегрузка задевает только часть пользователей | Операционная сложность: N раз больше окружений для мониторинга, деплоя, патчинга |
| Постепенный canary rollout деплоя по ячейкам естественным образом | Кросс-cell операции (агрегация, кросс-tenant отчётность) сложнее |
| Легче планировать capacity — ячейка одинакового размера, масштабирование = добавление ячеек | Балансировка нагрузки между ячейками при неравномерном росте клиентов ("hot cell") |
| Проще выполнять требования по резидентности данных (разные ячейки в разных регионах) | Router/control plane — critical single point, требует отдельного, повышенного уровня надёжности |

### 📝 Фраза для интервью
> "Cell-based architecture — способ ограничить blast radius на большом масштабе: систему разбивают на несколько идентичных, полностью изолированных ячеек — каждая со своим app-слоем, БД, кешем, очередями — и каждая обслуживает свой набор клиентов. В отличие от обычного шардирования БД, изолируется весь стек, а не только данные, поэтому баг или перегрузка в одной ячейке не задевает остальные, а деплой можно раскатывать ячейка за ячейкой как встроенный canary. Платите за это операционной сложностью — N окружений вместо одного — и нужен тонкий, максимально надёжный router/control plane, который направляет запрос в нужную ячейку. Это паттерн, который AWS и другие крупные облачные провайдеры описывают как стандартный подход к масштабированию на very-large-scale."

---

## 14. FinOps и Cost-Awareness как измерение System Design

### 🎯 Что спрашивают
> "Как будет масштабироваться стоимость этого решения, и как вы будете её контролировать?"

### Простое объяснение
До недавнего времени senior system design интервью проверяло в основном latency/throughput/availability/consistency trade-offs. К 2025-2026 в это же обсуждение почти всегда добавляется **cost** как равноправное нефункциональное требование — интервьюер ожидает, что вы явно проговорите, во что выливается предложенный дизайн по деньгам при росте нагрузки, и какие рычаги есть для контроля.

### Основные cost levers, которые стоит упомянуть

| Область | Рычаг |
|---|---|
| **Compute** | Autoscaling под реальную нагрузку (не постоянный over-provisioning "на всякий случай"), spot/preemptible инстансы для fault-tolerant batch-задач, правильный instance sizing |
| **Storage** | Tiering горячих/тёплых/холодных данных (SSD → HDD → object storage → archive), lifecycle policies для автоматического перевода старых данных в дешёвый tier |
| **Данные между зонами/регионами** | Cross-AZ и особенно cross-region трафик часто оказывается недооценённой статьёй расходов — по возможности держать chatty-компоненты в одной зоне |
| **Managed vs self-hosted** | Managed-сервис (RDS, managed Kafka) дороже по прямой цене, но экономит operational cost (on-call, поддержка) — это тоже часть TCO, а не только "инфраструктура дешевле" |
| **Кеширование** | Снижает не только latency, но и cost — каждый закешированный запрос не долетает до дорогого downstream (БД, сторонний API, LLM-вызов) |
| **Наблюдаемость** | Логи/трейсы/метрики с высокой кардинальностью и долгим retention сами становятся заметной статьёй расходов на масштабе — нужна политика retention и sampling |
| **LLM/AI-компоненты** | Стоимость растёт с токенами, а не только с числом запросов — см. раздел 11 и файл 14 (model routing, prompt caching, batch API) |

### FinOps-практики, которые стоит назвать на senior/staff уровне
- **Cost attribution / tagging**: каждый ресурс помечен тегами (team/service/feature), чтобы видеть, кто и за что платит — без этого невозможен любой дальнейший контроль.
- **Budgets & alerts**: автоматические алерты при отклонении расходов от прогноза, а не постфактум разбор счёта в конце месяца.
- **Showback/chargeback**: команды видят (showback) или реально платят (chargeback) за свою долю инфраструктурных расходов — стимулирует cost-aware решения на уровне команды, а не только платформы.
- Cost — explicit trade-off наравне с latency: например, "мы можем срезать p99 latency ещё на 30%, но это удвоит infrastructure cost — оправдано ли это бизнес-требованием?" — именно такой ответ ожидают от senior/staff кандидата.

### 📝 Фраза для интервью
> "Cost — такое же нефункциональное требование, как latency и availability, и я закладываю его в дизайн явно, а не как afterthought. Основные рычаги — autoscaling и правильный instance sizing вместо постоянного over-provisioning, storage tiering для холодных данных, кеширование, чтобы не бить лишний раз по дорогим downstream-зависимостям, и осознанный выбор managed vs self-hosted с учётом operational cost, а не только price tag. На платформенном уровне — cost attribution через теги и алерты на отклонение от бюджета, чтобы cost-проблема обнаруживалась не в конце месяца по счёту от облака. Для систем с LLM-компонентом это особенно заметно, потому что стоимость растёт с токенами, а не только с количеством запросов."

---

## 🎤 Частые вопросы System Design

### "Как спроектировать Twitter?"
> "Read-heavy система. Fan-out on write для home timeline (предвычисляем при твите). Celebrities — fan-out on read (слишком много followers). Redis для timeline cache, Kafka для event streaming."

### "Как спроектировать Instagram?"
> "Media storage в S3/CDN. Metadata в PostgreSQL с sharding по user_id. News feed — push model для обычных, pull для celebrities. Кеширование на всех уровнях."

### "Как обеспечить 99.99% availability?"
> "Это 52 минуты downtime в год. Redundancy на всех уровнях: multi-AZ, load balancers, database replicas. Graceful degradation, circuit breakers, health checks."

### "Как обрабатывать 1M запросов в секунду?"
> "CDN для статики, aggressive caching, horizontal scaling, async processing через очереди, database sharding, load balancing на нескольких уровнях."

### "Что такое eventual consistency?"
> "Данные станут консистентными через некоторое время, не мгновенно. Пример: репликация между дата-центрами. Подходит когда availability важнее строгой консистентности."

### "Как спроектировать RAG-чатбота или AI-агента — на что смотрит интервьюер?"
> "На то же, что и в любом system design: throughput, latency budget, отказоустойчивость, cost. Отдельно — что генерация LLM обычно доминирует latency budget (секунды против миллисекунд у обычного DB-запроса), поэтому в первую очередь думаю про streaming ответа (SSE), timeout/circuit breaker/fallback вокруг LLM-вызова как вокруг любого нестабильного внешнего API, кеширование повторяющихся ответов и явный cost control (per-tenant token budget, model routing). Сами техники retrieval/prompting/агентов — это отдельный слой поверх этого системного каркаса."

### "Что такое CDC и когда его использовать вместо публикации события из кода приложения?"
> "CDC (Change Data Capture, например Debezium) читает транзакционный лог самой БД (binlog/WAL) и превращает каждый commit в событие — вместо того, чтобы приложение делало два независимых write (в БД и в очередь/индекс), которые могут разойтись при сбое между ними. Использую для инвалидации кеша, синхронизации поискового индекса, event-driven интеграции между сервисами без прямых синхронных вызовов."

### "Что такое cell-based architecture и чем отличается от шардирования?"
> "Система разбивается на несколько идентичных, изолированных 'ячеек' (cell) — каждая со своим app-слоем, БД, кешем — обслуживающих свой набор клиентов. В отличие от шардирования БД, где изолируются только данные, здесь изолируется весь стек, что ограничивает blast radius бага или перегрузки одной ячейкой и позволяет раскатывать деплой ячейка за ячейкой. Платится это операционной сложностью — N окружений вместо одного."

### "Как оценивать стоимость системы на System Design интервью?"
> "Проговариваю cost как отдельное нефункциональное требование наравне с latency/availability: как расходы растут с нагрузкой (compute, storage tiering, cross-region трафик, managed vs self-hosted), и какие рычаги контроля есть — кеширование, autoscaling вместо over-provisioning, cost attribution по тегам, алерты на отклонение от бюджета. Для систем с LLM-компонентом отдельно — что стоимость растёт с токенами, а не только с числом запросов."

---

## 📚 Чек-лист для подготовки

### Базовый уровень
- [ ] Framework 4 шагов (Requirements → High-Level → Deep Dive → Trade-offs)
- [ ] CAP теорема, CP vs AP на практике
- [ ] Consistent hashing, virtual nodes
- [ ] Load balancing (L4 vs L7, алгоритмы)
- [ ] Caching strategies (cache-aside, write-through, write-behind), инвалидация

### Средний уровень
- [ ] Database scaling: vertical/horizontal, read replicas, sharding
- [ ] Message queues vs event streaming (RabbitMQ vs Kafka)
- [ ] Rate limiting (token bucket, sliding window, distributed)
- [ ] Классические кейсы (URL shortener, Twitter, Instagram)
- [ ] Microservices patterns: Circuit Breaker, Saga, API Gateway
- [ ] CDC (Debezium-style) vs transactional outbox — синхронизация данных без dual-write

### Продвинутый / Senior-Staff уровень
- [ ] Проектирование систем с LLM/AI-компонентом: latency budget, отказоустойчивость LLM-вызова, кеширование, cost control, streaming (SSE)
- [ ] Cell-based architecture — ограничение blast radius на очень большом масштабе
- [ ] FinOps / cost-awareness как явное измерение дизайна наравне с latency/availability/consistency
- [ ] Умение проговорить trade-offs между cost, latency и надёжностью на конкретных цифрах
