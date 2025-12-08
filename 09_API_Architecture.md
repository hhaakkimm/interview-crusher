# API Design & Architecture - Руководство для Senior Backend интервью

> 💡 **Как думать об архитектуре на интервью:**
> "Хорошая архитектура — это баланс между гибкостью и простотой. Начинаю с простого монолита, выделяю сервисы по мере роста. Главное — loose coupling, high cohesion."

---

## 1. REST API Best Practices

### 🎯 Что спрашивают
> "Как спроектировать хороший REST API?"

### Naming Conventions
```http
# ✅ Правильно: существительные, множественное число
GET    /users           # Список пользователей
GET    /users/{id}      # Конкретный пользователь
POST   /users           # Создать
PUT    /users/{id}      # Полное обновление
PATCH  /users/{id}      # Частичное обновление
DELETE /users/{id}      # Удалить

# Вложенные ресурсы
GET    /users/{id}/orders
POST   /users/{id}/orders

# ❌ Неправильно
GET /getUsers
POST /createUser
GET /user/list
POST /users/create
```

### HTTP Status Codes
```
2xx Success:
  200 OK              — Успешный GET/PUT/PATCH
  201 Created         — Успешный POST
  204 No Content      — Успешный DELETE

4xx Client Error:
  400 Bad Request     — Невалидные данные
  401 Unauthorized    — Не аутентифицирован
  403 Forbidden       — Нет прав
  404 Not Found       — Ресурс не найден
  409 Conflict        — Конфликт (duplicate)
  422 Unprocessable   — Валидация не прошла
  429 Too Many Requests — Rate limit

5xx Server Error:
  500 Internal Error  — Ошибка сервера
  502 Bad Gateway     — Upstream error
  503 Service Unavailable — Временно недоступен
```

### Pagination
```http
GET /users?page=2&per_page=20

# Response
{
    "data": [...],
    "pagination": {
        "page": 2,
        "per_page": 20,
        "total": 150,
        "total_pages": 8
    },
    "links": {
        "self": "/users?page=2",
        "first": "/users?page=1",
        "prev": "/users?page=1",
        "next": "/users?page=3",
        "last": "/users?page=8"
    }
}
```

### Versioning
```http
# URL versioning (рекомендуется)
GET /api/v1/users
GET /api/v2/users

# Header versioning
GET /users
Accept: application/vnd.api+json; version=2

# Query param
GET /users?version=2
```

### 📝 Фраза для интервью
> "REST — ресурсо-ориентированный стиль. Правильные HTTP методы и статус коды. Pagination для больших списков. Versioning через URL prefix для backward compatibility."

---

## 2. Error Handling

### 🎯 Что спрашивают
> "Как обрабатывать ошибки в API?"

### Структурированные ошибки
```json
{
    "error": {
        "code": "VALIDATION_ERROR",
        "message": "Validation failed",
        "details": [
            {
                "field": "email",
                "message": "Invalid email format"
            },
            {
                "field": "age",
                "message": "Must be at least 18"
            }
        ],
        "request_id": "req_abc123",
        "documentation_url": "https://api.example.com/docs/errors#VALIDATION_ERROR"
    }
}
```

### Централизованная обработка (FastAPI)
```python
from fastapi import FastAPI, HTTPException
from fastapi.exceptions import RequestValidationError

app = FastAPI()

class ApiError(Exception):
    def __init__(self, code: str, message: str, status: int = 400):
        self.code = code
        self.message = message
        self.status = status

@app.exception_handler(ApiError)
async def api_error_handler(request, exc: ApiError):
    return JSONResponse(
        status_code=exc.status,
        content={
            "error": {
                "code": exc.code,
                "message": exc.message,
                "request_id": request.state.request_id
            }
        }
    )

@app.exception_handler(RequestValidationError)
async def validation_handler(request, exc):
    return JSONResponse(
        status_code=422,
        content={
            "error": {
                "code": "VALIDATION_ERROR",
                "message": "Invalid request",
                "details": exc.errors()
            }
        }
    )
```

### 📝 Фраза для интервью
> "Централизованный exception handler для консистентных ответов. Структурированные ошибки с кодом, сообщением, деталями. request_id для трейсинга. Не показываем stack trace в production."

---

## 3. Idempotency

### 🎯 Что спрашивают
> "Что такое идемпотентность и зачем она нужна?"

### Простое объяснение
Идемпотентная операция при повторном вызове даёт тот же результат.

```
GET, PUT, DELETE — идемпотентны по определению
POST — НЕ идемпотентен (создаёт новый ресурс каждый раз)
```

### Проблема
```
Client → POST /payments → Server (success)
       ← ... network timeout ...
Client → POST /payments → Server (duplicate payment!)
```

### Решение: Idempotency Key
```python
@app.post("/payments")
async def create_payment(
    payment: PaymentCreate,
    idempotency_key: str = Header(...)
):
    # Проверяем, была ли уже такая операция
    existing = await redis.get(f"idempotency:{idempotency_key}")
    if existing:
        return json.loads(existing)  # Возвращаем сохранённый результат
    
    # Выполняем операцию
    result = await process_payment(payment)
    
    # Сохраняем результат
    await redis.setex(
        f"idempotency:{idempotency_key}",
        86400,  # 24 часа
        json.dumps(result)
    )
    
    return result
```

### 📝 Фраза для интервью
> "Idempotency-Key header для защиты от дублирования при retry. Клиент генерирует UUID, сервер кеширует результат. Критично для payments, создания ресурсов."

---

## 4. gRPC vs REST vs GraphQL

### 🎯 Что спрашивают
> "Когда использовать gRPC/GraphQL вместо REST?"

### Сравнение

| Aspect | REST | gRPC | GraphQL |
|--------|------|------|---------|
| Protocol | HTTP/JSON | HTTP/2 + Protobuf | HTTP/JSON |
| Performance | Medium | High | Medium |
| Schema | OpenAPI (optional) | Protobuf (required) | GraphQL SDL |
| Streaming | No (SSE/WS) | Yes, built-in | Subscriptions |
| Browser | Native | Needs proxy | Native |
| Use case | Public APIs | Microservices | Mobile, complex queries |

### gRPC для микросервисов
```protobuf
// user.proto
service UserService {
    rpc GetUser(GetUserRequest) returns (User);
    rpc ListUsers(ListUsersRequest) returns (stream User);
}

message User {
    int32 id = 1;
    string name = 2;
    string email = 3;
}
```

### GraphQL для гибких запросов
```graphql
# Клиент запрашивает только нужные поля
query {
    user(id: 123) {
        name
        orders(limit: 5) {
            id
            total
        }
    }
}
```

### 📝 Фраза для интервью
> "REST для public APIs — простой, понятный. gRPC для internal microservices — быстрый, строгие контракты, streaming. GraphQL когда клиенту нужна гибкость в запросах — mobile apps, dashboards."

---

## 5. Microservices vs Monolith

### 🎯 Что спрашивают
> "Когда переходить на микросервисы?"

### Monolith First
```
Monolith: Один deployable unit
├── Simple to develop
├── Simple to deploy
├── Simple to debug
└── Becomes complex at scale

Microservices: Many deployable units
├── Independent scaling
├── Technology freedom
├── Team autonomy
└── Operational complexity
```

### Когда микросервисы имеют смысл
- Команда > 50 человек
- Разные части нужно масштабировать независимо
- Разные requirements по tech stack
- Deployment независимости критична

### Проблемы микросервисов
- **Network latency** между сервисами
- **Distributed transactions** сложны
- **Debugging** труднее
- **Data consistency** — eventual
- **Operational overhead** — мониторинг, деплой

### 📝 Фраза для интервью
> "Start with monolith, extract services по мере роста. Микросервисы — когда команда большая, нужна независимость деплоя, разное масштабирование. Цена: operational complexity, distributed debugging."

---

## 6. Event-Driven Architecture

### 🎯 Что спрашивают
> "Что такое event-driven архитектура?"

### Patterns

#### Event Notification
```python
# Order Service публикует event
event = {
    "type": "order.created",
    "data": {"order_id": 123, "user_id": 456}
}
kafka.publish("orders", event)

# Email Service подписан
@kafka.subscribe("orders")
def handle_order_created(event):
    if event["type"] == "order.created":
        send_confirmation_email(event["data"]["user_id"])
```

#### Event Sourcing
```python
# Храним события, не состояние
events = [
    {"type": "account.created", "data": {"balance": 0}},
    {"type": "money.deposited", "data": {"amount": 100}},
    {"type": "money.withdrawn", "data": {"amount": 30}},
]

# Восстанавливаем состояние из событий
def get_balance(events):
    balance = 0
    for event in events:
        if event["type"] == "money.deposited":
            balance += event["data"]["amount"]
        elif event["type"] == "money.withdrawn":
            balance -= event["data"]["amount"]
    return balance  # 70
```

#### CQRS (Command Query Responsibility Segregation)
```
Commands (Write) → Write Model → Event Store
                                      ↓
Queries (Read) ← Read Model ← Projections
```

### 📝 Фраза для интервью
> "Event-driven для loose coupling — сервисы общаются через события, не зная друг о друге. Event Sourcing хранит историю изменений. CQRS разделяет модели чтения и записи для оптимизации."

---

## 7. API Gateway Pattern

### 🎯 Что спрашивают
> "Зачем нужен API Gateway?"

### Responsibilities
```
                    ┌─────────────────────────────────┐
Client ────────────▶│          API Gateway            │
                    │  • Authentication               │
                    │  • Rate Limiting                │
                    │  • Request Routing              │
                    │  • Load Balancing               │
                    │  • Request/Response Transform   │
                    │  • Caching                      │
                    │  • Monitoring                   │
                    └─────────────────┬───────────────┘
                           ┌──────────┼──────────┐
                           ▼          ▼          ▼
                       ┌──────┐  ┌──────┐   ┌──────┐
                       │Users │  │Orders│   │Products│
                       └──────┘  └──────┘   └──────┘
```

### Backend for Frontend (BFF)
```
Mobile App ────▶ Mobile BFF ────▶ Services
Web App    ────▶ Web BFF    ────▶ Services
Partner    ────▶ Partner BFF ────▶ Services
```

### 📝 Фраза для интервью
> "API Gateway — единая точка входа. Централизует auth, rate limiting, routing. BFF pattern когда разные клиенты нужны разные API форматы — mobile оптимизирован для bandwidth, web для features."

---

## 8. Database per Service

### 🎯 Что спрашивают
> "Как управлять данными в микросервисах?"

### Проблема
```
Service A ──┐
            ├──▶ Shared Database ◀──┬── Service B
Service C ──┘                       │
                                    └── Tight coupling!
```

### Решение: Database per Service
```
Service A ──▶ Database A
Service B ──▶ Database B
Service C ──▶ Database C
```

### Как синхронизировать данные?

#### 1. Event-driven sync
```python
# Order Service
def create_order(order):
    db.insert(order)
    kafka.publish("orders", {
        "type": "order.created",
        "data": order
    })

# Analytics Service слушает и обновляет свою базу
@kafka.subscribe("orders")
def handle_order(event):
    analytics_db.insert_order_stats(event["data"])
```

#### 2. Saga для distributed transactions
```python
# Choreography Saga
Order Service → "order.created"
                    ↓
Payment Service listens → "payment.processed"
                              ↓
Inventory Service listens → "inventory.reserved"
                                ↓
Shipping Service listens → "shipment.created"
```

### 📝 Фраза для интервью
> "Database per service для loose coupling — каждый сервис владеет своими данными. Синхронизация через events. Distributed transactions через Saga pattern с compensation при ошибках."

---

## 🎤 Частые вопросы Architecture

### "Как обеспечить backward compatibility в API?"
> "URL versioning (/v1/, /v2/). Добавление полей — не breaking. Deprecation policy: предупреждаем, даём время (6 months минимум), потом удаляем."

### "Что такое strangler pattern?"
> "Постепенная миграция с монолита на микросервисы. Новая функциональность — в новых сервисах. Старая — постепенно переписывается. Proxy маршрутизирует запросы."

### "Service discovery — как сервисы находят друг друга?"
> "Consul, Eureka, или Kubernetes DNS. Сервис регистрируется при старте. Клиент запрашивает адреса у registry. Health checks для актуальности."

### "Как обрабатывать failures между сервисами?"
> "Circuit breaker для fast-fail. Retry с exponential backoff. Timeout на каждый call. Fallback на cached data или degraded response."

### "Что такое sidecar pattern?"
> "Вспомогательный контейнер рядом с основным. Envoy для service mesh — обрабатывает networking, observability, security. Приложение не знает о инфраструктуре."
