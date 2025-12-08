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

### 📝 Фраза для интервью
> "ORM мапит таблицы на классы Python. Преимущества: удобство, защита от SQL injection. Недостаток — скрытые запросы, особенно N+1 problem. Решается через select_related/prefetch_related. Миграции — это версионирование схемы БД, позволяющее откатываться."

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

