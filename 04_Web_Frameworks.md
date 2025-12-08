# Web Frameworks - Полное руководство

## Содержание
1. [Application Settings, Exceptions](#application-settings-exceptions)
2. [Request, Response, Routing, Views, Handlers/Controllers](#request-response-routing)
3. [Database, ORM, Migrations](#database-orm-migrations)
4. [Templates (Django, Jinja2)](#templates)
5. [Listeners (Receiver/Signals), Middleware](#listeners-и-middleware)
6. [Data Validation](#data-validation)
7. [Customize Admin Site](#customize-admin-site)
8. [Security (Auth, CORS, SQL Injection)](#security)
9. [Internationalization and Localization](#i18n-и-l10n)
10. [Performance and Optimization](#performance-и-optimization)
11. [Websockets](#websockets)
12. [Background Tasks (Asyncio, Celery, Dramatiq)](#background-tasks)
13. [Deployment (ASGI, WSGI, Nginx, Gunicorn)](#deployment)
14. [Cache](#cache)
15. [Streaming](#streaming)

---

## Application Settings, Exceptions

### Конфигурация приложения

#### Django
```python
# settings.py
DEBUG = os.environ.get('DEBUG', 'False') == 'True'
SECRET_KEY = os.environ.get('SECRET_KEY')
ALLOWED_HOSTS = os.environ.get('ALLOWED_HOSTS', '').split(',')

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get('DB_NAME'),
        'USER': os.environ.get('DB_USER'),
        'PASSWORD': os.environ.get('DB_PASSWORD'),
        'HOST': os.environ.get('DB_HOST', 'localhost'),
    }
}
```

#### FastAPI
```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    debug: bool = False
    secret_key: str
    database_url: str
    
    class Config:
        env_file = ".env"

settings = Settings()
```

### Обработка исключений

#### Django
```python
# views.py
from django.http import JsonResponse

def custom_exception_handler(request, exception):
    return JsonResponse({'error': str(exception)}, status=500)

# В urls.py
handler500 = 'myapp.views.custom_exception_handler'
```

#### FastAPI
```python
from fastapi import FastAPI, HTTPException
from fastapi.responses import JSONResponse

app = FastAPI()

@app.exception_handler(Exception)
async def global_exception_handler(request, exc):
    return JSONResponse(status_code=500, content={"error": str(exc)})

class CustomException(Exception):
    def __init__(self, message: str, code: int = 400):
        self.message = message
        self.code = code

@app.exception_handler(CustomException)
async def custom_exception_handler(request, exc):
    return JSONResponse(status_code=exc.code, content={"error": exc.message})
```

### Best Practices
- ✅ Храните секреты в переменных окружения
- ✅ Разделяйте настройки dev/staging/prod
- ✅ Централизованная обработка ошибок
- ✅ Логируйте все исключения

---

## Request, Response, Routing

### Django Views
```python
# Function-based view
from django.http import JsonResponse

def get_user(request, user_id):
    if request.method == 'GET':
        user = User.objects.get(id=user_id)
        return JsonResponse({'name': user.name})

# Class-based view
from django.views import View

class UserView(View):
    def get(self, request, user_id):
        user = User.objects.get(id=user_id)
        return JsonResponse({'name': user.name})

# urls.py
urlpatterns = [
    path('users/<int:user_id>/', get_user),
    path('users-cbv/<int:user_id>/', UserView.as_view()),
]
```

### FastAPI
```python
from fastapi import FastAPI, Path, Query
from pydantic import BaseModel

app = FastAPI()

class UserCreate(BaseModel):
    name: str
    email: str

@app.get("/users/{user_id}")
async def get_user(user_id: int = Path(..., gt=0)):
    return {"user_id": user_id}

@app.post("/users/", status_code=201)
async def create_user(user: UserCreate):
    return {"name": user.name}

@app.get("/search/")
async def search(q: str = Query(..., min_length=3)):
    return {"query": q}
```

### Flask
```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/users/<int:user_id>', methods=['GET'])
def get_user(user_id):
    return jsonify({'user_id': user_id})

@app.route('/users/', methods=['POST'])
def create_user():
    data = request.get_json()
    return jsonify(data), 201
```

---

## Database, ORM, Migrations

### Django ORM
```python
# models.py
from django.db import models

class User(models.Model):
    name = models.CharField(max_length=100)
    email = models.EmailField(unique=True)
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        indexes = [models.Index(fields=['email'])]

# Queries
users = User.objects.filter(name__icontains='иван')
user = User.objects.get(id=1)
User.objects.create(name='Иван', email='ivan@mail.ru')
```

### SQLAlchemy
```python
from sqlalchemy import create_engine, Column, Integer, String
from sqlalchemy.orm import declarative_base, sessionmaker

Base = declarative_base()

class User(Base):
    __tablename__ = 'users'
    id = Column(Integer, primary_key=True)
    name = Column(String(100))
    email = Column(String(100), unique=True)

engine = create_engine('postgresql://user:pass@localhost/db')
Session = sessionmaker(bind=engine)
session = Session()

# CRUD
session.add(User(name='Иван', email='ivan@mail.ru'))
session.commit()

users = session.query(User).filter(User.name.like('%иван%')).all()
```

### Migrations

#### Django
```bash
python manage.py makemigrations
python manage.py migrate
python manage.py migrate myapp 0003  # До конкретной миграции
```

#### Alembic (SQLAlchemy)
```bash
alembic init migrations
alembic revision --autogenerate -m "Add users table"
alembic upgrade head
alembic downgrade -1
```

---

## Templates

### Django Templates
```html
<!-- base.html -->
<!DOCTYPE html>
<html>
<head><title>{% block title %}{% endblock %}</title></head>
<body>
    {% block content %}{% endblock %}
</body>
</html>

<!-- user.html -->
{% extends "base.html" %}
{% block title %}{{ user.name }}{% endblock %}
{% block content %}
    <h1>{{ user.name }}</h1>
    {% for post in posts %}
        <p>{{ post.title|truncatewords:20 }}</p>
    {% empty %}
        <p>Нет постов</p>
    {% endfor %}
{% endblock %}
```

### Jinja2
```python
from jinja2 import Environment, FileSystemLoader

env = Environment(loader=FileSystemLoader('templates'))
template = env.get_template('user.html')
html = template.render(user=user, posts=posts)
```

```html
<!-- Jinja2 template -->
{% macro input(name, type='text') %}
    <input type="{{ type }}" name="{{ name }}">
{% endmacro %}

{{ input('email', 'email') }}
```

---

## Listeners и Middleware

### Django Signals
```python
from django.db.models.signals import post_save
from django.dispatch import receiver

@receiver(post_save, sender=User)
def user_created(sender, instance, created, **kwargs):
    if created:
        send_welcome_email(instance.email)
```

### Django Middleware
```python
class LoggingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        print(f"Request: {request.path}")
        response = self.get_response(request)
        print(f"Response: {response.status_code}")
        return response

# settings.py
MIDDLEWARE = ['myapp.middleware.LoggingMiddleware', ...]
```

### FastAPI Middleware
```python
from fastapi import FastAPI
from starlette.middleware.base import BaseHTTPMiddleware

class LoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        print(f"Request: {request.url}")
        response = await call_next(request)
        return response

app.add_middleware(LoggingMiddleware)
```

---

## Data Validation

### Pydantic (FastAPI)
```python
from pydantic import BaseModel, EmailStr, validator, Field
from typing import Optional

class UserCreate(BaseModel):
    name: str = Field(..., min_length=2, max_length=50)
    email: EmailStr
    age: Optional[int] = Field(None, ge=18, le=120)
    
    @validator('name')
    def name_must_be_capitalized(cls, v):
        return v.title()

# Использование
user = UserCreate(name='иван', email='ivan@mail.ru')
print(user.name)  # 'Иван'
```

### Django Forms
```python
from django import forms

class UserForm(forms.Form):
    name = forms.CharField(max_length=100)
    email = forms.EmailField()
    
    def clean_name(self):
        name = self.cleaned_data['name']
        if len(name) < 2:
            raise forms.ValidationError("Слишком короткое имя")
        return name.title()
```

---

## Customize Admin Site

### Django Admin
```python
from django.contrib import admin

@admin.register(User)
class UserAdmin(admin.ModelAdmin):
    list_display = ['name', 'email', 'created_at']
    list_filter = ['created_at']
    search_fields = ['name', 'email']
    ordering = ['-created_at']
    readonly_fields = ['created_at']
    
    fieldsets = [
        ('Основное', {'fields': ['name', 'email']}),
        ('Даты', {'fields': ['created_at'], 'classes': ['collapse']}),
    ]
    
    actions = ['activate_users']
    
    @admin.action(description='Активировать пользователей')
    def activate_users(self, request, queryset):
        queryset.update(is_active=True)
```

---

## Security

### Authentication
```python
# Django
from django.contrib.auth import authenticate, login

def login_view(request):
    user = authenticate(username=username, password=password)
    if user:
        login(request, user)

# FastAPI + JWT
from fastapi_jwt_auth import AuthJWT

@app.post('/login')
def login(user: UserLogin, Authorize: AuthJWT = Depends()):
    access_token = Authorize.create_access_token(subject=user.email)
    return {"access_token": access_token}
```

### CORS
```python
# Django
CORS_ALLOWED_ORIGINS = ["https://example.com"]

# FastAPI
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://example.com"],
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### SQL Injection Prevention
```python
# ❌ НИКОГДА
query = f"SELECT * FROM users WHERE id = {user_id}"

# ✅ Параметризованные запросы
cursor.execute("SELECT * FROM users WHERE id = %s", [user_id])

# ✅ ORM
User.objects.filter(id=user_id)
```

---

## I18n и L10n

### Django
```python
# settings.py
USE_I18N = True
LANGUAGE_CODE = 'ru-ru'
LANGUAGES = [('en', 'English'), ('ru', 'Русский')]

# В шаблонах
{% load i18n %}
{% trans "Привет" %}

# В коде
from django.utils.translation import gettext as _
message = _("Добро пожаловать")

# Генерация файлов перевода
# python manage.py makemessages -l ru
```

---

## Performance и Optimization

### Django
```python
# select_related (ForeignKey)
users = User.objects.select_related('profile').all()

# prefetch_related (ManyToMany)
posts = Post.objects.prefetch_related('tags').all()

# Пагинация
from django.core.paginator import Paginator
paginator = Paginator(users, 25)
page = paginator.get_page(request.GET.get('page'))

# Индексы
class User(models.Model):
    email = models.EmailField(db_index=True)
```

### FastAPI
```python
# Async endpoints
@app.get("/users/")
async def get_users():
    users = await User.all()  # Tortoise ORM
    return users
```

---

## Websockets

### FastAPI
```python
from fastapi import WebSocket

@app.websocket("/ws/{client_id}")
async def websocket_endpoint(websocket: WebSocket, client_id: str):
    await websocket.accept()
    while True:
        data = await websocket.receive_text()
        await websocket.send_text(f"Получено: {data}")
```

### Django Channels
```python
# consumers.py
from channels.generic.websocket import AsyncWebsocketConsumer

class ChatConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        await self.accept()
    
    async def receive(self, text_data):
        await self.send(text_data=f"Echo: {text_data}")
```

---

## Background Tasks

### Celery
```python
# tasks.py
from celery import Celery

app = Celery('tasks', broker='redis://localhost:6379')

@app.task
def send_email(to, subject, body):
    # Отправка email
    pass

# Вызов
send_email.delay('user@mail.ru', 'Тема', 'Текст')
send_email.apply_async(args=['user@mail.ru', 'Тема', 'Текст'], countdown=60)
```

### FastAPI Background Tasks
```python
from fastapi import BackgroundTasks

def send_notification(email: str):
    # Отправка
    pass

@app.post("/users/")
async def create_user(user: UserCreate, background_tasks: BackgroundTasks):
    background_tasks.add_task(send_notification, user.email)
    return {"message": "Пользователь создан"}
```

---

## Deployment

### WSGI (Django/Flask)
```bash
# Gunicorn
gunicorn myapp.wsgi:application -w 4 -b 0.0.0.0:8000

# uWSGI
uwsgi --http :8000 --wsgi-file myapp/wsgi.py --processes 4
```

### ASGI (FastAPI/Starlette)
```bash
# Uvicorn
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4

# Hypercorn
hypercorn main:app --bind 0.0.0.0:8000 --workers 4
```

### Nginx
```nginx
upstream app {
    server 127.0.0.1:8000;
}

server {
    listen 80;
    server_name example.com;
    
    location / {
        proxy_pass http://app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
    
    location /static/ {
        alias /app/static/;
    }
}
```

---

## Cache

### Django
```python
# settings.py
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://localhost:6379',
    }
}

# Использование
from django.core.cache import cache

cache.set('key', 'value', timeout=300)
value = cache.get('key')

# Декоратор
from django.views.decorators.cache import cache_page

@cache_page(60 * 15)  # 15 минут
def my_view(request):
    pass
```

### FastAPI
```python
from fastapi_cache import FastAPICache
from fastapi_cache.decorator import cache

@app.get("/users/")
@cache(expire=60)
async def get_users():
    return await fetch_users()
```

---

## Streaming

### FastAPI Streaming Response
```python
from fastapi.responses import StreamingResponse

async def generate_data():
    for i in range(100):
        yield f"data: {i}\n\n"
        await asyncio.sleep(0.1)

@app.get("/stream")
async def stream():
    return StreamingResponse(generate_data(), media_type="text/event-stream")
```

### Django Streaming
```python
from django.http import StreamingHttpResponse

def generate():
    for i in range(100):
        yield f"data: {i}\n"

def stream_view(request):
    return StreamingHttpResponse(generate(), content_type="text/event-stream")
```
