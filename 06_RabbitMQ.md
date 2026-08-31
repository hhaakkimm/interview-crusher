# RabbitMQ - Руководство для технического интервью

> 💡 **Как объяснить RabbitMQ на интервью за 30 секунд:**
> "RabbitMQ — это брокер сообщений. Представьте почтовое отделение: отправитель бросает письмо в ящик, почта сортирует и доставляет получателю. Отправителю не нужно ждать, пока получатель заберёт письмо. Это даёт асинхронность и развязку между сервисами."

---

## 1. Событийная архитектура

### 🎯 Что спрашивают на интервью
> "Что такое message broker и зачем он нужен?"

### Синхронная vs Асинхронная архитектура

**Синхронно (HTTP):**
```
User → API → Payment Service → Email Service → Response
         |_____waiting_______|______waiting________|
```
Проблема: если Email Service упал — вся цепочка сломана.

**Асинхронно (Message Queue):**
```
User → API → Response (сразу!)
         ↓
      RabbitMQ → Payment Service
               → Email Service
               → Analytics Service
```

### Преимущества
- **Асинхронность**: ответ пользователю мгновенный
- **Развязка**: сервисы не знают друг о друге
- **Надёжность**: сообщение сохраняется, пока не обработано
- **Масштабирование**: добавляем consumer'ов при нагрузке

### 📝 Фраза для интервью
> "Message broker позволяет перейти от синхронной к асинхронной архитектуре. Сервисы обмениваются сообщениями через очередь, не зная друг о друге. Это даёт отказоустойчивость — если получатель недоступен, сообщение сохранится в очереди."

---

## 2. Компоненты AMQP Protocol

### 🎯 Что спрашивают
> "Объясните архитектуру RabbitMQ"

### Схема прохождения сообщения
```
Producer → Exchange → Binding → Queue → Consumer
              │
         routing key
```

### Компоненты простым языком

| Компонент | Аналогия | Назначение |
|-----------|----------|------------|
| **Producer** | Отправитель письма | Публикует сообщения |
| **Exchange** | Сортировочный центр | Маршрутизирует сообщения |
| **Binding** | Правило сортировки | Связывает exchange с queue |
| **Routing Key** | Адрес на конверте | Ключ для маршрутизации |
| **Queue** | Почтовый ящик | Хранит сообщения |
| **Consumer** | Получатель | Обрабатывает сообщения |
| **Channel** | Виртуальное соединение | Мультиплексирование внутри connection |

### Пример кода

```python
import pika

# Подключение
connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Объявляем инфраструктуру
channel.exchange_declare(exchange='orders', exchange_type='direct')
channel.queue_declare(queue='order_processing', durable=True)
channel.queue_bind(queue='order_processing', exchange='orders', routing_key='new_order')

# Producer: отправляем
channel.basic_publish(
    exchange='orders',
    routing_key='new_order',
    body='{"order_id": 123}'
)

# Consumer: получаем
def callback(ch, method, properties, body):
    print(f"Получено: {body}")
    ch.basic_ack(delivery_tag=method.delivery_tag)

channel.basic_consume(queue='order_processing', on_message_callback=callback)
channel.start_consuming()
```

### 📝 Фраза для интервью
> "Producer отправляет сообщение в Exchange с routing key. Exchange по правилам binding направляет сообщение в нужную Queue. Consumer подписывается на Queue и обрабатывает сообщения. Channel — это легковесное виртуальное соединение внутри TCP connection."

---

## 3. Типы Exchange

### 🎯 Что спрашивают
> "Какие типы Exchange есть в RabbitMQ?"

### Fanout — всем подряд
```
Exchange ─┬─► Queue1 (все сообщения)
          ├─► Queue2 (все сообщения)
          └─► Queue3 (все сообщения)
```
**Use case**: Уведомления всем сервисам, логирование

### Direct — точное совпадение
```
Exchange ─── routing_key="error" ──► error_queue
         └── routing_key="info"  ──► info_queue
```
**Use case**: Разные обработчики для разных типов событий

### Topic — паттерны
```
Exchange ─── "order.*.created" ──► new_orders (order.book.created, order.phone.created)
         └── "order.#"         ──► all_orders (любые order.*)
```
- `*` — ровно одно слово
- `#` — ноль или более слов

**Use case**: Гибкая подписка на события

### 📝 Фраза для интервью
> "Fanout рассылает всем подписчикам. Direct — точное совпадение routing key. Topic — паттерны со звёздочкой и решёткой. Выбор зависит от задачи: fanout для broadcast, direct для простой маршрутизации, topic для гибких подписок."

---

## 4. Acknowledgements — подтверждения

### 🎯 Что спрашивают
> "Как гарантировать, что сообщение обработано?"

### Проблема
Что если consumer получил сообщение и упал до обработки?

### Решение — ручные подтверждения

```python
def callback(ch, method, properties, body):
    try:
        process(body)  # Обрабатываем
        ch.basic_ack(delivery_tag=method.delivery_tag)  # ✅ Успех
    except Exception:
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)  # ❌ Вернуть в очередь

# ❌ Опасно: auto_ack=True — сообщение удаляется при получении
channel.basic_consume(queue='q', on_message_callback=callback, auto_ack=True)

# ✅ Безопасно: ручное подтверждение
channel.basic_consume(queue='q', on_message_callback=callback, auto_ack=False)
```

### Типы подтверждений
- **ack** — обработано успешно, удалить
- **nack** — ошибка, можно вернуть в очередь (requeue=True)
- **reject** — отклонить (для одного сообщения)

### 📝 Фраза для интервью
> "При auto_ack сообщение удаляется сразу при доставке — если consumer упадёт, данные потеряются. С ручным ack сообщение удаляется только после подтверждения обработки. При ошибке можно сделать nack с requeue для повторной попытки."

---

## 5. Prefetch — контроль нагрузки

### 🎯 Что спрашивают
> "Как распределить нагрузку между consumer'ами?"

### Проблема
```
Consumer1 [████████░░] — обрабатывает 8 тяжёлых задач
Consumer2 [░░░░░░░░░░] — простаивает
```
RabbitMQ по умолчанию раздаёт round-robin, не учитывая занятость.

### Решение — prefetch_count

```python
channel.basic_qos(prefetch_count=1)  # Не более 1 unacked сообщения
```

Теперь Consumer1 не получит новое сообщение, пока не сделает ack.

### Выбор значения
- `prefetch_count=1` — равномерное распределение, но много round-trips
- `prefetch_count=10-50` — баланс производительности и распределения
- Зависит от времени обработки сообщения

### 📝 Фраза для интервью
> "Prefetch ограничивает количество unacked сообщений на consumer. С prefetch=1 каждый consumer получает новое сообщение только после подтверждения предыдущего. Это обеспечивает честное распределение нагрузки."

---

## 6. Dead Letter Exchange (DLX)

### 🎯 Что спрашивают
> "Что делать с сообщениями, которые не удалось обработать?"

### Проблема
Если сообщение всегда вызывает ошибку, оно будет ходить по кругу вечно.

### Решение — Dead Letter Exchange
Сообщения "мёртвых писем" — тех, что не удалось доставить.

```python
# DLX — куда отправлять проблемные сообщения
channel.exchange_declare(exchange='dlx', exchange_type='direct')
channel.queue_declare(queue='dead_letters')
channel.queue_bind(queue='dead_letters', exchange='dlx', routing_key='dead')

# Основная очередь с DLX
channel.queue_declare(
    queue='orders',
    arguments={
        'x-dead-letter-exchange': 'dlx',
        'x-dead-letter-routing-key': 'dead'
    }
)
```

### Когда сообщение попадает в DLX
1. **Reject/Nack с requeue=False**
2. **TTL истёк**
3. **Очередь переполнена** (x-max-length)

### Use Case: Retry с задержкой
```
main_queue → reject → DLX → retry_queue (TTL=60s) → main_queue
```

### 📝 Фраза для интервью
> "Dead Letter Exchange — это 'кладбище' для проблемных сообщений. Сообщения попадают туда при reject, истечении TTL или переполнении очереди. Это позволяет анализировать ошибки и реализовать retry-логику."

---

## 7. Durability — сохранность данных

### 🎯 Что спрашивают
> "Что будет с сообщениями при перезапуске RabbitMQ?"

### Три уровня durability

1. **Durable Queue** — очередь сохраняется при рестарте
```python
channel.queue_declare(queue='orders', durable=True)
```

2. **Persistent Messages** — сообщения сохраняются на диск
```python
properties = pika.BasicProperties(delivery_mode=2)  # 2 = persistent
channel.basic_publish(exchange='', routing_key='orders', 
                      body=msg, properties=properties)
```

3. **Publisher Confirms** — подтверждение от брокера
```python
channel.confirm_delivery()
channel.basic_publish(...)  # Бросит exception если не доставлено
```

### Чек-лист для надёжной доставки
- ✅ `durable=True` для очереди
- ✅ `delivery_mode=2` для сообщений
- ✅ Publisher confirms
- ✅ Manual ack у consumer

### 📝 Фраза для интервью
> "Для сохранности нужны durable очереди и persistent сообщения. Но это не гарантирует доставку — нужны ещё publisher confirms и ручные ack у consumer. Это снижает производительность, поэтому выбор зависит от требований."

---

## 8. Масштабирование и HA

### 🎯 Что спрашивают
> "Как масштабировать RabbitMQ?"

### Масштабирование Consumer'ов
Просто запускаем больше consumer'ов — RabbitMQ распределит нагрузку.

### Clustering — несколько узлов
```bash
# На node2
rabbitmqctl stop_app
rabbitmqctl join_cluster rabbit@node1
rabbitmqctl start_app
```

### Quorum Queues — теперь единственный поддерживаемый тип реплицируемой durable очереди
```python
channel.queue_declare(
    queue='orders',
    arguments={'x-queue-type': 'quorum'}
)
```
Данные реплицируются на несколько узлов через **Raft consensus**, автоматический failover.

#### RabbitMQ 4.0 (август 2024) — конец classic mirrored queues

До версии 4.0 в RabbitMQ было два способа реплицировать очередь: старые **classic mirrored queues** (репликация на уровне Erlang-процессов, без формального консенсуса) и более новые **quorum queues** (Raft-based, добавлены в 3.8). Classic mirrored queues были помечены deprecated ещё в 3.x, а в **RabbitMQ 4.0 они полностью удалены** — quorum queues стали единственным рекомендованным и по факту единственным поддерживаемым типом реплицируемой durable очереди.

Почему это важно на интервью:
- Quorum queues дают формальные гарантии сохранности данных через Raft consensus (запись подтверждена, только если её видит большинство узлов) — mirrored queues такой гарантии не давали и были подвержены edge-case потере данных при split-brain.
- Если в опыте фигурирует RabbitMQ версии до 4.0 с mirrored queues, стоит явно сказать, что это устаревший подход, а миграционный путь — на quorum queues.
- Quorum queues пока не поддерживают некоторые фичи classic-очередей (например, priority queues) — в редких случаях это влияет на выбор, но для подавляющего большинства durable-очередей это уже не вопрос выбора, а стандарт.

### Khepri — новый metadata store (замена Mnesia)

Метаданные кластера (какие очереди/exchange'и/bindings существуют, где реплики) исторически хранились в **Mnesia** — встроенной Erlang-БД, которая плохо переживала split-brain и требовала ручного вмешательства при рассинхронизации узлов. **Khepri** — новый metadata store на основе Raft (тот же принцип консенсуса, что и у quorum queues), доступный как опциональная (пока не включённая по умолчанию везде) альтернатива Mnesia, начиная с недавних версий RabbitMQ.

Что важно знать: Khepri даёт более предсказуемое поведение при network partition (тот же формальный Raft-консенсус, что убрал двусмысленность и у очередей) — это логичное продолжение того же архитектурного сдвига, что и переход mirrored queues → quorum queues: RabbitMQ последовательно заменяет Erlang-специфичные, менее формальные механизмы репликации/консенсуса на Raft везде, где раньше был "best effort".

### Federation — между дата-центрами
Для географически распределённых систем.

### 📝 Фраза для интервью
> "Consumer'ы масштабируются добавлением инстансов. Для HA используем кластер с quorum queues — единственный поддерживаемый тип реплицируемой durable очереди начиная с RabbitMQ 4.0, где classic mirrored queues были полностью удалены как менее надёжный, не Raft-based механизм. На уровне метаданных кластера то же самое происходит с Khepri — Raft-based заменой legacy Mnesia, более устойчивой к split-brain. Federation — для связи между дата-центрами."

---

## 9. RabbitMQ Streams — log-based очереди (Kafka-подобная семантика)

### 🎯 Что спрашивают
> "Чем RabbitMQ Streams отличаются от обычных (classic/quorum) очередей?"

### Простое объяснение
**RabbitMQ Streams** — отдельный, log-based тип очереди (появился как feature preview в 3.9, к 2026 — зрелая, first-class возможность), концептуально близкий к Kafka: сообщения не удаляются сразу после обработки, а хранятся как append-only лог, который можно **перечитывать** (replay) с любой точки, и который несколько consumer'ов могут читать независимо, каждый со своим offset.

```python
# rabbitmq-stream-client (отдельный протокол, не классический AMQP 0-9-1)
from rstream import Producer, Consumer, ConsumerOffsetSpecification, OffsetType

async with Producer("localhost", username="guest", password="guest") as producer:
    await producer.send(stream="orders-stream", message=b'{"order_id": 123}')

async with Consumer("localhost", username="guest", password="guest") as consumer:
    await consumer.subscribe(
        stream="orders-stream",
        offset_specification=ConsumerOffsetSpecification(OffsetType.FIRST),  # replay с начала
    )
```

Когда выбирать Streams вместо classic/quorum очередей:

| Критерий | Classic/Quorum queue | Streams |
|----------|----------------------|---------|
| Модель | Сообщение удаляется после ack | Append-only log, сообщение хранится по retention-политике независимо от ack |
| Replay | Нет (once consumed — ушло) | Да, можно перечитать с любого offset |
| Множественные независимые consumer'ы одного потока | Нужно fanout-exchange + отдельные очереди на каждого | Нативно — offset у каждого consumer свой |
| Throughput | Хорош для типичных задач/очередей | Оптимизирован под очень высокий throughput (последовательное чтение с диска, zero-copy) |
| Типичный use case | Задачи, RPC, разовая обработка событий | Event sourcing, аудит-логи, high-throughput аналитика — тот же паттерн, что и Kafka topics |

### 📝 Фраза для интервью
> "RabbitMQ Streams — это log-based тип очереди внутри того же RabbitMQ-кластера, семантически близкий к Kafka: append-only, поддерживает replay, несколько consumer'ов читают независимо со своим offset, оптимизирован под высокий throughput. Использую его, когда нужен Kafka-подобный паттерн (event sourcing, replay, аудит), но не хочется поднимать отдельный Kafka-кластер, если RabbitMQ уже есть в инфраструктуре. Для классических задач/RPC/разовой обработки событий обычные quorum queues всё ещё более уместны и проще в эксплуатации."

---

## 10. RabbitMQ vs Kafka vs NATS vs Cloud-native очереди — как выбирать

### 🎯 Что спрашивают
> "Почему RabbitMQ, а не Kafka/NATS/managed cloud messaging?"

### Простое объяснение
К 2026 выбор брокера сообщений — это не "RabbitMQ vs Kafka" по умолчанию, как было раньше. Добавились как минимум два важных игрока: **NATS/NATS JetStream** как лёгкая альтернатива, и managed cloud-сервисы (SQS/SNS, EventBridge, Google Pub/Sub) как вариант вообще не эксплуатировать брокер самостоятельно.

| Решение | Сильные стороны | Когда выбрать |
|---------|------------------|----------------|
| **RabbitMQ** | Богатый роутинг (exchanges: direct/topic/fanout), зрелая экосистема, quorum queues для надёжности, Streams для log-семантики | Классическая межсервисная асинхронность со сложной маршрутизацией, self-hosted или в managed-варианте (CloudAMQP и др.) |
| **Kafka** | Durable log с долгим retention, огромная пропускная способность, зрелая экосистема (Connect, Schema Registry, Streams API) | Event streaming/event sourcing на масштабе компании, множество независимых consumer'ов одних и тех же событий, долгий replay |
| **NATS / NATS JetStream** | Очень лёгкий, простой в эксплуатации, низкая latency; JetStream добавляет persistence/replay поверх базового pub/sub | Микросервисы, где важна простота операций и низкий overhead, а не богатая маршрутизация RabbitMQ или экосистема Kafka; набирает популярность в cloud-native/Kubernetes-инфраструктуре как "лёгкий брокер по умолчанию" |
| **Managed cloud (SQS/SNS, EventBridge, Google Pub/Sub)** | Zero-ops — не нужно администрировать кластер, встроенное масштабирование, оплата по использованию | Команда уже в конкретном облаке и не хочет держать отдельную messaging-инфраструктуру; типичный default для cloud-native стартапов |

Честный вывод для интервью: самостоятельно хостить RabbitMQ или Kafka имеет смысл, когда нужен контроль над семантикой маршрутизации/retention, которого managed cloud-сервисы не дают, или когда стоимость масштаба перевешивает cloud-native pricing. Многие команды в 2026 стартуют с managed cloud messaging просто потому, что это резко снижает operational overhead, и переходят на self-hosted RabbitMQ/Kafka только когда упираются в конкретные ограничения (маршрутизация, throughput, стоимость на масштабе, требования к data residency).

### 📝 Фраза для интервью
> "RabbitMQ выбираю, когда нужна гибкая маршрутизация (topic/direct/fanout exchanges) и зрелая экосистема для классической межсервисной асинхронности. Kafka — когда нужен durable event log с долгим retention и много независимых consumer'ов одних и тех же событий. NATS/JetStream — когда важны простота эксплуатации и низкий overhead, особенно в cloud-native/Kubernetes-окружении, а богатая маршрутизация RabbitMQ не нужна. А managed cloud messaging (SQS/EventBridge/Pub-Sub) — когда команда уже в облаке и не хочет тратить operational bandwidth на администрирование брокера вообще — это всё чаще дефолтный первый выбор, а self-hosted брокер обосновывается конкретными ограничениями, а не берётся 'по умолчанию'."

---

## 🎤 Частые вопросы на интервью

### "Зачем нужен Message Broker?"
> "Развязка сервисов, асинхронная обработка, гарантированная доставка, балансировка нагрузки, устойчивость к пиковым нагрузкам."

### "RabbitMQ vs Kafka?"
> "RabbitMQ — традиционный брокер, умный роутинг, push-модель, сообщение удаляется после обработки. Kafka — distributed log, pull-модель, сообщения хранятся долго, для event streaming и replay событий."

### "Как гарантировать exactly-once?"
> "RabbitMQ гарантирует at-least-once или at-most-once. Для exactly-once нужна идемпотентность на стороне consumer — например, проверка по уникальному ID сообщения перед обработкой."

### "Что такое exchange default?"
> "Пустая строка '' — default exchange. Сообщения направляются напрямую в очередь, имя которой совпадает с routing_key. Удобно для простых случаев."

### "Как обрабатывать poison messages?"
> "Сообщения, которые всегда вызывают ошибку. Решение: счётчик retry в headers, после N попыток отправлять в DLX для ручного анализа."

### "Что такое Virtual Host?"
> "Логическое разделение брокера. У каждого vhost свои exchanges, queues, permissions. Используется для изоляции окружений: production, staging, разные приложения."

### "Как работает round-robin в RabbitMQ?"
> "По умолчанию сообщения распределяются по consumer'ам по очереди, независимо от их загрузки. Для честного распределения нужен prefetch_count=1."

### "Что такое Channel и зачем он нужен?"
> "Виртуальное соединение внутри TCP connection. Создание connection дорогое, поэтому мультиплексируем через channels. Типично: один channel на thread."

### "Как реализовать RPC через RabbitMQ?"
> "Отправляем запрос с correlation_id и reply_to (очередь для ответа). Сервер обрабатывает и отправляет ответ в reply_to с тем же correlation_id."

### "Что такое Quorum Queues?"
> "Очереди с репликацией на несколько узлов через Raft consensus. С RabbitMQ 4.0 (2024) полностью заменили classic mirrored queues, которые были удалены из ядра как менее надёжный механизм без формального консенсуса. Надёжнее и производительнее для HA — сейчас это стандарт, а не одна из опций."

### "Как мониторить RabbitMQ?"
> "Management plugin (веб-интерфейс), Prometheus + rabbitmq_exporter, rabbitmqctl команды. Отслеживать: queue depth, consumer count, message rates."

### "Что происходит когда очередь переполняется?"
> "Зависит от настроек: reject-publish (отклонять новые), drop-head (удалять старые). При x-max-length сообщения могут идти в DLX."

### "Как обеспечить порядок сообщений?"
> "Один consumer на очередь гарантирует порядок. При нескольких consumer'ах — использовать consistent hashing exchange или разные очереди по ключу."

### "Что такое Lazy Queues?"
> "Очереди, которые хранят сообщения на диске, не в памяти. Медленнее, но позволяют хранить миллионы сообщений без исчерпания RAM."

### "RabbitMQ vs Amazon SQS?"
> "RabbitMQ: self-hosted, сложный роутинг, больше контроля. SQS: managed, проще, интеграция с AWS, меньше операционных затрат."

### "Как масштабировать producer'ов?"
> "Producer'ы stateless, просто запускаем больше. Важно использовать connection pooling и не создавать connection на каждое сообщение."

### "Что такое Shovel plugin?"
> "Перемещает сообщения между брокерами. Используется для связи кластеров, миграции данных, backup очередей."

### "Как реализовать delayed messages?"
> "Через DLX с TTL: сообщение в очередь с TTL, после истечения попадает в основную очередь через DLX. Или плагин rabbitmq_delayed_message_exchange."

### "Что изменилось в RabbitMQ 4.0?"
> "Главное — classic mirrored queues полностью удалены; quorum queues (Raft-based) стали единственным поддерживаемым типом реплицируемой durable очереди. Это убирает менее надёжный, не консенсус-based механизм репликации в пользу формальных Raft-гарантий сохранности данных."

### "Что такое Khepri в RabbitMQ?"
> "Новый metadata store кластера на основе Raft — опциональная замена legacy Mnesia. Даёт более предсказуемое поведение при network partition/split-brain, продолжая тот же архитектурный сдвиг к Raft-консенсусу, что и quorum queues."

### "Чем RabbitMQ Streams отличаются от обычных очередей?"
> "Streams — log-based тип очереди: сообщения хранятся append-only и могут быть перечитаны (replay), несколько consumer'ов читают независимо со своим offset — семантика ближе к Kafka, чем к классической AMQP-очереди. Использую, когда нужен replay/event sourcing/высокий throughput внутри уже существующего RabbitMQ-кластера."

### "Когда выбрать NATS вместо RabbitMQ?"
> "Когда важны простота эксплуатации и низкий overhead больше, чем богатая маршрутизация RabbitMQ или экосистема Kafka — особенно в cloud-native/Kubernetes-инфраструктуре. NATS JetStream добавляет persistence/replay поверх лёгкого pub/sub, но экосистема и тулинг у него меньше, чем у RabbitMQ/Kafka."

### "Когда лучше взять managed cloud messaging вместо своего RabbitMQ?"
> "Когда команда уже в конкретном облаке и не хочет тратить operational bandwidth на администрирование брокера — SQS/EventBridge/Pub-Sub дают zero-ops масштабирование и оплату по использованию. Self-hosted RabbitMQ/Kafka обосновываю конкретными ограничениями managed-сервисов (гибкость маршрутизации, стоимость на масштабе, data residency), а не беру по умолчанию."

---

## 📚 Чек-лист для подготовки

### Базовый уровень
- [ ] Producer/Exchange/Binding/Queue/Consumer, AMQP-протокол
- [ ] Типы Exchange: fanout, direct, topic
- [ ] Acknowledgements: ack/nack/reject

### Средний уровень
- [ ] Prefetch и распределение нагрузки
- [ ] Dead Letter Exchange, retry с задержкой
- [ ] Durability: durable queue, persistent messages, publisher confirms

### Продвинутый уровень
- [ ] RabbitMQ 4.0: удаление classic mirrored queues, quorum queues как стандарт
- [ ] Khepri как Raft-based замена Mnesia
- [ ] RabbitMQ Streams: log-based семантика, replay, отличие от classic/quorum
- [ ] RabbitMQ vs Kafka vs NATS/JetStream vs managed cloud messaging — decision framework
- [ ] Масштабирование и HA, clustering

