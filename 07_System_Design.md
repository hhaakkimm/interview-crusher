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
