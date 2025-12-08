# RabbitMQ - Полное руководство

## Содержание
1. [Событийная архитектура: producer/message/consumer](#событийная-архитектура)
2. [AMQ Protocol: publisher, channel, exchange, message, routing-key, queue, consumer](#amq-protocol)
3. [Get vs Consume](#get-vs-consume)
4. [Виды exchanges: fanout, direct, topic](#виды-exchanges)
5. [Basic.Ack, Basic.Reject, Basic.Nack](#acknowledgements)
6. [Prefetch и Consumer transactions](#prefetch-и-transactions)
7. [Message properties и Priority](#message-properties-и-priority)
8. [Dead Letter Exchanges](#dead-letter-exchanges)
9. [Автоматическое истечение queues/messages](#автоматическое-истечение)
10. [Temporary и Permanent queues](#temporary-и-permanent-queues)
11. [Speed vs Guaranteed delivery](#speed-vs-guaranteed-delivery)
12. [Virtual hosts](#virtual-hosts)
13. [Сжатие сообщений через gzip](#сжатие-сообщений)
14. [HA queues](#ha-queues)
15. [Scaling с clusters/Shovel plugin](#scaling-rabbitmq)
16. [Cross-cluster message distribution](#cross-cluster-distribution)

---

## Событийная архитектура

### Слабосвязанная архитектура
```
┌──────────┐     ┌─────────────┐     ┌──────────┐
│ Producer │────►│  RabbitMQ   │────►│ Consumer │
└──────────┘     │  (Broker)   │     └──────────┘
                 └─────────────┘
```

### Преимущества
- **Асинхронность** — producer не ждёт consumer
- **Масштабируемость** — добавление consumers
- **Отказоустойчивость** — сообщения сохраняются
- **Развязка** — сервисы независимы

### Use Cases
- Обработка заказов в e-commerce
- Email/SMS уведомления
- Обработка изображений
- Микросервисная коммуникация

---

## AMQ Protocol

### Компоненты
```
Producer → Channel → Exchange → Binding → Queue → Channel → Consumer
                         │
                    routing-key
```

| Компонент | Описание |
|-----------|----------|
| Publisher | Отправляет сообщения |
| Channel | Виртуальное соединение внутри connection |
| Exchange | Маршрутизирует сообщения в queues |
| Message | Данные + метаданные |
| Routing Key | Ключ для маршрутизации |
| Queue | Буфер для хранения сообщений |
| Consumer | Получает и обрабатывает сообщения |

### Пример Python (pika)
```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Объявление exchange и queue
channel.exchange_declare(exchange='orders', exchange_type='direct')
channel.queue_declare(queue='order_processing')
channel.queue_bind(queue='order_processing', exchange='orders', routing_key='new_order')

# Публикация
channel.basic_publish(
    exchange='orders',
    routing_key='new_order',
    body='{"order_id": 123}'
)

# Потребление
def callback(ch, method, properties, body):
    print(f"Получено: {body}")
    ch.basic_ack(delivery_tag=method.delivery_tag)

channel.basic_consume(queue='order_processing', on_message_callback=callback)
channel.start_consuming()
```

---

## Get vs Consume

### Basic.Get (Pull)
```python
# Получить одно сообщение
method, properties, body = channel.basic_get(queue='myqueue', auto_ack=True)
if method:
    print(body)
```
- ❌ Неэффективно для потока сообщений
- ✅ Подходит для единичных запросов

### Basic.Consume (Push)
```python
# Подписка на сообщения
channel.basic_consume(queue='myqueue', on_message_callback=callback)
channel.start_consuming()
```
- ✅ Эффективно для потока
- ✅ Сообщения доставляются автоматически

---

## Виды Exchanges

### Fanout
```
Exchange ─┬─► Queue1
          ├─► Queue2
          └─► Queue3
```
Все сообщения во все привязанные queues.

### Direct
```
Exchange ─── routing_key="error" ──► error_queue
         └── routing_key="info" ───► info_queue
```
Точное совпадение routing key.

### Topic
```
Exchange ─── "order.*.created" ──► new_orders_queue
         └── "order.#" ──────────► all_orders_queue
```
Pattern matching: `*` = одно слово, `#` = ноль или более слов.

### Headers
Маршрутизация по заголовкам, не routing key.

```python
# Fanout
channel.exchange_declare(exchange='logs', exchange_type='fanout')

# Direct
channel.exchange_declare(exchange='direct_logs', exchange_type='direct')
channel.queue_bind(queue='errors', exchange='direct_logs', routing_key='error')

# Topic
channel.exchange_declare(exchange='topic_logs', exchange_type='topic')
channel.queue_bind(queue='all_orders', exchange='topic_logs', routing_key='order.#')
```

---

## Acknowledgements

### Basic.Ack (подтверждение)
```python
def callback(ch, method, properties, body):
    try:
        process(body)
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except Exception:
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)
```

### Basic.Reject (отклонение одного)
```python
ch.basic_reject(delivery_tag=method.delivery_tag, requeue=False)
```

### Basic.Nack (отклонение нескольких)
```python
ch.basic_nack(delivery_tag=method.delivery_tag, multiple=True, requeue=True)
```

### Auto-ack
```python
# ❌ Опасно: сообщение удаляется сразу при доставке
channel.basic_consume(queue='q', on_message_callback=cb, auto_ack=True)

# ✅ Безопасно: ручное подтверждение
channel.basic_consume(queue='q', on_message_callback=cb, auto_ack=False)
```

---

## Prefetch и Transactions

### Prefetch (QoS)
```python
# Ограничить количество неподтверждённых сообщений
channel.basic_qos(prefetch_count=10)  # Максимум 10 без ack
```

**Рекомендации**:
- `prefetch_count=1` — равномерное распределение
- `prefetch_count=10-50` — баланс производительности

### Consumer Transactions
```python
channel.tx_select()  # Начать транзакцию

try:
    # Операции
    channel.tx_commit()
except:
    channel.tx_rollback()
```
❌ Транзакции медленные, используйте publisher confirms.

---

## Message Properties и Priority

### Properties
```python
properties = pika.BasicProperties(
    content_type='application/json',
    delivery_mode=2,  # Persistent
    priority=5,
    expiration='60000',  # TTL в мс
    headers={'x-retry-count': 0},
    correlation_id='request-123',
    reply_to='callback_queue'
)
channel.basic_publish(exchange='', routing_key='q', body=msg, properties=properties)
```

### Priority Queue
```python
# Объявить очередь с приоритетами
channel.queue_declare(
    queue='priority_queue',
    arguments={'x-max-priority': 10}
)

# Отправить с приоритетом
props = pika.BasicProperties(priority=8)
channel.basic_publish(exchange='', routing_key='priority_queue', body=msg, properties=props)
```

---

## Dead Letter Exchanges

### Настройка DLX
```python
# Dead Letter Exchange
channel.exchange_declare(exchange='dlx', exchange_type='direct')
channel.queue_declare(queue='dead_letters')
channel.queue_bind(queue='dead_letters', exchange='dlx', routing_key='dead')

# Основная очередь с DLX
channel.queue_declare(
    queue='main_queue',
    arguments={
        'x-dead-letter-exchange': 'dlx',
        'x-dead-letter-routing-key': 'dead'
    }
)
```

### Когда сообщение попадает в DLX
- Rejected с `requeue=False`
- TTL истёк
- Queue переполнена (`x-max-length`)

### Use Case: Retry механизм
```
main_queue → DLX → retry_queue (TTL) → main_queue
```

---

## Автоматическое истечение

### Message TTL
```python
# Per-message TTL
props = pika.BasicProperties(expiration='60000')  # 60 сек

# Per-queue TTL
channel.queue_declare(queue='ttl_queue', arguments={'x-message-ttl': 60000})
```

### Queue TTL
```python
# Очередь удалится через 10 минут неактивности
channel.queue_declare(queue='temp', arguments={'x-expires': 600000})
```

---

## Temporary и Permanent Queues

### Temporary Queues
```python
# Auto-delete: удаляется когда нет consumers
channel.queue_declare(queue='temp', auto_delete=True)

# Exclusive: только для этого connection
result = channel.queue_declare(queue='', exclusive=True)
temp_queue = result.method.queue
```

### Permanent Queues
```python
channel.queue_declare(
    queue='orders',
    durable=True,  # Выживает рестарт
    arguments={
        'x-max-length': 10000,  # Макс. сообщений
        'x-max-length-bytes': 104857600,  # Макс. 100MB
        'x-overflow': 'reject-publish'  # Отклонять новые
    }
)
```

---

## Speed vs Guaranteed Delivery

| Настройка | Скорость | Надёжность |
|-----------|----------|------------|
| `delivery_mode=1` (transient) | ⚡⚡⚡ | ❌ |
| `delivery_mode=2` (persistent) | ⚡⚡ | ✅ |
| Publisher confirms | ⚡ | ✅✅ |
| Transactions | ⚡ | ✅✅✅ |

### Publisher Confirms
```python
channel.confirm_delivery()

try:
    channel.basic_publish(exchange='', routing_key='q', body=msg, mandatory=True)
    print("Сообщение доставлено")
except pika.exceptions.UnroutableError:
    print("Сообщение не доставлено")
```

### Best Practices
- **Высокая скорость**: transient + auto_ack
- **Надёжность**: persistent + manual ack + confirms + durable queue

---

## Virtual Hosts

### Изоляция окружений
```bash
# Создание vhost
rabbitmqctl add_vhost production
rabbitmqctl add_vhost staging

# Права пользователя
rabbitmqctl set_permissions -p production myuser ".*" ".*" ".*"
```

### Подключение
```python
credentials = pika.PlainCredentials('user', 'password')
connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost', virtual_host='production', credentials=credentials)
)
```

---

## Сжатие сообщений

```python
import gzip
import json

def publish_compressed(channel, exchange, routing_key, data):
    json_data = json.dumps(data).encode()
    compressed = gzip.compress(json_data)
    
    props = pika.BasicProperties(
        content_type='application/json',
        content_encoding='gzip'
    )
    channel.basic_publish(exchange=exchange, routing_key=routing_key, 
                          body=compressed, properties=props)

def decompress_message(body, properties):
    if properties.content_encoding == 'gzip':
        return json.loads(gzip.decompress(body))
    return json.loads(body)
```

---

## HA Queues

### Mirrored Queues (Classic)
```bash
# Политика для всех очередей
rabbitmqctl set_policy ha-all ".*" '{"ha-mode":"all"}' --apply-to queues

# Зеркалирование на N узлов
rabbitmqctl set_policy ha-two "^ha\." '{"ha-mode":"exactly","ha-params":2}' --apply-to queues
```

### Quorum Queues (Рекомендуется)
```python
channel.queue_declare(
    queue='quorum_queue',
    arguments={'x-queue-type': 'quorum'}
)
```

---

## Scaling RabbitMQ

### Cluster
```bash
# На node2
rabbitmqctl stop_app
rabbitmqctl reset
rabbitmqctl join_cluster rabbit@node1
rabbitmqctl start_app
```

### Shovel Plugin
```bash
rabbitmq-plugins enable rabbitmq_shovel
rabbitmq-plugins enable rabbitmq_shovel_management
```

Конфигурация через Management UI или rabbitmq.conf.

---

## Cross-cluster Distribution

### Federation Plugin
```bash
rabbitmq-plugins enable rabbitmq_federation
rabbitmq-plugins enable rabbitmq_federation_management
```

### Use Cases
- Географически распределённые системы
- Репликация между дата-центрами
- Multi-region архитектура

### Альтернативы RabbitMQ
- **Apache Kafka** — для event streaming
- **Amazon SQS** — managed queue
- **Redis Streams** — простая очередь
- **NATS** — lightweight messaging
