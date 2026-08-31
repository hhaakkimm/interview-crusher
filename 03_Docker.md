# Docker - Руководство для технического интервью

> 💡 **Как объяснить Docker на интервью за 30 секунд:**
> "Docker — это инструмент контейнеризации. Представьте, что вы упаковываете приложение со всеми его зависимостями в 'коробку' (контейнер), которая будет работать одинаково везде: на моём ноутбуке, на сервере тестирования и в продакшене. Это решает проблему 'у меня работает, а у тебя нет'."

---

## 1. Теория - Контейнер vs Виртуальная машина

### 🎯 Что спрашивают на интервью
> "В чём разница между контейнером и виртуальной машиной?"

### Простое объяснение
**Виртуальная машина** — это как отдельная квартира в доме. У неё своя сантехника, электрика, всё отдельное. Занимает много места и ресурсов.

**Контейнер** — это как комната в коммунальной квартире. Комнаты изолированы, но кухня и ванная общие (ядро ОС). Занимает мало места, запускается мгновенно.

### Ключевые отличия для ответа

| | Контейнер | VM |
|---|----------|-----|
| **Что изолируется** | Процессы приложения | Вся ОС целиком |
| **Размер** | Мегабайты (10-500 MB) | Гигабайты (2-10 GB) |
| **Запуск** | Секунды | Минуты |
| **Накладные расходы** | Минимальные | Значительные |
| **Ядро ОС** | Общее с хостом | Своё у каждой VM |

### Когда что использовать
- **Контейнеры**: микросервисы, CI/CD, когда нужна одна ОС (Linux)
- **VM**: нужны разные ОС, полная изоляция безопасности, legacy системы

### 📝 Фраза для интервью
> "Контейнеры используют ядро хост-системы и изолируют только приложение, а VM эмулирует целый компьютер с отдельной ОС. Поэтому контейнеры легче, быстрее запускаются и потребляют меньше ресурсов."

---

## 2. Docker CLI - Основные команды

### 🎯 Что спрашивают
> "Расскажите основные команды Docker"

### Команды которые нужно знать наизусть

```bash
# Скачать образ
docker pull nginx:1.21

# Запустить контейнер
docker run -d -p 8080:80 --name my-nginx nginx

# Посмотреть что работает
docker ps          # работающие
docker ps -a       # все (включая остановленные)

# Зайти внутрь контейнера
docker exec -it my-nginx bash

# Остановить/удалить
docker stop my-nginx
docker rm my-nginx

# Логи
docker logs my-nginx
docker logs -f my-nginx  # в реальном времени
```

### Разбор флагов docker run
```bash
docker run -d -p 8080:80 -v /data:/app/data -e DB_HOST=localhost --name app myimage
```
- `-d` — detached (в фоне)
- `-p 8080:80` — проброс порта (хост:контейнер)
- `-v /data:/app/data` — монтирование папки
- `-e DB_HOST=localhost` — переменная окружения
- `--name app` — имя контейнера

### 📝 Фраза для интервью
> "Основной цикл работы: pull для скачивания образа, run для запуска контейнера, exec для входа внутрь, logs для просмотра логов, stop/rm для остановки. Для многоконтейнерных приложений использую docker-compose."

---

## 3. Dockerfile - Создание образов

### 🎯 Что спрашивают
> "Как устроен Dockerfile? Напишите простой пример."

### Структура Dockerfile с объяснениями

```dockerfile
# FROM — базовый образ (всегда первая строка)
# Это как фундамент, на котором строим
FROM python:3.11-slim

# WORKDIR — рабочая директория внутри контейнера
# Аналог "cd /app" при каждой следующей команде
WORKDIR /app

# COPY — копируем файлы с хоста в контейнер
# Сначала зависимости (меняются редко)
COPY requirements.txt .

# RUN — выполнить команду при СБОРКЕ образа
# Устанавливаем зависимости
RUN pip install --no-cache-dir -r requirements.txt

# Потом копируем код (меняется часто)
COPY . .

# ENV — переменные окружения
ENV PYTHONUNBUFFERED=1

# EXPOSE — документация: какой порт использует приложение
EXPOSE 8000

# CMD — команда при ЗАПУСКЕ контейнера
CMD ["python", "app.py"]
```

### Важно понимать разницу
- **RUN** — выполняется при `docker build` (сборка)
- **CMD** — выполняется при `docker run` (запуск)
- **COPY** vs **ADD** — COPY просто копирует, ADD умеет распаковывать архивы и качать по URL (используйте COPY, если не нужны эти фичи)

### BuildKit — билдер по умолчанию (уже не opt-in)

Раньше BuildKit включали вручную (`DOCKER_BUILDKIT=1 docker build ...`). Сейчас это дефолтный билдер и в Docker Engine, и в `docker build`/`docker buildx build` — включать явно не нужно, старый builder остался только для обратной совместимости.

Что это даёт на практике (хороший ответ на вопрос про оптимизацию сборки):

```dockerfile
# Cache mounts — кешируем директорию пакетного менеджера МЕЖДУ сборками,
# не запекая кеш в слой образа
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# Secrets на этапе сборки — не остаются в истории слоёв,
# в отличие от ARG/ENV с паролем/токеном
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm install
```

```bash
docker build --secret id=npmrc,src=$HOME/.npmrc .

# buildx — мультиплатформенная сборка одной командой (важно для Apple Silicon / ARM в облаке)
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:latest --push .
```

- BuildKit параллелит независимые стадии multistage-билда, а не строит их строго последовательно.
- `cache mounts` и `--mount=type=secret` закрывают два частых промаха: раздутые слои от кеша пакетного менеджера и утечку секретов в историю образа.

### 📝 Фраза для интервью
> "Dockerfile описывает как собрать образ: FROM указывает базовый образ, COPY копирует файлы, RUN выполняет команды при сборке, CMD определяет что запускать. Важно правильно упорядочивать инструкции для эффективного кеширования — редко меняющиеся файлы сверху. Собираю через BuildKit (он и так дефолтный) — использую cache mounts вместо кеша, запечённого в слой, и `--mount=type=secret` вместо ARG для секретов на этапе сборки."

---

## 4. Multistage Build

### 🎯 Что спрашивают
> "Что такое multistage build и зачем он нужен?"

### Простое объяснение
Представьте: чтобы испечь торт, нужна куча инструментов (миксеры, формы). Но на стол гостям вы выносите только торт, не все инструменты. Multistage — это когда мы на одном этапе собираем (с компилятором, npm, всеми dev-зависимостями), а в финальный образ кладём только результат.

### Пример для Go

```dockerfile
# Этап 1: Сборка (800 MB образ с компилятором)
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN go build -o main .

# Этап 2: Финальный образ (только 15 MB!)
FROM alpine:latest
COPY --from=builder /app/main .
CMD ["./main"]
```

**Результат**: образ 15 MB вместо 800 MB!

### Когда использовать
- Компилируемые языки (Go, Rust, Java)
- Frontend (React/Vue + nginx)
- Когда нужны dev-зависимости только для сборки

### 📝 Фраза для интервью
> "Multistage build позволяет использовать несколько FROM в одном Dockerfile. На первом этапе собираем приложение со всеми инструментами, на втором берём только артефакт сборки. Это значительно уменьшает размер финального образа и убирает лишние зависимости."

---

## 5. Volumes - Хранение данных

### 🎯 Что спрашивают
> "Как контейнеры хранят данные? Что будет если контейнер удалить?"

### Главное правило
**Контейнеры эфемерны** — данные внутри контейнера удаляются вместе с ним!

Для сохранения данных используем **Volumes**:

```bash
# Named volume (Docker управляет)
docker run -v mydata:/app/data postgres

# Bind mount (монтируем папку хоста)
docker run -v /home/user/data:/app/data myapp

# Только для чтения
docker run -v ./config:/app/config:ro myapp
```

### Когда что использовать
| Тип | Когда использовать |
|-----|-------------------|
| Named volume | База данных, production данные |
| Bind mount | Разработка (чтобы видеть изменения кода) |
| tmpfs | Временные данные (только в памяти) |

### 📝 Фраза для интервью
> "Контейнеры stateless по дизайну — при удалении данные теряются. Для персистентных данных используем volumes: named volumes для production, bind mounts для разработки. Базы данных обязательно должны хранить данные в volume."

---

## 6. Networks - Взаимодействие контейнеров

### 🎯 Что спрашивают
> "Как контейнеры общаются друг с другом?"

### Простое объяснение
Docker создаёт виртуальную сеть. Контейнеры в одной сети могут общаться по **имени контейнера** как по DNS-имени.

```yaml
# docker-compose.yml
services:
  api:
    image: myapi
    environment:
      # Обращаемся к контейнеру db по имени!
      - DATABASE_URL=postgres://user:pass@db:5432/mydb
  
  db:
    image: postgres
```

### Типы сетей
| Тип | Описание | Когда использовать |
|-----|----------|-------------------|
| bridge | Изолированная сеть (по умолчанию) | Обычные приложения |
| host | Сеть хоста напрямую | Максимальная производительность |
| none | Без сети | Полная изоляция |

### Изоляция сервисов
```yaml
# Frontend видит только API, база изолирована
services:
  frontend:
    networks: [frontend]
  api:
    networks: [frontend, backend]
  db:
    networks: [backend]

networks:
  frontend:
  backend:
    internal: true  # нет доступа в интернет
```

### 📝 Фраза для интервью
> "Docker создаёт виртуальные сети. Контейнеры в одной сети обращаются друг к другу по имени контейнера. Для безопасности создаём отдельные сети для разных уровней приложения — база данных не должна быть доступна напрямую из интернета."

---

## 7. Docker Compose

### 🎯 Что спрашивают
> "Что такое docker-compose и когда его использовать?"

### Простое объяснение
Вместо того чтобы запускать 5 контейнеров 5 командами с кучей флагов, описываем всё в одном файле и запускаем одной командой.

### Пример полного стека

```yaml
version: '3.8'

services:
  web:
    build: ./frontend
    ports:
      - "3000:3000"
    depends_on:
      - api
  
  api:
    build: ./backend
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/mydb
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis
  
  db:
    image: postgres:14
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=secret
  
  redis:
    image: redis:alpine

volumes:
  postgres_data:
```

```bash
docker compose up -d      # Запустить всё
docker compose logs -f    # Логи всех сервисов
docker compose down       # Остановить всё
```

> ⚠️ **`docker compose` (v2, без дефиса) — стандарт.** Это Go-based плагин к Docker CLI, встроенный по умолчанию в Docker Desktop и современный Docker Engine. Старый `docker-compose` (v1, на Python) в EOL и обновлений не получает — если видите его в чужом проекте или CI, это сигнал легаси-тулинга, стоит мигрировать. В Compose v2 поле `version:` в начале файла необязательно (Compose Specification объединила версии в одну) и в новых файлах обычно опускается.

### 📝 Фраза для интервью
> "Docker Compose — это инструмент для запуска многоконтейнерных приложений. Описываем все сервисы, сети и volumes в YAML файле и управляем ими как единым целым. Идеален для разработки и тестирования. Использую `docker compose` (v2, встроенный CLI-плагин) — легаси `docker-compose` через дефис уже EOL."

---

## 8. Оптимизация образов

### 🎯 Что спрашивают
> "Как уменьшить размер Docker образа?"

### Чек-лист оптимизации

1. **Используйте slim/alpine образы**
   ```dockerfile
   # ❌ 900 MB
   FROM python:3.11
   # ✅ 120 MB
   FROM python:3.11-slim
   # ✅ 50 MB
   FROM python:3.11-alpine
   ```

2. **Объединяйте RUN команды**
   ```dockerfile
   # ❌ 3 слоя
   RUN apt-get update
   RUN apt-get install -y curl
   RUN rm -rf /var/lib/apt/lists/*
   
   # ✅ 1 слой
   RUN apt-get update && \
       apt-get install -y curl && \
       rm -rf /var/lib/apt/lists/*
   ```

3. **Используйте .dockerignore**
   ```
   node_modules
   .git
   *.md
   tests/
   ```

4. **Multistage builds** для компилируемых языков

5. **Идите дальше alpine — distroless/hardened minimal образы** для финального stage (подробно в разделе 9): меньше пакетов внутри = меньше CVE-поверхность, а не только меньше мегабайт.

### 📝 Фраза для интервью
> "Для оптимизации: выбираю slim/alpine или, для production, distroless/hardened minimal базовые образы, объединяю RUN команды чтобы уменьшить слои, использую .dockerignore, применяю multistage для компилируемых языков. Это может уменьшить образ с гигабайт до десятков мегабайт и заодно резко сократить количество CVE, которые нужно патчить."

---

## 9. Distroless и Hardened Minimal образы

### 🎯 Что спрашивают на интервью
> "Чем distroless отличается от alpine? Зачем нужны 'hardened' образы?"

### Простое объяснение
`alpine` уже сильно уменьшает образ по сравнению с Debian/Ubuntu, но внутри всё ещё остаётся полноценный shell, package manager (`apk`) и набор системных утилит — потенциальная attack surface, если атакующий получит RCE в приложении. **Distroless**-образы (концепция Google, продолженная проектами вроде Chainguard/Wolfi) идут на шаг дальше: в финальном образе нет shell'а, package manager'а и большинства бинарей — только рантайм и ваше приложение.

### Сравнение подходов

| База | Что внутри | Размер | Attack surface | Когда использовать |
|------|-----------|--------|-----------------|---------------------|
| `debian`/`ubuntu` | Полная ОС, shell, apt, coreutils | Сотни MB — GB | Большой | Легаси, нужны системные тулы для отладки внутри контейнера |
| `alpine` | Минимальный shell (busybox), apk | Единицы-десятки MB | Средний | Универсальный компромисс, но musl libc иногда ломает совместимость (некоторые Python wheels, glibc-зависимости) |
| **Distroless** (`gcr.io/distroless/*`) | Только рантайм (напр. libc + бинарник), без shell | Ещё меньше alpine | Минимальный | Production, compiled-языки, финальный stage multistage build |
| **Chainguard/Wolfi-style "hardened minimal"** | distroless + непрерывный ребилд под свежие CVE-патчи, SBOM из коробки | Сравнимо с distroless | Минимальный + низкий CVE-счётчик | Compliance-чувствительные production-системы, где важна provenance "из коробки" |

### Пример: distroless в multistage build

```dockerfile
FROM golang:1.22 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o main .

# Никакого shell, никакого apk/apt — только бинарник и минимальный рантайм
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /app/main /main
ENTRYPOINT ["/main"]
```

### Компромисс, который стоит проговорить на интервью
- **Минус**: нет shell → нельзя просто `docker exec -it container sh` для дебага; отладка требует ephemeral debug-контейнеров (`kubectl debug` с отдельным debug-образом) или sidecar-контейнеров.
- **Плюс**: резко меньше CVE на образ (меньше пакетов = меньше что сканировать и патчить), меньше размер → быстрее pull/деплой, меньше поверхность атаки при компрометации приложения.
- На практике distroless/hardened-минимальные образы — ожидаемый стандарт для production-сервисов на Go/Rust/статически линкуемой Java (native image); для интерпретируемых языков (Python/Node) полный distroless сложнее (нужен интерпретатор), но Chainguard/Wolfi-based образы всё чаще покрывают и этот случай.

### 📝 Фраза для интервью
> "Alpine — хороший компромисс по размеру, но внутри всё ещё живой shell и package manager. Distroless и hardened-минимальные образы (Chainguard/Wolfi) убирают их полностью, оставляя только рантайм — это резко снижает CVE-поверхность и размер образа. Плачу за это усложнённым дебагом в runtime (нет `sh` внутри), поэтому для отладки использую ephemeral debug-контейнеры, а не exec в prod-контейнер напрямую."

---

## 10. Rootless-контейнеры и non-root по умолчанию

### 🎯 Что спрашивают на интервью
> "Почему нельзя запускать контейнер от root? Что такое rootless Docker?"

### Простое объяснение
По умолчанию процесс внутри контейнера исторически запускался от `root` (UID 0). Если атакующий вырвется из приложения (RCE) и затем найдёт способ выйти за границы контейнера (container escape через уязвимость в runtime/ядре), root внутри контейнера часто маппится на root снаружи — урон резко больше. К 2025-2026 запуск от non-root — это baseline-ожидание security review, а не "nice to have".

### Два разных уровня non-root

| Уровень | Что решает | Как включить |
|---------|-----------|---------------|
| **Non-root user внутри контейнера** | Процесс приложения не root даже внутри своего namespace | `USER` в Dockerfile |
| **Rootless Docker daemon** | Сам `dockerd` и все контейнеры работают под непривилегированным пользователем хоста, без root-демона вообще | `dockerd-rootless-setuptool.sh install` |

### Non-root user в Dockerfile

```dockerfile
FROM node:20-slim

RUN addgroup --system app && adduser --system --ingroup app app
WORKDIR /app
COPY --chown=app:app . .
RUN npm ci --omit=dev

USER app
CMD ["node", "server.js"]
```

Для distroless-образов (раздел 9) есть готовый тег `nonroot` (`gcr.io/distroless/static-debian12:nonroot`) с уже настроенным непривилегированным пользователем — создавать вручную не нужно.

### Дополнительное hardening (что ещё спрашивают senior-кандидатов)

```bash
docker run \
  --read-only \                        # файловая система контейнера read-only
  --tmpfs /tmp \                       # кроме явно разрешённых writable-путей
  --cap-drop=ALL \                     # убрать все Linux capabilities...
  --cap-add=NET_BIND_SERVICE \         # ...и вернуть только нужные
  --security-opt no-new-privileges \   # запретить эскалацию привилегий через setuid
  myapp
```

- **`--cap-drop=ALL` + точечный `--cap-add`** — принцип least privilege на уровне Linux capabilities, а не всё-или-ничего root/non-root.
- **`--read-only`** — если приложению не нужно писать на диск (кроме `/tmp`/явных volume), read-only файловая система лишает атакующего возможности закрепиться в контейнере.
- В Kubernetes то же самое декларативно через `securityContext` (`runAsNonRoot: true`, `readOnlyRootFilesystem: true`, `capabilities.drop: ["ALL"]`) — именно это проверяют admission-политики (Pod Security Standards, OPA Gatekeeper, Kyverno) в hardened-кластерах.

### 📝 Фраза для интервью
> "Non-root — это baseline, не опция. В Dockerfile всегда добавляю непривилегированного `USER`, для distroless беру готовый tag `:nonroot`. На уровне рантайма добавляю `--cap-drop=ALL` с точечным возвратом нужных capabilities и `--read-only` файловую систему где возможно. В Kubernetes то же самое задаю через `securityContext` — и это то, что enforce'ят admission-политики уровня кластера, а не просто договорённость команды."

---

## 11. Supply Chain Security образов: SBOM и подпись

### 🎯 Что спрашивают на интервью
> "Как убедиться, что образ, который деплоится в production, — это именно тот образ, что собрала ваша CI, и в нём нет известных уязвимых зависимостей?"

### Простое объяснение
"Software supply chain security" — про доверие ко всей цепочке: от исходников и зависимостей до собранного артефакта и того, что реально задеплоено. Для Docker-образов это сводится к двум связанным практикам: **SBOM** (что внутри образа) и **подпись/provenance** (кто и как это собрал, не подменили ли образ по дороге).

### SBOM (Software Bill of Materials)
Список всех пакетов/библиотек/версий внутри образа — как "состав продукта" на упаковке. Нужен для: быстрого ответа на "используем ли мы уязвимую версию условного log4j/openssl" без пересборки и пересканирования, compliance-требований (всё больше регуляторов и enterprise-заказчиков требуют SBOM), и как вход для сканеров уязвимостей.

```bash
# Docker Scout — встроен в Docker CLI, даёт SBOM + CVE-анализ образа
docker scout sbom myapp:latest
docker scout cves myapp:latest

# syft — популярный open-source генератор SBOM (форматы SPDX/CycloneDX),
# провайдер-агностичный, часто используется как шаг в CI
syft myapp:latest -o spdx-json > sbom.json
```

### Подпись образов: cosign / Sigstore
Даже с идеальным SBOM остаётся вопрос: "этот тег в registry — точно то, что собрала моя CI, а не что-то подменённое?" Подпись криптографически привязывает конкретный digest к тому, кто его собрал.

```bash
# Sigstore/cosign — де-факто стандарт подписи OCI-образов,
# часто с keyless-подписью через OIDC (identity CI-job'а, а не долгоживущий приватный ключ)
cosign sign --yes myregistry/myapp@sha256:abcdef...

# Верификация перед деплоем — можно встроить как admission-контроль в Kubernetes
cosign verify \
  --certificate-identity="https://github.com/myorg/myrepo/.github/workflows/ci.yml@refs/heads/main" \
  --certificate-oidc-issuer="https://token.actions.githubusercontent.com" \
  myregistry/myapp@sha256:abcdef...
```

### Зачем это реально нужно
- **Reproducibility/provenance** — доказуемо, что образ в production собран из проверенного source, а не подменён на каком-то этапе (compromised registry, MITM, инсайдер с доступом в registry).
- **Быстрая реакция на 0-day** — при новой критичной CVE в популярной библиотеке SBOM позволяет за минуты найти все затронутые образы по всей организации, вместо ручного аудита.
- **Compliance** — фреймворки вроде SLSA (см. также файл про CI/CD и DevOps) и регуляторные требования всё чаще прямо требуют SBOM и provenance для критичного ПО.
- В Kubernetes это можно **enforce**, а не просто "иметь": admission-контроллер (Kyverno, OPA Gatekeeper) отклоняет под, если образ не подписан доверенным ключом/identity или не проходит SBOM-based vulnerability gate.

### 📝 Фраза для интервью
> "SBOM — это состав образа: какие пакеты и версии внутри, генерирую через `syft` или `docker scout sbom` как шаг в CI. Подпись через cosign/Sigstore, обычно keyless через OIDC-identity CI-джобы, а не статический ключ — доказывает, что задеплоенный образ реально собран моим пайплайном, а не подменён. В hardened-кластерах это не просто документация, а enforced admission-политика: под с неподписанным образом или с критичными CVE просто не запустится."

---

## 12. Dev Containers — воспроизводимые окружения разработки

### 🎯 Что спрашивают на интервью
> "Как быстро онбордить нового разработчика без 'у меня не собирается' на его машине?"

### Простое объяснение
Тот же принцип, что "работает одинаково в проде", применили к **локальной разработке**: `devcontainer.json` описывает окружение разработки (базовый образ, тулы, extensions, post-create команды) как код, и любой IDE/инструмент, поддерживающий Dev Containers Specification, поднимает идентичное окружение в контейнере.

```jsonc
// .devcontainer/devcontainer.json
{
  "name": "backend-api",
  "image": "mcr.microsoft.com/devcontainers/python:3.12",
  "features": {
    "ghcr.io/devcontainers/features/docker-in-docker:2": {},
    "ghcr.io/devcontainers/features/node:1": { "version": "20" }
  },
  "postCreateCommand": "pip install -r requirements-dev.txt && pre-commit install",
  "customizations": {
    "vscode": {
      "extensions": ["ms-python.python", "charliermarsh.ruff"]
    }
  },
  "forwardPorts": [8000, 5432],
  "remoteUser": "vscode"
}
```

### Почему это стало стандартом
- **Спецификация открытая** (Dev Containers Specification) — не привязана к одному вендору: поддерживается VS Code, JetBrains, GitHub Codespaces, а также CLI (`devcontainer` CLI) для CI.
- **Онбординг за минуты**: клонировал репозиторий → "Reopen in Container" → получил тот же Python/Node/системные библиотеки/линтеры/pre-commit hooks, что у всей команды, без "инструкции на 50 шагов" в README.
- **Ближе к прод-окружению**, чем venv/nvm на голой машине разработчика — тот же базовый образ можно переиспользовать (или держать похожим на) образ, который реально деплоится.
- Естественно расширяется в облачные ephemeral dev-окружения (GitHub Codespaces и аналоги) — та же конфигурация поднимает и локальный контейнер, и облачную VM с контейнером.

### 📝 Фраза для интервью
> "Dev Containers решают ту же проблему, что Docker в проде, только для локальной разработки — `devcontainer.json` в репозитории описывает окружение как код: базовый образ, тулы, extensions, post-create шаги. Любой разработчик через VS Code, JetBrains или Codespaces получает идентичное окружение одной командой 'Reopen in Container', вместо ручной настройки и вечного 'у меня не воспроизводится'."

---

## 13. Контейнеры в эпоху AI-агентов

### 🎯 Что спрашивают на интервью
> "Как безопасно выполнять код, который сгенерировал или предложил AI-агент?"

### Простое объяснение
Два новых, но уже частых на практике use case, где контейнеризация — стандартный ответ:

**1. Локальный inference LLM.** Открытые модели (класса Llama/Mistral/Qwen) часто гоняют локально или on-prem через контейнеризованные рантаймы (например, Ollama или vLLM в Docker-образе) — тот же паттерн "воспроизводимое окружение + изоляция зависимостей (версии CUDA, драйверы)", что и для любого другого сервиса, но с GPU passthrough (`--gpus all` / NVIDIA Container Toolkit).

**2. Sandboxing для AI-агентов, исполняющих код.** Агенты (code-interpreter функциональность, автономные coding-агенты) на практике выполняют сгенерированный или подсказанный LLM код — а это по сути произвольный, недоверенный код, который нельзя пускать в основное окружение. Контейнер — минимальный практичный барьер изоляции для этого:

```bash
# Типичный паттерн: эфемерный, максимально урезанный контейнер
# на одно выполнение агентского кода
docker run --rm \
  --network none \                  # без сети, если агенту не нужен внешний доступ
  --read-only --tmpfs /tmp \
  --cap-drop=ALL \
  --memory=512m --cpus=1 \
  --pids-limit=100 \                # защита от fork-бомб
  agent-sandbox:python-3.12 \
  python /workspace/generated_script.py
```

Что важно проговорить на интервью:
- Обычный контейнер делит ядро с хостом — для **действительно недоверенного** кода (например, multi-tenant платформа, где код от разных внешних пользователей/агентов выполняется на общей инфраструктуре) senior-уровня ответ — усилить изоляцию поверх обычных контейнеров: **gVisor** (userspace-эмуляция syscalls) или **Firecracker** microVMs (тот же принцип, что использует AWS Lambda) — тоньше и быстрее полной VM, но с изоляцией, близкой к VM, а не только namespaces/cgroups.
- Ключевые ограничения для агентского sandbox: `--network none` или строгий egress allow-list (иначе агент с сетевым доступом может стать вектором exfiltration — см. раздел про indirect prompt injection в файле про Prompt Engineering), жёсткие лимиты CPU/memory/pids, `--read-only` файловая система, короткий TTL контейнера (эфемерный, на одно задание, не переиспользуемый между запросами разных пользователей).
- Это прямое пересечение с разделами 10-11 (rootless, capabilities, read-only fs, подпись образов) — просто применённое к модели угрозы "недоверенный код", а не только defense-in-depth для "моего же приложения".

### 📝 Фраза для интервью
> "В контексте AI вижу два новых применения контейнеров: контейнеризованный inference локальных LLM (Ollama/vLLM с GPU passthrough) — обычная задача воспроизводимого окружения; и sandboxing для кода, который выполняют AI-агенты — это уже задача изоляции недоверенного кода. Для неё беру эфемерный контейнер с `--network none`, `--cap-drop=ALL`, жёсткими лимитами ресурсов и read-only FS, а для по-настоящему multi-tenant сценариев усиливаю изоляцию через gVisor или Firecracker microVMs, а не полагаюсь только на namespaces/cgroups обычного контейнера."

---

## 🎤 Частые вопросы на интервью

### "Что такое Docker?"
> "Платформа для контейнеризации. Упаковывает приложение с зависимостями в изолированный контейнер, который работает одинаково на любой машине."

### "Чем образ отличается от контейнера?"
> "Образ — это шаблон, read-only, как класс в ООП. Контейнер — это запущенный экземпляр образа, как объект класса. Из одного образа можно запустить много контейнеров."

### "Что такое слои в Docker?"
> "Каждая инструкция в Dockerfile создаёт слой. Слои кешируются — если слой не изменился, он не пересобирается. Поэтому важно размещать редко меняющиеся инструкции выше."

### "Как передать секреты в контейнер?"
> "Через переменные окружения (-e) для некритичных данных, Docker secrets для production в Swarm, или монтирование файлов с секретами. Никогда не хардкодить в Dockerfile!"

### "Что такое health check?"
> "Проверка здоровья контейнера. Docker периодически выполняет команду и помечает контейнер healthy/unhealthy. Это нужно для оркестраторов чтобы перезапускать нездоровые контейнеры."

### "Чем ENTRYPOINT отличается от CMD?"
> "CMD — команда по умолчанию, легко перезаписывается при docker run. ENTRYPOINT — основная команда, CMD становится её аргументами. ENTRYPOINT для исполняемых файлов, CMD для параметров."

### "Что такое Docker Daemon?"
> "Фоновый процесс, который управляет Docker объектами: образами, контейнерами, сетями, volumes. Docker CLI общается с daemon через REST API."

### "Как уменьшить размер образа?"
> "Использовать slim/alpine базовые образы, multistage builds, объединять RUN команды, удалять кеш пакетных менеджеров, использовать .dockerignore."

### "Что происходит при docker run?"
> "Docker проверяет наличие образа локально, если нет — скачивает из registry. Создаёт writable layer поверх образа, настраивает сеть и storage, запускает процесс из CMD/ENTRYPOINT."

### "Как отладить контейнер который сразу падает?"
> "docker logs container_name для просмотра логов. docker run -it image sh для запуска с интерактивной оболочкой. docker run --entrypoint sh image для переопределения entrypoint."

### "Что такое Docker context?"
> "Все файлы и папки, которые отправляются Docker daemon при сборке. Большой context замедляет сборку. Используйте .dockerignore для исключения ненужных файлов."

### "Как обновить контейнер без downtime?"
> "Blue-green deployment: запустить новый контейнер, переключить traffic, остановить старый. Или rolling update в Docker Swarm/Kubernetes."

### "Docker Swarm vs Kubernetes?"
> "Swarm встроен в Docker, проще в настройке, для небольших кластеров. Kubernetes более мощный, больше возможностей, industry standard, но сложнее."

### "Что такое dangling images?"
> "Образы без тега, обычно остаются после пересборки. Занимают место. Удалить: docker image prune. Все неиспользуемые: docker system prune."

### "Как ограничить ресурсы контейнера?"
> "Флаги --memory, --cpus при docker run. Например: docker run --memory=512m --cpus=0.5 nginx. Важно для production чтобы один контейнер не съел все ресурсы."

### "Что такое Docker registry?"
> "Хранилище образов. Docker Hub — публичный registry. Можно поднять приватный (Harbor, GitLab Registry) или использовать облачные (ECR, GCR, ACR)."

### "Как работает docker build cache?"
> "Каждая инструкция создаёт слой с хешем. Если файлы и команда не изменились — используется кеш. При изменении слоя все последующие пересобираются. Поэтому COPY package.json перед COPY ."

### "Можно ли запустить GUI приложение в Docker?"
> "Да, через X11 forwarding или VNC. На Linux: docker run -e DISPLAY -v /tmp/.X11-unix:/tmp/.X11-unix. Но это редкий use case."

### "Что такое OCI?"
> "Open Container Initiative — стандарт для container runtime и image format. Docker соответствует OCI, поэтому образы совместимы с другими runtime (containerd, CRI-O)."

### "Чем BuildKit отличается от legacy builder?"
> "BuildKit — текущий дефолтный билдер Docker (не opt-in уже несколько лет), даёт параллельную сборку независимых стадий multistage-билда, cache mounts (кеш пакетного менеджера между сборками без запекания в слой) и `--mount=type=secret` для секретов на этапе сборки без утечки в историю образа. Legacy builder остался только для обратной совместимости."

### "docker-compose или docker compose?"
> "`docker compose` (v2, без дефиса) — стандарт: Go-based плагин, встроенный в Docker CLI. Legacy `docker-compose` (v1, Python) в EOL. В Compose v2 поле `version:` в файле уже не нужно — спецификация объединена."

### "Зачем нужны distroless образы, если уже есть alpine?"
> "Alpine всё ещё содержит shell и package manager — рабочую поверхность для атакующего после RCE. Distroless и hardened-минимальные образы (Chainguard/Wolfi) убирают их полностью, оставляя только рантайм и бинарник — меньше CVE, меньше размер, но и сложнее дебажить (нет `sh` внутри, нужны ephemeral debug-контейнеры)."

### "Почему нельзя просто гонять контейнер от root?"
> "Root внутри контейнера при container escape часто маппится на root снаружи — урон резко больше. Non-root user через `USER` в Dockerfile — baseline, плюс `--cap-drop=ALL` с точечным `--cap-add`, `--read-only` файловая система, а в Kubernetes — то же самое через `securityContext`, что enforce'ят admission-политики кластера."

### "Что такое SBOM и зачем он нужен?"
> "Software Bill of Materials — список всех пакетов и версий внутри образа. Генерирую через `syft` или `docker scout sbom` как шаг CI. Нужен для быстрого поиска затронутых образов при новой CVE и для compliance. Дополняется подписью образа (cosign/Sigstore) — доказательством, что задеплоенный образ реально собран моей CI, а не подменён."

### "Что такое Dev Containers?"
> "`devcontainer.json` в репозитории описывает окружение разработки как код — базовый образ, тулы, extensions, post-create шаги. Открытая спецификация, поддержана VS Code, JetBrains, GitHub Codespaces. Даёт идентичное окружение любому разработчику одной командой, вместо ручного онбординга."

### "Как безопасно выполнять код, который сгенерировал AI-агент?"
> "Эфемерный контейнер на одно выполнение: `--network none` (или строгий egress allow-list), `--read-only`, `--cap-drop=ALL`, жёсткие лимиты CPU/memory/pids-limit. Для настоящего multi-tenant сценария с недоверенным кодом от разных пользователей — усиливаю изоляцию через gVisor или Firecracker microVM, а не только namespaces/cgroups обычного контейнера."

---

## 📚 Чек-лист для подготовки

### Базовый уровень
- [ ] Контейнер vs виртуальная машина
- [ ] Основные команды CLI (run, exec, logs, ps)
- [ ] Структура Dockerfile (FROM, COPY, RUN, CMD)
- [ ] Volumes: named volume vs bind mount vs tmpfs
- [ ] Networks и DNS между контейнерами

### Средний уровень
- [ ] Multistage build
- [ ] BuildKit: cache mounts, `--mount=type=secret`, buildx
- [ ] `docker compose` (v2) vs legacy `docker-compose`
- [ ] Оптимизация размера образа (slim/alpine, .dockerignore, объединение RUN)
- [ ] ENTRYPOINT vs CMD, health checks

### Продвинутый уровень
- [ ] Distroless / hardened minimal образы (Chainguard/Wolfi) vs alpine
- [ ] Rootless-контейнеры, non-root по умолчанию, Linux capabilities (`--cap-drop`)
- [ ] Supply chain security: SBOM (syft, docker scout), подпись образов (cosign/Sigstore)
- [ ] Dev Containers (`devcontainer.json`) для reproducible dev-окружений
- [ ] Sandboxing недоверенного/AI-агентского кода (gVisor, Firecracker microVM)
- [ ] OCI-стандарт, container runtime (containerd, CRI-O)
