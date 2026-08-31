# Web Frameworks - Руководство для технического интервью

> 💡 **Как объяснить웹 фреймворки на интервью:**
> "Web framework — это скелет веб-приложения. Он берёт на себя рутину: маршрутизацию URL, обработку запросов, взаимодействие с базой, безопасность. Я пишу бизнес-логику, а фреймворк — инфраструктуру."

---

## 1. Request → Response цикл

### 🎯 Что спрашивают на интервью
> "Опишите путь HTTP запроса в веб-приложении"

### Простое объяснение

```
Пользователь → Nginx → Gunicorn → Django/FastAPI → База данных
                                        ↓
Пользователь ← Nginx ← Gunicorn ← Response ← ORM ←──┘
```

1. **Request приходит** от браузера
2. **Nginx** (reverse proxy) принимает, может отдать статику
3. **Gunicorn/Uvicorn** (WSGI/ASGI сервер) передаёт в приложение
4. **Routing** определяет какой handler/view обработает
5. **Middleware** обрабатывает до/после (логирование, auth)
6. **Handler/View** — ваша бизнес-логика
7. **ORM** — запросы к базе данных
8. **Response** возвращается обратно

### 📝 Фраза для интервью
> "Request проходит через reverse proxy, WSGI/ASGI сервер, middleware слой, попадает в view/handler по routing, выполняет логику (возможно с ORM), и response возвращается обратно через те же слои."

---

## 2. Routing и Handlers

### 🎯 Что спрашивают
> "Как работает маршрутизация?"

### Django

```python
# urls.py — централизованная маршрутизация
urlpatterns = [
    path('users/', views.user_list),
    path('users/<int:pk>/', views.user_detail),
    path('api/', include('api.urls')),  # Подключаем другой файл
]

# views.py
def user_detail(request, pk):
    user = get_object_or_404(User, pk=pk)
    return JsonResponse({'name': user.name})
```

### FastAPI

```python
# Декораторы прямо над функцией
@app.get("/users/{user_id}")
async def get_user(user_id: int):
    return {"user_id": user_id}

@app.post("/users/", status_code=201)
async def create_user(user: UserCreate):
    return {"name": user.name}
```

### Разница в подходах
- **Django**: URLconf отдельно от view, всё в одном месте
- **FastAPI**: маршруты как декораторы, рядом с кодом

### 📝 Фраза для интервью
> "Router сопоставляет URL с handler'ом. В Django маршруты централизованы в urls.py, во Flask/FastAPI — декораторы над функциями. Обе стратегии валидны, Django лучше для больших проектов, декораторы — для микросервисов."

---

## 3. ORM и Миграции

### 🎯 Что спрашивают
> "Что такое ORM и зачем нужны миграции?"

### ORM простым языком
Вместо SQL пишем Python:

```python
# SQL
cursor.execute("SELECT * FROM users WHERE age > 18")

# ORM (Django)
users = User.objects.filter(age__gt=18)

# ORM (SQLAlchemy)
users = session.query(User).filter(User.age > 18).all()
```

### Преимущества ORM
- ✅ Не пишем SQL вручную
- ✅ Защита от SQL injection (параметризация)
- ✅ Работаем с объектами Python
- ✅ Абстракция от конкретной СУБД

### Недостатки
- ❌ Скрытые запросы (N+1 problem)
- ❌ Сложные запросы проще писать на SQL
- ❌ Небольшой overhead

### Миграции — версионирование схемы

```bash
# Django
python manage.py makemigrations  # Создать миграцию
python manage.py migrate         # Применить

# Alembic (SQLAlchemy)
alembic revision --autogenerate -m "Add users"
alembic upgrade head
```

### N+1 Problem — частый вопрос!

```python
# ❌ N+1: 1 запрос на посты + N запросов на авторов
for post in Post.objects.all():
    print(post.author.name)  # Каждый раз запрос!

# ✅ 2 запроса с select_related (JOIN)
for post in Post.objects.select_related('author').all():
    print(post.author.name)  # Уже загружено
```

### Django Async ORM и Async Views (production-ready к 2026)

Раньше async в Django был "второсортным" — async view существовали (с Django 3.1), но ORM оставался синхронным, и внутри async view приходилось оборачивать ORM-вызовы в `sync_to_async`. К 2026 ситуация другая: Django последовательно добавил async-версии основных ORM-методов, и async стал реальным production-путём, а не экспериментом.

```python
from django.http import JsonResponse

# Async view
async def get_user(request, user_id: int):
    user = await User.objects.aget(id=user_id)                    # async ORM-метод
    orders = [o async for o in Order.objects.filter(user=user)]   # async iterator
    return JsonResponse({"name": user.name, "orders_count": len(orders)})

# Основные async-методы ORM: aget, acreate, asave, adelete,
# aupdate_or_create, abulk_create, aexists, acount, afirst, alast
```

Что важно на интервью:
- Django можно полностью развернуть на ASGI-сервере (Daphne, Uvicorn, Hypercorn) вместо WSGI/Gunicorn — тогда WebSocket, Server-Sent Events и async views работают нативно, без синхронной прослойки.
- Не весь ORM стал async "до конца" — часть сложных транзакционных сценариев и низкоуровневых операций всё ещё синхронные внутри, но набора async-методов достаточно для типичных CRUD-путей.
- Практический вывод: противопоставление "Django — синхронный, FastAPI — асинхронный" устарело. Выбор между ними сейчас чаще про экосистему (batteries-included ORM/admin у Django vs минимализм и API-first дизайн у FastAPI), а не про доступность async.

### 📝 Фраза для интервью
> "ORM мапит таблицы на классы Python. Преимущества: удобство, защита от SQL injection. Недостаток — скрытые запросы, особенно N+1 problem. Решается через select_related/prefetch_related. Миграции — это версионирование схемы БД, позволяющее откатываться. Отдельно стоит знать, что Django больше не чисто синхронный — async views и растущий набор async ORM-методов (aget, acreate, afilter) делают полностью ASGI-native Django рабочим production-вариантом, а не экспериментом."

---

## 4. Middleware

### 🎯 Что спрашивают
> "Что такое Middleware?"

### Простое объяснение
Middleware — это "слои лука" вокруг вашего handler:

```
Request → [Auth MW] → [Logging MW] → [CORS MW] → Handler
                                                    ↓
Response ← [Auth MW] ← [Logging MW] ← [CORS MW] ← Response
```

Каждый middleware может:
- Обработать request ДО handler
- Обработать response ПОСЛЕ handler
- Прервать цепочку (например, auth failed)

### Пример: логирование

```python
# Django
class LoggingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        print(f"[REQ] {request.method} {request.path}")
        response = self.get_response(request)
        print(f"[RES] {response.status_code}")
        return response

# FastAPI
@app.middleware("http")
async def log_requests(request: Request, call_next):
    print(f"[REQ] {request.method} {request.url}")
    response = await call_next(request)
    print(f"[RES] {response.status_code}")
    return response
```

### Типичные Middleware
- **Auth**: проверка токенов
- **CORS**: заголовки для cross-origin
- **Logging**: логирование запросов
- **Rate Limiting**: ограничение запросов
- **Compression**: gzip response

### 📝 Фраза для интервью
> "Middleware — это слой обработки до и после handler. Request проходит через цепочку middleware туда, response — обратно. Используется для cross-cutting concerns: логирование, аутентификация, CORS."

---

## 5. Валидация данных

### 🎯 Что спрашивают
> "Как валидировать входящие данные?"

### Pydantic (FastAPI) — современный подход

```python
from pydantic import BaseModel, EmailStr, Field, validator

class UserCreate(BaseModel):
    name: str = Field(..., min_length=2, max_length=50)
    email: EmailStr
    age: int = Field(..., ge=18, le=120)
    
    @validator('name')
    def name_must_be_capitalized(cls, v):
        return v.title()

# Автоматическая валидация
@app.post("/users/")
async def create_user(user: UserCreate):  # ← Pydantic валидирует
    return user
```

**Преимущества:**
- ✅ Декларативная валидация
- ✅ Автоматическая документация (OpenAPI)
- ✅ Type hints = самодокументирование

### Pydantic v2 — новый стандарт (Rust-ядро, breaking changes)

С выходом Pydantic v2 (ядро валидации переписано на Rust, `pydantic-core`) FastAPI и большая часть экосистемы перешли на v2 как стандарт — это одна из самых практических тем на интервью: "мигрировали ли вы с Pydantic v1 на v2, что сломалось?"

| v1 | v2 | Что изменилось |
|----|----|-----------------|
| `@validator` | `@field_validator` | Новый декоратор, другой набор аргументов (нужно явно указывать `mode="before"/"after"`) |
| `class Config:` | `model_config = ConfigDict(...)` | Конфигурация модели — атрибут, а не вложенный класс |
| `.dict()` | `.model_dump()` | Сериализация в dict |
| `.json()` | `.model_dump_json()` | Сериализация в JSON-строку |
| `orm_mode = True` | `from_attributes=True` | Чтение полей из ORM-объектов/произвольных атрибутов |
| `parse_obj()` | `model_validate()` | Валидация из dict/объекта |
| `__root__` | `RootModel` | Модели-обёртки над не-dict значением |

```python
# Pydantic v2
from pydantic import BaseModel, ConfigDict, field_validator

class UserCreate(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    name: str
    email: str

    @field_validator("name")
    @classmethod
    def name_must_be_capitalized(cls, v: str) -> str:
        return v.title()

user_dict = user.model_dump()          # было .dict()
user = UserCreate.model_validate(obj)  # было .parse_obj(obj)
```

Что важно на интервью:
- Главная практическая выгода v2 — заметный прирост производительности валидации/сериализации за счёт Rust-ядра, особенно чувствительный на hot path с большими моделями и списками.
- v1 и v2 могут какое-то время сосуществовать (compatibility-namespace `pydantic.v1`), но это временное решение на время миграции, а не стратегия на постоянку.
- Современный FastAPI рассчитан на Pydantic v2 — если в проекте всё ещё v1, это технический долг, который стоит явно проговорить на интервью, а не скрывать.

### Django Forms

```python
from django import forms

class UserForm(forms.Form):
    name = forms.CharField(max_length=100)
    email = forms.EmailField()
    
    def clean_name(self):
        name = self.cleaned_data['name']
        if name.lower() == 'admin':
            raise forms.ValidationError("Имя admin запрещено")
        return name
```

### 📝 Фраза для интервью
> "Валидация обязательна для безопасности. Pydantic в FastAPI — декларативно, через аннотации типов. Django Forms — через классы форм. Важно валидировать на сервере, не доверяя клиенту."

---

## 6. Authentication и Authorization

### 🎯 Что спрашивают
> "Разница между authentication и authorization?"

### Простое объяснение
- **Authentication** (WHO): "Кто ты?" — проверка личности
- **Authorization** (WHAT): "Что тебе можно?" — проверка прав

### JWT Authentication (популярный подход)

```python
# Login → получаем токен
@app.post("/login")
def login(credentials: LoginRequest):
    user = verify_password(credentials.username, credentials.password)
    if not user:
        raise HTTPException(401, "Неверные учётные данные")
    
    token = jwt.encode(
        {"sub": user.id, "exp": datetime.utcnow() + timedelta(hours=1)},
        SECRET_KEY
    )
    return {"access_token": token}

# Защищённый endpoint
@app.get("/profile")
def get_profile(token: str = Depends(oauth2_scheme)):
    payload = jwt.decode(token, SECRET_KEY)
    user_id = payload["sub"]
    return get_user(user_id)
```

### Безопасность — обязательные практики

```python
# ❌ SQL Injection
query = f"SELECT * FROM users WHERE name = '{name}'"

# ✅ Параметризованные запросы
cursor.execute("SELECT * FROM users WHERE name = %s", [name])

# ❌ Хранение паролей
user.password = password

# ✅ Хеширование
from passlib.hash import bcrypt
user.password_hash = bcrypt.hash(password)
```

### 📝 Фраза для интервью
> "Authentication — проверка личности (логин/пароль, JWT токен). Authorization — проверка прав (роли, permissions). JWT популярен для stateless API — токен содержит payload и подпись, сервер не хранит сессии."

---

## 7. Background Tasks

### 🎯 Что спрашивают
> "Как выполнять долгие операции?"

### Проблема
HTTP request должен ответить быстро (секунды). Что если нужно:
- Отправить 1000 email?
- Обработать видео?
- Сгенерировать отчёт?

### Celery — стандартное решение

```python
# tasks.py
from celery import Celery

app = Celery('tasks', broker='redis://localhost:6379')

@app.task
def send_email(to, subject, body):
    # Долгая операция
    smtp.send(to, subject, body)
    return "sent"

# Вызов из view
@app.post("/orders/")
def create_order(order: Order):
    save_order(order)
    send_email.delay(order.email, "Заказ создан", "...")  # Асинхронно!
    return {"status": "created"}
```

### FastAPI BackgroundTasks — для простых случаев

```python
from fastapi import BackgroundTasks

def send_notification(email: str):
    # Простая задача
    requests.post(webhook_url, json={"email": email})

@app.post("/users/")
def create_user(user: User, background_tasks: BackgroundTasks):
    save_user(user)
    background_tasks.add_task(send_notification, user.email)
    return {"status": "created"}  # Ответ сразу
```

### Когда что использовать
| Решение | Когда |
|---------|-------|
| FastAPI BackgroundTasks | Простые задачи, не критично |
| Celery | Надёжность, retry, мониторинг |
| RabbitMQ + consumer | Микросервисная архитектура |

### 📝 Фраза для интервью
> "Долгие операции нельзя делать в request-response цикле. Celery с Redis/RabbitMQ — стандарт для Python. Задача ставится в очередь, worker обрабатывает асинхронно. Есть retry, мониторинг, приоритеты."

---

## 8. Caching

### 🎯 Что спрашивают
> "Как ускорить веб-приложение?"

### Уровни кеширования

```
Browser Cache → CDN → Nginx → Redis → Database
                 ↑      ↑       ↑
              Статика  Page   Query
```

### Redis/Memcached кеширование

```python
# Django
from django.core.cache import cache

def get_user(user_id):
    key = f'user:{user_id}'
    user = cache.get(key)
    if not user:
        user = User.objects.get(id=user_id)
        cache.set(key, user, timeout=300)  # 5 минут
    return user

# Декоратор для view
from django.views.decorators.cache import cache_page

@cache_page(60 * 15)  # 15 минут
def product_list(request):
    products = Product.objects.all()
    return render(request, 'products.html', {'products': products})
```

### Инвалидация — сложная проблема

```python
def update_user(user_id, data):
    user = User.objects.get(id=user_id)
    user.name = data['name']
    user.save()
    cache.delete(f'user:{user_id}')  # Инвалидация!
```

### 📝 Фраза для интервью
> "Кеширование на нескольких уровнях: браузер, CDN, приложение (Redis), база (query cache). Главная проблема — инвалидация. При изменении данных кеш нужно очистить или обновить."

---

## 9. Deployment

### 🎯 Что спрашивают
> "Как развернуть Python веб-приложение?"

### WSGI vs ASGI
- **WSGI**: синхронный (Django, Flask)
- **ASGI**: асинхронный (FastAPI, Django 3+)

```bash
# WSGI (Gunicorn)
gunicorn myapp.wsgi:application -w 4 -b 0.0.0.0:8000

# ASGI (Uvicorn)
uvicorn main:app --workers 4 --host 0.0.0.0 --port 8000
```

### ASGI-экосистема: Starlette, Uvicorn/Hypercorn, Litestar

К 2026 ASGI — это не "экспериментальная альтернатива WSGI", а стандартный интерфейс для нового поколения Python веб-фреймворков:

- **Uvicorn** и **Hypercorn** — основные ASGI-серверы (аналог Gunicorn для WSGI); Hypercorn дополнительно поддерживает HTTP/2 "из коробки".
- **Starlette** — лёгкий ASGI-toolkit (роутинг, middleware, WebSocket, background tasks), на котором построен FastAPI: сам FastAPI добавляет поверх Starlette Pydantic-валидацию и автогенерацию OpenAPI. Полезно на интервью: вопрос "из чего состоит FastAPI" — это по сути "Starlette + Pydantic".
- **Litestar** — более новый ASGI-фреймворк, конкурент FastAPI: встроенный DI-контейнер, архитектура на плагинах, строгая типизация из коробки. Не обязательно знать его в деталях, но стоит знать, что он существует и решает те же задачи, что FastAPI, с иным архитектурным акцентом — DI и структура приложения заданы фреймворком, а не собираются через россыпь `Depends()`.

```bash
# Hypercorn — альтернатива Uvicorn с поддержкой HTTP/2
hypercorn main:app --workers 4 --bind 0.0.0.0:8000
```

### Типичный стек

```
Internet → Nginx → Gunicorn/Uvicorn → Django/FastAPI → PostgreSQL
              ↓                                            ↑
           Static                                       Redis
```

### Nginx конфигурация

```nginx
upstream app {
    server 127.0.0.1:8000;
}

server {
    listen 80;
    
    location /static/ {
        alias /app/static/;
    }
    
    location / {
        proxy_pass http://app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### 📝 Фраза для интервью
> "Nginx как reverse proxy принимает запросы, отдаёт статику, проксирует динамику на Gunicorn/Uvicorn. Gunicorn для WSGI приложений, Uvicorn для ASGI. Количество workers обычно 2-4 на CPU core."

---

## 10. FastAPI/ASGI как backend для AI-агентов и MCP-серверов

### 🎯 Что спрашивают
> "Где вы используете FastAPI в контексте AI/LLM-проектов?"

### Простое объяснение
Один из самых частых практических кейсов для senior backend-разработчика в 2026 — не "обычный" CRUD API, а backend для AI-агента: сервис, который проксирует запросы к LLM, стримит ответы клиенту, хранит память/сессии и выступает как **MCP-сервер** (Model Context Protocol) — оборачивает внутренние API/базы компании в стандартный интерфейс для LLM-агентов. Подробно про сам протокол MCP — в файле про Prompt/Context Engineering (раздел про Agents и Tool Use); здесь важно то, что FastAPI/ASGI стал де-факто стандартным транспортным слоем для такого сервера.

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

@app.post("/chat")
async def chat(message: str):
    async def event_stream():
        async for token in llm_stream(message):   # стримим токены от LLM
            yield f"data: {token}\n\n"
    return StreamingResponse(event_stream(), media_type="text/event-stream")

# MCP-сервер на Python (официальный SDK) поднимается поверх того же
# ASGI-стека — тот же Uvicorn/Starlette слой, что и у обычного REST API
```

Что важно на интервью:
- Async-native обработка запросов у FastAPI/ASGI хорошо ложится на природу LLM-вызовов — они I/O-bound (ждём сеть до провайдера модели), поэтому async даёт реальный выигрыш в конкурентности, а не только "модно".
- SSE (Server-Sent Events, подробнее — в файле про API Architecture) — стандартный способ стримить токены ответа LLM клиенту поверх обычного HTTP, без сложности WebSocket.
- MCP-сервер — это, по сути, ещё один ASGI-эндпоинт (или отдельный процесс, общающийся через stdio) — тот же набор знаний (роутинг, DI, валидация, ASGI-сервер), что и для обычного API, просто другой протокол поверх него.

### 📝 Фраза для интервью
> "В AI-проектах FastAPI обычно используется не просто как REST-обёртка над бизнес-логикой, а как backend для агента: стримит токены LLM клиенту через SSE, хранит контекст/память сессии, и часто сам выступает MCP-сервером — оборачивает внутренние API компании в протокол, который может вызывать любой MCP-совместимый агент. Async здесь не формальность — LLM-вызовы I/O-bound, и ASGI даёт реальную конкурентность на этом фоне."

---

## 11. HTMX и "HTML over the wire" — контр-тренд SPA

### 🎯 Что спрашивают
> "Обязательно ли для CRUD-приложения делать отдельный SPA-фронтенд на React/Vue?"

### Простое объяснение
Последние несколько лет заметен контр-тренд: не каждому приложению нужен полноценный SPA с отдельным frontend-стеком, стейт-менеджментом и build-пайплайном. **HTMX** (и близкие подходы вроде Hotwire/Turbo в Rails-мире) позволяют получать интерактивность уровня SPA, посылая с сервера **HTML-фрагменты**, а не JSON, и подменяя ими часть DOM — без написания отдельного frontend-приложения.

```html
<!-- Кнопка отправляет GET на /users/42/edit и подменяет div ответом-HTML -->
<div id="user-42">
  <button hx-get="/users/42/edit" hx-target="#user-42" hx-swap="outerHTML">
    Редактировать
  </button>
</div>
```

```python
# FastAPI/Django просто рендерят HTML-партиал в ответ на этот запрос,
# а не JSON — как для обычного серверного шаблона
@app.get("/users/{user_id}/edit", response_class=HTMLResponse)
async def edit_user_form(user_id: int, request: Request):
    user = await get_user(user_id)
    return templates.TemplateResponse("user_edit_partial.html", {"request": request, "user": user})
```

Когда это уместно:
- ✅ CRUD-тяжёлые внутренние инструменты, админки, dashboard'ы — где основная работа фронтенда это "показать данные и форму", а не богатый клиентский стейт.
- ✅ Команда без отдельных frontend-инженеров — экономит на поддержке двух стеков (API-контракт + отдельный SPA).
- ❌ Приложения со сложным клиентским состоянием, офлайн-режимом, богатой интерактивностью (редакторы, канвасы, real-time коллаборация) — там SPA/полноценный frontend-фреймворк всё ещё оправдан.

### 📝 Фраза для интервью
> "HTMX — это не замена React/Vue, а альтернатива для случаев, где полноценный SPA — overkill. Сервер отдаёт готовые HTML-фрагменты вместо JSON, а HTMX подменяет ими часть страницы — получаем SPA-подобную интерактивность без отдельного frontend-стека и build-пайплайна. Хороший пример того, что архитектурное решение должно исходить из сложности UI-состояния, а не из 'все сейчас делают SPA'."

---

## 🎤 Частые вопросы на интервью

### "Django vs FastAPI?"
> "Django — batteries included, полный стек (ORM, admin, auth). FastAPI — минималистичный, async, автодокументация OpenAPI. Django для full-stack, FastAPI для API-only микросервисов."

### "Что такое N+1 problem?"
> "Когда ORM делает 1 запрос на коллекцию и N запросов на связанные объекты. Решается through eager loading: select_related в Django (JOIN), prefetch_related (отдельный запрос)."

### "Как масштабировать веб-приложение?"
> "Горизонтально: больше инстансов за load balancer. Кеширование на всех уровнях. Асинхронные задачи через Celery. Read replicas для базы. CDN для статики."

### "Что такое CSRF?"
> "Cross-Site Request Forgery — когда злоумышленник заставляет браузер пользователя отправить запрос на ваш сайт. Защита: CSRF токен в формах, который проверяется на сервере."

### "REST vs GraphQL?"
> "REST: ресурсо-ориентированный, фиксированные endpoints, отдельный запрос на каждый ресурс. GraphQL: запрашиваем только нужные поля, один endpoint, сложнее кешировать."

### "Что такое WSGI и ASGI?"
> "WSGI — синхронный интерфейс между веб-сервером и Python приложением. ASGI — асинхронный, поддерживает WebSocket и long-polling. Django использует WSGI (ASGI с 3+), FastAPI — ASGI."

### "Как работает JWT токен?"
> "JSON Web Token состоит из header, payload и signature. Сервер генерирует токен с секретным ключом, клиент отправляет его в каждом запросе. Сервер проверяет подпись, не храня сессию."

### "Что такое XSS?"
> "Cross-Site Scripting — внедрение вредоносного JS кода на страницу. Защита: экранирование вывода, Content Security Policy, HttpOnly cookies."

### "Синхронный vs асинхронный Python?"
> "Синхронный блокирует на I/O операциях. Async/await позволяет обрабатывать другие запросы во время ожидания. FastAPI async нативно, Django — с ограничениями."

### "Что такое Dependency Injection?"
> "Паттерн, когда зависимости передаются извне, а не создаются внутри. FastAPI Depends() — отличный пример. Упрощает тестирование и модульность."

### "Как реализовать rate limiting?"
> "Ограничение количества запросов. На уровне Nginx (limit_req), middleware (django-ratelimit), или Redis (счётчики с TTL). Можно по IP, по user, по endpoint."

### "Что такое OpenAPI/Swagger?"
> "Спецификация для описания REST API. FastAPI генерирует автоматически из типов Pydantic. Даёт интерактивную документацию и генерацию клиентов."

### "Как обрабатывать ошибки в API?"
> "Централизованный exception handler. Возвращать структурированные ошибки с кодом, сообщением, деталями. HTTP статусы: 4xx для клиентских, 5xx для серверных."

### "Django ORM vs SQLAlchemy?"
> "Django ORM проще, тесно интегрирован с Django. SQLAlchemy мощнее, гибче, два паттерна (Core и ORM). SQLAlchemy для сложных запросов и non-Django проектов."

### "Что такое CORS и как настроить?"
> "Cross-Origin Resource Sharing — механизм разрешения запросов с другого домена. Настраивается через заголовки Access-Control-Allow-Origin. Django-cors-headers, FastAPI CORSMiddleware."

### "Как тестировать веб-приложение?"
> "Unit тесты для логики, интеграционные для API (pytest + TestClient), e2e для полного flow (Selenium, Playwright). Fixtures для данных, моки для внешних сервисов."

### "Что такое Gunicorn workers?"
> "Рабочие процессы, обрабатывающие запросы параллельно. Sync workers для WSGI, аsync для ASGI. Формула: 2 * CPU + 1. Слишком много — overhead на память."

### "Как логировать в production?"
> "Структурированные логи (JSON), уровни (DEBUG, INFO, ERROR), корреляционные ID для трейсинга. ELK stack или cloud logging (CloudWatch, Stackdriver)."

### "Что такое сессии и как они работают?"
> "Хранение состояния пользователя между запросами. Session ID в cookie, данные на сервере (Redis, DB). Stateless альтернатива — JWT в каждом запросе."

### "Pydantic v1 vs v2 — в чём разница?"
> "v2 переписал ядро валидации на Rust (pydantic-core) — заметный прирост производительности. Поменялось API: `@validator` → `@field_validator`, `class Config` → `model_config = ConfigDict()`, `.dict()`/`.json()` → `.model_dump()`/`.model_dump_json()`, `orm_mode` → `from_attributes`. Современный FastAPI рассчитан на v2, v1 — устаревшая база с временным compatibility-слоем `pydantic.v1` для миграции."

### "Django — только синхронный фреймворк?"
> "Уже нет. С Django 3.1+ есть async views, а ORM постепенно получил async-методы (aget, acreate, afilter и т.д.) — можно полностью развернуть Django на ASGI-сервере. Это не так гибко, как FastAPI, где async — по умолчанию, но для типичных CRUD-путей async Django — рабочий production-вариант, а не эксперимент."

### "Что такое ASGI и чем Starlette отличается от FastAPI?"
> "ASGI — асинхронный стандарт интерфейса между сервером и приложением, преемник WSGI, поддерживает WebSocket/SSE/long-lived соединения. Starlette — минималистичный ASGI-toolkit (роутинг, middleware, WebSocket), на котором построен FastAPI: FastAPI добавляет поверх Starlette Pydantic-валидацию и автогенерацию OpenAPI-документации."

### "Слышали про Litestar?"
> "Да, это ещё один ASGI-фреймворк, конкурент FastAPI — со встроенным DI-контейнером и архитектурой на плагинах из коробки. Не такой распространённый, как FastAPI, но стоит знать, что он есть и решает похожие задачи с несколько иным акцентом на строгую структуру приложения."

### "Где вы используете FastAPI в AI/LLM-проектах?"
> "Как backend для агента: стримит ответы LLM клиенту через SSE, управляет памятью/сессией диалога, часто сам является MCP-сервером — оборачивает внутренние API компании в протокол, который могут вызывать LLM-агенты. Async здесь даёт реальную выгоду, потому что вызовы к LLM — I/O-bound."

### "Что такое HTMX и когда его использовать вместо SPA?"
> "HTMX — библиотека, которая позволяет серверу отвечать HTML-фрагментами и подменять ими часть DOM по атрибутам (hx-get, hx-post, hx-swap), без отдельного frontend-фреймворка. Хорошо подходит для CRUD-тяжёлых внутренних инструментов и админок; для приложений со сложным клиентским состоянием (редакторы, real-time коллаборация) SPA всё ещё оправдан."

---

## 📚 Чек-лист для подготовки

### Базовый уровень
- [ ] Request → Response цикл, роль Nginx/Gunicorn/Uvicorn
- [ ] Routing и handlers (Django urls.py vs FastAPI-декораторы)
- [ ] ORM basics, миграции
- [ ] Middleware
- [ ] Валидация данных (Pydantic, Django Forms)

### Средний уровень
- [ ] N+1 problem и его решение
- [ ] Authentication vs Authorization, JWT
- [ ] Background tasks (Celery, BackgroundTasks)
- [ ] Кеширование и инвалидация
- [ ] WSGI vs ASGI, деплой (Nginx + Gunicorn/Uvicorn)

### Продвинутый уровень
- [ ] Pydantic v2: breaking changes, миграция с v1
- [ ] Django async views и async ORM (aget/acreate/afilter)
- [ ] ASGI-экосистема: Starlette как база FastAPI, Litestar как альтернатива
- [ ] FastAPI/ASGI backends для AI-агентов и MCP-серверов
- [ ] HTMX и "HTML over the wire" как альтернатива SPA для CRUD-приложений
- [ ] Rate limiting, CORS, CSRF/XSS защита

