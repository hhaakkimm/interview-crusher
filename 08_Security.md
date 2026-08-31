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

## 3. Passkeys и WebAuthn — passwordless по умолчанию (2025-2026)

### 🎯 Что спрашивают
> "Что вы думаете про passkeys? Стоит ли отказываться от паролей?"

### Простое объяснение
К 2025-2026 **passkeys** (на базе стандарта **WebAuthn / FIDO2**, продвигаемого FIDO Alliance при поддержке Apple, Google и Microsoft) стали мейнстримным и рекомендуемым по умолчанию способом аутентификации для новых систем — вместо классической пары "пароль + опциональный 2FA". Это не просто "ещё один MFA-фактор", а принципиально другая модель: на устройстве пользователя генерируется **пара ключей (приватный/публичный)** для конкретного сайта — приватный ключ никогда не покидает устройство (хранится в Secure Enclave/TPM или менеджере паролей с синхронизацией), сервер хранит только публичный ключ.

```
Регистрация passkey:
  Устройство генерирует keypair (private, public) для домена example.com
  → Публичный ключ уходит и сохраняется на сервере
  → Приватный ключ остаётся на устройстве (не передаётся никогда)

Вход:
  Сервер присылает challenge (случайную строку)
  → Устройство подписывает challenge приватным ключом
    (разблокировка — biometric/PIN локально, не передаётся серверу)
  → Сервер проверяет подпись публичным ключом
  → Нет пароля, нет OTP, нет ничего, что можно перехватить или украдено с сервера
```

### Passkeys vs Password + OTP/2FA

| Критерий | Password + OTP (2FA) | Passkey (WebAuthn) |
|---|---|---|
| Что хранит сервер | Хеш пароля (+ секрет для OTP) | Только публичный ключ — бесполезен для атакующего без приватной пары |
| Фишинг | Пароль/OTP можно ввести на поддельном сайте | **Phishing-resistant по конструкции**: подпись криптографически привязана к origin (домену) — не сработает на похожем, но чужом домене |
| Утечка базы сервера | Компрометирует хеши паролей (offline brute-force) | Утечка публичных ключей бесполезна атакующему |
| UX | Ввод пароля + отдельный шаг за OTP (SMS/app) | Один жест (biometric/PIN) — быстрее и без необходимости помнить пароль |
| Credential stuffing / reuse паролей | Реальный риск (пользователи переиспользуют пароли) | Неприменимо — ключ уникален для каждого сайта by design |

### Что важно сказать на интервью
- Passkeys не отменяют полностью нужду в fallback-механизме (потеря устройства, ещё не у всех клиентов есть поддержка) — на практике внедряются постепенно, с паролем/OTP как запасным путём на переходный период.
- Синхронизация passkey между устройствами пользователя обычно идёт через экосистему платформы (iCloud Keychain, Google Password Manager) — это снижает "потерял телефон — потерял доступ" риск, но добавляет вопрос доверия к провайдеру синхронизации.
- Для backend-разработчика задача — интегрировать WebAuthn API (регистрация/аутентификация ceremony), хранить публичные ключи с привязкой к user_id и device, обрабатывать multiple credentials на пользователя (несколько устройств).
- Это резко снижает эффективность целого класса атак — фишинга и credential stuffing — потому что нет общего секрета (пароля), который можно украсть, угадать или переиспользовать.

### 📝 Фраза для интервью
> "Passkeys на базе WebAuthn/FIDO2 — сейчас рекомендуемый по умолчанию способ аутентификации для новых систем, а не просто дополнительный MFA-фактор. На устройстве генерируется пара ключей для конкретного сайта, приватный ключ никогда не покидает устройство, сервер хранит только публичный ключ — это делает passkey phishing-resistant by design, потому что подпись криптографически привязана к origin и не сработает на поддельном домене. В отличие от password+OTP, здесь нет общего секрета, который можно украсть с сервера, перехватить или переиспользовать между сайтами. На практике внедряю как основной путь с password+2FA как fallback на переходный период, пока не у всех клиентов есть поддержка WebAuthn."

---

## 4. OAuth 2.0 и OpenID Connect

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

## 5. JWT Security

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

## 6. HTTPS и TLS

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

## 7. Secrets Management

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

### Эволюция: от статичных секретов к short-lived, OIDC-federated credentials (текущий best practice)

Даже env vars и secrets manager из примеров выше обычно означают **долгоживущий статичный credential** (API key, service account JSON, DB password), который где-то хранится и может утечь — из CI-лога, из скомпрометированного раннера, из неаккуратного дампа переменных окружения. Текущий best practice для CI/CD и межсервисной аутентификации к облаку — вообще не хранить такой долгоживущий секрет:

```
❌ Старый подход (статичный секрет в CI):
   GitHub Actions secret: AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY
   → Один и тот же credential живёт месяцами, лежит в секретах репозитория,
     годится для входа откуда угодно, если утечёт

✅ OIDC-федерация (текущий стандарт):
   1. CI job запрашивает у CI-провайдера (GitHub Actions) короткоживущий OIDC-токен
      (JWT, подписанный CI-провайдером, с claims: repo, branch, workflow)
   2. CI обменивает этот токен в облаке (AWS STS / GCP Workload Identity /
      Azure AD) на временный IAM-токен через preconfigured trust relationship
   3. Временный токен живёт минуты-часы и привязан к конкретному репозиторию/ветке
   → В CI вообще не хранится долгоживущий облачный credential
```

```yaml
# Пример: GitHub Actions OIDC → AWS без единого статичного секрета
permissions:
  id-token: write   # разрешает job запросить OIDC-токен
jobs:
  deploy:
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/ci-deploy-role
          aws-region: us-east-1
          # AWS доверяет OIDC-токену от GitHub Actions (trust policy на роль),
          # обменивает его на временные креды — ничего похожего на
          # AWS_SECRET_ACCESS_KEY в секретах репозитория не нужно
```

Что важно сказать на интервью:
- Главная выгода — снимается целый класс инцидентов "утёк долгоживущий credential из CI/лога/раннера": даже если токен из одного запуска утечёт, он и так истекает за минуты и привязан к конкретному контексту (repo/branch), а не даёт доступ навсегда.
- Централизованные secret store (Vault, cloud KMS/Secrets Manager) остаются нужны для секретов, которые в принципе не могут быть short-lived (например, ключи шифрования, сторонние API-ключи без поддержки federation) — но и там подход эволюционирует к динамическим, короткоживущим leases (Vault dynamic secrets), а не статичным значениям, которые лежат месяцами.
- Rotation policy на статичные секреты, где OIDC-федерация невозможна, всё ещё нужна как второй рубеж защиты.

### 📝 Фраза для интервью
> "Секреты — в secrets manager (AWS Secrets Manager, HashiCorp Vault) или environment variables как минимальный вариант, никогда в git и коде. Но текущий best practice для CI/CD и межсервисной аутентификации к облаку — уходить от долгоживущих статичных credentials вообще: CI-провайдер выдаёт короткоживущий OIDC-токен, который обменивается на временный cloud IAM credential (GitHub Actions OIDC → AWS STS/GCP Workload Identity) — токен живёт минуты и привязан к конкретному репозиторию/ветке, поэтому утечка из лога или раннера не даёт постоянного доступа. Там, где short-lived кредлы невозможны, использую Vault/KMS с rotation policy и audit log как второй рубеж."

---

## 8. Input Validation

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

## 9. Supply Chain Security

### 🎯 Что спрашивают
> "Как вы защищаетесь от атак через зависимости и CI/CD pipeline?"

### Простое объяснение
Supply chain security — защита не самого вашего кода, а всего, из чего он собирается и разворачивается: сторонние зависимости (npm/PyPI/Maven пакеты), CI/CD pipeline, сборочные артефакты, base images для контейнеров. Это выделилось в отдельную, обязательную для senior-инженера тему, потому что атака через зависимость или скомпрометированный build-процесс задевает сразу всех потребителей пакета/образа — атакующему не нужно ломать конкретно вашу систему.

### Классы атак на supply chain

| Атака | Как работает | Защита |
|---|---|---|
| **Dependency confusion** | Атакующий публикует в публичном registry (npm/PyPI) пакет с тем же именем, что и внутренний приватный пакет компании — package manager может по ошибке подтянуть публичную (вредоносную) версию вместо приватной | Приватные пакеты — через scoped-имена (`@company/pkg`) и/или через приватный registry-прокси, который не даёт fallback на публичный registry по умолчанию для внутренних имён |
| **Typosquatting** | Публикация пакета с именем, похожим на популярный (`reqeusts` вместо `requests`) в расчёте на опечатку разработчика | Внимательная проверка имени/автора пакета перед установкой, lockfile (package-lock.json/poetry.lock) для фиксации проверенных версий, автоматизированные сканеры зависимостей в CI |
| **Compromised dependency / maintainer account** | Легитимный пакет получает вредоносное обновление (через взлом аккаунта мейнтейнера или скомпрометированный upstream) | Pin точных версий в lockfile, review diff при обновлении критичных зависимостей, SBOM для видимости, что реально используется в проде |
| **Compromised build pipeline** | Вредоносный код внедряется не в исходники, а в сам процесс сборки/деплоя (CI runner, build script) | Artifact signing и provenance-проверка (см. ниже), изолированные/эфемерные CI-раннеры, минимальные права CI-токенов (см. раздел 7, OIDC) |

### SBOM, artifact signing и SLSA

```
SBOM (Software Bill of Materials):
  Машиночитаемый список всех зависимостей (прямых и транзитивных)
  в конкретном билде — что именно, каких версий, откуда.
  → Позволяет за минуты понять "используем ли мы уязвимую библиотеку X"
    при появлении новой уязвимости, вместо ручного аудита.

Artifact signing / provenance (Sigstore / cosign):
  Каждый собранный артефакт (контейнер-образ, пакет) подписывается
  криптографически на этапе сборки — потребитель может проверить,
  что артефакт действительно собран заявленным CI-pipeline
  из заявленного исходного кода, а не подменён по пути.

SLSA (Supply-chain Levels for Software Artifacts):
  Фреймворк уровней зрелости supply chain security — от базовой
  (билд воспроизводим и задокументирован) до строгой (билд полностью
  изолирован, herметичен, все шаги верифицируемы) — общий язык
  для описания "насколько защищена цепочка поставки" конкретного проекта.
```

Что важно сказать на интервью:
- Для внутренних/приватных пакетов — всегда scoped-имена и приватный registry (Artifactory/Nexus/GitHub Packages) с явно настроенным приоритетом над публичным registry, а не полагаться на "надеюсь, имя не совпадёт".
- Lockfile — не формальность, а граница доверия: без него каждый `npm install` может подтянуть новую версию транзитивной зависимости без review.
- SBOM и artifact signing — это то, что сейчас всё чаще требуют enterprise-заказчики и регуляторы как часть due diligence, а не "экзотика для больших корпораций".

### 📝 Фраза для интервью
> "Supply chain security — это защита не своего кода, а всего вокруг него: зависимостей, CI pipeline, артефактов сборки. Реальные классы атак — dependency confusion (вредоносный публичный пакет с именем как у внутреннего) и typosquatting (опечатка в имени пакета); защищаюсь scoped-именами для внутренних пакетов, приватным registry-прокси и строгими lockfile. Для артефактов — SBOM, чтобы за минуты понимать, какие зависимости реально в проде при появлении новой уязвимости, и artifact signing/provenance (Sigstore/cosign, ориентируясь на уровни SLSA) — чтобы потребитель мог проверить, что артефакт собран заявленным pipeline из заявленного кода, а не подменён где-то по дороге."

---

## 10. OWASP Top 10 для LLM-приложений

### 🎯 Что спрашивают
> "Чем отличается безопасность систем с LLM-компонентом от классической web security?"

### Простое объяснение
Помимо классического OWASP Top 10 (раздел 1), для систем с LLM/агентным компонентом существует отдельный, устоявшийся к 2025-2026 список — **OWASP Top 10 for LLM Applications**. Он не заменяет классический список (SQL injection, broken auth и т.д. никуда не делись), а добавляет категории рисков, специфичные именно для LLM-компонента:

| Категория (обобщённо) | Суть |
|---|---|
| **Prompt Injection** (в т.ч. indirect) | Инструкции, подмешанные во входные данные или в контент, который LLM подгружает сам, заставляют модель отклониться от заданного поведения |
| **Insecure Output Handling** | Ответ LLM используется downstream без валидации (например, вставляется в HTML/SQL/shell-команду как доверенный ввод) — по сути XSS/injection, но источник — генерация модели, а не пользователь напрямую |
| **Training Data / Model Poisoning** | Компрометация данных, на которых обучалась/дообучалась модель, для внедрения скрытого нежелательного поведения |
| **Excessive Agency** | LLM-агенту выданы более широкие права/tools, чем реально нужно для задачи — при ошибке или манипуляции агент может воспользоваться ими во вред |
| **Sensitive Information Disclosure** | Модель раскрывает секреты/PII, попавшие в промпт, обучающие данные или контекст |
| **Supply Chain (для моделей/плагинов)** | Риски, связанные с сторонними моделями, LoRA-адаптерами, plugin/tool-экосистемой — тот же класс атак, что в разделе 9, но применительно к ML-артефактам |

Подробный разбор конкретных техник защиты (prompt injection, indirect injection, sanitization, least privilege для tool-доступа агента) — см. **14_Prompt_Engineering.md**, раздел Prompt Security; здесь важно на интервью по security показать, что вы знаете о существовании этого отдельного списка и понимаете его связь с Excessive Agency/Insecure Output Handling как продолжением классических принципов (least privilege, никогда не доверяй входу, валидируй перед использованием) в новом контексте.

### 📝 Фраза для интервью
> "Для систем с LLM-компонентом кроме классического OWASP Top 10 действует отдельный OWASP Top 10 for LLM Applications — prompt injection (включая indirect, через данные, которые агент подгружает сам), insecure output handling, когда ответ модели используется downstream без валидации, excessive agency, когда у агента больше прав/tools, чем нужно для задачи, и утечка чувствительных данных через промпт или контекст. По сути это те же базовые принципы — never trust input, least privilege, validate before use — применённые к новому источнику непроверенного ввода, которым стала сама генерация модели. Детали техник защиты (input/output sanitization, least privilege для tool-доступа) я разбираю в контексте prompt engineering."

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

### "Passkeys vs пароль + 2FA — что рекомендовать для нового проекта?"
> "Для нового проекта — passkeys (WebAuthn/FIDO2) как основной путь входа: на устройстве генерируется пара ключей для конкретного домена, приватный ключ никогда не покидает устройство, поэтому это phishing-resistant by design — нет общего секрета, который можно украсть с сервера или ввести на поддельном сайте. Password + OTP оставляю как fallback на переходный период, пока не все клиенты/пользователи готовы к passkeys."

### "Что такое dependency confusion и как от него защититься?"
> "Атака, при которой вредоносный пакет публикуется в публичном registry с тем же именем, что и внутренний приватный пакет компании — package manager может по ошибке подтянуть публичную версию. Защита — scoped-имена для внутренних пакетов (`@company/pkg`), приватный registry-прокси без автоматического fallback на публичный registry, строгие lockfile."

### "Что такое SBOM и зачем он нужен?"
> "Software Bill of Materials — машиночитаемый список всех зависимостей билда, прямых и транзитивных. Позволяет за минуты понять, затрагивает ли новая обнаруженная уязвимость ваш прод, вместо ручного аудита зависимостей. Часто идёт в паре с artifact signing (Sigstore/cosign) и ориентиром на уровни SLSA."

### "Как правильно управлять секретами в CI/CD сейчас?"
> "Уходить от долгоживущих статичных credentials в секретах репозитория к OIDC-федерации: CI-провайдер (например GitHub Actions) выдаёт короткоживущий подписанный токен, который обменивается на временный cloud IAM credential (AWS STS, GCP Workload Identity) — токен живёт минуты и привязан к конкретному repo/branch. Там, где federation невозможна — централизованный secret store (Vault, cloud Secrets Manager/KMS) с rotation policy, а не .env файл со статичными значениями."

### "Существует ли отдельный OWASP-список для LLM-приложений?"
> "Да, OWASP Top 10 for LLM Applications — дополняет классический список категориями вроде prompt injection, insecure output handling, excessive agency и утечки чувствительных данных через контекст модели. Не заменяет классический Top 10, а добавляет риски, специфичные для LLM/агентного компонента системы."

---

## 📚 Чек-лист для подготовки

### Базовый уровень
- [ ] OWASP Top 10 (injection, broken auth, XSS, CSRF, IDOR)
- [ ] Authentication vs Authorization, RBAC
- [ ] OAuth 2.0 Authorization Code flow, OpenID Connect
- [ ] JWT: структура, типичные уязвимости и защита

### Средний уровень
- [ ] Passkeys / WebAuthn — как работают, phishing-resistance, чем отличаются от password+2FA
- [ ] HTTPS/TLS handshake, security headers (HSTS, CSP, X-Frame-Options)
- [ ] Secrets management: secrets manager/Vault, эволюция к short-lived OIDC-federated credentials в CI/CD
- [ ] Input validation: whitelist подход, серверная валидация

### Продвинутый / Senior уровень
- [ ] Supply chain security: SBOM, artifact signing/provenance (Sigstore/cosign), SLSA
- [ ] Dependency confusion и typosquatting — как атака работает и как защищаться (scoped packages, private registry, lockfile)
- [ ] OWASP Top 10 for LLM Applications: prompt injection, insecure output handling, excessive agency
- [ ] Умение связать классические принципы (least privilege, never trust input) с новыми угрозами (LLM-компоненты, supply chain)
