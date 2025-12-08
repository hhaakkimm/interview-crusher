# Базы данных - Руководство для технического интервью

> 💡 **Как объяснить базы данных на интервью:**
> "База данных — это организованное хранилище данных. SQL базы (PostgreSQL, MySQL) хранят данные в таблицах со строгой схемой, поддерживают ACID транзакции. NoSQL (MongoDB, Redis) — гибкие схемы, горизонтальное масштабирование."

---

## 1. Нормализация

### 🎯 Что спрашивают на интервью
> "Что такое нормализация и нормальные формы?"

### Простое объяснение
Нормализация — это **устранение дублирования данных** путём разбиения таблиц.

### 1NF — Атомарность
Каждая ячейка содержит одно значение.

```
❌ Плохо                          ✅ Хорошо
| name | phones           |       | name | phone   |
|------|------------------|       |------|---------|
| Иван | 123-456, 789-012 |       | Иван | 123-456 |
                                  | Иван | 789-012 |
```

### 2NF — Полная функциональная зависимость
Все неключевые атрибуты зависят от **всего** первичного ключа.

### 3NF — Нет транзитивных зависимостей

```
❌ Плохо (product_name зависит от product_id, не от order_id)
| order_id | product_id | product_name |

✅ Хорошо (разделили)
orders:   | order_id | product_id |
products: | product_id | product_name |
```

### 📝 Фраза для интервью
> "Нормализация убирает дублирование данных. 1NF — атомарные значения. 2NF — зависимость от всего ключа. 3NF — нет транзитивных зависимостей. Это уменьшает аномалии при insert/update/delete, но может потребовать больше JOIN."

---

## 2. Индексы

### 🎯 Что спрашивают
> "Зачем нужны индексы и как они работают?"

### Простое объяснение
Индекс — это как оглавление в книге. Без него придётся листать всю книгу (Full Scan).

### B-tree (по умолчанию)

```sql
CREATE INDEX idx_email ON users(email);

-- Теперь быстро:
SELECT * FROM users WHERE email = 'ivan@mail.ru';  -- O(log n)

-- А не Full Scan:
-- O(n) — перебор всех строк
```

### Когда индекс НЕ помогает

```sql
-- ❌ Функция на колонке
SELECT * FROM users WHERE LOWER(email) = 'ivan@mail.ru';
-- Решение: функциональный индекс
CREATE INDEX idx_email_lower ON users(LOWER(email));

-- ❌ LIKE с % в начале
SELECT * FROM users WHERE email LIKE '%@gmail.com';
-- Индекс не используется!

-- ❌ Малая селективность
SELECT * FROM users WHERE is_active = true;
-- Если 90% записей active — индекс бесполезен
```

### Составные индексы

```sql
CREATE INDEX idx_status_date ON orders(status, created_at);

-- ✅ Использует индекс
SELECT * FROM orders WHERE status = 'pending';
SELECT * FROM orders WHERE status = 'pending' AND created_at > '2024-01-01';

-- ❌ НЕ использует (порядок важен!)
SELECT * FROM orders WHERE created_at > '2024-01-01';
```

### 📝 Фраза для интервью
> "Индекс ускоряет поиск с O(n) до O(log n). B-tree — самый распространённый. Индексы занимают место и замедляют INSERT/UPDATE, поэтому нужен баланс. Важно понимать, когда индекс не используется: функции на колонке, LIKE с %, низкая селективность."

---

## 3. SQL Basic

### 🎯 Что спрашивают
> "Напишите простой SQL запрос с JOIN"

### CRUD — базовые операции

```sql
-- Create
INSERT INTO users (name, email) VALUES ('Иван', 'ivan@mail.ru');

-- Read
SELECT name, email FROM users WHERE id = 1;

-- Update
UPDATE users SET name = 'Пётр' WHERE id = 1;

-- Delete
DELETE FROM users WHERE id = 1;
```

### JOIN — объединение таблиц

```sql
-- INNER JOIN: только совпадения
SELECT u.name, o.total
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- LEFT JOIN: все из левой + совпадения из правой
SELECT u.name, o.total
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;
-- Пользователи без заказов тоже попадут (o.total = NULL)
```

### Визуализация JOIN

```
INNER JOIN:  A ∩ B       — только пересечение
LEFT JOIN:   A           — вся левая + пересечение
RIGHT JOIN:      B       — вся правая + пересечение  
FULL JOIN:   A ∪ B       — обе + пересечение
```

### 📝 Фраза для интервью
> "INNER JOIN возвращает только совпадающие строки. LEFT JOIN — все из левой таблицы и совпадения из правой. Если совпадения нет, колонки правой таблицы будут NULL."

---

## 4. SQL Advanced — Window Functions

### 🎯 Что спрашивают
> "Как получить ранг/нумерацию в SQL?"

### Проблема
Как пронумеровать строки? Как найти топ-3 в каждой категории?

### ROW_NUMBER, RANK, DENSE_RANK

```sql
SELECT 
    name,
    department,
    salary,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) as row_num,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) as rank,
    DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) as dense_rank
FROM employees;
```

| name | salary | ROW_NUMBER | RANK | DENSE_RANK |
|------|--------|------------|------|------------|
| A    | 100    | 1          | 1    | 1          |
| B    | 100    | 2          | 1    | 1          |
| C    | 90     | 3          | 3    | 2          |

- **ROW_NUMBER**: уникальные номера
- **RANK**: одинаковые значения = одинаковый ранг, потом пропуск
- **DENSE_RANK**: без пропусков

### Running Total (накопительная сумма)

```sql
SELECT 
    date,
    amount,
    SUM(amount) OVER (ORDER BY date) as running_total
FROM transactions;
```

### Топ-3 в каждой категории

```sql
WITH ranked AS (
    SELECT 
        product_name,
        category,
        sales,
        ROW_NUMBER() OVER (PARTITION BY category ORDER BY sales DESC) as rn
    FROM products
)
SELECT * FROM ranked WHERE rn <= 3;
```

### 📝 Фраза для интервью
> "Window functions вычисляют значение для каждой строки на основе 'окна' связанных строк. PARTITION BY группирует, ORDER BY сортирует. Это позволяет делать рейтинги, running totals, сравнение с предыдущей строкой — без GROUP BY."

---

## 5. Транзакции и ACID

### 🎯 Что спрашивают
> "Что такое ACID?"

### ACID простым языком

| Свойство | Что значит | Пример |
|----------|-----------|--------|
| **Atomicity** | Всё или ничего | Перевод денег: -100 и +100 либо оба, либо никак |
| **Consistency** | Данные всегда валидны | Баланс не может стать отрицательным |
| **Isolation** | Транзакции не видят друг друга | Пока я редактирую, другой не видит мои изменения |
| **Durability** | После COMMIT — навсегда | Даже при crash данные сохранятся |

### Пример транзакции

```sql
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
    -- Если всё ок:
COMMIT;
    -- Если ошибка:
-- ROLLBACK;
```

### Уровни изоляции

| Уровень | Проблема | Описание |
|---------|----------|----------|
| READ UNCOMMITTED | Dirty Read | Видим незакоммиченные данные |
| READ COMMITTED | Non-repeatable Read | Данные могут измениться между чтениями |
| REPEATABLE READ | Phantom Read | Могут появиться новые строки |
| SERIALIZABLE | Нет | Полная изоляция, но медленнее |

### 📝 Фраза для интервью
> "ACID — гарантии транзакций. Atomicity — всё или ничего. Consistency — данные валидны. Isolation — транзакции изолированы. Durability — данные не теряются. Уровень изоляции — компромисс между согласованностью и производительностью."

---

## 6. EXPLAIN — анализ запросов

### 🎯 Что спрашивают
> "Как оптимизировать медленный запрос?"

### EXPLAIN — ваш главный инструмент

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@mail.ru';
```

### Что смотреть в выводе

```
Seq Scan on users  (cost=0.00..1234.00 rows=1 width=100)
  Filter: (email = 'test@mail.ru'::text)
  Rows Removed by Filter: 99999
  Actual time: 15.123..145.456 ms
```

| Что видим | Что значит | Что делать |
|-----------|-----------|------------|
| **Seq Scan** | Полный перебор | Нужен индекс! |
| **Index Scan** | Использует индекс | ✅ Хорошо |
| **Rows Removed** | Много отфильтровано | Плохая селективность |
| **Actual time** | Реальное время | Цель оптимизации |

### Типичные проблемы

```sql
-- 1. Нет индекса
EXPLAIN SELECT * FROM users WHERE email = 'x';
-- Видим Seq Scan → добавить индекс

-- 2. Функция на колонке
EXPLAIN SELECT * FROM users WHERE LOWER(email) = 'x';
-- Индекс не используется → функциональный индекс

-- 3. Неправильный порядок в составном индексе
-- Индекс: (status, date)
EXPLAIN SELECT * FROM orders WHERE date > '2024-01-01';
-- Индекс не используется → изменить порядок или добавить отдельный
```

### 📝 Фраза для интервью
> "EXPLAIN ANALYZE показывает план выполнения запроса. Seq Scan — плохо, значит полный перебор. Index Scan — хорошо. Смотрим actual time, rows removed, есть ли nested loops с большим количеством итераций. На основе этого добавляем индексы или переписываем запрос."

---

## 7. Репликация

### 🎯 Что спрашивают
> "Как масштабировать базу данных?"

### Типы репликации

```
┌──────────────┐
│    Master    │ ← Все записи сюда
│   (Primary)  │
└──────┬───────┘
       │ репликация
  ┌────┴────┐
  ▼         ▼
┌─────┐   ┌─────┐
│Slave│   │Slave│ ← Читаем отсюда (read replicas)
└─────┘   └─────┘
```

### Синхронная vs Асинхронная

| Тип | Скорость | Надёжность |
|-----|----------|------------|
| **Синхронная** | Медленнее | Данные гарантированно на replica |
| **Асинхронная** | Быстрее | Возможна потеря при crash master |

### Read/Write Splitting

```python
# Записи → Master
def create_user(user_data):
    master_db.execute("INSERT INTO users ...")

# Чтения → Replica
def get_user(user_id):
    replica_db.execute("SELECT * FROM users WHERE id = %s", user_id)
```

### Шардирование (Sharding)

```
Shard 1: users с id 1-1000000
Shard 2: users с id 1000001-2000000
Shard 3: users с id 2000001-3000000
```

**Проблемы**: cross-shard запросы, изменение схемы, распределённые транзакции.

### 📝 Фраза для интервью
> "Репликация для HA и масштабирования чтения: master принимает записи, реплики — чтения. Шардирование для масштабирования записи: данные делятся по ключу. Но это усложняет архитектуру: cross-shard запросы, распределённые транзакции."

---

## 8. SQL vs NoSQL

### 🎯 Что спрашивают
> "Когда использовать SQL, когда NoSQL?"

### Сравнение

| Аспект | SQL (PostgreSQL) | NoSQL (MongoDB) |
|--------|------------------|-----------------|
| Схема | Строгая, определена заранее | Гибкая, документы могут отличаться |
| Транзакции | Полные ACID | Ограниченные (обычно на документ) |
| Масштабирование | Вертикальное + read replicas | Горизонтальное (шардирование) |
| Запросы | Мощный SQL, JOIN | Простые запросы, embedded документы |
| Use case | Финансы, ERP, сложные связи | Каталоги, CMS, прототипы |

### MongoDB — когда подходит

```javascript
// Документ с вложенной структурой
{
    _id: ObjectId("..."),
    name: "Ноутбук",
    specs: {
        cpu: "Intel i7",
        ram: "16GB"
    },
    reviews: [
        { user: "Иван", rating: 5 },
        { user: "Пётр", rating: 4 }
    ]
}
```

**Когда использовать:**
- Быстрое прототипирование
- Данные естественно вложены (не нужны JOIN)
- Схема часто меняется
- Горизонтальное масштабирование важнее транзакций

### PostgreSQL — когда подходит

- Сложные связи между данными
- Нужны ACID транзакции
- Сложные аналитические запросы
- Финансовые данные

### 📝 Фраза для интервью
> "Выбор зависит от задачи. SQL для транзакций, сложных связей, аналитики. NoSQL для гибкой схемы, горизонтального масштабирования, когда данные естественно вложены. Часто используют оба: PostgreSQL для основных данных, MongoDB для каталогов, Redis для кеша."

---

## 9. Денормализация

### 🎯 Что спрашивают
> "Зачем нарушать нормальные формы?"

### Когда денормализовать

Нормализация хороша для консистентности, но:
- Много JOIN = медленно
- Read-heavy нагрузка
- Аналитические запросы

### Пример

```sql
-- Нормализованно: 3 JOIN для каждого запроса
SELECT 
    o.id, u.name, p.title
FROM orders o
JOIN users u ON o.user_id = u.id
JOIN products p ON o.product_id = p.id;

-- Денормализованно: всё в одной таблице
SELECT id, user_name, product_title FROM orders_denorm;
```

### Способы денормализации

1. **Дублирование полей**: `user_name` в orders
2. **Computed columns**: `order_count` в users
3. **Materialized Views**: предвычисленные агрегации

### Проблема: консистентность

```python
# При обновлении user.name нужно обновить везде!
def update_user_name(user_id, new_name):
    db.execute("UPDATE users SET name = %s WHERE id = %s", new_name, user_id)
    db.execute("UPDATE orders SET user_name = %s WHERE user_id = %s", new_name, user_id)
    # Или использовать триггеры
```

### 📝 Фраза для интервью
> "Денормализация — осознанное дублирование для ускорения чтения. Применяется для read-heavy нагрузки, OLAP систем. Цена — сложность поддержания консистентности при update. Нужно использовать триггеры или application-level логику."

---

## 🎤 Частые вопросы на интервью

### "Что такое deadlock?"
> "Взаимная блокировка: транзакция A ждёт ресурс B, транзакция B ждёт ресурс A. База детектит и откатывает одну из транзакций. Предотвращение: всегда захватывать ресурсы в одном порядке."

### "Что такое индекс и когда он не помогает?"
> "Индекс — структура для быстрого поиска. Не помогает: при LIKE '%...', при функциях на колонке, при низкой селективности, при запросе большей части таблицы."

### "Как найти второй по величине элемент?"
```sql
-- Способ 1: OFFSET
SELECT salary FROM employees ORDER BY salary DESC LIMIT 1 OFFSET 1;

-- Способ 2: Подзапрос
SELECT MAX(salary) FROM employees WHERE salary < (SELECT MAX(salary) FROM employees);

-- Способ 3: DENSE_RANK
SELECT salary FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) as rn
    FROM employees
) t WHERE rn = 2;
```

### "Чем PRIMARY KEY отличается от UNIQUE?"
> "PRIMARY KEY = UNIQUE + NOT NULL + один на таблицу. UNIQUE допускает NULL (один или несколько, зависит от СУБД) и может быть много на таблицу."

### "Что такое connection pool?"
> "Пул готовых соединений с базой. Создание соединения дорогое (TCP, auth). Пул держит соединения открытыми и переиспользует. Параметры: min, max connections, timeout."

### "Что такое VIEW?"
> "Виртуальная таблица, результат сохранённого запроса. Не хранит данные, вычисляется при каждом обращении. Используется для упрощения сложных запросов и контроля доступа."

### "Что такое Materialized View?"
> "VIEW, которая физически хранит данные. Быстрее при чтении, но нужно обновлять (REFRESH). Для отчётов, агрегаций, OLAP."

### "Чем TRUNCATE отличается от DELETE?"
> "DELETE — DML, удаляет построчно, можно WHERE, пишет в лог, медленнее. TRUNCATE — DDL, удаляет всё мгновенно, сбрасывает счётчики, нельзя откатить (зависит от СУБД)."

### "Что такое VACUUM в PostgreSQL?"
> "Сборка мусора. PostgreSQL использует MVCC — старые версии строк не удаляются сразу. VACUUM освобождает место. AUTOVACUUM делает это автоматически."

### "Что такое WAL?"
> "Write-Ahead Log — журнал всех изменений перед записью в основные файлы. Обеспечивает durability и crash recovery. Используется для репликации."

### "Как работает B-tree индекс?"
> "Сбалансированное дерево. Ключи отсортированы, поиск O(log n). Эффективен для =, <, >, BETWEEN, ORDER BY. Не эффективен для LIKE '%...'."

### "Что такое covering index?"
> "Индекс, содержащий все колонки запроса. Не нужно обращаться к таблице (index-only scan). Ускоряет чтение, но занимает больше места."

### "Как удалить дубликаты?"
```sql
-- С CTE и ROW_NUMBER
WITH duplicates AS (
    SELECT id, ROW_NUMBER() OVER (PARTITION BY email ORDER BY id) as rn
    FROM users
)
DELETE FROM users WHERE id IN (SELECT id FROM duplicates WHERE rn > 1);
```

### "Что такое OLTP vs OLAP?"
> "OLTP — транзакционная обработка: много мелких операций, строгие транзакции. OLAP — аналитика: сложные запросы, агрегации, чтение больших объёмов."

### "Как работает оптимизатор запросов?"
> "Анализирует запрос, генерирует возможные планы, оценивает стоимость (на основе статистики), выбирает оптимальный. ANALYZE обновляет статистику."

### "Что такое prepared statements?"
> "Предкомпилированные запросы с параметрами. Защита от SQL injection, более быстрое выполнение при повторении. Параметры передаются отдельно от запроса."

### "PostgreSQL vs MySQL?"
> "PostgreSQL: мощнее, больше типов данных, лучше стандарты SQL, JSONB. MySQL: проще, быстрее для простых запросов, популярнее в web. Для новых проектов рекомендую PostgreSQL."

### "Что такое foreign key constraint?"
> "Ссылочная целостность: значение в колонке должно существовать в другой таблице. Предотвращает orphan записи. ON DELETE CASCADE/SET NULL для автоматических действий."

### "Как реализовать soft delete?"
> "Вместо DELETE — UPDATE deleted_at = NOW(). Добавить WHERE deleted_at IS NULL во все запросы. Или использовать поле is_deleted. Можно восстановить данные."

