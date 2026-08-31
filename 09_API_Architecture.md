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

### OpenAPI 3.1 и JSON Schema

OpenAPI 3.0 (многолетний стандарт для REST-документации) использовал собственный, слегка урезанный диалект JSON Schema — с нестыковками вроде `nullable: true` вместо union-типа, и не полным набором ключевых слов JSON Schema. **OpenAPI 3.1** полностью выровнял схемы с JSON Schema (draft 2020-12) — теперь schema-секция OpenAPI-документа это валидный JSON Schema, который можно переиспользовать напрямую в других инструментах (валидаторы, генераторы моков, codegen), без "переводчика" между двумя похожими, но не идентичными форматами.

```yaml
# OpenAPI 3.1 — nullable через стандартный JSON Schema union-тип
properties:
  middle_name:
    type: ["string", "null"]   # вместо OpenAPI 3.0-специфичного nullable: true
```

На интервью стоит знать: современные генераторы OpenAPI-спецификации (включая FastAPI) уже используют 3.1, и вопрос "чем 3.1 отличается от 3.0" — это по сути вопрос про выравнивание с JSON Schema, а не про новые фичи REST как таковые.

### 📝 Фраза для интервью
> "REST — ресурсо-ориентированный стиль. Правильные HTTP методы и статус коды. Pagination для больших списков. Versioning через URL prefix для backward compatibility. Документирую через OpenAPI — начиная с версии 3.1 её схемы полностью выровнены с JSON Schema, что упрощает переиспользование схем в других инструментах."

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

### GraphQL at scale: Apollo Federation

Одна монолитная GraphQL-схема на весь бэкенд плохо масштабируется на организацию из многих команд — она становится единой точкой конфликтов при каждом изменении. **GraphQL Federation** (паттерн, популяризированный Apollo) решает это разбиением графа на **subgraphs**, каждый из которых владеет своей частью схемы и своим сервисом, а gateway на лету компонует их в единый граф для клиента.

```graphql
# Subgraph "users" — владеет типом User
type User @key(fields: "id") {
  id: ID!
  name: String!
}

# Subgraph "orders" — расширяет User своим полем, не трогая users-сервис
extend type User @key(fields: "id") {
  id: ID! @external
  orders: [Order!]!
}
```

Gateway маршрутизирует части запроса к нужным subgraph-сервисам и склеивает ответ — с точки зрения клиента это по-прежнему один GraphQL endpoint. Это стандартный ответ на вопрос "как GraphQL используется в компании с десятками команд", а не "у нас один огромный резолвер на всё".

### tRPC — типобезопасность без схемы (full-stack TypeScript)

Для команд, где и фронтенд, и бэкенд на TypeScript в одном репозитории (monorepo), **tRPC** предлагает третий путь помимо REST/GraphQL: вызываешь backend-процедуры с фронтенда как обычные типизированные функции, без отдельного схема-слоя (OpenAPI/GraphQL SDL) и без codegen-шага — типы выводятся напрямую из серверного кода через TypeScript generics.

```typescript
// Сервер
const appRouter = router({
  getUser: publicProcedure.input(z.number()).query(({ input }) => getUser(input)),
});

// Клиент — типы user выведены автоматически, без генерации .d.ts
const user = await trpc.getUser.query(42);
```

Компромисс: это работает только когда весь стек на TypeScript и в одном репозитории (или тесно связанных пакетах) — в отличие от REST/GraphQL/gRPC, tRPC не даёт языконезависимого контракта для внешних/мобильных/сторонних клиентов. На интервью достаточно знать, что это существует и для какой ниши создано — не для публичных или полиглотных API.

### MCP — новая парадигма API для AI-агентов (кратко)

Отдельная категория, которая появилась в 2024-2025 и не вписывается в REST/GraphQL/gRPC — **MCP (Model Context Protocol)**. Если REST/GraphQL/gRPC проектируются для людей и сервисов, вызывающих API по заранее известному контракту, то MCP — протокол именно для того, чтобы **LLM-агент** мог обнаружить и вызвать инструменты/данные сервера динамически, во время рассуждения. Подробный разбор протокола (архитектура, tools/resources/prompts, security-риски вроде indirect prompt injection) — в файле про Prompt Engineering, раздел про Agents и Tool Use; здесь важно зафиксировать для интервью, что это отдельный, четвёртый класс API-контракта — не замена REST/GraphQL/gRPC, а протокол над ними для agentic use case.

### 📝 Фраза для интервью
> "REST для public APIs — простой, понятный. gRPC для internal microservices — быстрый, строгие контракты, streaming. GraphQL когда клиенту нужна гибкость в запросах — mobile apps, dashboards; на масштабе многих команд — не монолитная схема, а Apollo Federation с разбиением на subgraphs. tRPC знаю как нишевый вариант для full-stack TypeScript монорепозиториев — типобезопасность без схемы и codegen, но не для полиглотных/внешних клиентов. А MCP — вообще отдельная категория, не замена этим протоколам: контракт для того, чтобы LLM-агент мог динамически обнаруживать и вызывать инструменты сервера, а не человек/сервис по заранее известному API."

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

### AsyncAPI — контракт для событийных API

Для REST есть OpenAPI, для событийных систем (Kafka, RabbitMQ, MQTT) — аналогичный по духу стандарт **AsyncAPI**: декларативное описание топиков/очередей, форматов сообщений (payload schema), кто publisher, кто consumer. Решает ту же проблему, что OpenAPI для REST — документация, которая не расходится с реальностью, и возможность сгенерировать клиентский код/валидаторы из спецификации.

```yaml
# asyncapi.yaml (упрощённо)
channels:
  order.created:
    publish:
      message:
        payload:
          type: object
          properties:
            order_id: { type: integer }
            user_id: { type: integer }
```

На практике AsyncAPI распространён заметно меньше, чем OpenAPI — многие команды до сих пор документируют события неформально (README, комментарии в коде). На интервью полезно знать сам факт существования стандарта и что он покрывает: если спросят "как вы документируете контракты Kafka-топиков", ответ "у нас AsyncAPI-спецификация как единый источник правды" — сильный ответ.

### 📝 Фраза для интервью
> "Event-driven для loose coupling — сервисы общаются через события, не зная друг о друге. Event Sourcing хранит историю изменений. CQRS разделяет модели чтения и записи для оптимизации. Контракты событий (топики, схемы сообщений) документирую через AsyncAPI — это тот же принцип, что OpenAPI для REST, но для событийных систем вроде Kafka/RabbitMQ."

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

## 9. Real-time и Streaming APIs: SSE, WebSockets, gRPC Streaming

### 🎯 Что спрашивают
> "Как стримить данные клиенту в реальном времени? Чем Server-Sent Events отличается от WebSocket?"

### Простое объяснение
К 2026 в API-архитектуре сосуществуют несколько способов доставки данных клиенту не по принципу "один запрос — один ответ", и выбор между ними — частый архитектурный вопрос, особенно с ростом LLM/AI-продуктов, которым нужно стримить токены ответа по мере генерации.

| Подход | Направление | Транспорт | Когда использовать |
|--------|-------------|-----------|---------------------|
| **REST polling** | Клиент → сервер (периодически) | обычный HTTP | Просто, но задержка = интервал опроса; лишняя нагрузка на сервер |
| **Long polling** | Клиент → сервер (держит соединение) | обычный HTTP | Переходный вариант там, где WebSocket/SSE недоступны (legacy-инфра, строгие прокси) |
| **WebSockets** | Двунаправленно | свой протокол поверх TCP (upgrade от HTTP) | Chat, совместное редактирование, игры — когда клиент тоже должен часто слать данные |
| **SSE (Server-Sent Events)** | Только сервер → клиент | обычный HTTP (`text/event-stream`) | **Стандарт для стриминга ответов LLM/AI** — токен за токеном, простое переподключение, работает через обычную HTTP-инфраструктуру/прокси/CDN |
| **gRPC streaming** | Однонаправленно или двунаправленно | HTTP/2 + Protobuf | Внутренний межсервисный стриминг с типизированным контрактом (не для браузера напрямую) |

### SSE — почему это стандарт для LLM-стриминга

```python
from fastapi.responses import StreamingResponse

@app.post("/chat")
async def chat(message: str):
    async def event_stream():
        async for token in llm.stream(message):
            yield f"data: {token}\n\n"       # формат SSE: "data: <payload>\n\n"
        yield "event: done\ndata: {}\n\n"
    return StreamingResponse(event_stream(), media_type="text/event-stream")
```

```javascript
// Клиент — встроенный браузерный EventSource, без сторонних библиотек
const es = new EventSource("/chat");
es.onmessage = (e) => appendToken(e.data);
```

Почему именно SSE, а не WebSocket, стал стандартом для LLM-ответов (OpenAI, Anthropic и другие API стримят именно так):
- Ответ модели течёт только в одну сторону (сервер → клиент) — двунаправленность WebSocket тут не нужна.
- SSE работает поверх обычного HTTP/1.1 — не нужен отдельный upgrade-протокол, легче проходит через существующие прокси, балансировщики, corporate firewalls.
- Встроенная в браузер автоматическая переподписка при разрыве соединения (`EventSource`), без ручной реализации reconnect-логики.
- Проще на бэкенде: обычный HTTP-хендлер с потоковым ответом, а не отдельная WebSocket-инфраструктура.

### 📝 Фраза для интервью
> "Для чисто одностороннего стриминга сервер → клиент — особенно для ответов LLM — использую SSE: он проще WebSocket, работает поверх обычного HTTP, и клиенту не нужна сторонняя библиотека. WebSocket оставляю для действительно двунаправленных сценариев — чат, совместное редактирование, игры. gRPC streaming — для внутреннего межсервисного стриминга с типизированным протобаф-контрактом, не для браузера напрямую. Polling — только когда ничего из вышеперечисленного не доступно по инфраструктурным причинам."

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

### "Чем SSE отличается от WebSocket?"
> "SSE — однонаправленный поток сервер → клиент поверх обычного HTTP, с автопереподключением из коробки (EventSource). WebSocket — полнодуплексный протокол, нужен для случаев, когда клиент тоже часто шлёт данные (чат, совместное редактирование). Стриминг LLM-ответов — классический SSE-кейс, потому что данные текут только в одну сторону."

### "Что нового в OpenAPI 3.1?"
> "Полное выравнивание с JSON Schema (draft 2020-12) — до этого OpenAPI 3.0 использовал собственный урезанный диалект со своими нестыковками (например, `nullable: true` вместо union-типа). Теперь schema-часть спецификации — валидный JSON Schema, который можно переиспользовать в других инструментах без адаптации."

### "Что такое AsyncAPI?"
> "Аналог OpenAPI для событийных систем — декларативное описание топиков/очередей Kafka/RabbitMQ и схем сообщений. Решает ту же проблему: контракт, который не расходится с реальностью, и возможность сгенерировать код/валидацию из спецификации."

### "Как масштабировать GraphQL на много команд?"
> "GraphQL Federation (Apollo-style): схема разбивается на subgraphs, каждым владеет своя команда/сервис, gateway на лету компонует их в единый граф для клиента. Так избегаем единой монолитной схемы, которая становится точкой конфликтов при росте организации."

### "Слышали про tRPC?"
> "Да — способ получить end-to-end типобезопасность между TypeScript-фронтендом и бэкендом без отдельного схема-слоя (OpenAPI/GraphQL) и без codegen: типы выводятся напрямую из серверного кода. Работает только когда весь стек на TS в одном репозитории — не подходит для публичных или полиглотных API."

### "Что такое MCP и как он соотносится с REST/GraphQL/gRPC?"
> "MCP (Model Context Protocol) — не замена REST/GraphQL/gRPC, а отдельная категория API-контракта для agentic use case: он даёт LLM-агенту возможность динамически обнаруживать и вызывать инструменты/данные сервера во время рассуждения, а не по заранее зашитому контракту, как в классическом API-вызове. Подробно — тема Prompt/Context Engineering и tool use."

---

## 📚 Чек-лист для подготовки

### Базовый уровень
- [ ] REST naming conventions, HTTP статус-коды, pagination, versioning
- [ ] Структурированная обработка ошибок
- [ ] Idempotency и idempotency key

### Средний уровень
- [ ] REST vs gRPC vs GraphQL — когда что выбрать
- [ ] Microservices vs monolith, trade-offs
- [ ] Event-driven архитектура, Event Sourcing, CQRS
- [ ] API Gateway, BFF pattern
- [ ] Database per service, Saga pattern

### Продвинутый уровень
- [ ] OpenAPI 3.1 и выравнивание с JSON Schema
- [ ] AsyncAPI для событийных контрактов
- [ ] GraphQL Federation (Apollo-style subgraphs)
- [ ] tRPC — типобезопасность без схемы для full-stack TS
- [ ] SSE vs WebSocket vs gRPC streaming vs polling — decision framework
- [ ] SSE как стандарт стриминга LLM-ответов
- [ ] MCP как новая парадигма API для AI-агентов (в общих чертах)
