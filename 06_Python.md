# Python - Руководство для технического интервью

> 💡 **Как объяснить Python на интервью за 30 секунд:**
> "Python — высокоуровневый интерпретируемый язык с динамической типизацией. Читаемость — главный приоритет. Используется везде: web (Django, FastAPI), data science (pandas, numpy), ML (PyTorch, TensorFlow), автоматизация. Отлично подходит для быстрой разработки."

---

## 1. Синтаксис и основы

### 🎯 Что спрашивают на интервью
> "Расскажите про базовые конструкции Python"

### Функции
```python
# Позиционные и именованные аргументы
def greet(name, greeting="Привет"):
    return f"{greeting}, {name}!"

# *args, **kwargs
def flexible(*args, **kwargs):
    print(args)    # кортеж позиционных
    print(kwargs)  # словарь именованных

flexible(1, 2, 3, name="Иван", age=25)
# (1, 2, 3)
# {'name': 'Иван', 'age': 25}
```

### Классы
```python
class Animal:
    def __init__(self, name):
        self.name = name
    
    def speak(self):
        raise NotImplementedError

class Dog(Animal):
    def speak(self):
        return f"{self.name} говорит: Гав!"

dog = Dog("Шарик")
print(dog.speak())  # Шарик говорит: Гав!
```

### Обработка ошибок
```python
try:
    result = 10 / 0
except ZeroDivisionError as e:
    print(f"Ошибка: {e}")
except Exception as e:
    print(f"Неизвестная ошибка: {e}")
else:
    print("Успех!")  # Если нет исключений
finally:
    print("Выполнится всегда")
```

### 📝 Фраза для интервью
> "Python использует duck typing — важно не кто объект, а что он умеет. Функции — first-class citizens, можно передавать как аргументы. Классы поддерживают множественное наследование через MRO."

---

## 2. Дзен Python (PEP 20)

### 🎯 Что спрашивают
> "Какие принципы Python вы знаете?"

### Ключевые принципы
```python
import this  # Показывает все принципы
```

| Принцип | Значение |
|---------|----------|
| **Beautiful is better than ugly** | Читаемый код > хитрый |
| **Explicit is better than implicit** | Явное лучше неявного |
| **Simple is better than complex** | Простое решение > сложное |
| **Readability counts** | Код читается чаще, чем пишется |
| **Errors should never pass silently** | Не глушите ошибки |

### 📝 Фраза для интервью
> "Дзен Python — это философия языка. Главное: читаемость, явность, простота. 'Explicit is better than implicit' — не используйте магию, пишите понятно."

---

## 3. ООП: Классы и наследование

### 🎯 Что спрашивают
> "Объясните инкапсуляцию, наследование, полиморфизм в Python"

### Инкапсуляция
```python
class BankAccount:
    def __init__(self, balance):
        self._balance = balance      # Protected (конвенция)
        self.__secret = "hidden"     # Private (name mangling)
    
    @property
    def balance(self):
        return self._balance
    
    @balance.setter
    def balance(self, value):
        if value < 0:
            raise ValueError("Баланс не может быть отрицательным")
        self._balance = value

acc = BankAccount(100)
print(acc.balance)      # 100
acc.balance = 200       # Используем setter
print(acc._BankAccount__secret)  # name mangling
```

### Множественное наследование и MRO
```python
class A:
    def method(self):
        return "A"

class B(A):
    def method(self):
        return "B"

class C(A):
    def method(self):
        return "C"

class D(B, C):
    pass

print(D().method())  # "B"
print(D.__mro__)     # D -> B -> C -> A -> object
```

### Полиморфизм
```python
def make_sound(animal):
    print(animal.speak())  # Duck typing!

make_sound(Dog("Шарик"))  # Работает
make_sound(Cat("Мурка"))  # Тоже работает
```

### 📝 Фраза для интервью
> "Python использует duck typing — если объект имеет нужный метод, он подходит. Инкапсуляция через конвенции: _ protected, __ private с name mangling. MRO определяет порядок поиска методов при множественном наследовании."

---

## 4. PEP 8 и Code Style

### 🎯 Что спрашивают
> "Какие правила оформления кода вы знаете?"

### Основные правила
```python
# ✅ Правильно
import os
import sys
from typing import List, Dict

CONSTANT_VALUE = 42  # UPPER_CASE для констант

class MyClass:  # CamelCase для классов
    def my_method(self, param_one):  # snake_case для методов
        local_variable = param_one * 2
        return local_variable

def calculate_total(items: List[int]) -> int:
    return sum(items)

# ❌ Неправильно
import os, sys  # Импорты на одной строке
def f(x):return x*2  # Всё в одну строку
myVariable = 1  # camelCase для переменных
```

### Инструменты
- **black** — автоформатирование
- **flake8** — линтер
- **isort** — сортировка импортов
- **mypy** — проверка типов

### 📝 Фраза для интервью
> "Следую PEP 8: snake_case для функций и переменных, CamelCase для классов, UPPER_CASE для констант. Использую black для автоформатирования, flake8 для линтинга, mypy для проверки типов."

---

## 5. Генераторы

### 🎯 Что спрашивают
> "Что такое генератор и чем отличается от списка?"

### Простое объяснение
Генератор вычисляет значения **лениво** — по одному, не храня всё в памяти.

```python
# Список — всё в памяти сразу
numbers_list = [x**2 for x in range(1000000)]  # ~8 MB

# Генератор — вычисляется по мере необходимости
numbers_gen = (x**2 for x in range(1000000))   # ~100 bytes!

# Функция-генератор
def countdown(n):
    while n > 0:
        yield n  # Возвращает значение и "засыпает"
        n -= 1

for num in countdown(3):
    print(num)  # 3, 2, 1
```

### Когда использовать
- Большие данные (файлы, БД)
- Бесконечные последовательности
- Pipeline обработки данных

### 📝 Фраза для интервью
> "Генератор — ленивый итератор, вычисляет значения по требованию через yield. Не хранит все данные в памяти, только текущее состояние. Идеален для больших файлов и потоков данных."

---

## 6. Comprehensions

### 🎯 Что спрашивают
> "Что такое list comprehension?"

### Виды comprehensions
```python
# List comprehension
squares = [x**2 for x in range(10)]

# С условием
evens = [x for x in range(10) if x % 2 == 0]

# Dict comprehension
word_lengths = {word: len(word) for word in ["apple", "banana"]}

# Set comprehension
unique_chars = {char for char in "hello"}  # {'h', 'e', 'l', 'o'}

# Вложенные
matrix = [[1, 2], [3, 4], [5, 6]]
flat = [num for row in matrix for num in row]  # [1, 2, 3, 4, 5, 6]
```

### 📝 Фраза для интервью
> "Comprehensions — декларативный способ создания коллекций. Читабельнее и быстрее циклов. Используйте для простых трансформаций, для сложной логики — обычный цикл."

---

## 7. Итераторы

### 🎯 Что спрашивают
> "Что такое итератор? Чем отличается от iterable?"

### Разница
```python
# Iterable — объект с __iter__(), возвращающим iterator
# Iterator — объект с __next__(), возвращающим следующий элемент

my_list = [1, 2, 3]  # Iterable
iterator = iter(my_list)  # Iterator

print(next(iterator))  # 1
print(next(iterator))  # 2
print(next(iterator))  # 3
print(next(iterator))  # StopIteration!
```

### Создание своего итератора
```python
class Counter:
    def __init__(self, max_val):
        self.max_val = max_val
        self.current = 0
    
    def __iter__(self):
        return self
    
    def __next__(self):
        if self.current >= self.max_val:
            raise StopIteration
        self.current += 1
        return self.current

for num in Counter(3):
    print(num)  # 1, 2, 3
```

### 📝 Фраза для интервью
> "Iterable имеет __iter__(), iterator имеет __next__(). for-цикл вызывает iter() для получения итератора, затем next() до StopIteration. Генератор — это итератор, созданный функцией с yield."

---

## 8. Декораторы

### 🎯 Что спрашивают
> "Что такое декоратор? Напишите пример."

### Простое объяснение
Декоратор — функция, которая оборачивает другую функцию.

```python
import functools
import time

def timer(func):
    @functools.wraps(func)  # Сохраняет метаданные оригинальной функции
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        print(f"{func.__name__} выполнилась за {time.time() - start:.2f}с")
        return result
    return wrapper

@timer
def slow_function():
    time.sleep(1)
    return "Готово"

slow_function()  # slow_function выполнилась за 1.00с
```

### Декоратор с аргументами
```python
def repeat(times):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for _ in range(times):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator

@repeat(times=3)
def greet(name):
    print(f"Привет, {name}!")

greet("Иван")  # Выведет 3 раза
```

### 📝 Фраза для интервью
> "Декоратор — функция высшего порядка, оборачивающая другую функцию. @decorator эквивалентно func = decorator(func). functools.wraps сохраняет __name__ и __doc__ оригинала."

---

## 9. Context Managers

### 🎯 Что спрашивают
> "Что такое context manager? Напишите свой."

### Простое объяснение
Гарантирует выполнение кода при входе и выходе из блока.

```python
# Стандартное использование
with open("file.txt", "r") as f:
    content = f.read()
# Файл автоматически закрыт!

# Создание через класс
class Timer:
    def __enter__(self):
        self.start = time.time()
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        print(f"Время: {time.time() - self.start:.2f}с")
        return False  # Не подавлять исключения

with Timer():
    time.sleep(1)  # Время: 1.00с

# Через contextlib
from contextlib import contextmanager

@contextmanager
def temp_change_dir(path):
    old_dir = os.getcwd()
    os.chdir(path)
    try:
        yield
    finally:
        os.chdir(old_dir)
```

### 📝 Фраза для интервью
> "Context manager гарантирует выполнение cleanup-кода через __enter__ и __exit__. with statement автоматически вызывает их. contextlib.contextmanager упрощает создание через генератор."

---

## 10. Asyncio Basic

### 🎯 Что спрашивают
> "Как работает async/await в Python?"

### Простое объяснение
Asyncio позволяет выполнять I/O-операции параллельно, не создавая потоки.

```python
import asyncio

async def fetch_data(url):
    print(f"Начинаю загрузку {url}")
    await asyncio.sleep(1)  # Симуляция I/O
    print(f"Загружено {url}")
    return f"Данные из {url}"

async def main():
    # Последовательно — 3 секунды
    # result1 = await fetch_data("url1")
    # result2 = await fetch_data("url2")
    
    # Параллельно — 1 секунда!
    results = await asyncio.gather(
        fetch_data("url1"),
        fetch_data("url2"),
        fetch_data("url3")
    )
    print(results)

asyncio.run(main())
```

### Ключевые концепции
- **Coroutine** — async функция
- **Task** — запланированная корутина
- **Event Loop** — планировщик
- **await** — точка переключения

### 📝 Фраза для интервью
> "Asyncio использует кооперативную многозадачность: корутины добровольно отдают управление при await. Event loop переключает между ними. Это не параллелизм — один поток, но эффективно для I/O-bound задач."

---

## 11. Typing

### 🎯 Что спрашивают
> "Зачем нужны type hints в Python?"

### Основы
```python
from typing import List, Dict, Optional, Union, Callable, TypeVar

def greet(name: str) -> str:
    return f"Привет, {name}!"

def process_items(items: List[int]) -> Dict[str, int]:
    return {"sum": sum(items), "count": len(items)}

def find_user(user_id: int) -> Optional[User]:  # User | None
    return db.get(user_id)

# Union для нескольких типов
def parse(value: Union[str, int]) -> str:
    return str(value)

# Generics
T = TypeVar('T')
def first(items: List[T]) -> T:
    return items[0]
```

### Преимущества
- IDE автодополнение
- Раннее обнаружение ошибок (mypy)
- Самодокументирование
- Рефакторинг безопаснее

### 📝 Фраза для интервью
> "Type hints — опциональная статическая типизация. Не влияют на runtime, но помогают IDE и mypy находить ошибки. Optional[X] = X | None, Union[A, B] = A или B."

---

## 12. Логирование

### 🎯 Что спрашивают
> "Как правильно логировать в Python?"

### Базовое использование
```python
import logging

# Настройка
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)

logger = logging.getLogger(__name__)

logger.debug("Отладочная информация")
logger.info("Информационное сообщение")
logger.warning("Предупреждение")
logger.error("Ошибка")
logger.exception("Ошибка с traceback")  # В except блоке
```

### Production логирование
```python
import structlog  # Структурированные логи

logger = structlog.get_logger()
logger.info("user_logged_in", user_id=123, ip="192.168.1.1")
# {"event": "user_logged_in", "user_id": 123, "ip": "192.168.1.1", "timestamp": ...}
```

### 📝 Фраза для интервью
> "Использую модуль logging, не print. Уровни: DEBUG, INFO, WARNING, ERROR, CRITICAL. В production — структурированные логи (JSON) для парсинга. Sentry для ошибок, ELK для агрегации."

---

## 13. Virtual Environments

### 🎯 Что спрашивают
> "Зачем нужны виртуальные окружения?"

### Проблема
Разные проекты требуют разные версии библиотек.

### Решения
```bash
# venv (встроенный)
python -m venv .venv
source .venv/bin/activate  # Linux/Mac
pip install -r requirements.txt

# Poetry (современный подход)
poetry init
poetry add requests
poetry install

# pipenv
pipenv install requests
pipenv shell
```

### 📝 Фраза для интервью
> "Виртуальное окружение изолирует зависимости проекта. venv встроен, poetry удобнее для управления зависимостями. requirements.txt или pyproject.toml фиксируют версии."

---

## 14. Multi-threading и GIL

### 🎯 Что спрашивают
> "Что такое GIL и как с ним бороться?"

### GIL простым языком
GIL (Global Interpreter Lock) — только один поток может выполнять Python-код одновременно.

```python
import threading
import multiprocessing

# Threading — для I/O-bound задач
def download(url):
    # Во время ожидания сети GIL освобождается
    response = requests.get(url)

threads = [threading.Thread(target=download, args=(url,)) for url in urls]
for t in threads: t.start()
for t in threads: t.join()

# Multiprocessing — для CPU-bound задач
def compute(n):
    return sum(i**2 for i in range(n))

with multiprocessing.Pool(4) as pool:
    results = pool.map(compute, [10**6, 10**6, 10**6, 10**6])
```

### Когда что использовать
| Задача | Решение |
|--------|---------|
| I/O-bound (сеть, диск) | threading или asyncio |
| CPU-bound (вычисления) | multiprocessing |
| Простые задачи | asyncio |
| Тяжёлые вычисления | multiprocessing или Cython |

### 📝 Фраза для интервью
> "GIL позволяет только одному потоку выполнять Python-код. Для I/O-bound задач это не проблема — GIL освобождается при ожидании. Для CPU-bound используйте multiprocessing, который создаёт отдельные процессы."

---

## 15. Профилирование

### 🎯 Что спрашивают
> "Как найти узкие места в коде?"

### Инструменты
```python
import cProfile
import timeit

# timeit — для микробенчмарков
result = timeit.timeit('sum(range(100))', number=10000)
print(f"Время: {result:.4f}с")

# cProfile — профилирование функций
cProfile.run('my_function()')

# line_profiler — построчное
# pip install line_profiler
@profile
def slow_function():
    # ...

# memory_profiler — память
# pip install memory_profiler
@profile
def memory_hungry():
    # ...
```

### 📝 Фраза для интервью
> "timeit для микробенчмарков, cProfile для общего профилирования, line_profiler для построчного анализа. memory_profiler для отслеживания памяти. Профилируйте перед оптимизацией!"

---

## 16. Memory Management и GC

### 🎯 Что спрашивают
> "Как Python управляет памятью?"

### Reference Counting
```python
import sys

a = [1, 2, 3]
print(sys.getrefcount(a))  # 2 (сам объект + аргумент функции)

b = a  # Ещё одна ссылка
print(sys.getrefcount(a))  # 3

del b  # Удаляем ссылку
print(sys.getrefcount(a))  # 2
```

### Garbage Collector
```python
import gc

# Циклические ссылки — reference counting не справляется
a = []
b = []
a.append(b)
b.append(a)
# GC найдёт и удалит

gc.collect()  # Принудительная сборка
gc.disable()  # Отключить (не рекомендуется)
```

### Memory Leaks
```python
# Частые причины утечек
# 1. Замыкания, удерживающие большие объекты
# 2. Глобальные кеши без ограничений
# 3. Циклические ссылки с __del__

# Решение: weakref
import weakref
cache = weakref.WeakValueDictionary()
```

### 📝 Фраза для интервью
> "Python использует reference counting + cyclic garbage collector. Объект удаляется при refcount=0. GC находит циклические ссылки. Утечки: бесконечные кеши, замыкания. weakref для кешей без удержания объектов."

---

## 🎤 Частые вопросы на интервью

### "Чем list отличается от tuple?"
> "List мутабельный, tuple — нет. Tuple хешируемый (можно в dict/set), быстрее создаётся, занимает меньше памяти. Используйте tuple для неизменяемых данных."

### "Что такое *args и **kwargs?"
> "*args собирает позиционные аргументы в tuple, **kwargs — именованные в dict. Позволяют создавать гибкие функции с переменным числом аргументов."

### "Как работает dict под капотом?"
> "Hash table. Ключ хешируется, хеш определяет позицию. O(1) для get/set в среднем. Ключи должны быть hashable (immutable). С Python 3.7 сохраняет порядок вставки."

### "Что такое mutable default argument?"
```python
# ❌ Опасно!
def append_to(item, lst=[]):
    lst.append(item)
    return lst

append_to(1)  # [1]
append_to(2)  # [1, 2] — тот же список!

# ✅ Правильно
def append_to(item, lst=None):
    if lst is None:
        lst = []
    lst.append(item)
    return lst
```

### "Чем is отличается от ==?"
> "is проверяет идентичность (один объект в памяти), == проверяет равенство значений. a is b означает id(a) == id(b)."

### "Что такое slots?"
> "__slots__ ограничивает атрибуты класса, экономит память, ускоряет доступ. Вместо __dict__ — фиксированный список атрибутов."

### "Что такое метаклассы?"
> "Класс класса. Позволяет изменять создание классов. type — базовый метакласс. Используется в ORM, API frameworks. Сложно, используйте редко."

### "Как реализовать Singleton?"
```python
class Singleton:
    _instance = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
```

### "Что такое descriptor?"
> "Объект с __get__, __set__, __delete__. Контролирует доступ к атрибуту. property — пример дескриптора. Используется в ORM для полей."

### "Как работает import?"
> "Проверяет sys.modules (кеш), ищет в sys.path, компилирует в .pyc, выполняет код модуля. Модуль выполняется один раз, потом кешируется."

### "Что такое __name__ == '__main__'?"
> "Проверка, запущен ли файл напрямую или импортирован. При прямом запуске __name__ == '__main__', при импорте — имя модуля."

### "Чем отличаются staticmethod, classmethod, обычный метод?"
> "Обычный получает self. classmethod получает cls (сам класс). staticmethod не получает ни то, ни другое — обычная функция в namespace класса."

### "Что такое dataclass?"
```python
from dataclasses import dataclass

@dataclass
class Point:
    x: float
    y: float
    
# Автоматически: __init__, __repr__, __eq__
p = Point(1.0, 2.0)
```

### "Как сделать класс hashable?"
> "Реализовать __hash__ и __eq__. Объект должен быть immutable. __hash__ возвращает int, __eq__ сравнивает объекты."
