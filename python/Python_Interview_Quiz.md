# Python Interview Quiz: Comprehensive Guide

**Level:** Intermediate to Hard  
**Target:** Backend engineers, technical interviews, real-world scenarios  
**Total Questions:** 50 + 5 Bonus Challenge Questions  

---

## Table of Contents

1. [Core Python Concepts](#core-python-concepts)
2. [OOP & Design Patterns](#oop--design-patterns)
3. [Data Structures & Algorithms](#data-structures--algorithms)
4. [Concurrency & Performance](#concurrency--performance)
5. [Libraries & Real-world Use Cases](#libraries--real-world-use-cases)
6. [Bonus: Challenge Questions](#bonus-challenge-questions)

---

# Core Python Concepts

## Question 1: Mutable Default Arguments (Debugging)
**Difficulty:** Medium

```python
def append_item(item, container=[]):
    container.append(item)
    return container

result1 = append_item(1)
result2 = append_item(2)
print(result1)  # What's printed?
print(result2)  # What's printed?
print(result1 is result2)  # True or False?
```

**Expected Output:** `[1, 2]`, `[1, 2]`, `True`

**Constraints:** Explain why this happens and provide the correct implementation.

**Hints:**
- Default arguments are evaluated once at function definition time
- They are shared across all function calls
- This is a classic Python pitfall

---

## Question 2: List vs Tuple Performance
**Difficulty:** Medium

You have 1 million integers. You need to:
1. Store them in a data structure
2. Access elements by index 1,000 times
3. Never modify the structure after creation

**Question:** Which would you choose—list or tuple? Why?

**Constraints:** Consider memory usage, performance, and semantics.

**Expected Approach:**
- Tuple: immutable, slightly smaller memory footprint, signals intent
- List: mutable, negligible performance difference for indexing
- Correct answer: tuple (semantic clarity + minor memory savings)

---

## Question 3: String Interning and Identity (Tricky)
**Difficulty:** Medium

```python
a = "python"
b = "python"
print(a is b)  # True

c = "python" + ""
d = "python"
print(c is d)  # True or False?

e = "".join(["python"])
f = "python"
print(e is f)  # True or False?
```

**Expected Output:** `True`, `True`, `False`

**Question:** Explain the behavior in each case.

**Constraints:** CPython-specific behavior.

**Hints:**
- String interning optimization in CPython
- When strings are created at parse time vs runtime
- `.join()` creates a new string object

---

## Question 4: Dictionary Ordering and Insertion Order
**Difficulty:** Medium

```python
d = {"b": 1}
d["a"] = 2
d["c"] = 3

# In Python 3.7+, what's the iteration order?
for key in d:
    print(key)

# What about dict.keys() vs set(d.keys())?
```

**Question:** Explain Python 3.7+ dictionary implementation and why insertion order is guaranteed.

**Constraints:** Be specific about CPython implementation details.

**Expected Approach:**
- Python 3.7+ dicts use compact array-based layout
- Insertion order is guaranteed by language spec
- `set(d.keys())` loses order (unordered hash set)

---

## Question 5: Generator Expression vs List Comprehension (Optimization)
**Difficulty:** Hard

```python
# Scenario: Process 10 million numbers, keep only even numbers
numbers = range(10_000_000)

# Option A: List comprehension
evens_list = [x for x in numbers if x % 2 == 0]

# Option B: Generator expression
evens_gen = (x for x in numbers if x % 2 == 0)

# Option C: filter() with lambda
evens_filter = filter(lambda x: x % 2 == 0, numbers)
```

**Question:** Compare memory usage, performance, and use cases for each approach.

**Constraints:** Consider: memory footprint, time to first result, time to iterate all, readability.

**Expected Approach:**
- List comprehension: O(n) memory, all data upfront, fastest iteration
- Generator: O(1) memory, lazy evaluation, best for one-pass processing
- filter(): similar to generator, less readable in Python
- Answer: Generator for streaming large datasets, list for multiple accesses

---

## Question 6: Function Closure and Variable Capture (Debugging)
**Difficulty:** Hard

```python
functions = []
for i in range(3):
    def func():
        return i
    functions.append(func)

print([f() for f in functions])  # What's printed?

# Fix this to print [0, 1, 2]
```

**Expected Output:** `[2, 2, 2]`

**Question:** Why? Provide a corrected version.

**Constraints:** Explain closure semantics.

**Expected Approach:**
- Closure captures the variable `i`, not its value
- By the time any function is called, `i == 2`
- Fix: use default argument `def func(x=i)` or factory function

---

## Question 7: *args and **kwargs Unpacking (Real-world)
**Difficulty:** Medium

```python
def process_data(a, b, *args, verbose=False, **kwargs):
    print(f"a={a}, b={b}")
    print(f"args={args}")
    print(f"verbose={verbose}")
    print(f"kwargs={kwargs}")

data = {'key1': 'val1', 'key2': 'val2'}
args_data = (1, 2, 3)

# How to call process_data to unpack both?
```

**Question:** Write the function call that unpacks both `args_data` and `data`.

**Expected Answer:**
```python
process_data(*args_data, **data)
# or with keyword args:
process_data(args_data[0], args_data[1], *args_data[2:], **data)
```

**Constraints:** Show understanding of unpacking order (positional before keyword).

---

## Question 8: Shallow vs Deep Copy (Debugging)
**Difficulty:** Medium

```python
import copy

original = [[1, 2], [3, 4]]

shallow = copy.copy(original)
shallow[0][0] = 999

deep = copy.deepcopy(original)
deep[0][0] = 999

print(original)  # What's printed?
```

**Expected Output:** `[[999, 2], [3, 4]]`

**Question:** Why is the first element modified? When would you use each?

**Constraints:** Explain the difference in memory layout.

---

## Question 9: Decorators with Arguments (OOP)
**Difficulty:** Hard

```python
def repeat(times):
    def decorator(func):
        def wrapper(*args, **kwargs):
            results = []
            for _ in range(times):
                results.append(func(*args, **kwargs))
            return results
        return wrapper
    return decorator

@repeat(3)
def greet(name):
    return f"Hello, {name}!"

print(greet("Alice"))
```

**Question:** Predict the output. Explain decorator stacking.

**Expected Output:** `['Hello, Alice!', 'Hello, Alice!', 'Hello, Alice!']`

**Constraints:** Explain the decorator parameter passing mechanism.

---

## Question 10: functools.wraps and Metadata Preservation
**Difficulty:** Hard

```python
import functools
import time

def timer(func):
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        elapsed = time.time() - start
        print(f"Took {elapsed:.4f}s")
        return result
    return wrapper

@timer
def compute():
    """Compute expensive operation."""
    return sum(range(10_000_000))

print(compute.__name__)  # What's printed?
print(compute.__doc__)   # What's printed?
```

**Expected Output:** `'wrapper'`, `None`

**Question:** How to preserve original function metadata? Show the corrected version.

**Expected Approach:**
```python
def timer(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        ...
    return wrapper
```

---

# OOP & Design Patterns

## Question 11: Static Methods vs Class Methods (Conceptual)
**Difficulty:** Medium

```python
class Calculator:
    pi = 3.14159
    
    @staticmethod
    def add(a, b):
        return a + b
    
    @classmethod
    def from_string(cls, expr):
        a, b = map(float, expr.split('+'))
        return cls.add(a, b)
    
    def instance_method(self):
        return self.pi

calc = Calculator()
print(calc.add(2, 3))          # Works?
print(Calculator.add(2, 3))    # Works?
print(calc.from_string("2+3")) # Works? Return type?
```

**Expected Output:** `5`, `5`, `5.0`

**Question:** When would you use static vs class methods? Provide a realistic example.

**Constraints:** Consider factory patterns and inheritance.

---

## Question 12: Abstract Classes and Interfaces (Design)
**Difficulty:** Hard

```python
from abc import ABC, abstractmethod

class DataStore(ABC):
    @abstractmethod
    def get(self, key):
        pass
    
    @abstractmethod
    def set(self, key, value):
        pass

# Can you instantiate DataStore()?
# What if a subclass implements only get()?

class PartialStore(DataStore):
    def get(self, key):
        return None

store = PartialStore()  # Error or not?
```

**Expected Output:** TypeError: Can't instantiate abstract class PartialStore...

**Question:** How does ABC enforce contracts? Show a complete implementation.

**Constraints:** Include `abstractmethod` and `abstractproperty`.

---

## Question 13: Property Decorators and Lazy Evaluation (Real-world)
**Difficulty:** Medium

```python
class User:
    def __init__(self, user_id):
        self.user_id = user_id
        self._profile = None
    
    @property
    def profile(self):
        if self._profile is None:
            # Simulate expensive API call
            self._profile = {"name": "Alice", "age": 30}
        return self._profile
    
    @profile.setter
    def profile(self, value):
        self._profile = value

user = User(1)
print(user.profile)      # Triggers fetch?
print(user.profile)      # Triggers fetch again?
user.profile = {"name": "Bob"}
print(user.profile)      # What's printed?
```

**Expected Output:** Fetch only on first access (lazy loading)

**Question:** Explain lazy evaluation and when it's beneficial.

---

## Question 14: Singleton Pattern Implementation (Design)
**Difficulty:** Hard

Implement a thread-safe Singleton in Python. Show at least 2 approaches:
1. Using metaclass
2. Using decorator

**Constraints:**
- Must be thread-safe
- Must prevent multiple instantiations
- Show how to test it

**Expected Approach:**
```python
# Metaclass approach
class SingletonMeta(type):
    _instances = {}
    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]

# Decorator approach
def singleton(cls):
    instances = {}
    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]
    return get_instance
```

---

## Question 15: Inheritance and Method Resolution Order (MRO) (Debugging)
**Difficulty:** Hard

```python
class A:
    def method(self):
        print("A")

class B(A):
    def method(self):
        print("B")

class C(A):
    def method(self):
        print("C")

class D(B, C):
    pass

d = D()
d.method()

# What's printed? What's the MRO?
print(D.__mro__)
```

**Expected Output:** `'B'` and `(D, B, C, A, object)`

**Question:** Explain C3 linearization and why MRO matters.

**Constraints:** Show how `super()` works with MRO.

---

## Question 16: Composition vs Inheritance (Design Decision)
**Difficulty:** Hard

**Scenario:** You're building a notification system. You want to send notifications via Email, SMS, and Slack.

**Option A: Inheritance**
```python
class Notifier(ABC):
    @abstractmethod
    def send(self, message): pass

class EmailNotifier(Notifier): ...
class SMSNotifier(Notifier): ...
```

**Option B: Composition**
```python
class NotificationService:
    def __init__(self, channels):
        self.channels = channels
    
    def send(self, message):
        for channel in self.channels:
            channel.send(message)
```

**Question:** Which is better for this scenario? Why? What if you need multi-channel notifications?

**Constraints:** Discuss extensibility, testability, and the Liskov Substitution Principle.

---

## Question 17: SOLID Principles - Single Responsibility
**Difficulty:** Hard

**Problematic Code:**
```python
class UserManager:
    def create_user(self, email, password):
        # Validate email
        if '@' not in email:
            raise ValueError("Invalid email")
        
        # Hash password
        hashed = self.hash_password(password)
        
        # Save to DB
        db.save({'email': email, 'password': hashed})
        
        # Send welcome email
        self.send_email(email, "Welcome!")
    
    def hash_password(self, pwd): ...
    def send_email(self, email, msg): ...
```

**Question:** Identify SRP violations. Refactor this into smaller, focused classes.

**Expected Approach:**
- Separate: validation, password hashing, persistence, email
- Use dependency injection
- Create: `Validator`, `PasswordHasher`, `UserRepository`, `EmailService`

---

## Question 18: Open/Closed Principle (Design)
**Difficulty:** Hard

**Problem:** You have a payment processing system. Currently, it supports Credit Card and PayPal. Tomorrow, you need to add Stripe.

**Current Code:**
```python
def process_payment(payment_type, amount):
    if payment_type == 'credit_card':
        # CC logic
    elif payment_type == 'paypal':
        # PayPal logic
    elif payment_type == 'stripe':
        # New logic
```

**Question:** Refactor this to follow Open/Closed Principle (open for extension, closed for modification).

**Expected Approach:**
```python
class PaymentProcessor(ABC):
    @abstractmethod
    def process(self, amount): pass

class CreditCardProcessor(PaymentProcessor): ...
class PayPalProcessor(PaymentProcessor): ...

def process_payment(processor: PaymentProcessor, amount):
    return processor.process(amount)
```

---

## Question 19: Dependency Injection (Real-world)
**Difficulty:** Hard

**Scenario:** You have a `UserService` that fetches users from a database. In tests, you want to use a mock database.

**Bad Code:**
```python
class UserService:
    def __init__(self):
        self.db = RealDatabase()
    
    def get_user(self, user_id):
        return self.db.fetch(user_id)
```

**Question:** Show how to use dependency injection to make this testable.

**Expected Approach:**
```python
class UserService:
    def __init__(self, db):
        self.db = db
    
    def get_user(self, user_id):
        return self.db.fetch(user_id)

# In production:
service = UserService(RealDatabase())

# In tests:
service = UserService(MockDatabase())
```

---

## Question 20: Factory Pattern (Design)
**Difficulty:** Medium

Create a factory for different types of loggers (Console, File, CloudWatch). The factory should:
1. Return the correct logger based on config
2. Be extensible without modifying existing code
3. Support singleton instances

**Expected Approach:**
```python
class LoggerFactory:
    _loggers = {}
    
    @staticmethod
    def get_logger(log_type):
        if log_type not in LoggerFactory._loggers:
            if log_type == 'console':
                LoggerFactory._loggers[log_type] = ConsoleLogger()
            elif log_type == 'file':
                LoggerFactory._loggers[log_type] = FileLogger()
        return LoggerFactory._loggers[log_type]
```

---

# Data Structures & Algorithms

## Question 21: Stack Implementation and Use Cases (Real-world)
**Difficulty:** Medium

**Problem:** Implement a stack with O(1) push, pop, and **min()** operations.

```python
class MinStack:
    def push(self, val): pass
    def pop(self): pass
    def top(self): pass
    def getMin(self): pass

# Example
ms = MinStack()
ms.push(3)
ms.push(2)
ms.push(1)
print(ms.getMin())  # 1
ms.pop()
print(ms.getMin())  # 2
```

**Constraints:** All operations O(1) time and space.

**Expected Approach:**
- Maintain two stacks: one for values, one for minimums
- When pushing, track minimum at each level

---

## Question 22: Queue Implementation using Collections (Real-world)
**Difficulty:** Medium

Implement a Circular Queue with fixed size using `collections.deque`.

**Requirements:**
1. Fixed capacity (e.g., 5)
2. Handle overflow gracefully (reject or overwrite)
3. Track size efficiently

```python
from collections import deque

class CircularQueue:
    def __init__(self, capacity):
        self.queue = deque(maxlen=capacity)
    
    def enqueue(self, val):
        # What happens if queue is full?
        pass
    
    def dequeue(self):
        pass
    
    def is_full(self):
        pass
    
    def is_empty(self):
        pass
```

**Question:** Explain deque's `maxlen` behavior and when it's useful.

---

## Question 23: LRU Cache Implementation (Hard)
**Difficulty:** Hard

**Problem:** Implement an LRU (Least Recently Used) Cache.

```python
class LRUCache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = {}
    
    def get(self, key):
        # Return value, mark as recently used
        pass
    
    def put(self, key, value):
        # Set value, mark as recently used
        # Evict LRU item if at capacity
        pass

# Example
cache = LRUCache(2)
cache.put(1, 1)
cache.put(2, 2)
print(cache.get(1))  # 1 (1 is now most recently used)
cache.put(3, 3)      # Evicts key 2
print(cache.get(2))  # -1 (not found)
```

**Constraints:** All operations O(1) average time.

**Expected Approach:**
- Use `OrderedDict` or combination of dict + doubly-linked list
- Move accessed items to end
- Evict from front when at capacity

---

## Question 24: Hash Table Collision Handling (Interview)
**Difficulty:** Hard

**Problem:** Implement a simple hash table with collision handling.

**Question:** Compare two collision resolution strategies:
1. **Chaining:** Use linked lists for collisions
2. **Open Addressing:** Use linear/quadratic probing

For each:
- Time complexity: insertion, lookup, deletion
- Space complexity
- Best/worst case scenarios
- Real-world use cases

**Expected Approach:**
- Chaining: O(1) average, O(n) worst case
- Open addressing: O(1) to O(n) depending on load factor and probing strategy
- Chaining better for sparse tables
- Open addressing better for cache locality

---

## Question 25: Sorting Algorithms - Complexity Analysis (Interview)
**Difficulty:** Hard

**Problem:** You're designing a data processing pipeline. You have 1 million user records to sort by age.

**Options:**
1. Python's built-in `sorted()` (Timsort)
2. Quicksort
3. Counting sort
4. Radix sort

**Question:** Which would you choose and why?

**Constraints:** Consider:
- Worst-case time complexity
- Space complexity
- Stability (do equal elements maintain order?)
- Data characteristics (is data partially sorted?)
- Real-world performance

**Expected Approach:**
- Python's `sorted()`: O(n log n), stable, optimized for real-world data
- Counting/Radix: O(n) but only for integers in limited range
- Answer: Use `sorted()` unless you have specific constraints

---

## Question 26: Binary Search Trees and Balancing (Interview)
**Difficulty:** Hard

**Problem:** Implement a self-balancing BST (AVL Tree) insert operation.

**Requirements:**
1. Insert maintains balance (left and right subtree heights differ by ≤1)
2. Detect imbalance after insertion
3. Rebalance using rotations

**Question:** Explain the four rotation cases (LL, LR, RL, RR) for rebalancing.

**Expected Approach:**
```python
class AVLNode:
    def __init__(self, val):
        self.val = val
        self.left = None
        self.right = None
        self.height = 1

def insert(node, val):
    if not node:
        return AVLNode(val)
    
    if val < node.val:
        node.left = insert(node.left, val)
    else:
        node.right = insert(node.right, val)
    
    # Update height
    node.height = 1 + max(height(node.left), height(node.right))
    
    # Check balance and rebalance
    balance = height(node.left) - height(node.right)
    
    # LL case
    if balance > 1 and val < node.left.val:
        return rotate_right(node)
    
    # RR case
    if balance < -1 and val > node.right.val:
        return rotate_left(node)
    
    # LR case
    if balance > 1 and val > node.left.val:
        node.left = rotate_left(node.left)
        return rotate_right(node)
    
    # RL case
    if balance < -1 and val < node.right.val:
        node.right = rotate_right(node.right)
        return rotate_left(node)
    
    return node
```

---

## Question 27: Graph Traversal - BFS vs DFS (Optimization)
**Difficulty:** Medium

**Problem:** Given a graph, find the shortest path from source to destination.

**Question:** Would you use BFS or DFS? Why?

**Expected Approach:**
- BFS finds shortest path in unweighted graphs
- DFS finds any path, not necessarily shortest
- For weighted graphs, use Dijkstra or Bellman-Ford

---

## Question 28: Dynamic Programming - Fibonacci (Real-world)
**Difficulty:** Hard

**Problem:** Compute the nth Fibonacci number efficiently.

**Approaches:**
1. Recursion: O(2^n)
2. Memoization (top-down DP): O(n)
3. Tabulation (bottom-up DP): O(n)
4. Matrix exponentiation: O(log n)

**Question:** Implement all three and explain trade-offs.

**Expected Approach:**
```python
# Memoization
def fib(n, memo={}):
    if n in memo:
        return memo[n]
    if n <= 1:
        return n
    memo[n] = fib(n-1, memo) + fib(n-2, memo)
    return memo[n]

# Tabulation
def fib(n):
    if n <= 1:
        return n
    dp = [0] * (n + 1)
    dp[1] = 1
    for i in range(2, n + 1):
        dp[i] = dp[i-1] + dp[i-2]
    return dp[n]

# Space optimization
def fib(n):
    if n <= 1:
        return n
    a, b = 0, 1
    for _ in range(2, n + 1):
        a, b = b, a + b
    return b
```

---

## Question 29: String Algorithms - Pattern Matching (Interview)
**Difficulty:** Hard

**Problem:** Find all occurrences of a pattern in a text.

**Text:** "ABABDABACDABABCABAB"  
**Pattern:** "ABABCABAB"

**Approaches:**
1. Naive: O(n*m)
2. KMP (Knuth-Morris-Pratt): O(n+m)
3. Boyer-Moore: O(n/m) average case
4. Rabin-Karp: O(n+m) with hashing

**Question:** Explain KMP algorithm and build the failure function.

**Expected Approach:**
- Failure function: for each position, track longest proper prefix which is also suffix
- Avoid unnecessary re-comparisons

---

## Question 30: Time and Space Complexity Analysis (Interview)
**Difficulty:** Hard

**Code:**
```python
def find_pairs(arr, target):
    """Find all pairs in array that sum to target."""
    pairs = []
    seen = set()
    
    for num in arr:
        complement = target - num
        if complement in seen:
            pairs.append((num, complement))
        seen.add(num)
    
    return pairs
```

**Question:**
1. What's the time complexity?
2. What's the space complexity?
3. Can you optimize further?
4. What if the array is sorted?

**Expected Approach:**
- Current: O(n) time, O(n) space (excellent)
- Two-pointer for sorted: O(n) time, O(1) space
- Trade-off: sorting takes O(n log n), but saves space

---

# Concurrency & Performance

## Question 31: Global Interpreter Lock (GIL) (Conceptual)
**Difficulty:** Hard

**Question:** Explain Python's GIL. Why does it exist? What are the implications?

```python
import threading
import time

counter = 0

def increment():
    global counter
    for _ in range(1_000_000):
        counter += 1

# Single thread
start = time.time()
increment()
print(f"Time: {time.time() - start}")
print(f"Counter: {counter}")

# Two threads
counter = 0
start = time.time()
t1 = threading.Thread(target=increment)
t2 = threading.Thread(target=increment)
t1.start()
t2.start()
t1.join()
t2.join()
print(f"Time: {time.time() - start}")
print(f"Counter: {counter}")  # Why not 2,000,000?
```

**Expected Approach:**
- GIL prevents true parallelism for CPU-bound tasks
- Only one thread executes Python bytecode at a time
- I/O-bound tasks can release GIL
- Counter won't be 2,000,000 due to race conditions
- Solutions: multiprocessing, async, Cython, PyPy

---

## Question 32: Threading vs Multiprocessing (Design Decision)
**Difficulty:** Hard

**Scenario:** You need to:
1. Fetch data from 100 URLs concurrently
2. Process 10 GB of data in parallel
3. Run 10 tasks that alternate between CPU and I/O

**Question:** For each scenario, choose threading or multiprocessing. Justify.

**Expected Approach:**
1. URLs: Threading (I/O-bound, fast context switching)
2. Data: Multiprocessing (CPU-bound, bypasses GIL)
3. Mixed: Async/asyncio (best for mixed workloads)

---

## Question 33: Async/Await and Event Loop (Real-world)
**Difficulty:** Hard

```python
import asyncio

async def fetch_data(url):
    print(f"Fetching {url}")
    await asyncio.sleep(2)  # Simulate network call
    print(f"Done {url}")
    return f"Data from {url}"

async def main():
    tasks = [
        fetch_data("url1"),
        fetch_data("url2"),
        fetch_data("url3"),
    ]
    results = await asyncio.gather(*tasks)
    return results

asyncio.run(main())
```

**Question:**
1. How long does this take? (2s or 6s?)
2. Explain `asyncio.gather()` vs `await` sequentially
3. What's the difference between `asyncio.create_task()` and `await`?

**Expected Approach:**
- Takes ~2s (concurrent)
- `gather()` runs tasks concurrently
- Sequential `await` takes ~6s
- `create_task()` schedules immediately, `await` waits

---

## Question 34: Race Conditions and Thread Safety (Debugging)
**Difficulty:** Hard

**Problem:** Identify and fix the race condition.

```python
import threading

class BankAccount:
    def __init__(self, balance):
        self.balance = balance
    
    def withdraw(self, amount):
        if self.balance >= amount:
            # Race condition here!
            self.balance -= amount

# Concurrent withdrawals
account = BankAccount(1000)

def withdraw_task():
    for _ in range(100):
        account.withdraw(5)

threads = [threading.Thread(target=withdraw_task) for _ in range(10)]
for t in threads:
    t.start()
for t in threads:
    t.join()

print(account.balance)  # Expected: 1000 - 5*100*10 = 500
```

**Question:** Why is this wrong? How to fix it?

**Expected Approach:**
- Check and update are not atomic
- Solution: use `threading.Lock()`

```python
class BankAccount:
    def __init__(self, balance):
        self.balance = balance
        self.lock = threading.Lock()
    
    def withdraw(self, amount):
        with self.lock:
            if self.balance >= amount:
                self.balance -= amount
```

---

## Question 35: Deadlock Detection and Prevention (Interview)
**Difficulty:** Hard

**Problem:** Analyze this code for deadlock potential.

```python
import threading

lock1 = threading.Lock()
lock2 = threading.Lock()

def thread_a():
    with lock1:
        print("A acquired lock1")
        import time; time.sleep(0.1)
        with lock2:
            print("A acquired lock2")

def thread_b():
    with lock2:
        print("B acquired lock2")
        import time; time.sleep(0.1)
        with lock1:
            print("B acquired lock1")

t1 = threading.Thread(target=thread_a)
t2 = threading.Thread(target=thread_b)
t1.start()
t2.start()
```

**Question:**
1. Will this deadlock?
2. How to prevent it?
3. What's the general rule for lock ordering?

**Expected Approach:**
- Likely to deadlock (A holds lock1, waits for lock2; B holds lock2, waits for lock1)
- Prevention: always acquire locks in same order
- Use lock ordering: `if id(lock1) < id(lock2): acquire lock1 first`

---

## Question 36: Iterators and Generators (Performance)
**Difficulty:** Medium

```python
def gen_squares(n):
    for i in range(n):
        yield i ** 2

# vs

def list_squares(n):
    return [i ** 2 for i in range(n)]

# Memory usage for n=10,000,000?
gen = gen_squares(10_000_000)
lst = list_squares(10_000_000)
```

**Question:** Compare memory usage, performance, and when each is appropriate.

**Expected Approach:**
- Generator: O(1) memory, produces values on-demand
- List: O(n) memory, all values upfront
- Generator for streaming, list for multiple access

---

## Question 37: Context Managers and Resource Management (Real-world)
**Difficulty:** Medium

**Problem:** Write a context manager for database connections.

```python
class DatabaseConnection:
    def __enter__(self):
        print("Connecting to database...")
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        print("Closing connection...")
        if exc_type:
            print(f"Exception occurred: {exc_type}")
        return False  # Don't suppress exceptions

# Usage
with DatabaseConnection() as db:
    print("Using database")
    # Connection auto-closes
```

**Question:** Explain the protocol and compare with `@contextmanager` decorator.

**Expected Approach:**
```python
from contextlib import contextmanager

@contextmanager
def database_connection():
    print("Connecting...")
    try:
        yield None
    finally:
        print("Closing...")
```

---

## Question 38: Profiling and Optimization (Real-world)
**Difficulty:** Hard

**Problem:** You have a function that processes 1 million records and it's slow.

```python
def process_records(records):
    result = []
    for record in records:
        if is_valid(record):
            processed = transform(record)
            result.append(processed)
    return result

def is_valid(record):
    # Expensive check
    return 'required_field' in record and len(record['data']) > 0

def transform(record):
    # Expensive transformation
    return {'id': record['id'], 'value': record['data'].upper()}
```

**Question:**
1. How would you profile this?
2. What are likely bottlenecks?
3. Optimize for speed and memory.

**Expected Approach:**
- Use `cProfile` or `line_profiler`
- Bottlenecks: likely `is_valid()` or `transform()`
- Optimizations:
  - Cache expensive computations
  - Use list comprehension (faster than append loop)
  - Consider vectorization with NumPy for large datasets
  - Use generators if result will be processed sequentially

---

## Question 39: Memory Leaks in Python (Debugging)
**Difficulty:** Hard

**Problem:** Identify and fix memory leak.

```python
class Cache:
    instances = []
    
    def __init__(self):
        Cache.instances.append(self)
        self.data = [0] * 1_000_000

# Creating many objects
for _ in range(1000):
    cache = Cache()

# Caches are never garbage collected!
# Why? What's the fix?
```

**Question:** Why is this a leak? How to fix?

**Expected Approach:**
- `Cache.instances` holds references to all created instances
- They never get garbage collected
- Fix: use weak references

```python
import weakref

class Cache:
    instances = []
    
    def __init__(self):
        Cache.instances.append(weakref.ref(self))
```

---

## Question 40: Caching Strategies (Optimization)
**Difficulty:** Hard

**Problem:** You have an API that's called 1000 times/second with 100 unique queries.

**Options:**
1. LRU Cache: keep most-used 50 queries
2. TTL Cache: keep queries for 5 minutes
3. Write-through Cache: update before invalidating
4. Write-back Cache: invalidate first, update later

**Question:** Which caching strategy is best? Why?

**Expected Approach:**
- Use TTL cache for time-sensitive data
- Use LRU for frequently accessed data
- Write-through: safer, slower
- Write-back: faster, risk of inconsistency
- For this scenario: LRU (working set is 100 queries, keep 50 for hit rate) or TTL (5-min staleness is acceptable)

---

# Libraries & Real-world Use Cases

## Question 41: Requests Module and Error Handling (Real-world)
**Difficulty:** Medium

**Problem:** Fetch data from multiple endpoints with retries and timeout.

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

def get_with_retries(url, max_retries=3):
    """Fetch URL with exponential backoff."""
    session = requests.Session()
    
    # Configure retries
    retry = Retry(
        total=max_retries,
        backoff_factor=0.5,
        status_forcelist=(500, 502, 503, 504),
    )
    
    adapter = HTTPAdapter(max_retries=retry)
    session.mount("http://", adapter)
    session.mount("https://", adapter)
    
    try:
        response = session.get(url, timeout=5)
        response.raise_for_status()
        return response.json()
    except requests.ConnectionError:
        # Handle connection error
        pass
    except requests.Timeout:
        # Handle timeout
        pass
    except requests.HTTPError as e:
        # Handle HTTP errors (4xx, 5xx)
        pass
```

**Question:** Explain each error type and best practices for handling.

---

## Question 42: Pandas Data Manipulation (Real-world)
**Difficulty:** Hard

**Problem:** Given a CSV with user data, find top 10 users by purchase amount, group by region, and calculate average purchase frequency.

```python
import pandas as pd

# Load data
df = pd.read_csv('users.csv')
# Columns: user_id, region, purchase_amount, purchase_date

# Task 1: Top 10 users by purchase amount
# Task 2: Group by region, get stats
# Task 3: Calculate average purchase frequency
```

**Expected Approach:**
```python
# Top 10
top_users = df.nlargest(10, 'purchase_amount')

# Group by region
region_stats = df.groupby('region')['purchase_amount'].agg(['sum', 'mean', 'count'])

# Purchase frequency (purchases per user per month)
df['purchase_date'] = pd.to_datetime(df['purchase_date'])
df['year_month'] = df['purchase_date'].dt.to_period('M')
frequency = df.groupby('user_id').size() / df.groupby('user_id')['purchase_date'].apply(
    lambda x: (x.max() - x.min()).days / 30
)
```

---

## Question 43: File Reading/Writing - Optimization (Real-world)
**Difficulty:** Hard

**Problem:** Read a 10 GB CSV file, filter rows, and write to new file.

```python
# Bad approach: load entire file
df = pd.read_csv('huge_file.csv')
df_filtered = df[df['status'] == 'active']
df_filtered.to_csv('output.csv')

# Better approach?
```

**Question:** How to handle large files efficiently?

**Expected Approach:**
```python
# Use chunks
chunk_size = 50000
chunks = pd.read_csv('huge_file.csv', chunksize=chunk_size)

with open('output.csv', 'w') as f:
    for i, chunk in enumerate(chunks):
        filtered = chunk[chunk['status'] == 'active']
        filtered.to_csv(f, header=(i==0), index=False)

# Or use iterator
import csv
with open('huge_file.csv', 'r') as infile, open('output.csv', 'w') as outfile:
    reader = csv.DictReader(infile)
    writer = csv.DictWriter(outfile, fieldnames=reader.fieldnames)
    writer.writeheader()
    for row in reader:
        if row['status'] == 'active':
            writer.writerow(row)
```

---

## Question 44: Multiprocessing - Parallel Processing (Real-world)
**Difficulty:** Hard

**Problem:** Process 1 million images in parallel, apply filter, save results.

```python
from multiprocessing import Pool, cpu_count
import os

def process_image(image_path):
    """Heavy CPU operation."""
    img = load_image(image_path)
    filtered = apply_filter(img)
    save_image(filtered)

image_paths = [f"images/{f}" for f in os.listdir('images/')]

# How to parallelize?
```

**Expected Approach:**
```python
with Pool(cpu_count()) as pool:
    pool.map(process_image, image_paths)

# With progress tracking
from tqdm import tqdm
with Pool(cpu_count()) as pool:
    list(tqdm(pool.imap(process_image, image_paths), total=len(image_paths)))

# Or with chunks for better performance
pool.map(process_image, image_paths, chunksize=100)
```

---

## Question 45: Email Sending and Error Handling (Real-world)
**Difficulty:** Medium

**Problem:** Send bulk emails with error handling and retry logic.

```python
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
import time

def send_email(smtp_server, sender, password, recipient, subject, body, max_retries=3):
    """Send email with exponential backoff."""
    for attempt in range(max_retries):
        try:
            msg = MIMEMultipart()
            msg['From'] = sender
            msg['To'] = recipient
            msg['Subject'] = subject
            msg.attach(MIMEText(body, 'html'))
            
            with smtplib.SMTP(smtp_server, 587) as server:
                server.starttls()
                server.login(sender, password)
                server.send_message(msg)
            
            return True
        
        except smtplib.SMTPAuthenticationError:
            print("Auth failed, not retrying")
            return False
        
        except smtplib.SMTPException as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                print(f"Attempt {attempt+1} failed, retrying in {wait_time}s")
                time.sleep(wait_time)
            else:
                print(f"Failed after {max_retries} attempts")
                return False

# Usage
send_email(
    smtp_server="smtp.gmail.com",
    sender="your_email@gmail.com",
    password="app_password",
    recipient="user@example.com",
    subject="Test",
    body="<h1>Hello</h1>"
)
```

**Question:** Discuss best practices, rate limiting, and async email sending.

---

## Question 46: Unit Testing Best Practices (Real-world)
**Difficulty:** Hard

**Problem:** Write comprehensive unit tests for a payment processor.

```python
class PaymentProcessor:
    def process(self, amount, payment_method):
        if amount <= 0:
            raise ValueError("Amount must be positive")
        
        if payment_method == 'card':
            return self.process_card(amount)
        elif payment_method == 'paypal':
            return self.process_paypal(amount)
        else:
            raise ValueError("Unknown payment method")

import unittest
from unittest.mock import patch, MagicMock

class TestPaymentProcessor(unittest.TestCase):
    def setUp(self):
        self.processor = PaymentProcessor()
    
    def test_invalid_amount(self):
        # Test negative amount
        pass
    
    def test_unknown_method(self):
        # Test invalid payment method
        pass
    
    @patch.object(PaymentProcessor, 'process_card')
    def test_card_processing(self, mock_card):
        # Test card processing
        pass
```

**Question:** Identify test cases needed. Discuss mocking and fixtures.

---

## Question 47: Logging Best Practices (Real-world)
**Difficulty:** Medium

**Problem:** Set up structured logging for a production application.

```python
import logging
import logging.handlers
import json

# Configure logging
logger = logging.getLogger(__name__)

# What about:
# 1. Different log levels (DEBUG, INFO, WARNING, ERROR, CRITICAL)
# 2. Rotating file handler (don't let logs grow unbounded)
# 3. Structured logging (JSON format for parsing)
# 4. Different handlers for different levels
```

**Expected Approach:**
```python
import logging
import logging.handlers

logger = logging.getLogger('myapp')
logger.setLevel(logging.DEBUG)

# File handler with rotation
file_handler = logging.handlers.RotatingFileHandler(
    'app.log',
    maxBytes=10_000_000,  # 10 MB
    backupCount=5
)
file_handler.setLevel(logging.INFO)

# Console handler for errors
console_handler = logging.StreamHandler()
console_handler.setLevel(logging.ERROR)

# Formatter
formatter = logging.Formatter(
    '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
file_handler.setFormatter(formatter)
console_handler.setFormatter(formatter)

logger.addHandler(file_handler)
logger.addHandler(console_handler)
```

---

## Question 48: JSON Serialization and Custom Objects (Real-world)
**Difficulty:** Medium

**Problem:** Serialize custom Python objects to JSON.

```python
import json
from datetime import datetime

class User:
    def __init__(self, id, name, created_at):
        self.id = id
        self.name = name
        self.created_at = created_at

user = User(1, "Alice", datetime.now())

# How to convert to JSON?
# json.dumps(user)  # TypeError: Object of type User is not JSON serializable
```

**Question:** Show two approaches (custom encoder vs `__dict__`).

**Expected Approach:**
```python
# Approach 1: Custom encoder
class UserEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, User):
            return {
                'id': obj.id,
                'name': obj.name,
                'created_at': obj.created_at.isoformat()
            }
        return super().default(obj)

json.dumps(user, cls=UserEncoder)

# Approach 2: Convert to dict
json.dumps(user.__dict__, default=str)
```

---

## Question 49: SQLAlchemy ORM and N+1 Query Problem (Real-world)
**Difficulty:** Hard

**Problem:** Identify and fix N+1 query problem.

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy import Column, Integer, String, ForeignKey

Base = declarative_base()

class User(Base):
    __tablename__ = 'users'
    id = Column(Integer, primary_key=True)
    name = Column(String)
    posts = relationship('Post')

class Post(Base):
    __tablename__ = 'posts'
    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey('users.id'))
    title = Column(String)

# N+1 problem
users = session.query(User).all()
for user in users:
    print(user.posts)  # 1 query for users + N queries for posts = N+1

# Fix?
```

**Expected Approach:**
```python
from sqlalchemy.orm import joinedload

# Option 1: eager loading
users = session.query(User).options(joinedload(User.posts)).all()

# Option 2: explicit join
from sqlalchemy import join
users = session.query(User).options(contains_eager(User.posts)).join(Post).all()

# Option 3: select load
users = session.query(User).options(selectinload(User.posts)).all()
```

---

## Question 50: Debugging with pdb and Exception Handling (Real-world)
**Difficulty:** Medium

**Problem:** Debug a complex issue using pdb.

```python
def complex_function(data):
    result = []
    for item in data:
        processed = transform(item)
        if validate(processed):
            result.append(processed)
    return result

# Add breakpoint and explain debugging process
```

**Question:** Show how to use pdb to inspect variables, step through code, and set conditional breakpoints.

**Expected Approach:**
```python
import pdb

def complex_function(data):
    result = []
    for item in data:
        pdb.set_trace()  # Start debugger here
        processed = transform(item)
        if validate(processed):
            result.append(processed)
    return result

# In Python 3.7+:
def complex_function(data):
    result = []
    for item in data:
        breakpoint()  # Cleaner syntax
        processed = transform(item)
        ...

# Commands:
# l (list) - show code
# n (next) - step over
# s (step) - step into
# c (continue) - run to next breakpoint
# p variable - print variable
# pp object - pretty print
# w (where) - print stack
# h (help) - show commands
```

---

# Bonus: Challenge Questions

These are real interview take-home problems. Spend 1-2 hours on each.

## Challenge 1: Rate Limiter Implementation (Hard)

**Problem:** Design and implement a distributed rate limiter that supports:
- Token bucket algorithm
- Sliding window log
- Sliding window counter

**Requirements:**
1. Support multiple strategies
2. Thread-safe
3. Distributed support (Redis-backed)
4. Per-user and per-IP limits
5. Burst handling

**Constraints:**
- Time complexity: O(1) per request
- Space complexity: O(users)
- Handle 10,000 requests/second

**Deliverables:**
- Implementation with tests
- Benchmark results
- Documentation of design decisions

---

## Challenge 2: Async Web Crawler (Hard)

**Problem:** Build an async web crawler that:
1. Crawls a domain recursively
2. Respects robots.txt and rate limits
3. Handles errors gracefully
4. Stores results in database
5. Monitors progress and memory

**Requirements:**
- Max 10 concurrent requests
- 1s delay between requests to same domain
- Timeout after 30s
- Handle 404, 500, timeouts
- Use asyncio, not threading

**Deliverables:**
- Async implementation
- Error handling strategy
- Memory profiling
- Example output

---

## Challenge 3: Design a Cache Layer (Hard)

**Problem:** Design and implement a multi-level cache system:
- L1: In-memory LRU cache (10 MB)
- L2: Redis cache (1 GB)
- L3: Database

**Requirements:**
1. Automatic promotion on hits
2. TTL support
3. Cache invalidation strategy
4. Metrics (hit rate, miss rate)
5. Handle cache stampede

**Constraints:**
- Sub-millisecond L1 access
- <10ms L2 access
- Eventual consistency acceptable

**Deliverables:**
- Architecture diagram
- Implementation with tests
- Performance benchmarks
- Load test results

---

## Challenge 4: Background Job Queue (Hard)

**Problem:** Design a robust background job queue system using Celery/RQ:
1. Process 1000 jobs/second
2. Handle job retries with exponential backoff
3. Track job progress
4. Priority queues
5. Dead letter queue for failures

**Requirements:**
- Process jobs out of order (not FIFO)
- Store results for 24 hours
- Alert on job failures
- Scale horizontally
- Graceful shutdown

**Deliverables:**
- Queue implementation
- Worker implementation
- Monitoring dashboard
- Failure handling strategy

---

## Challenge 5: Data Pipeline with Error Recovery (Hard)

**Problem:** Build an ETL pipeline that:
1. **Extract:** Read from multiple CSV/JSON sources
2. **Transform:** Clean, validate, and enrich data
3. **Load:** Write to database with transactional guarantees

**Requirements:**
- Process 100 GB of data daily
- Handle partial failures (skip bad records, log errors)
- Idempotent (can retry without duplicates)
- Monitor data quality
- Rollback on critical errors

**Constraints:**
- Run in <4 hours
- <10% error rate acceptable
- Memory: <2 GB at peak

**Deliverables:**
- Pipeline implementation
- Data validation schema
- Error handling & logging
- Performance metrics
- Example test data & results

---

## Answer Key & Evaluation Guide

This section would contain:
1. Expected answers for conceptual questions
2. Code solutions for implementation questions
3. Grading rubric for challenge questions
4. Common mistakes and how to identify them
5. Interview evaluation notes

**Scoring Guide:**
- Easy questions (1-2 points each)
- Medium questions (3-5 points each)
- Hard questions (5-10 points each)
- Challenge questions (20-50 points each)
- Total: 200+ points

**Assessment Criteria:**
- Correctness (does it work?)
- Efficiency (time/space complexity)
- Code quality (readability, maintainability)
- Testing (edge cases, error handling)
- Communication (explains reasoning)

---

## Resources for Further Learning

### Books
- "Fluent Python" by Luciano Ramalho
- "High Performance Python" by Micha Gorelick & Ian Ozsvald
- "Python in Practice" by Mark Summerfield

### Online Courses
- Real Python tutorials
- DataCamp Python track
- LeetCode System Design

### Practice Platforms
- LeetCode (coding problems)
- HackerRank (Python challenges)
- CodeSignal (interview prep)
- Codewars (algorithms)

### Tools & Libraries to Master
- `cProfile`, `line_profiler` - profiling
- `pytest` - testing framework
- `black`, `pylint` - code quality
- `asyncio` - async programming
- `pandas`, `numpy` - data processing
- `requests`, `httpx` - HTTP clients
- `sqlalchemy` - ORM

---

## Final Tips for Interviewees

1. **Ask clarifying questions** before coding
2. **Start with brute force**, then optimize
3. **Consider edge cases** (empty input, None, large data)
4. **Think about trade-offs** (time vs space, simplicity vs performance)
5. **Test your code** with examples
6. **Explain your reasoning** as you code
7. **Discuss alternatives** and why you chose your approach
8. **Handle errors gracefully** in real-world code
9. **Use design patterns** appropriately
10. **Profile before optimizing** (don't optimize prematurely)

---

Generated: 2026-05-04 | Python 3.9+ Compatible
