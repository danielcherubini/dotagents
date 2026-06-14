# Python Code Review Guide

> Python code review guide covering type annotations, async/await, testing, exception handling, and performance optimization. Covers Python 3.12+ features.

## Table of Contents

- [Type Annotations](#type-annotations)
- [Async Programming](#async-programming)
- [Exception Handling](#exception-handling)
- [Common Pitfalls](#common-pitfalls)
- [Testing Best Practices](#testing-best-practices)
- [Performance Optimization](#performance-optimization)
- [Code Style](#code-style)
- [Review Checklist](#review-checklist)

---

## Type Annotations

### Basic Type Annotations

```python
# ❌ No type annotations, IDE can't help
def process_data(data, count):
    return data[:count]

# ✅ Use type annotations
def process_data(data: str, count: int) -> str:
    return data[:count]

# ✅ Complex types use typing module
from typing import Optional, Union

def find_user(user_id: int) -> Optional[User]:
    """Returns user or None"""
    return db.get(user_id)

def handle_input(value: Union[str, int]) -> str:
    """Accepts string or integer"""
    return str(value)
```

### Container Type Annotations

```python
from typing import List, Dict, Set, Tuple, Sequence

# ❌ Imprecise types
def get_names(users: list) -> list:
    return [u.name for u in users]

# ✅ Precise container types (Python 3.9+ can use list[User] directly)
def get_names(users: List[User]) -> List[str]:
    return [u.name for u in users]

# ✅ Use Sequence for read-only sequences (more flexible)
def process_items(items: Sequence[str]) -> int:
    return len(items)

# ✅ Dictionary types
def count_words(text: str) -> Dict[str, int]:
    words: Dict[str, int] = {}
    for word in text.split():
        words[word] = words.get(word, 0) + 1
    return words

# ✅ Tuple (fixed length and type)
def get_point() -> Tuple[float, float]:
    return (1.0, 2.0)

# ✅ Variable-length tuple
def get_scores() -> Tuple[int, ...]:
    return (90, 85, 92, 88)
```

### Generics and TypeVar

```python
from typing import TypeVar, Generic, List, Callable

T = TypeVar('T')
K = TypeVar('K')
V = TypeVar('V')

# ✅ Generic function
def first(items: List[T]) -> T | None:
    return items[0] if items else None

# ✅ Constrained TypeVar
from typing import Hashable
H = TypeVar('H', bound=Hashable)

def dedupe(items: List[H]) -> List[H]:
    return list(set(items))

# ✅ Generic class
class Cache(Generic[K, V]):
    def __init__(self) -> None:
        self._data: Dict[K, V] = {}

    def get(self, key: K) -> V | None:
        return self._data.get(key)

    def set(self, key: K, value: V) -> None:
        self._data[key] = value
```

### Callable and Callback Functions

```python
from typing import Callable, Awaitable

# ✅ Function type annotation
Handler = Callable[[str, int], bool]

def register_handler(name: str, handler: Handler) -> None:
    handlers[name] = handler

# ✅ Async callback
AsyncHandler = Callable[[str], Awaitable[dict]]

async def fetch_with_handler(
    url: str,
    handler: AsyncHandler
) -> dict:
    return await handler(url)

# ✅ Function that returns a function
def create_multiplier(factor: int) -> Callable[[int], int]:
    def multiplier(x: int) -> int:
        return x * factor
    return multiplier
```

### TypedDict and Structured Data

```python
from typing import TypedDict, Required, NotRequired

# ✅ Define dictionary structure
class UserDict(TypedDict):
    id: int
    name: str
    email: str
    age: NotRequired[int]  # Python 3.11+

def create_user(data: UserDict) -> User:
    return User(**data)

# ✅ Partial required fields
class ConfigDict(TypedDict, total=False):
    debug: bool
    timeout: int
    host: Required[str]  # This one is required
```

### Protocol and Structural Subtyping

```python
from typing import Protocol, runtime_checkable

# ✅ Define protocol (duck-typing type check)
class Readable(Protocol):
    def read(self, size: int = -1) -> bytes: ...

class Closeable(Protocol):
    def close(self) -> None: ...

# Composed protocol
class ReadableCloseable(Readable, Closeable, Protocol):
    pass

def process_stream(stream: Readable) -> bytes:
    return stream.read()

# ✅ Runtime-checkable protocol
@runtime_checkable
class Drawable(Protocol):
    def draw(self) -> None: ...

def render(obj: object) -> None:
    if isinstance(obj, Drawable):  # Runtime check
        obj.draw()
```

---

## Async Programming

### async/await Basics

```python
import asyncio

# ❌ Synchronous blocking call
def fetch_all_sync(urls: list[str]) -> list[str]:
    results = []
    for url in urls:
        results.append(requests.get(url).text)  # Serial execution
    return results

# ✅ Async concurrent call
async def fetch_url(url: str) -> str:
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.text()

async def fetch_all(urls: list[str]) -> list[str]:
    tasks = [fetch_url(url) for url in urls]
    return await asyncio.gather(*tasks)  # Concurrent execution
```

### Async Context Manager

```python
from contextlib import asynccontextmanager
from typing import AsyncIterator

# ✅ Async context manager class
class AsyncDatabase:
    async def __aenter__(self) -> 'AsyncDatabase':
        await self.connect()
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb) -> None:
        await self.disconnect()

# ✅ Using decorator
@asynccontextmanager
async def get_connection() -> AsyncIterator[Connection]:
    conn = await create_connection()
    try:
        yield conn
    finally:
        await conn.close()

async def query_data():
    async with get_connection() as conn:
        return await conn.fetch("SELECT * FROM users")
```

### Async Iterators

```python
from typing import AsyncIterator

# ✅ Async generator
async def fetch_pages(url: str) -> AsyncIterator[dict]:
    page = 1
    while True:
        data = await fetch_page(url, page)
        if not data['items']:
            break
        yield data
        page += 1

# ✅ Using async iteration
async def process_all_pages():
    async for page in fetch_pages("https://api.example.com"):
        await process_page(page)
```

### Task Management and Cancellation

```python
import asyncio

# ❌ Forgetting to handle cancellation
async def bad_worker():
    while True:
        await do_work()  # Can't be properly cancelled

# ✅ Properly handle cancellation
async def good_worker():
    try:
        while True:
            await do_work()
    except asyncio.CancelledError:
        await cleanup()  # Clean up resources
        raise  # Re-raise so caller knows it was cancelled

# ✅ Timeout control
async def fetch_with_timeout(url: str) -> str:
    try:
        async with asyncio.timeout(10):  # Python 3.11+
            return await fetch_url(url)
    except asyncio.TimeoutError:
        return ""

# ✅ Task groups (Python 3.11+)
async def fetch_multiple():
    async with asyncio.TaskGroup() as tg:
        task1 = tg.create_task(fetch_url("url1"))
        task2 = tg.create_task(fetch_url("url2"))
    # Automatically waits after all tasks complete, exceptions propagate
    return task1.result(), task2.result()
```

### Mixing Sync and Async

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

# ✅ Run sync function in async code
async def run_sync_in_async():
    loop = asyncio.get_event_loop()
    # Use thread pool to execute blocking operation
    result = await loop.run_in_executor(
        None,  # Default thread pool
        blocking_io_function,
        arg1, arg2
    )
    return result

# ✅ Run async function in sync code
def run_async_in_sync():
    return asyncio.run(async_function())

# ❌ Don't use time.sleep in async code
async def bad_delay():
    time.sleep(1)  # Blocks the entire event loop!

# ✅ Use asyncio.sleep
async def good_delay():
    await asyncio.sleep(1)
```

### Semaphores and Rate Limiting

```python
import asyncio

# ✅ Use semaphore to limit concurrency
async def fetch_with_limit(urls: list[str], max_concurrent: int = 10):
    semaphore = asyncio.Semaphore(max_concurrent)

    async def fetch_one(url: str) -> str:
        async with semaphore:
            return await fetch_url(url)

    return await asyncio.gather(*[fetch_one(url) for url in urls])

# ✅ Use asyncio.Queue for producer-consumer
async def producer_consumer():
    queue: asyncio.Queue[str] = asyncio.Queue(maxsize=100)

    async def producer():
        for item in items:
            await queue.put(item)
        await queue.put(None)  # End signal

    async def consumer():
        while True:
            item = await queue.get()
            if item is None:
                break
            await process(item)
            queue.task_done()

    await asyncio.gather(producer(), consumer())
```

---

## Exception Handling

### Exception Catching Best Practices

```python
# ❌ Catching too broad
try:
    result = risky_operation()
except:  # Catches everything, even KeyboardInterrupt!
    pass

# ❌ Catching Exception but not handling
try:
    result = risky_operation()
except Exception:
    pass  # Swallows all exceptions, hard to debug

# ✅ Catch specific exceptions
try:
    result = risky_operation()
except ValueError as e:
    logger.error(f"Invalid value: {e}")
    raise
except IOError as e:
    logger.error(f"IO error: {e}")
    return default_value

# ✅ Multiple exception types
try:
    result = parse_and_process(data)
except (ValueError, TypeError, KeyError) as e:
    logger.error(f"Data error: {e}")
    raise DataProcessingError(str(e)) from e
```

### Exception Chains

```python
# ❌ Losing original exception info
try:
    result = external_api.call()
except APIError as e:
    raise RuntimeError("API failed")  # Lost the reason

# ✅ Use from to preserve exception chain
try:
    result = external_api.call()
except APIError as e:
    raise RuntimeError("API failed") from e

# ✅ Explicitly break exception chain (rare case)
try:
    result = external_api.call()
except APIError:
    raise RuntimeError("API failed") from None
```

### Custom Exceptions

```python
# ✅ Define business exception hierarchy
class AppError(Exception):
    """Application base exception"""
    pass

class ValidationError(AppError):
    """Data validation error"""
    def __init__(self, field: str, message: str):
        self.field = field
        self.message = message
        super().__init__(f"{field}: {message}")

class NotFoundError(AppError):
    """Resource not found"""
    def __init__(self, resource: str, id: str | int):
        self.resource = resource
        self.id = id
        super().__init__(f"{resource} with id {id} not found")

# Usage
def get_user(user_id: int) -> User:
    user = db.get(user_id)
    if not user:
        raise NotFoundError("User", user_id)
    return user
```

### Exceptions in Context Managers

```python
from contextlib import contextmanager

# ✅ Properly handle exceptions in context managers
@contextmanager
def transaction():
    conn = get_connection()
    try:
        yield conn
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        conn.close()

# ✅ Use ExceptionGroup (Python 3.11+)
def process_batch(items: list) -> None:
    errors = []
    for item in items:
        try:
            process(item)
        except Exception as e:
            errors.append(e)

    if errors:
        raise ExceptionGroup("Batch processing failed", errors)
```

---

## Common Pitfalls

### Mutable Default Arguments

```python
# ❌ Mutable default arguments
def add_item(item, items=[]):  # Bug! Shared across calls
    items.append(item)
    return items

# Problem demo
add_item(1)  # [1]
add_item(2)  # [1, 2] instead of [2]!

# ✅ Use None as default
def add_item(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items

# ✅ Or use dataclass field
from dataclasses import dataclass, field

@dataclass
class Container:
    items: list = field(default_factory=list)
```

### Mutable Class Attributes

```python
# ❌ Using mutable class attributes
class User:
    permissions = []  # Shared across all instances!

# Problem demo
u1 = User()
u2 = User()
u1.permissions.append("admin")
print(u2.permissions)  # ["admin"] - unexpectedly shared!

# ✅ Initialize in __init__
class User:
    def __init__(self):
        self.permissions = []

# ✅ Use dataclass
@dataclass
class User:
    permissions: list = field(default_factory=list)
```

### Closures in Loops

```python
# ❌ Closure capturing loop variable
funcs = []
for i in range(3):
    funcs.append(lambda: i)

print([f() for f in funcs])  # [2, 2, 2] instead of [0, 1, 2]!

# ✅ Use default parameter to capture value
funcs = []
for i in range(3):
    funcs.append(lambda i=i: i)

print([f() for f in funcs])  # [0, 1, 2]

# ✅ Use functools.partial
from functools import partial

funcs = [partial(lambda x: x, i) for i in range(3)]
```

### is vs ==

```python
# ❌ Using is to compare values
if x is 1000:  # May not work!
    pass

# Python caches small integers (-5 to 256)
a = 256
b = 256
a is b  # True

a = 257
b = 257
a is b  # False!

# ✅ Use == for value comparison
if x == 1000:
    pass

# ✅ is only for None and singletons
if x is None:
    pass

if x is True:  # Strict boolean check
    pass
```

### String Concatenation Performance

```python
# ❌ Concatenating strings in loop
result = ""
for item in large_list:
    result += str(item)  # O(n²) complexity

# ✅ Use join
result = "".join(str(item) for item in large_list)  # O(n)

# ✅ Use StringIO for building large strings
from io import StringIO

buffer = StringIO()
for item in large_list:
    buffer.write(str(item))
result = buffer.getvalue()
```

---

## Testing Best Practices

### pytest Basics

```python
import pytest

# ✅ Clear test naming
def test_user_creation_with_valid_email():
    user = User(email="test@example.com")
    assert user.email == "test@example.com"

def test_user_creation_with_invalid_email_raises_error():
    with pytest.raises(ValidationError):
        User(email="invalid")

# ✅ Parameterized testing
@pytest.mark.parametrize("input,expected", [
    ("hello", "HELLO"),
    ("World", "WORLD"),
    ("", ""),
    ("123", "123"),
])
def test_uppercase(input: str, expected: str):
    assert input.upper() == expected

# ✅ Test exceptions
def test_division_by_zero():
    with pytest.raises(ZeroDivisionError) as exc_info:
        1 / 0
    assert "division by zero" in str(exc_info.value)
```

### Fixtures

```python
import pytest
from typing import Generator

# ✅ Basic fixture
@pytest.fixture
def user() -> User:
    return User(name="Test User", email="test@example.com")

def test_user_name(user: User):
    assert user.name == "Test User"

# ✅ Fixture with cleanup
@pytest.fixture
def database() -> Generator[Database, None, None]:
    db = Database()
    db.connect()
    yield db
    db.disconnect()  # Cleanup after test

# ✅ Async fixture
@pytest.fixture
async def async_client() -> AsyncGenerator[AsyncClient, None]:
    async with AsyncClient() as client:
        yield client

# ✅ Shared fixtures (conftest.py)
# conftest.py
@pytest.fixture(scope="session")
def app():
    """App instance shared across test session"""
    return create_app()

@pytest.fixture(scope="module")
def db(app):
    """Database connection shared per test module"""
    return app.db
```

### Mock and Patch

```python
from unittest.mock import Mock, patch, AsyncMock

# ✅ Mock external dependencies
def test_send_email():
    mock_client = Mock()
    mock_client.send.return_value = True

    service = EmailService(client=mock_client)
    result = service.send_welcome_email("user@example.com")

    assert result is True
    mock_client.send.assert_called_once_with(
        to="user@example.com",
        subject="Welcome!",
        body=ANY,
    )

# ✅ Patch module-level function
@patch("myapp.services.external_api.call")
def test_with_patched_api(mock_call):
    mock_call.return_value = {"status": "ok"}

    result = process_data()

    assert result["status"] == "ok"

# ✅ Async Mock
async def test_async_function():
    mock_fetch = AsyncMock(return_value={"data": "test"})

    with patch("myapp.client.fetch", mock_fetch):
        result = await get_data()

    assert result == {"data": "test"}
```

### Test Organization

```python
# ✅ Organize related tests in class
class TestUserAuthentication:
    """User authentication related tests"""

    def test_login_with_valid_credentials(self, user):
        assert authenticate(user.email, "password") is True

    def test_login_with_invalid_password(self, user):
        assert authenticate(user.email, "wrong") is False

    def test_login_locks_after_failed_attempts(self, user):
        for _ in range(5):
            authenticate(user.email, "wrong")
        assert user.is_locked is True

# ✅ Use marks to tag tests
@pytest.mark.slow
def test_large_data_processing():
    pass

@pytest.mark.integration
def test_database_connection():
    pass

# Run specific marked tests: pytest -m "not slow"
```

### Coverage and Quality

```python
# pytest.ini or pyproject.toml
[tool.pytest.ini_options]
addopts = "--cov=myapp --cov-report=term-missing --cov-fail-under=80"
testpaths = ["tests"]

# ✅ Test edge cases
def test_empty_input():
    assert process([]) == []

def test_none_input():
    with pytest.raises(TypeError):
        process(None)

def test_large_input():
    large_data = list(range(100000))
    result = process(large_data)
    assert len(result) == 100000
```

---

## Performance Optimization

### Data Structure Selection

```python
# ❌ List lookup O(n)
if item in large_list:  # Slow
    pass

# ✅ Set lookup O(1)
large_set = set(large_list)
if item in large_set:  # Fast
    pass

# ✅ Use collections module
from collections import Counter, defaultdict, deque

# Counting
word_counts = Counter(words)
most_common = word_counts.most_common(10)

# Default dictionary
graph = defaultdict(list)
graph[node].append(neighbor)

# Deque (O(1) operations at both ends)
queue = deque()
queue.appendleft(item)  # O(1) vs list.insert(0, item) O(n)
```

### Generators and Iterators

```python
# ❌ Loading all data at once
def get_all_users():
    return [User(row) for row in db.fetch_all()]  # High memory usage

# ✅ Use generator
def get_all_users():
    for row in db.fetch_all():
        yield User(row)  # Lazy loading

# ✅ Generator expression
sum_of_squares = sum(x**2 for x in range(1000000))  # Doesn't create list

# ✅ itertools module
from itertools import islice, chain, groupby

# Take only first 10
first_10 = list(islice(infinite_generator(), 10))

# Chain multiple iterators
all_items = chain(list1, list2, list3)

# Grouping
for key, group in groupby(sorted(items, key=get_key), key=get_key):
    process_group(key, list(group))
```

### Caching

```python
from functools import lru_cache, cache

# ✅ LRU cache
@lru_cache(maxsize=128)
def expensive_computation(n: int) -> int:
    return sum(i**2 for i in range(n))

# ✅ Unlimited cache (Python 3.9+)
@cache
def fibonacci(n: int) -> int:
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

# ✅ Manual caching (when more control needed)
class DataService:
    def __init__(self):
        self._cache: dict[str, Any] = {}
        self._cache_ttl: dict[str, float] = {}

    def get_data(self, key: str) -> Any:
        if key in self._cache:
            if time.time() < self._cache_ttl[key]:
                return self._cache[key]

        data = self._fetch_data(key)
        self._cache[key] = data
        self._cache_ttl[key] = time.time() + 300  # 5 minutes
        return data
```

### Parallel Processing

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

# ✅ Use thread pool for IO-bound work
def fetch_all_urls(urls: list[str]) -> list[str]:
    with ThreadPoolExecutor(max_workers=10) as executor:
        results = list(executor.map(fetch_url, urls))
    return results

# ✅ Use process pool for CPU-bound work
def process_large_dataset(data: list) -> list:
    with ProcessPoolExecutor() as executor:
        results = list(executor.map(heavy_computation, data))
    return results

# ✅ Use as_completed to get results as they finish
from concurrent.futures import as_completed

with ThreadPoolExecutor() as executor:
    futures = {executor.submit(fetch, url): url for url in urls}
    for future in as_completed(futures):
        url = futures[future]
        try:
            result = future.result()
        except Exception as e:
            print(f"{url} failed: {e}")
```

---

## Code Style

### PEP 8 Essentials

```python
# ✅ Naming conventions
class MyClass:  # Class names PascalCase
    MAX_SIZE = 100  # Constants UPPER_SNAKE_CASE

    def method_name(self):  # Methods snake_case
        local_var = 1  # Variables snake_case

# ✅ Import order
# 1. Standard library
import os
import sys
from typing import Optional

# 2. Third-party libraries
import numpy as np
import pandas as pd

# 3. Local modules
from myapp import config
from myapp.utils import helper

# ✅ Line length limit (79 or 88 characters)
# Line breaks for long expressions
result = (
    long_function_name(arg1, arg2, arg3)
    + another_long_function(arg4, arg5)
)

# ✅ Blank line conventions
class MyClass:
    """Class docstring"""

    def method_one(self):
        pass

    def method_two(self):  # One blank line between methods
        pass


def top_level_function():  # Two blank lines between top-level definitions
    pass
```

### Docstrings

```python
# ✅ Google style docstrings
def calculate_area(width: float, height: float) -> float:
    """Calculate rectangle area.

    Args:
        width: Rectangle width (must be positive).
        height: Rectangle height (must be positive).

    Returns:
        Rectangle area.

    Raises:
        ValueError: If width or height is negative.

    Example:
        >>> calculate_area(3, 4)
        12.0
    """
    if width < 0 or height < 0:
        raise ValueError("Dimensions must be positive")
    return width * height

# ✅ Class docstrings
class DataProcessor:
    """Tool class for processing and transforming data.

    Attributes:
        source: Data source path.
        format: Output format ('json' or 'csv').

    Example:
        >>> processor = DataProcessor("data.csv")
        >>> processor.process()
    """
```

### Modern Python Features

```python
# ✅ f-strings (Python 3.6+)
name = "World"
print(f"Hello, {name}!")

# With expressions
print(f"Result: {1 + 2 = }")  # "Result: 1 + 2 = 3"

# ✅ Walrus operator (Python 3.8+)
if (n := len(items)) > 10:
    print(f"List has {n} items")

# ✅ Positional-only parameter separator (Python 3.8+)
def greet(name, /, greeting="Hello", *, punctuation="!"):
    """name can only be passed positionally, punctuation keyword-only"""
    return f"{greeting}, {name}{punctuation}"

# ✅ Pattern matching (Python 3.10+)
def handle_response(response: dict):
    match response:
        case {"status": "ok", "data": data}:
            return process_data(data)
        case {"status": "error", "message": msg}:
            raise APIError(msg)
        case _:
            raise ValueError("Unknown response format")

# ✅ Type parameter defaults (Python 3.13+)
from typing import TypeVar, Generic

T = TypeVar('T', default=str)  # Default type if not specified

class Container(Generic[T]):
    def __init__(self, value: T) -> None:
        self.value = value

# Usage: Container() now defaults to str
container = Container("hello")  # T inferred as str
```

### Python 3.12+ Improvements

```python
# ✅ tomllib (standard library TOML parser)
import tomllib
with open("pyproject.toml", "rb") as f:
    config = tomllib.load(f)  # No need for third-party toml package

# ✅ Improved error messages with 'Did you mean...'
# Python 3.12+ shows helpful suggestions for common mistakes

# ✅ f-string improvements: self-documenting expressions
value = 42
print(f"{value=}")  # "value=42"
```

---

## Review Checklist

### Type Safety
- [ ] Functions have type annotations (parameters and return values)
- [ ] Use `Optional` to explicitly indicate possible None
- [ ] Generic types used correctly
- [ ] mypy check passes (no errors)
- [ ] Avoid using `Any`, add comments explaining when necessary

### Async Code
- [ ] async/await properly paired
- [ ] No blocking calls in async code
- [ ] CancelledError properly handled
- [ ] Using `asyncio.gather` or `TaskGroup` for concurrent execution
- [ ] Resources properly cleaned up (async context manager)

### Exception Handling
- [ ] Catching specific exception types, no bare `except:`
- [ ] Exception chains use `from` to preserve cause
- [ ] Custom exceptions inherit from appropriate base classes
- [ ] Exception messages are meaningful for debugging

### Data Structures
- [ ] No mutable default arguments (list, dict, set)
- [ ] Class attributes are not mutable objects
- [ ] Correct data structure selected (set vs list lookup)
- [ ] Large datasets use generators instead of lists

### Testing
- [ ] Test coverage meets target (recommend ≥80%)
- [ ] Test names clearly describe test scenarios
- [ ] Edge cases have test coverage
- [ ] Mocks properly isolate external dependencies
- [ ] Async code has corresponding async tests

### Code Style
- [ ] Follows PEP 8 style guide
- [ ] Functions and classes have docstrings
- [ ] Import order correct (standard library, third-party, local)
- [ ] Naming is consistent and meaningful
- [ ] Uses modern Python features (f-strings, walrus operator, etc.)

### Performance
- [ ] Avoid creating objects repeatedly in loops
- [ ] String concatenation uses join
- [ ] Caching used appropriately (@lru_cache)
- [ ] IO/CPU-bound work uses appropriate parallelization
