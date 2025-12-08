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

### 📝 Фраза для интервью
> "Rolling update — стандарт в Kubernetes, постепенная замена подов. Blue-green для instant rollback. Canary для постепенного rollout с мониторингом метрик. Выбор зависит от риска и требований."

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

### 📝 Фраза для интервью
> "GitOps — Git как единый источник правды для инфраструктуры. ArgoCD/Flux следят за repo и синхронизируют cluster. Declarative, auditable, легко откатить через git revert."

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
