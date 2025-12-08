# Security - Руководство для Senior Backend интервью

> 💡 **Как думать о безопасности на интервью:**
> "Безопасность — это слои защиты. Defense in depth: даже если один слой пробит, есть следующий. Принцип наименьших привилегий. Никогда не доверяй входящим данным."

---

## 1. OWASP Top 10

### 🎯 Что спрашивают
> "Какие уязвимости вы знаете? Как защищаться?"

### Критические уязвимости

#### 1. Injection (SQL, NoSQL, Command)
```python
# ❌ SQL Injection
query = f"SELECT * FROM users WHERE name = '{user_input}'"
# user_input = "'; DROP TABLE users; --"

# ✅ Параметризованные запросы
cursor.execute("SELECT * FROM users WHERE name = %s", [user_input])

# ✅ ORM
User.objects.filter(name=user_input)
```

#### 2. Broken Authentication
```python
# ❌ Плохо
if user.password == submitted_password:  # Plain text!

# ✅ Хорошо
from passlib.hash import bcrypt
if bcrypt.verify(submitted_password, user.password_hash):
```

#### 3. XSS (Cross-Site Scripting)
```html
<!-- ❌ Уязвимо -->
<div>{{ user_input }}</div>
<!-- user_input = "<script>steal(document.cookie)</script>" -->

<!-- ✅ Экранирование (Django автоматически) -->
<div>{{ user_input|escape }}</div>
```

#### 4. CSRF (Cross-Site Request Forgery)
```html
<!-- Защита: CSRF токен -->
<form method="POST">
    {% csrf_token %}
    <input type="hidden" name="csrfmiddlewaretoken" value="...">
</form>
```

#### 5. Insecure Direct Object Reference (IDOR)
```python
# ❌ Уязвимо — пользователь может изменить order_id
@app.get("/orders/{order_id}")
def get_order(order_id: int):
    return db.get_order(order_id)

# ✅ Проверка владельца
@app.get("/orders/{order_id}")
def get_order(order_id: int, current_user: User = Depends(get_current_user)):
    order = db.get_order(order_id)
    if order.user_id != current_user.id:
        raise HTTPException(403, "Forbidden")
    return order
```

### 📝 Фраза для интервью
> "OWASP Top 10 — чек-лист основных уязвимостей. SQL injection решается параметризацией. XSS — экранированием и CSP. CSRF — токенами. Главное: валидация на сервере, принцип наименьших привилегий."

---

## 2. Authentication vs Authorization

### 🎯 Что спрашивают
> "Разница между аутентификацией и авторизацией?"

### Простое объяснение
```
Authentication (AuthN): КТО ты?
  → Login, password, JWT, OAuth

Authorization (AuthZ): ЧТО тебе можно?
  → Roles, permissions, policies
```

### Способы аутентификации

| Метод | Описание | Когда использовать |
|-------|----------|-------------------|
| Session-based | Cookie с session ID | Traditional web apps |
| JWT | Stateless token | APIs, microservices |
| API Key | Простой ключ | Server-to-server |
| OAuth 2.0 | Делегированный доступ | Third-party auth |

### Role-Based Access Control (RBAC)
```python
class Permission(Enum):
    READ_USERS = "read:users"
    WRITE_USERS = "write:users"
    DELETE_USERS = "delete:users"

ROLES = {
    "admin": [Permission.READ_USERS, Permission.WRITE_USERS, Permission.DELETE_USERS],
    "editor": [Permission.READ_USERS, Permission.WRITE_USERS],
    "viewer": [Permission.READ_USERS],
}

def has_permission(user, permission):
    return permission in ROLES.get(user.role, [])
```

### 📝 Фраза для интервью
> "Authentication — проверка личности, authorization — проверка прав. RBAC группирует permissions в роли. Для APIs — JWT с claims, для web — sessions с CSRF protection."

---

## 3. OAuth 2.0 и OpenID Connect

### 🎯 Что спрашивают
> "Объясните OAuth 2.0 flow"

### OAuth 2.0 Authorization Code Flow
```
┌──────┐                                    ┌──────────────┐
│ User │                                    │   Provider   │
│      │                                    │ (Google/FB)  │
└──┬───┘                                    └──────┬───────┘
   │                                               │
   │  1. Click "Login with Google"                 │
   │ ─────────────────────────────────────────────▶│
   │                                               │
   │  2. Redirect to Google login                  │
   │ ◀─────────────────────────────────────────────│
   │                                               │
   │  3. User logs in, grants permission           │
   │ ─────────────────────────────────────────────▶│
   │                                               │
   │  4. Redirect back with authorization code     │
   │ ◀─────────────────────────────────────────────│
   │                                               │
   │                    ┌─────────┐                │
   │                    │Your App │                │
   │                    └────┬────┘                │
   │                         │                     │
   │  5. Exchange code for tokens (server-to-server)
   │                         │────────────────────▶│
   │                         │                     │
   │  6. Access token + ID token                   │
   │                         │◀────────────────────│
   │                         │                     │
   │  7. Use access token to get user info         │
   │                         │────────────────────▶│
```

### Tokens
- **Access Token**: для доступа к API (короткоживущий)
- **Refresh Token**: для получения нового access token
- **ID Token** (OpenID Connect): информация о пользователе (JWT)

### 📝 Фраза для интервью
> "OAuth 2.0 — протокол авторизации для делегированного доступа. Authorization Code flow для web apps — code обменивается на токены server-to-server. OpenID Connect добавляет authentication — ID token с информацией о пользователе."

---

## 4. JWT Security

### 🎯 Что спрашивают
> "Какие проблемы безопасности у JWT?"

### Структура JWT
```
Header.Payload.Signature

eyJhbGciOiJIUzI1NiJ9.         # {"alg": "HS256"}
eyJ1c2VyX2lkIjoxMjN9.         # {"user_id": 123}
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c  # Signature
```

### Уязвимости и защита

| Уязвимость | Решение |
|------------|---------|
| **alg: none** | Проверять алгоритм |
| **Weak secret** | Сильный ключ (256+ bit) |
| **Token theft** | Короткий TTL, HttpOnly cookie |
| **No revocation** | Blacklist или short-lived tokens |

### Best Practices
```python
# ✅ Правильная генерация
import jwt
import secrets

SECRET_KEY = secrets.token_hex(32)  # 256 bit

token = jwt.encode(
    {
        "sub": user_id,
        "exp": datetime.utcnow() + timedelta(minutes=15),  # Короткий TTL!
        "iat": datetime.utcnow(),
        "jti": str(uuid.uuid4()),  # Unique ID для revocation
    },
    SECRET_KEY,
    algorithm="HS256"
)

# ✅ Проверка
try:
    payload = jwt.decode(
        token, 
        SECRET_KEY, 
        algorithms=["HS256"],  # Явно указываем!
        options={"require": ["exp", "sub"]}
    )
except jwt.ExpiredSignatureError:
    raise AuthError("Token expired")
```

### 📝 Фраза для интервью
> "JWT stateless — сервер не хранит сессии. Уязвимости: alg:none attack, weak secrets, невозможность отзыва. Защита: короткий TTL (15 min), refresh tokens, явная проверка алгоритма, blacklist для logout."

---

## 5. HTTPS и TLS

### 🎯 Что спрашивают
> "Как работает HTTPS?"

### TLS Handshake (упрощённо)
```
Client                                 Server
   │                                      │
   │  1. ClientHello (supported ciphers)  │
   │ ────────────────────────────────────▶│
   │                                      │
   │  2. ServerHello + Certificate        │
   │ ◀────────────────────────────────────│
   │                                      │
   │  3. Verify cert, generate pre-master │
   │     secret, encrypt with public key  │
   │ ────────────────────────────────────▶│
   │                                      │
   │  4. Both derive session key          │
   │ ◀───────────────────────────────────▶│
   │                                      │
   │  5. Encrypted communication          │
   │ ◀═══════════════════════════════════▶│
```

### Важные заголовки
```nginx
# HSTS — только HTTPS
Strict-Transport-Security: max-age=31536000; includeSubDomains

# Запрет встраивания в iframe
X-Frame-Options: DENY

# CSP — контроль источников контента
Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline'
```

### 📝 Фраза для интервью
> "HTTPS = HTTP + TLS. TLS обеспечивает шифрование (конфиденциальность), целостность (MAC), аутентификацию сервера (сертификат). HSTS header заставляет браузер использовать только HTTPS."

---

## 6. Secrets Management

### 🎯 Что спрашивают
> "Как управлять секретами в production?"

### ❌ Никогда не делайте
```python
# Хардкод в коде
DATABASE_URL = "postgres://user:password@host/db"

# В git
# .env с секретами закоммичен
```

### ✅ Правильные подходы

#### Environment Variables
```python
import os
DATABASE_URL = os.environ["DATABASE_URL"]
```

#### Secrets Manager (AWS/GCP/Azure)
```python
import boto3

def get_secret(secret_name):
    client = boto3.client('secretsmanager')
    response = client.get_secret_value(SecretId=secret_name)
    return response['SecretString']

db_password = get_secret("prod/db/password")
```

#### HashiCorp Vault
```python
import hvac

client = hvac.Client(url='https://vault.example.com')
secret = client.secrets.kv.read_secret_version(path='database')
password = secret['data']['data']['password']
```

### 📝 Фраза для интервью
> "Secrets в environment variables или secrets manager (AWS Secrets Manager, HashiCorp Vault). Никогда в git. Rotation policy для периодической смены. Audit log для отслеживания доступа."

---

## 7. Input Validation

### 🎯 Что спрашивают
> "Как валидировать входящие данные?"

### Принципы
1. **Validate on server** — клиент не доверяем
2. **Whitelist > Blacklist** — разрешаем известное
3. **Fail securely** — при ошибке отклоняем

### Pydantic для строгой валидации
```python
from pydantic import BaseModel, EmailStr, Field, validator
import re

class UserCreate(BaseModel):
    username: str = Field(..., min_length=3, max_length=20, regex="^[a-zA-Z0-9_]+$")
    email: EmailStr
    age: int = Field(..., ge=18, le=120)
    
    @validator('username')
    def no_sql_injection(cls, v):
        dangerous = ["'", '"', ";", "--", "/*", "*/"]
        if any(char in v for char in dangerous):
            raise ValueError("Invalid characters")
        return v

# Автоматическая валидация при создании
user = UserCreate(username="john_doe", email="john@example.com", age=25)
```

### 📝 Фраза для интервью
> "Валидация на сервере обязательна. Whitelist подход — разрешаем только известные паттерны. Pydantic для типизированной валидации. Экранирование при выводе для защиты от XSS."

---

## 🎤 Частые вопросы Security

### "Как хранить пароли?"
> "Bcrypt или Argon2 — алгоритмы с солью и итерациями. Никогда MD5/SHA без соли. Cost factor подбираем чтобы hash занимал ~100ms."

### "Что такое CORS?"
> "Cross-Origin Resource Sharing — механизм разрешения запросов с других доменов. Браузер делает preflight OPTIONS запрос. Access-Control-Allow-Origin header определяет разрешённые домены."

### "Как защитить API от DDoS?"
> "Rate limiting на нескольких уровнях: CDN, loadbalancer, application. WAF для фильтрации. Капча для подозрительных запросов. Autoscaling."

### "Что такое SQL injection second order?"
> "Данные сохраняются безопасно, но используются небезопасно позже. Решение: параметризация везде, даже для данных из базы."

### "Как реализовать безопасный logout?"
> "Для JWT: blacklist token'а в Redis до его expiry. Для sessions: удалить серверную сессию и cookie. Invalidate все токены при смене пароля."

### "Что такое security headers?"
> "HTTP headers для защиты: X-Content-Type-Options: nosniff, X-Frame-Options: DENY, CSP, HSTS. Проверяйте через securityheaders.com."
