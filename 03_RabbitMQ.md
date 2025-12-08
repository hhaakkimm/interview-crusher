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

### Quorum Queues (рекомендуется)
```python
channel.queue_declare(
    queue='orders',
    arguments={'x-queue-type': 'quorum'}
)
```
Данные реплицируются на несколько узлов, автоматический failover.

### Federation — между дата-центрами
Для географически распределённых систем.

### 📝 Фраза для интервью
> "Consumer'ы масштабируются добавлением инстансов. Для HA используем кластер с quorum queues — данные реплицируются, при отказе узла работа продолжается. Federation — для связи между дата-центрами."

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
> "Очереди с репликацией на несколько узлов через Raft consensus. Заменяют mirrored queues. Надёжнее и производительнее для HA."

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

