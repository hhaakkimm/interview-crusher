# Algorithms & Data Structures - Руководство для Senior Backend интервью

> 💡 **Как подходить к алгоритмическим задачам:**
> "Сначала уточняю constraints. Проговариваю brute-force решение. Оптимизирую. Обсуждаю trade-offs. Пишу код. Тестирую на edge cases."

---

## 1. Big O Notation

### 🎯 Что спрашивают
> "Какая сложность этого алгоритма?"

### Основные сложности

| Сложность | Название | Пример |
|-----------|----------|--------|
| O(1) | Constant | Доступ по индексу |
| O(log n) | Logarithmic | Binary search |
| O(n) | Linear | Линейный поиск |
| O(n log n) | Linearithmic | Merge sort |
| O(n²) | Quadratic | Bubble sort |
| O(2ⁿ) | Exponential | Fibonacci наивный |

### Как определить
```python
# O(1) — константа
def get_first(arr):
    return arr[0]

# O(n) — один цикл
def find_sum(arr):
    total = 0
    for x in arr:  # n раз
        total += x
    return total

# O(n²) — вложенные циклы
def find_pairs(arr):
    pairs = []
    for i in arr:      # n раз
        for j in arr:  # n раз
            pairs.append((i, j))
    return pairs

# O(log n) — делим пополам
def binary_search(arr, target):
    left, right = 0, len(arr) - 1
    while left <= right:
        mid = (left + right) // 2
        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            left = mid + 1
        else:
            right = mid - 1
    return -1
```

### 📝 Фраза для интервью
> "Time complexity: O(n) — линейно зависит от размера входных данных. Space complexity: O(1) — дополнительная память не зависит от входа."

---

## 2. Основные структуры данных

### 🎯 Что спрашивают
> "Какую структуру данных выбрать?"

### Сравнение

| Структура | Access | Search | Insert | Delete | Use case |
|-----------|--------|--------|--------|--------|----------|
| Array | O(1) | O(n) | O(n) | O(n) | Индексированный доступ |
| Linked List | O(n) | O(n) | O(1) | O(1) | Частые вставки |
| Hash Table | — | O(1)* | O(1)* | O(1)* | Key-value lookup |
| Binary Tree | — | O(log n)* | O(log n)* | O(log n)* | Сортированные данные |
| Heap | — | O(n) | O(log n) | O(log n) | Priority queue |

### Hash Table (Dict)
```python
# Python dict — hash table
cache = {}
cache["key"] = "value"  # O(1) insert
val = cache.get("key")  # O(1) lookup

# Коллизии решаются chaining или open addressing
# Worst case O(n) при плохой hash функции
```

### Stack и Queue
```python
from collections import deque

# Stack (LIFO)
stack = []
stack.append(1)  # push
stack.pop()      # pop

# Queue (FIFO)
queue = deque()
queue.append(1)     # enqueue
queue.popleft()     # dequeue

# Use cases:
# Stack: undo, parsing, DFS
# Queue: BFS, task scheduling
```

### Heap (Priority Queue)
```python
import heapq

# Min-heap
heap = []
heapq.heappush(heap, 3)
heapq.heappush(heap, 1)
heapq.heappush(heap, 2)
heapq.heappop(heap)  # 1 (минимальный)

# Use cases: Top K elements, median, scheduling
```

---

## 3. Частые паттерны

### Two Pointers
```python
# Найти пару с заданной суммой (отсортированный массив)
def two_sum_sorted(arr, target):
    left, right = 0, len(arr) - 1
    while left < right:
        current_sum = arr[left] + arr[right]
        if current_sum == target:
            return [left, right]
        elif current_sum < target:
            left += 1
        else:
            right -= 1
    return []

# O(n) time, O(1) space
```

### Sliding Window
```python
# Максимальная сумма подмассива размера k
def max_sum_subarray(arr, k):
    window_sum = sum(arr[:k])
    max_sum = window_sum
    
    for i in range(k, len(arr)):
        window_sum += arr[i] - arr[i - k]
        max_sum = max(max_sum, window_sum)
    
    return max_sum

# O(n) time, O(1) space
```

### Hash Map для подсчёта
```python
# Two Sum (неотсортированный)
def two_sum(nums, target):
    seen = {}
    for i, num in enumerate(nums):
        complement = target - num
        if complement in seen:
            return [seen[complement], i]
        seen[num] = i
    return []

# O(n) time, O(n) space
```

### Binary Search
```python
# Поиск в отсортированном массиве
def binary_search(arr, target):
    left, right = 0, len(arr) - 1
    while left <= right:
        mid = left + (right - left) // 2  # Избегаем overflow
        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            left = mid + 1
        else:
            right = mid - 1
    return -1

# O(log n) time
```

---

## 4. Деревья и графы

### Binary Tree Traversal
```python
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right

# Inorder (Left, Root, Right) — для BST даёт отсортированный список
def inorder(root):
    if not root:
        return []
    return inorder(root.left) + [root.val] + inorder(root.right)

# Preorder (Root, Left, Right) — копирование дерева
def preorder(root):
    if not root:
        return []
    return [root.val] + preorder(root.left) + preorder(root.right)

# BFS — по уровням
from collections import deque

def bfs(root):
    if not root:
        return []
    result = []
    queue = deque([root])
    while queue:
        node = queue.popleft()
        result.append(node.val)
        if node.left:
            queue.append(node.left)
        if node.right:
            queue.append(node.right)
    return result
```

### Graph BFS/DFS
```python
# Graph как adjacency list
graph = {
    'A': ['B', 'C'],
    'B': ['D', 'E'],
    'C': ['F'],
    'D': [], 'E': [], 'F': []
}

# BFS — кратчайший путь в невзвешенном графе
def bfs(graph, start):
    visited = set()
    queue = deque([start])
    visited.add(start)
    
    while queue:
        node = queue.popleft()
        print(node)
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)

# DFS — поиск пути, топологическая сортировка
def dfs(graph, start, visited=None):
    if visited is None:
        visited = set()
    visited.add(start)
    print(start)
    for neighbor in graph[start]:
        if neighbor not in visited:
            dfs(graph, neighbor, visited)
```

---

## 5. Dynamic Programming

### 🎯 Что спрашивают
> "Решите задачу с оптимизацией"

### Признаки DP задачи
1. **Overlapping subproblems** — одни подзадачи решаются многократно
2. **Optimal substructure** — оптимальное решение строится из оптимальных подрешений

### Fibonacci (классический пример)
```python
# Наивный O(2^n)
def fib_naive(n):
    if n <= 1:
        return n
    return fib_naive(n-1) + fib_naive(n-2)

# Memoization O(n)
def fib_memo(n, memo={}):
    if n in memo:
        return memo[n]
    if n <= 1:
        return n
    memo[n] = fib_memo(n-1, memo) + fib_memo(n-2, memo)
    return memo[n]

# Bottom-up O(n) time, O(1) space
def fib(n):
    if n <= 1:
        return n
    prev, curr = 0, 1
    for _ in range(2, n + 1):
        prev, curr = curr, prev + curr
    return curr
```

### Climb Stairs
```python
# Сколько способов подняться на n ступенек (1 или 2 за шаг)
def climb_stairs(n):
    if n <= 2:
        return n
    prev, curr = 1, 2
    for _ in range(3, n + 1):
        prev, curr = curr, prev + curr
    return curr
```

### Coin Change
```python
# Минимальное количество монет для суммы
def coin_change(coins, amount):
    dp = [float('inf')] * (amount + 1)
    dp[0] = 0
    
    for coin in coins:
        for x in range(coin, amount + 1):
            dp[x] = min(dp[x], dp[x - coin] + 1)
    
    return dp[amount] if dp[amount] != float('inf') else -1
```

---

## 6. SQL Алгоритмические задачи

### 🎯 Что спрашивают
> "Напишите SQL запрос"

### Второй по величине элемент
```sql
SELECT MAX(salary) 
FROM employees 
WHERE salary < (SELECT MAX(salary) FROM employees);

-- Или с DENSE_RANK
SELECT salary FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) as rn
    FROM employees
) t WHERE rn = 2;
```

### Дубликаты
```sql
-- Найти дубликаты email
SELECT email, COUNT(*) as count
FROM users
GROUP BY email
HAVING COUNT(*) > 1;

-- Удалить дубликаты (оставить первый)
DELETE FROM users
WHERE id NOT IN (
    SELECT MIN(id)
    FROM users
    GROUP BY email
);
```

### Накопительная сумма
```sql
SELECT 
    date,
    amount,
    SUM(amount) OVER (ORDER BY date) as running_total
FROM transactions;
```

---

## 🎤 Частые задачи на интервью

### "Разверните linked list"
```python
def reverse_list(head):
    prev = None
    current = head
    while current:
        next_node = current.next
        current.next = prev
        prev = current
        current = next_node
    return prev
```

### "Проверьте на палиндром"
```python
def is_palindrome(s):
    s = ''.join(c.lower() for c in s if c.isalnum())
    return s == s[::-1]
```

### "Найдите цикл в linked list"
```python
def has_cycle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow == fast:
            return True
    return False
```

### "Merge two sorted arrays"
```python
def merge(nums1, m, nums2, n):
    p1, p2, p = m - 1, n - 1, m + n - 1
    while p2 >= 0:
        if p1 >= 0 and nums1[p1] > nums2[p2]:
            nums1[p] = nums1[p1]
            p1 -= 1
        else:
            nums1[p] = nums2[p2]
            p2 -= 1
        p -= 1
```

### "Valid parentheses"
```python
def is_valid(s):
    stack = []
    mapping = {')': '(', '}': '{', ']': '['}
    for char in s:
        if char in mapping:
            if not stack or stack.pop() != mapping[char]:
                return False
        else:
            stack.append(char)
    return len(stack) == 0
```

### 📝 Фраза для интервью
> "Для этой задачи я бы использовал [структуру данных], потому что нужен [операция] за O(1). Time complexity O(n), space O(1). Edge cases: пустой input, один элемент, все одинаковые."
