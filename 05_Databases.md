# Базы данных - Полное руководство

## Содержание
1. [Нормальная форма](#нормальная-форма)
2. [Модель данных (Типы, Таблицы, Связи, Индексы)](#модель-данных)
3. [SQL Basic (CRUD, Joins, Pagination)](#sql-basic)
4. [SQL Middle (Aggregate, Group By, Subselects)](#sql-middle)
5. [SQL Advanced (Analytic, Recursive, Extensions)](#sql-advanced)
6. [Транзакции (ACID)](#транзакции-acid)
7. [Триггеры/Хранимые процедуры](#триггеры-и-процедуры)
8. [Отладка запросов (Explain, Slow logs)](#отладка-запросов)
9. [Представления/Материализованные представления](#представления)
10. [Репликация/Кластеризация](#репликация-и-кластеризация)
11. [Колоночные базы данных](#колоночные-базы-данных)
12. [NoSQL базы данных](#nosql-базы-данных)
13. [KeyValue хранилища (Redis)](#keyvalue-хранилища)
14. [Денормализация](#денормализация)
15. [Выбор типа БД, Шардирование](#выбор-типа-бд)

---

## Нормальная форма

### 1NF (Первая нормальная форма)
- Атомарные значения (нет списков в ячейках)
- Уникальные строки

```sql
-- ❌ Нарушение 1NF
| id | phones              |
|----|---------------------|
| 1  | 123-456, 789-012    |

-- ✅ 1NF
| id | phone   |
|----|---------|
| 1  | 123-456 |
| 1  | 789-012 |
```

### 2NF (Вторая нормальная форма)
- 1NF + все неключевые атрибуты зависят от всего первичного ключа

### 3NF (Третья нормальная форма)
- 2NF + нет транзитивных зависимостей

```sql
-- ❌ Нарушение 3NF
| order_id | product_id | product_name |  -- product_name зависит от product_id
|---------:|------------|--------------|

-- ✅ 3NF: разделить на две таблицы
orders (order_id, product_id)
products (product_id, product_name)
```

### BCNF и выше
Редко используются на практике из-за сложности.

---

## Модель данных

### Типы данных (PostgreSQL)
| Тип | Описание | Пример |
|-----|----------|--------|
| INTEGER | Целое число | 42 |
| BIGINT | Большое целое | 9223372036854775807 |
| DECIMAL(p,s) | Точное число | 123.45 |
| VARCHAR(n) | Строка переменной длины | 'Иван' |
| TEXT | Неограниченная строка | 'Большой текст...' |
| BOOLEAN | Логический | TRUE/FALSE |
| DATE | Дата | '2024-01-15' |
| TIMESTAMP | Дата и время | '2024-01-15 10:30:00' |
| UUID | Уникальный идентификатор | uuid_generate_v4() |
| JSONB | JSON (бинарный) | '{"key": "value"}' |
| ARRAY | Массив | ARRAY[1, 2, 3] |

### Связи
```sql
-- One-to-Many
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(id)
);

-- Many-to-Many
CREATE TABLE user_roles (
    user_id INT REFERENCES users(id),
    role_id INT REFERENCES roles(id),
    PRIMARY KEY (user_id, role_id)
);

-- One-to-One
CREATE TABLE profiles (
    user_id INT PRIMARY KEY REFERENCES users(id),
    bio TEXT
);
```

### Индексы
```sql
-- B-tree (по умолчанию)
CREATE INDEX idx_email ON users(email);

-- Уникальный индекс
CREATE UNIQUE INDEX idx_unique_email ON users(email);

-- Составной индекс
CREATE INDEX idx_name_date ON orders(status, created_at);

-- Частичный индекс
CREATE INDEX idx_active ON users(email) WHERE is_active = true;

-- GIN для JSON/массивов
CREATE INDEX idx_tags ON posts USING GIN(tags);
```

---

## SQL Basic

### CRUD операции
```sql
-- CREATE
INSERT INTO users (name, email) VALUES ('Иван', 'ivan@mail.ru');
INSERT INTO users (name, email) VALUES 
    ('Пётр', 'petr@mail.ru'),
    ('Анна', 'anna@mail.ru');

-- READ
SELECT * FROM users WHERE id = 1;
SELECT name, email FROM users WHERE is_active = true;

-- UPDATE
UPDATE users SET name = 'Иван Иванов' WHERE id = 1;

-- DELETE
DELETE FROM users WHERE id = 1;
```

### Joins
```sql
-- INNER JOIN (только совпадения)
SELECT u.name, o.total
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- LEFT JOIN (все из левой + совпадения)
SELECT u.name, o.total
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;

-- RIGHT JOIN
SELECT u.name, o.total
FROM users u
RIGHT JOIN orders o ON u.id = o.user_id;

-- FULL OUTER JOIN
SELECT u.name, o.total
FROM users u
FULL OUTER JOIN orders o ON u.id = o.user_id;
```

### Pagination
```sql
-- LIMIT/OFFSET
SELECT * FROM products ORDER BY id LIMIT 10 OFFSET 20;

-- Keyset pagination (эффективнее)
SELECT * FROM products WHERE id > 100 ORDER BY id LIMIT 10;
```

### ALTER
```sql
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
ALTER TABLE users DROP COLUMN phone;
ALTER TABLE users ALTER COLUMN name SET NOT NULL;
ALTER TABLE users ADD CONSTRAINT unique_email UNIQUE(email);
```

---

## SQL Middle

### Aggregate Functions
```sql
SELECT 
    COUNT(*) as total,
    SUM(amount) as sum,
    AVG(amount) as average,
    MIN(amount) as minimum,
    MAX(amount) as maximum
FROM orders;
```

### GROUP BY, HAVING
```sql
SELECT 
    user_id,
    COUNT(*) as order_count,
    SUM(total) as total_spent
FROM orders
WHERE status = 'completed'
GROUP BY user_id
HAVING SUM(total) > 1000
ORDER BY total_spent DESC;
```

### Subselects
```sql
-- В WHERE
SELECT * FROM users 
WHERE id IN (SELECT user_id FROM orders WHERE total > 100);

-- В FROM
SELECT avg_order.user_id, avg_order.avg_total
FROM (
    SELECT user_id, AVG(total) as avg_total
    FROM orders
    GROUP BY user_id
) avg_order
WHERE avg_order.avg_total > 50;

-- Коррелированный подзапрос
SELECT u.name, 
    (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id) as order_count
FROM users u;
```

### JSON операции
```sql
-- Создание
SELECT '{"name": "Иван"}'::jsonb;

-- Извлечение
SELECT data->>'name' FROM users;  -- Как текст
SELECT data->'address'->'city' FROM users;  -- Вложенный

-- Поиск
SELECT * FROM users WHERE data @> '{"role": "admin"}';
```

---

## SQL Advanced

### Window Functions (Аналитические)
```sql
-- ROW_NUMBER
SELECT 
    name,
    department,
    salary,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) as rank
FROM employees;

-- Running total
SELECT 
    date,
    amount,
    SUM(amount) OVER (ORDER BY date) as running_total
FROM transactions;

-- LAG/LEAD
SELECT 
    date,
    amount,
    LAG(amount) OVER (ORDER BY date) as prev_amount,
    LEAD(amount) OVER (ORDER BY date) as next_amount
FROM transactions;

-- RANK, DENSE_RANK, NTILE
SELECT 
    name,
    score,
    RANK() OVER (ORDER BY score DESC) as rank,
    DENSE_RANK() OVER (ORDER BY score DESC) as dense_rank,
    NTILE(4) OVER (ORDER BY score DESC) as quartile
FROM students;
```

### Recursive Queries
```sql
-- Иерархия категорий
WITH RECURSIVE category_tree AS (
    -- Базовый случай
    SELECT id, name, parent_id, 1 as level
    FROM categories
    WHERE parent_id IS NULL
    
    UNION ALL
    
    -- Рекурсивный случай
    SELECT c.id, c.name, c.parent_id, ct.level + 1
    FROM categories c
    INNER JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree;
```

### Extensions (PostgreSQL)
```sql
-- UUID
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
SELECT uuid_generate_v4();

-- Full-text search
CREATE EXTENSION IF NOT EXISTS pg_trgm;
SELECT * FROM products WHERE name % 'поиск';  -- Fuzzy match

-- PostGIS (геоданные)
CREATE EXTENSION IF NOT EXISTS postgis;
SELECT ST_Distance(point1, point2) FROM locations;
```

---

## Транзакции (ACID)

### ACID
- **Atomicity** — всё или ничего
- **Consistency** — данные всегда валидны
- **Isolation** — транзакции изолированы
- **Durability** — сохранённые данные не теряются

### Синтаксис
```sql
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;

-- При ошибке
ROLLBACK;

-- Savepoints
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    SAVEPOINT sp1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
    -- Ошибка? Откат к savepoint
    ROLLBACK TO sp1;
COMMIT;
```

### Уровни изоляции
| Уровень | Dirty Read | Non-repeatable | Phantom |
|---------|------------|----------------|---------|
| READ UNCOMMITTED | ✅ | ✅ | ✅ |
| READ COMMITTED | ❌ | ✅ | ✅ |
| REPEATABLE READ | ❌ | ❌ | ✅ |
| SERIALIZABLE | ❌ | ❌ | ❌ |

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN;
    -- Критичные операции
COMMIT;
```

---

## Триггеры и процедуры

### Триггеры
```sql
-- Функция для триггера
CREATE OR REPLACE FUNCTION update_modified() 
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Создание триггера
CREATE TRIGGER set_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION update_modified();

-- Аудит
CREATE OR REPLACE FUNCTION audit_log() 
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO audit_log (table_name, action, old_data, new_data)
    VALUES (TG_TABLE_NAME, TG_OP, row_to_json(OLD), row_to_json(NEW));
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

### Хранимые процедуры
```sql
-- Функция
CREATE OR REPLACE FUNCTION get_user_orders(p_user_id INT)
RETURNS TABLE(order_id INT, total DECIMAL) AS $$
BEGIN
    RETURN QUERY
    SELECT id, total FROM orders WHERE user_id = p_user_id;
END;
$$ LANGUAGE plpgsql;

-- Вызов
SELECT * FROM get_user_orders(1);

-- Процедура (PostgreSQL 11+)
CREATE OR REPLACE PROCEDURE transfer_money(
    p_from INT, p_to INT, p_amount DECIMAL
) AS $$
BEGIN
    UPDATE accounts SET balance = balance - p_amount WHERE id = p_from;
    UPDATE accounts SET balance = balance + p_amount WHERE id = p_to;
    COMMIT;
END;
$$ LANGUAGE plpgsql;

CALL transfer_money(1, 2, 100.00);
```

---

## Отладка запросов

### EXPLAIN
```sql
EXPLAIN SELECT * FROM users WHERE email = 'test@mail.ru';

EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@mail.ru';

EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT * FROM users WHERE email = 'test@mail.ru';
```

### Основные метрики
- **Seq Scan** — полный перебор (обычно плохо)
- **Index Scan** — использование индекса (хорошо)
- **Rows** — оценка количества строк
- **Cost** — оценка стоимости
- **Actual Time** — реальное время выполнения

### Slow Query Log
```sql
-- postgresql.conf
log_min_duration_statement = 1000  -- Логировать запросы > 1 сек
log_statement = 'all'  -- Логировать все запросы
```

---

## Представления

### Views (Представления)
```sql
CREATE VIEW active_users AS
SELECT id, name, email
FROM users
WHERE is_active = true;

-- Использование
SELECT * FROM active_users;

-- Обновляемое представление
CREATE VIEW user_summary AS
SELECT 
    u.id,
    u.name,
    COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name;
```

### Materialized Views
```sql
-- Создание
CREATE MATERIALIZED VIEW product_stats AS
SELECT 
    product_id,
    COUNT(*) as sales_count,
    SUM(quantity) as total_quantity
FROM order_items
GROUP BY product_id;

-- Обновление
REFRESH MATERIALIZED VIEW product_stats;

-- С индексом
CREATE UNIQUE INDEX idx_product_stats ON product_stats(product_id);
REFRESH MATERIALIZED VIEW CONCURRENTLY product_stats;
```

### Use Cases
- **Views** — упрощение сложных запросов, безопасность
- **Materialized Views** — кеширование агрегаций, отчёты

---

## Репликация и кластеризация

### Типы репликации
| Тип | Описание |
|-----|----------|
| Master-Slave | Один мастер, несколько реплик |
| Master-Master | Несколько мастеров |
| Synchronous | Данные подтверждаются на реплике |
| Asynchronous | Данные реплицируются с задержкой |

### PostgreSQL Streaming Replication
```bash
# На мастере (postgresql.conf)
wal_level = replica
max_wal_senders = 3

# На реплике
primary_conninfo = 'host=master port=5432 user=replicator'
```

### Кластеризация
- **PostgreSQL** — Patroni, Citus, pgpool
- **MySQL** — MySQL Cluster, Galera, ProxySQL
- **Cloud** — RDS Multi-AZ, Cloud SQL

---

## Колоночные базы данных

### Особенности
- Хранят данные по колонкам, не по строкам
- Оптимизированы для аналитики
- Отличное сжатие

### Примеры
| СУБД | Описание |
|------|----------|
| ClickHouse | Быстрая OLAP от Яндекса |
| BigQuery | Managed от Google |
| Redshift | Managed от AWS |
| Vertica | Enterprise |

### ClickHouse
```sql
CREATE TABLE events (
    event_date Date,
    user_id UInt32,
    event_type String
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, user_id);

-- Быстрые агрегации
SELECT event_type, COUNT(*) 
FROM events 
WHERE event_date >= '2024-01-01'
GROUP BY event_type;
```

---

## NoSQL базы данных

### MongoDB (Document)
```javascript
// CRUD
db.users.insertOne({name: 'Иван', email: 'ivan@mail.ru'})
db.users.find({name: 'Иван'})
db.users.updateOne({_id: ObjectId('...')}, {$set: {name: 'Пётр'}})
db.users.deleteOne({_id: ObjectId('...')})

// Индексы
db.users.createIndex({email: 1}, {unique: true})

// Агрегации
db.orders.aggregate([
    {$match: {status: 'completed'}},
    {$group: {_id: '$user_id', total: {$sum: '$amount'}}}
])
```

### Use Cases
- **Document (MongoDB)** — гибкая схема, JSON
- **Key-Value (Redis)** — кеш, сессии
- **Wide-column (Cassandra)** — время-ряды, логи
- **Graph (Neo4j)** — связи, рекомендации

---

## KeyValue хранилища

См. отдельный документ [02_Redis.md](./02_Redis.md)

---

## Денормализация

### Когда применять
- Ускорение чтения (меньше JOIN)
- Кеширование агрегаций
- OLAP системы

### Пример
```sql
-- Нормализованно
SELECT u.name, COUNT(o.id) as orders
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id;

-- Денормализованно
ALTER TABLE users ADD COLUMN order_count INT DEFAULT 0;

-- Обновление через триггер
CREATE TRIGGER update_order_count
AFTER INSERT ON orders
FOR EACH ROW
UPDATE users SET order_count = order_count + 1 WHERE id = NEW.user_id;
```

### Best Practices
- ✅ Денормализуйте для часто читаемых данных
- ✅ Поддерживайте консистентность триггерами
- ✅ Рассмотрите materialized views

---

## Выбор типа БД

### Матрица выбора
| Сценарий | Рекомендация |
|----------|--------------|
| OLTP, транзакции | PostgreSQL, MySQL |
| Аналитика, OLAP | ClickHouse, BigQuery |
| Кеширование | Redis |
| Документы, гибкая схема | MongoDB |
| Графы, связи | Neo4j |
| Временные ряды | TimescaleDB, InfluxDB |
| Масштабирование записи | Cassandra, ScyllaDB |

### Шардирование

### Стратегии
| Стратегия | Описание |
|-----------|----------|
| Range | По диапазону ключа (дата, ID) |
| Hash | По хешу ключа |
| Directory | Отдельная таблица маппинга |

```sql
-- Пример: шардирование по user_id
-- Шард 1: user_id % 4 = 0
-- Шард 2: user_id % 4 = 1
-- и т.д.
```

### PostgreSQL Citus
```sql
-- Распределённая таблица
SELECT create_distributed_table('orders', 'user_id');

-- Запросы работают как обычно
SELECT * FROM orders WHERE user_id = 123;
```

### Best Practices
- ✅ Начинайте с одной БД, масштабируйте при необходимости
- ✅ Выбирайте ключ шардирования по access pattern
- ✅ Избегайте cross-shard запросов
- ✅ Планируйте resharding
