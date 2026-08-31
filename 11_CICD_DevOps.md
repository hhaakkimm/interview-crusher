# CI/CD и DevOps - Руководство для Senior Backend интервью

> 💡 **Как думать о CI/CD на интервью:**
> "CI/CD — это автоматизация пути от кода до production. Чем чаще деплоим, тем меньше риск. Маленькие изменения легче откатить. Infrastructure as Code для воспроизводимости."

---

## 1. CI/CD Pipeline

### 🎯 Что спрашивают
> "Опишите ваш CI/CD pipeline"

### Типичный Pipeline
```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│  Code   │───▶│  Build  │───▶│  Test   │───▶│ Deploy  │───▶│ Monitor │
│  Push   │    │         │    │         │    │ Staging │    │         │
└─────────┘    └─────────┘    └─────────┘    └────┬────┘    └─────────┘
                                                  │
                                                  ▼
                                            ┌─────────┐
                                            │ Deploy  │
                                            │  Prod   │
                                            └─────────┘
```

### GitHub Actions Example
```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest pytest-cov
      
      - name: Run tests
        run: pytest --cov=app --cov-report=xml
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build Docker image
        run: |
          docker build -t myapp:${{ github.sha }} .
      
      - name: Push to Registry
        run: |
          docker push myregistry/myapp:${{ github.sha }}

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Deploy to Staging
        run: |
          kubectl set image deployment/myapp \
            myapp=myregistry/myapp:${{ github.sha }}

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Deploy to Production
        run: |
          kubectl set image deployment/myapp \
            myapp=myregistry/myapp:${{ github.sha }}
```

### 📝 Фраза для интервью
> "CI автоматически запускает тесты при каждом push. CD деплоит на staging автоматически, на production — после approval. Каждый commit — потенциально готов к деплою."

---

## 2. Deployment Strategies

### 🎯 Что спрашивают
> "Какие стратегии деплоя вы знаете?"

### Rolling Update
```
Step 1: [v1] [v1] [v1] [v1]
Step 2: [v2] [v1] [v1] [v1]
Step 3: [v2] [v2] [v1] [v1]
Step 4: [v2] [v2] [v2] [v1]
Step 5: [v2] [v2] [v2] [v2]
```
✅ Нет downtime
✅ Постепенный rollout
❌ Две версии одновременно

### Blue-Green Deployment
```
        Load Balancer
             │
    ┌────────┴────────┐
    ▼                 ▼
┌────────┐      ┌────────┐
│  Blue  │      │ Green  │
│  (v1)  │      │  (v2)  │
│ ACTIVE │      │ STANDBY│
└────────┘      └────────┘

Switch: просто переключаем LB на Green
Rollback: переключаем обратно на Blue
```
✅ Instant rollback
✅ Тестируем v2 до переключения
❌ Двойные ресурсы

### Canary Deployment
```
                Load Balancer
                     │
         ┌───────────┴───────────┐
         │ 95%              5%   │
         ▼                   ▼
    ┌────────┐          ┌────────┐
    │  v1    │          │  v2    │
    │(stable)│          │(canary)│
    └────────┘          └────────┘

Постепенно увеличиваем трафик на v2:
5% → 10% → 25% → 50% → 100%
```
✅ Минимальный риск
✅ Реальное тестирование
❌ Сложнее в реализации

### Progressive Delivery — ожидаемый default, а не nice-to-have (2025-2026)

Canary/blue-green решают вопрос "как физически раскатить новую версию с минимальным риском". Отдельный, комплементарный вопрос — "когда фича становится видна пользователю", и в 2026 senior-уровня ответ разделяет эти два события явно:

```
Deploy  ≠  Release

Deploy  — новый код оказался в production-инфраструктуре (может быть выключен)
Release — фича реально видна/доступна пользователям

Feature flags разрывают жёсткую связку "задеплоил = включил всем":
  Deploy v2 (флаг OFF для всех) → канареечный % пользователей (флаг ON) →
  → постепенный rollout по сегментам → 100% → удаление флага из кода
```

```python
# Пример: feature flag поверх обычного деплоя — код уже в проде,
# но включается независимо от деплоя, точечно и обратимо
if feature_flags.is_enabled("new_checkout_flow", user=current_user):
    return new_checkout_flow(request)
return legacy_checkout_flow(request)
```

Почему это теперь "default", а не опциональная практика:
- **Мгновенный kill switch** без отката деплоя — если фича сломалась, выключаем флагом за секунды, не гоняем rollback пайплайна.
- **Progressive delivery = canary/rolling update (инфраструктурный уровень) + feature flags (продуктовый уровень) + автоматический анализ метрик** (error rate, latency, business-метрики) перед тем, как расширять % трафика/пользователей — часто автоматизировано инструментами вроде Argo Rollouts/Flagger, которые сами откатывают canary при деградации SLO.
- Ключевой вопрос на интервью: "что если фича сломала прод, а деплой отменять поздно/дорого?" — ответ senior-уровня: разделение deploy/release через флаги именно для этого и существует, откат — это toggle флага, а не redeploy.

### 📝 Фраза для интервью
> "Rolling update — стандарт в Kubernetes, постепенная замена подов. Blue-green для instant rollback. Canary для постепенного rollout с мониторингом метрик. В 2026 это база, а не потолок — поверх добавляю feature flags, которые разделяют deploy и release: код уезжает в прод выключенным, включаю его постепенно по сегментам и могу мгновенно выключить без отката деплоя. Progressive delivery — это canary/rolling на инфраструктурном уровне плюс feature flags на продуктовом, часто с автоматическим анализом метрик (Argo Rollouts/Flagger), а не просто ручное 'выкатили и смотрим'."

---

## 3. Infrastructure as Code

### 🎯 Что спрашивают
> "Что такое Infrastructure as Code?"

### Terraform Example
```hcl
# main.tf
provider "aws" {
  region = "us-west-2"
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  
  tags = {
    Name        = "web-server"
    Environment = "production"
  }
}

resource "aws_security_group" "web" {
  name = "web-sg"
  
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_rds_instance" "db" {
  identifier        = "mydb"
  engine            = "postgres"
  engine_version    = "14"
  instance_class    = "db.t3.micro"
  allocated_storage = 20
  
  username = var.db_username
  password = var.db_password
}
```

### Workflow
```bash
terraform init      # Initialize
terraform plan      # Preview changes
terraform apply     # Apply changes
terraform destroy   # Remove all
```

### Преимущества IaC
- **Version control** — история изменений в git
- **Reproducibility** — одинаковые окружения
- **Documentation** — код описывает инфраструктуру
- **Automation** — CI/CD для инфраструктуры

### 📝 Фраза для интервью
> "Infrastructure as Code — декларативное описание инфраструктуры. Terraform для cloud-agnostic, CloudFormation для AWS. Version control, code review для инфраструктуры как для кода."

---

## 4. Kubernetes Essentials

### 🎯 Что спрашивают
> "Объясните основные концепции Kubernetes"

### Архитектура
```
┌─────────────────────────────────────────────────────────┐
│                    Control Plane                         │
│  ┌─────────┐  ┌────────────┐  ┌─────────┐  ┌─────────┐ │
│  │   API   │  │ Controller │  │Scheduler│  │  etcd   │ │
│  │ Server  │  │  Manager   │  │         │  │         │ │
│  └─────────┘  └────────────┘  └─────────┘  └─────────┘ │
└─────────────────────────┬───────────────────────────────┘
                          │
         ┌────────────────┼────────────────┐
         ▼                ▼                ▼
    ┌─────────┐      ┌─────────┐      ┌─────────┐
    │  Node   │      │  Node   │      │  Node   │
    │ ┌─────┐ │      │ ┌─────┐ │      │ ┌─────┐ │
    │ │ Pod │ │      │ │ Pod │ │      │ │ Pod │ │
    │ └─────┘ │      │ └─────┘ │      │ └─────┘ │
    │ kubelet │      │ kubelet │      │ kubelet │
    └─────────┘      └─────────┘      └─────────┘
```

### Основные ресурсы

```yaml
# Deployment — управляет репликами
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:1.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
---
# Service — сетевой доступ к подам
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
---
# Ingress — внешний доступ
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp
            port:
              number: 80
```

### 📝 Фраза для интервью
> "Pod — минимальная единица, обычно один контейнер. Deployment управляет репликами и rolling updates. Service для внутреннего network discovery. Ingress для внешнего HTTP traffic."

---

## 5. Kubernetes Operations

### 🎯 Что спрашивают
> "Как управлять приложением в Kubernetes?"

### Полезные команды
```bash
# Просмотр
kubectl get pods
kubectl get deployments
kubectl describe pod myapp-xxx

# Логи
kubectl logs myapp-xxx
kubectl logs -f myapp-xxx --tail=100

# Отладка
kubectl exec -it myapp-xxx -- /bin/sh
kubectl port-forward myapp-xxx 8080:8080

# Scaling
kubectl scale deployment myapp --replicas=5

# Updates
kubectl set image deployment/myapp myapp=myapp:2.0
kubectl rollout status deployment/myapp
kubectl rollout undo deployment/myapp
```

### ConfigMaps и Secrets
```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  DATABASE_HOST: "postgres.default.svc"
  LOG_LEVEL: "info"
---
# Secret
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
type: Opaque
stringData:
  DATABASE_PASSWORD: "supersecret"
---
# Использование в Pod
spec:
  containers:
  - name: myapp
    envFrom:
    - configMapRef:
        name: myapp-config
    - secretRef:
        name: myapp-secrets
```

### 📝 Фраза для интервью
> "ConfigMaps для конфигурации, Secrets для sensitive данных (зашифрованы at rest). Helm для templating и package management. kubectl rollout undo для быстрого отката."

---

## 6. GitOps

### 🎯 Что спрашивают
> "Что такое GitOps?"

### Принцип
```
Git Repository (source of truth)
        │
        ▼
┌───────────────┐
│   ArgoCD /    │ ◀── Watches for changes
│   Flux        │
└───────┬───────┘
        │
        ▼ Syncs
┌───────────────┐
│  Kubernetes   │
│   Cluster     │
└───────────────┘
```

### ArgoCD Application
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myapp-config
    targetRevision: HEAD
    path: kubernetes
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### Workflow
```
1. Developer pushes code → triggers CI
2. CI builds image, pushes to registry
3. CI updates image tag in config repo
4. ArgoCD detects change
5. ArgoCD syncs cluster with config repo
```

### GitOps (pull) vs традиционный push-based CI/CD — ключевой контраст для интервью

К 2025-2026 GitOps — стандартный паттерн деплоя именно в Kubernetes, и на senior-интервью важно чётко объяснить, чем он принципиально отличается от "CI просто кладёт манифесты в кластер":

| | Push-based CI/CD (традиционный) | GitOps (pull-based) |
|---|----------------------------------|----------------------|
| **Кто применяет изменения в кластер** | CI-раннер сам вызывает `kubectl apply`/`helm upgrade` с внешними credentials в кластер | Контроллер **внутри** кластера (ArgoCD/Flux) сам подтягивает изменения из git |
| **Credentials в кластер** | CI/CD система должна иметь прямой сетевой доступ и права в production-кластер | Не нужны — только у контроллера внутри кластера есть доступ; CI лишь пишет в git-репозиторий конфигурации |
| **Источник истины** | Пайплайн (может быть запущен вручную, drift не отслеживается) | Git-репозиторий — единственный source of truth |
| **Drift detection** | Обычно нет — если кто-то поменял ресурс вручную (`kubectl edit`), никто не узнает | Контроллер непрерывно сверяет реальное состояние с git и может **self-heal** — вернуть ручное изменение обратно |
| **Откат** | Повторный запуск pipeline с нужной версией | `git revert` — контроллер сам приводит кластер к прежнему состоянию |
| **Аудит** | Логи CI-системы | История git-коммитов = полный auditable-лог того, что и когда должно быть в кластере |

Главная идея, которую стоит проговорить: в push-модели CI/CD — это "рука", которая тянется в прод и что-то там меняет (шире поверхность атаки, шире права токена CI). В GitOps направление обратное — **ничто снаружи кластера не имеет прав его менять**, контроллер внутри сам "тянет" (pull) желаемое состояние из git и непрерывно сверяет его с реальностью, а не применяет изменения одноразово в момент деплоя.

### 📝 Фраза для интервью
> "GitOps — Git как единый источник правды для инфраструктуры. Ключевое отличие от традиционного push-based CI/CD — направление доступа: в push-модели CI-раннер сам заходит в кластер с credentials и что-то там меняет, в GitOps контроллер (ArgoCD/Flux) живёт внутри кластера и сам подтягивает (pull) состояние из git — внешним системам вообще не нужны права на прямое изменение кластера. Это даёт continuous drift detection и self-heal — если кто-то поменял ресурс руками, контроллер это откатит к состоянию из git, — и откат через `git revert` вместо повторного запуска pipeline."

---

## 7. Testing in CI/CD

### 🎯 Что спрашивают
> "Какие тесты должны быть в pipeline?"

### Testing Pyramid
```
          /\
         /  \
        / E2E\          ← Мало, медленные
       /______\
      /        \
     /Integration\      ← Средне
    /______________\
   /                \
  /    Unit Tests    \  ← Много, быстрые
 /____________________\
```

### Pipeline Stages
```yaml
stages:
  - lint          # Seconds
  - unit-test     # Seconds-Minutes
  - build         # Minutes
  - integration   # Minutes
  - security-scan # Minutes
  - deploy-staging
  - e2e-test      # Minutes-Hours
  - deploy-prod
```

### Security Scanning
```yaml
# SAST (Static Application Security Testing)
- name: Run Bandit
  run: bandit -r app/

# Dependency Scanning
- name: Safety Check
  run: safety check -r requirements.txt

# Container Scanning
- name: Trivy Scan
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:${{ github.sha }}
```

### 📝 Фраза для интервью
> "Тесты в pipeline: lint, unit (fast feedback), integration, security scan, e2e. Fail fast — тяжёлые тесты в конце. Security: SAST для кода, dependency scan, container scan."

---

## 8. Supply Chain Security в пайплайне: SLSA, подпись, SBOM

### 🎯 Что спрашивают
> "Как убедиться, что артефакт, который деплоится в production, действительно собран вашей CI и не подменён по дороге?"

### Простое объяснение
"Shift security left" начиналось с SAST/dependency scan (см. раздел 7) — проверяем код и зависимости до деплоя. Следующий логичный шаг, ставший стандартом ожиданий к 2025-2026, — расширить эту идею на **сам процесс сборки и его выход**: не только "нет ли уязвимости в коде", но и "точно ли то, что задеплоено, — это то, что собрала именно эта CI, из именно этого коммита, без постороннего вмешательства".

### SLSA (Supply-chain Levels for Software Artifacts)

Фреймворк с несколькими уровнями зрелости supply chain — не нужно запоминать формальные критерии каждого уровня дословно, важна сама идея прогрессии:

```
Уровень 0: Нет гарантий — сборка вручную на чьём-то ноутбуке
Уровень 1: Сборка автоматизирована, есть provenance-метаданные (что, из чего, чем собрано)
Уровень 2: Сборка на управляемой CI-платформе, provenance подписана
Уровень 3+: Сборка в изолированном, защищённом от tampering окружении,
            provenance проверяема и не может быть подделана даже
            скомпрометированным пользователем CI
```

Практическая суть для интервью: чем выше уровень, тем сложнее незаметно подменить артефакт между "CI собрала" и "прод задеплоил" — даже если у атакующего есть доступ к части системы (например, к одному CI-джобу, но не к самой платформе CI).

### Что это значит как шаги пайплайна

```yaml
# Иллюстративный набор шагов supply-chain security в пайплайне
- name: Generate SBOM
  run: syft . -o cyclonedx-json > sbom.json

- name: Sign build artifact / image
  run: cosign sign --yes myregistry/myapp@${{ steps.build.outputs.digest }}

- name: Generate provenance attestation (SLSA)
  uses: slsa-framework/slsa-github-generator@v2

- name: Verify third-party dependencies before build
  run: |
    # проверка контрольных сумм/подписей пакетов, а не слепой install
    npm audit signatures
```

- **SBOM как шаг пайплайна**, а не разовая инвентаризация — генерируется на каждый билд, версионируется вместе с артефактом (подробнее и с примерами инструментов — файл про Docker).
- **Подпись артефактов** (образов, бинарников, пакетов) через cosign/Sigstore с verify-шагом перед деплоем — деплой отказывает, если подпись не проходит.
- **Верификация зависимостей** — не просто "просканировать на известные CVE" (это уже есть в разделе 7), а убедиться, что скачанный пакет — это действительно тот пакет, который заявлен (защита от атак вроде dependency confusion и compromised upstream maintainer).

### 📝 Фраза для интервью
> "Supply chain security — естественное продолжение shift-left: раньше проверяли код и зависимости на уязвимости, теперь дополнительно доказываем происхождение самого артефакта. SLSA даёт шкалу зрелости — от 'собрано на чьём-то ноутбуке' до 'сборка в изолированном окружении с проверяемой provenance'. На практике это генерация SBOM на каждый билд, подпись артефактов через cosign/Sigstore с обязательной верификацией перед деплоем, и проверка подлинности зависимостей, а не только их сканирование на известные CVE."

---

## 9. Platform Engineering и Internal Developer Platforms

### 🎯 Что спрашивают
> "Что такое Platform Engineering и чем это отличается от классического DevOps/SRE?"

### Простое объяснение
К середине 2020-х в средних и крупных инженерных организациях накопилась усталость от "you build it, you run it в чистом виде" — каждая продуктовая команда тратит много когнитивной нагрузки на воспроизведение одной и той же инфраструктурной обвязки (CI/CD, observability, secrets, provisioning окружений). **Platform Engineering** — отдельная дисциплина и часто отдельная команда, которая строит **Internal Developer Platform (IDP)**: самообслуживаемую платформу, через которую продуктовые команды получают инфраструктуру и инструменты без необходимости самим разбираться в Kubernetes/Terraform/observability-стеке во всех деталях.

```
Без платформы:                      С Internal Developer Platform:
Product team сам:                   Product team:
- пишет Terraform                   - выбирает "golden path" из каталога
- настраивает CI/CD с нуля          - получает готовый CI/CD, observability,
- разбирается в K8s манифестах        secrets management "из коробки"
- настраивает observability
- договаривается о secrets          Platform team:
                                     - строит и поддерживает golden paths
Итог: дублирование усилий,          - владеет платформой как продуктом
      непоследовательные практики,    для внутренних разработчиков
      высокая когнитивная нагрузка
```

### Golden Paths и Backstage

**Golden path** — заранее одобренный, документированный и поддерживаемый "рецепт" для типовой задачи (создать новый сервис, добавить очередь, задеплоить cron job) — не единственный технически возможный путь, но путь с наименьшим сопротивлением и максимальной поддержкой со стороны платформы.

**Backstage** (изначально open-sourced Spotify, сейчас проект CNCF) — самый известный фреймворк для построения такого self-service портала:
- **Software Catalog** — единый реестр всех сервисов организации, кто владелец, какие у него зависимости, где документация.
- **Software Templates** — golden path в виде формы: разработчик выбирает шаблон ("новый REST-сервис на Go"), заполняет несколько полей, получает готовый репозиторий с CI/CD, observability, преднастроенными dashboard'ами — без ручного копипаста boilerplate.
- **TechDocs** — документация как код, рядом с сервисом, а не в отдельной вики, которая устаревает.
- **Plugins** — интеграции с CI/CD, Kubernetes, cost-дашбордами, security-сканерами прямо в едином UI.

### Почему это важно на senior/staff-уровне

- Это организационный, а не только технический тренд: появляется отдельная роль/команда "Platform Engineer", метрика успеха которой — не аптайм одного сервиса, а **developer experience и скорость onboarding/delivery** для всех остальных команд (частая метрика — DORA-метрики продуктовых команд как показатель эффективности платформы).
- Отличие от классического DevOps/SRE: DevOps/SRE исторически ближе к "команда эксплуатирует прод и помогает с pipeline", Platform Engineering — "команда строит платформу как **продукт**, с собственным roadmap, API/UI и внутренними 'клиентами' — другими инженерными командами".
- На интервью может звучать как "как вы будете масштабировать инженерную культуру на 50+ команд без хаоса практик?" — хороший ответ упоминает golden paths и IDP как способ дать свободу и стандартизацию одновременно, вместо жёсткого мандата "все обязаны делать так" или полного laissez-faire.

### 📝 Фраза для интервью
> "Platform Engineering — ответ на то, что в крупной организации каждая команда иначе тратит время на переизобретение одной и той же инфраструктурной обвязки. Отдельная платформенная команда строит Internal Developer Platform — самообслуживаемый портал (классика — Backstage) с каталогом сервисов и 'golden paths': готовыми, поддерживаемыми шаблонами для типовых задач, куда встроены CI/CD, observability, secrets из коробки. Это снижает когнитивную нагрузку продуктовых команд и стандартизирует практики, не превращаясь в жёсткий мандат — платформенная команда относится к платформе как к продукту с собственными внутренними 'клиентами'."

---

## 10. AI в SDLC: AI-ревью кода и контроль AI-generated изменений

### 🎯 Что спрашивают
> "Как вы контролируете качество и безопасность кода, который предложил AI-ассистент, в вашем pipeline?"

### Простое объяснение
К 2026 заметная часть кода в PR так или иначе сгенерирована или предложена AI-ассистентом (автодополнение, agentic coding-инструменты, целые PR от автономных агентов). Это не заменило human review, но добавило пайплайну новый слой и новый процессный вопрос: **как ревьюить код, у которого нет "живого" автора в привычном смысле**, и как не дать скорости генерации кода обогнать скорость его осмысленной проверки.

### AI-ревьюеры в PR-пайплайне

```yaml
# Иллюстративный шаг: AI code review bot как gate в pipeline
- name: AI code review
  uses: some-ai-review-action@v1
  with:
    focus: "security, correctness, test coverage, style consistency"
    # AI-ревьюер оставляет комментарии в PR как ещё один reviewer,
    # но не блокирует merge единолично — это дополнение к human review, не замена
```

Такие боты (встроенные в GitHub/GitLab, отдельные SaaS, или self-hosted на базе LLM API) сейчас обычно выполняют роль **первого прохода**: ловят очевидные баги, забытые edge cases, несоответствие стилю, потенциальные security-проблемы — до того, как на PR потратит время человек-ревьюер. Это ускоряет цикл, но не отменяет его.

### Процессный вопрос, который реально волнует senior/tech lead в 2026

Ключевой сдвиг — не в том, что AI пишет код (это уже нормализовано), а в том, как команды **гейтят** это на уровне процесса:

| Правило | Зачем |
|---------|-------|
| **Автор PR обязан объяснить любое AI-предложенное изменение** | Если человек, приложивший PR, не может объяснить, почему код работает именно так — риск "cargo cult" изменений, которые никто в команде не понимает, до того как они сломаются в проде |
| **Повышенная строгость ревью для AI-generated diff'ов**, особенно больших/автономных | Автономные coding-агенты могут сгенерировать правдоподобно выглядящий, но тонко неверный код (некорректная обработка edge case, subtly wrong конкурентность) — "выглядит нормально" AI-код требует не меньше, а иногда больше скепсиса |
| **AI-ревью боты как дополнение, не замена approval человека** | Автоматический approve только от бота — антипаттерн; ответственность за смёрженный код всё равно на человеке-approver'е |
| **Явная маркировка AI-generated PR/коммитов** (где это возможно) | Помогает при аудите/инцидентах быстро понять, был ли участок кода результатом автономной генерации, и приоритизировать его при post-incident review |

### 📝 Фраза для интервью
> "AI-ревью боты в PR-пайплайне — полезный первый проход: ловят очевидные баги и security-проблемы до human review, но не заменяют approval человека. Более важный процессный вопрос для меня как tech lead — требование, чтобы автор PR мог объяснить любое AI-предложенное изменение: если никто в команде не понимает, почему код работает именно так, это риск, который проявится в проде, а не в code review. К большим или полностью автономно сгенерированным diff'ам применяю повышенную, а не сниженную строгость ревью — правдоподобно выглядящий AI-код может быть тонко неверным именно там, где человек расслабляется и меньше вчитывается."

---

## 🎤 Частые вопросы CI/CD

### "Как откатить неудачный деплой?"
> "Kubernetes: kubectl rollout undo. Blue-green: переключить LB. GitOps: git revert config change. Главное — автоматические health checks и быстрое обнаружение проблем."

### "Как управлять secrets в CI/CD?"
> "Никогда не в git. GitHub Secrets, GitLab CI Variables, HashiCorp Vault. External Secrets Operator для Kubernetes. Rotate регулярно."

### "Trunk-based vs GitFlow?"
> "Trunk-based: короткоживущие feature branches, частые мержи в main. GitFlow: долгие ветки develop/release. Для CI/CD лучше trunk-based — чаще интегрируемся, меньше конфликтов."

### "Как тестировать инфраструктуру?"
> "Terratest для Terraform — пишем Go тесты. Kitchen-Terraform. Checkov для security policies. Plan review как code review."

### "Zero-downtime deployment?"
> "Rolling update с readiness probes. Graceful shutdown (SIGTERM handling). Database migrations compatible с обеими версиями. Feature flags для постепенного включения."

### "В чём принципиальная разница между GitOps и обычным push-based CI/CD?"
> "Направление доступа. В push-модели CI-раннер сам заходит в кластер с credentials и меняет его — это лишняя точка входа и широкие права у CI. В GitOps контроллер (ArgoCD/Flux) живёт внутри кластера и сам вытягивает состояние из git — внешним системам вообще не нужны права менять кластер напрямую. Плюс GitOps даёт непрерывный drift detection и self-heal, чего у push-модели обычно нет."

### "Зачем разделять deploy и release, если у меня уже есть canary?"
> "Canary снижает риск на инфраструктурном уровне — постепенно раскатывает трафик на новую версию. Feature flags решают другую задачу — независимо включают/выключают саму фичу для конкретных пользователей/сегментов, без передеплоя. Вместе это progressive delivery: если фича сломалась, я выключаю флаг за секунды, а не откатываю деплой."

### "Что такое SLSA и зачем он нужен, если у нас уже есть SAST/dependency scan?"
> "SAST и dependency scan проверяют код и зависимости на известные уязвимости — это про содержимое. SLSA — про происхождение самого артефакта: доказуемо ли, что задеплоенный бинарник/образ собран именно этой CI из именно этого коммита, без подмены по дороге. Это шкала зрелости supply chain, а не разовая проверка, и на практике реализуется через SBOM на каждый билд плюс подпись артефактов (cosign/Sigstore) с обязательной верификацией перед деплоем."

### "Что такое Platform Engineering и Backstage?"
> "Platform Engineering — отдельная команда, которая строит Internal Developer Platform как продукт для внутренних разработчиков, а не просто эксплуатирует прод. Backstage — самый распространённый фреймворк для такой платформы: каталог сервисов, golden paths в виде шаблонов (Software Templates) с готовым CI/CD и observability из коробки, документация рядом с кодом. Цель — снизить когнитивную нагрузку продуктовых команд, не жертвуя стандартизацией практик."

### "Как вы ревьюите PR, который в основном сгенерирован AI-агентом?"
> "AI-ревью боты в пайплайне — полезный первый проход на очевидные баги и security-проблемы, но не замена human approval. Ключевое требование — автор PR должен уметь объяснить любое AI-предложенное изменение; если не может — это красный флаг, а не готовый к merge код. К большим автономно сгенерированным diff'ам применяю более строгий, а не более мягкий ревью — правдоподобный AI-код может быть тонко неверным именно там, где человек меньше вчитывается."

---

## 📚 Чек-лист для подготовки

### Базовый уровень
- [ ] Этапы типичного CI/CD pipeline (build, test, deploy)
- [ ] Rolling update, blue-green, canary — разница и trade-off'ы
- [ ] Основы Infrastructure as Code (Terraform workflow)
- [ ] Kubernetes: Pod, Deployment, Service, Ingress
- [ ] ConfigMaps и Secrets

### Средний уровень
- [ ] GitOps (ArgoCD/Flux): pull-based модель, drift detection, self-heal
- [ ] GitOps vs традиционный push-based CI/CD — явный контраст
- [ ] Feature flags и разделение deploy/release (progressive delivery)
- [ ] Testing pyramid и security-сканирование в pipeline (SAST, dependency, container scan)
- [ ] Trunk-based development vs GitFlow

### Продвинутый уровень
- [ ] Supply chain security: SLSA levels, подпись артефактов (cosign/Sigstore), SBOM как шаг pipeline
- [ ] Верификация происхождения зависимостей (dependency confusion, compromised upstream)
- [ ] Platform Engineering / Internal Developer Platform, Backstage, golden paths
- [ ] Метрики успеха платформенной команды (developer experience, DORA-метрики)
- [ ] AI-ревью в PR-пайплайне: роль как первого прохода, а не замены approval
- [ ] Процесс гейтинга AI-generated изменений (объяснимость автором, повышенная строгость ревью)
