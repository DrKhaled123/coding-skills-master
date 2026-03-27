# Python Patterns — Professional Development Guide

> **VIP EXPERT NOTE:** Python's dynamic typing makes it *especially* vulnerable to
> runtime type errors that TypeScript would catch at compile time. Compensate with:
> type hints on every function, Pydantic/dataclass validation at boundaries,
> and mypy/pyright in strict mode in your CI pipeline. The patterns below
> represent production-hardened solutions from high-scale Python applications.

---

## 🔷 MUTABLE DEFAULT ARGUMENTS (Classic Python Bug)

```python
# ❌ BROKEN: Default list is created ONCE at function definition
def add_item(item: str, items: list = []) -> list:
    items.append(item)
    return items

add_item("apple")   # ['apple']
add_item("banana")  # ['apple', 'banana'] ← BUG! Shares same list
add_item("cherry")  # ['apple', 'banana', 'cherry']

# ✅ CORRECT: Use None as sentinel, create new object each call
def add_item(item: str, items: list[str] | None = None) -> list[str]:
    if items is None:
        items = []
    items.append(item)
    return items
```

> **VIP EXPERT NOTE:** This is Python's most infamous gotcha. Default arguments
> are evaluated ONCE when the function is defined, not each time it's called.
> This affects mutable types: lists, dicts, sets, and custom objects. The None
> sentinel pattern is the idiomatic fix. For dataclass defaults, use
> `default_factory=list` instead of `default=[]`.

---

## 🔷 CONTEXT MANAGERS FOR RESOURCE SAFETY

```python
# ❌ BROKEN: Resource leak if exception occurs
def read_and_process(filepath: str) -> str:
    f = open(filepath)
    data = f.read()    # if this raises → file never closed
    result = process(data)
    f.close()
    return result

# ✅ CORRECT: Context manager guarantees cleanup
from pathlib import Path

def read_and_process(filepath: Path) -> str:
    with filepath.open(encoding='utf-8') as f:
        data = f.read()
    return process(data)

# Custom context manager for complex resources
from contextlib import contextmanager
from typing import Generator

@contextmanager
def get_db_connection(url: str) -> Generator[Connection, None, None]:
    conn = create_connection(url)
    try:
        yield conn
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        conn.close()  # ALWAYS runs

# Usage
with get_db_connection(DATABASE_URL) as conn:
    execute_query(conn, "INSERT INTO users ...")
```

> **VIP EXPERT NOTE:** Context managers ensure cleanup happens regardless of
> exceptions. The `finally` block always executes, preventing resource leaks.
> Use `contextlib.ExitStack` for managing multiple resources dynamically.
> For async resources, use `@asynccontextmanager` (see Async Context Managers below).

---

## 🔷 DATACLASSES AND PYDANTIC FOR VALIDATED DATA

```python
# ❌ BROKEN: Raw dict — no validation, no type safety, no IDE support
def create_order(data: dict) -> dict:
    return {
        "user_id": data["user_id"],   # KeyError if missing
        "total": data["total"],        # no type validation
        "items": data.get("items", []),
    }

# ✅ CORRECT: Pydantic v2 model — validation at boundary
from pydantic import BaseModel, EmailStr, field_validator, Field
from decimal import Decimal
from uuid import UUID, uuid4

class OrderItem(BaseModel):
    product_id: UUID
    quantity: int = Field(gt=0, description="Must be positive")
    unit_price: Decimal = Field(ge=0)

    @property
    def subtotal(self) -> Decimal:
        return self.unit_price * self.quantity

class CreateOrderRequest(BaseModel):
    user_id: UUID
    items: list[OrderItem] = Field(min_length=1)
    shipping_address: str = Field(min_length=10)

    @field_validator('items')
    @classmethod
    def items_must_be_unique(cls, items: list[OrderItem]) -> list[OrderItem]:
        product_ids = [item.product_id for item in items]
        if len(product_ids) != len(set(product_ids)):
            raise ValueError('Duplicate product IDs in order')
        return items

    @property
    def total(self) -> Decimal:
        return sum(item.subtotal for item in self.items)

# FastAPI automatically validates and documents:
@app.post("/orders", response_model=OrderResponse)
async def create_order(request: CreateOrderRequest) -> OrderResponse:
    order = await order_service.create(request)
    return OrderResponse.from_order(order)

# VIP NOTE on Pydantic v1 vs v2:
#   v1: @validator('field')           → v2: @field_validator('field')
#   v1: class Config: ...             → v2: model_config = ConfigDict(...)
#   v1: .dict()                       → v2: .model_dump()
#   v1: .parse_obj(data)              → v2: .model_validate(data)
```

> **VIP EXPERT NOTE:** Pydantic validates at the boundary (API, file, DB) and
> converts everything to validated Python objects. Use Field() for constraints,
> @field_validator for cross-field validation, and @model_validator for complex
> logic. Pydantic v2 is 5-50x faster than v1 due to Rust core.

---

## 🔷 GENERATOR PATTERN FOR LARGE DATA PROCESSING

```python
# ❌ BROKEN: Loads entire dataset into memory
def get_all_records(db_url: str) -> list[dict]:
    conn = get_connection(db_url)
    return conn.execute("SELECT * FROM records").fetchall()  # 10M rows in memory!

def process_all():
    records = get_all_records(DATABASE_URL)  # OOM if large table
    for record in records:
        process(record)

# ✅ CORRECT: Generator — processes one at a time, O(1) memory
from typing import Generator, Iterator

def stream_records(
    db_url: str,
    batch_size: int = 1000,
) -> Iterator[dict]:
    conn = get_connection(db_url)
    offset = 0
    while True:
        batch = conn.execute(
            "SELECT * FROM records ORDER BY id LIMIT ? OFFSET ?",
            (batch_size, offset)
        ).fetchall()
        if not batch:
            break
        yield from batch
        offset += batch_size

def process_all():
    for record in stream_records(DATABASE_URL):
        process(record)  # only one record in memory at a time
```

> **VIP EXPERT NOTE:** Generators enable streaming processing with constant memory.
> Use for: large file processing, database pagination, API pagination, and
> pipeline transformations. Combine with `itertools` for powerful compositions.
> For type safety, prefer `Iterator[T]` for outputs, `Iterable[T]` for inputs.

---

## 🔷 EXCEPTION HANDLING BEST PRACTICES

```python
# ❌ BROKEN: Catches everything, loses information
def fetch_user(user_id: str) -> dict | None:
    try:
        return api_client.get(f"/users/{user_id}")
    except:  # catches SystemExit, KeyboardInterrupt, etc!
        return None

# ❌ ALSO BAD: Too broad, swallows bugs
def fetch_user(user_id: str) -> dict | None:
    try:
        return api_client.get(f"/users/{user_id}")
    except Exception:
        return None  # hides programming errors too

# ✅ CORRECT: Specific exceptions, preserve context
class UserNotFoundError(Exception):
    def __init__(self, user_id: str) -> None:
        super().__init__(f"User {user_id!r} not found")
        self.user_id = user_id

class ApiConnectionError(Exception):
    pass

def fetch_user(user_id: str) -> dict:
    """Fetch user by ID.
    
    Raises:
        UserNotFoundError: If user does not exist.
        ApiConnectionError: If the API is unreachable.
        ValueError: If user_id is empty.
    """
    if not user_id or not isinstance(user_id, str):
        raise ValueError(f"user_id must be a non-empty string, got {user_id!r}")
    
    try:
        response = api_client.get(f"/users/{user_id}")
    except requests.ConnectionError as exc:
        raise ApiConnectionError(f"Failed to reach API: {exc}") from exc
    
    if response.status_code == 404:
        raise UserNotFoundError(user_id)
    
    response.raise_for_status()  # raises for other 4xx/5xx
    return response.json()

# Caller handles specifically:
try:
    user = fetch_user(user_id)
except UserNotFoundError:
    return {"error": "User not found"}, 404
except ApiConnectionError as exc:
    logger.error("API unreachable", exc_info=exc)
    return {"error": "Service temporarily unavailable"}, 503
```

> **VIP EXPERT NOTE:** Never use bare `except:`. It catches SystemExit and
> KeyboardInterrupt, making your program unkillable. Create custom exceptions
> for your domain, use `raise ... from ...` for exception chaining, and document
> raises in docstrings. Let unexpected errors propagate — don't hide bugs.

---

## 🔷 ASYNC PYTHON BEST PRACTICES

```python
# ✅ CORRECT: Async with proper error handling and concurrency
import asyncio
import aiohttp
from typing import NamedTuple

class FetchResult(NamedTuple):
    user_id: str
    data: dict | None
    error: str | None

async def fetch_user_async(
    session: aiohttp.ClientSession,
    user_id: str,
    timeout: float = 10.0,
) -> FetchResult:
    try:
        async with session.get(
            f"/users/{user_id}",
            timeout=aiohttp.ClientTimeout(total=timeout),
        ) as response:
            if response.status == 404:
                return FetchResult(user_id=user_id, data=None, error="not_found")
            response.raise_for_status()
            data = await response.json()
            return FetchResult(user_id=user_id, data=data, error=None)
    except asyncio.TimeoutError:
        return FetchResult(user_id=user_id, data=None, error="timeout")
    except aiohttp.ClientError as exc:
        return FetchResult(user_id=user_id, data=None, error=str(exc))

async def fetch_many_users(user_ids: list[str]) -> list[FetchResult]:
    async with aiohttp.ClientSession(base_url="https://api.example.com") as session:
        tasks = [fetch_user_async(session, uid) for uid in user_ids]
        return await asyncio.gather(*tasks)  # concurrent, not sequential
```

> **VIP EXPERT NOTE:** Async Python enables high concurrency with single-threaded
> performance. Key rules: 1) Use `async with` for resource management, 2) Avoid
> blocking calls (use `asyncio.to_thread()` for CPU work), 3) Batch with
> `asyncio.gather()`, 4) Set timeouts on ALL network operations. Never mix sync
> and async code carelessly — it causes deadlock and performance issues.

---

## 🔷 TYPE HINTS BEST PRACTICES

```python
from typing import TypeVar, Generic, Callable, Protocol
from collections.abc import Iterable

# Generic types
T = TypeVar('T')

class Repository(Generic[T]):
    def __init__(self, model_class: type[T]) -> None:
        self.model_class = model_class
    
    def find_by_id(self, id: int) -> T | None:
        ...

# Protocol for duck typing
class Drawable(Protocol):
    def draw(self) -> None: ...

def render(items: Iterable[Drawable]) -> None:
    for item in items:
        item.draw()

# Callable types
Handler = Callable[[Request], Response]

def register_handler(handler: Handler) -> None:
    ...

# Type aliases
JsonDict = dict[str, object]
UserId = int
UserMap = dict[UserId, 'User']

# Use | for union types (Python 3.10+)
def find_user(user_id: int | str) -> User | None:
    ...

# Use ... for pass in stubs/stubs
class AbstractService:
    def process(self, data: bytes) -> bytes: ...
```

> **VIP EXPERT NOTE:** Type hints are documentation that can be machine-checked.
> Use mypy or pyright in strict mode in CI. Prefer `X | Y` over `Union[X, Y]`
> (Python 3.10+). Use Protocol for structural subtyping. Use TypeVar with bounds
> for generic constraints. Type hints have zero runtime cost (removed at compile
> time by __future__ annotations).

---

## 🔷 DECORATOR PATTERNS

```python
from functools import wraps
from typing import TypeVar, Callable, ParamSpec
import time
import logging

P = ParamSpec('P')
R = TypeVar('R')

# Timing decorator
def timed(func: Callable[P, R]) -> Callable[P, R]:
    @wraps(func)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        logging.info(f"{func.__name__} took {elapsed:.3f}s")
        return result
    return wrapper

# Retry decorator
def retry(max_attempts: int = 3, delay: float = 1.0):
    def decorator(func: Callable[P, R]) -> Callable[P, R]:
        @wraps(func)
        def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
            last_error: Exception | None = None
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    last_error = e
                    if attempt < max_attempts - 1:
                        time.sleep(delay * (2 ** attempt))  # exponential backoff
            raise last_error
        return wrapper
    return decorator

# Caching decorator with TTL
from functools import lru_cache

def ttl_cache(maxsize: int = 128, ttl: float = 300.0):
    def decorator(func: Callable[P, R]) -> Callable[P, R]:
        cache: dict = {}
        
        @wraps(func)
        def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
            key = (args, tuple(sorted(kwargs.items())))
            now = time.time()
            if key in cache:
                result, expiry = cache[key]
                if now < expiry:
                    return result
            result = func(*args, **kwargs)
            cache[key] = (result, now + ttl)
            return result
        return wrapper
    return decorator

# Usage
@timed
@retry(max_attempts=3)
def fetch_data(url: str) -> dict:
    return requests.get(url).json()

@ttl_cache(maxsize=100, ttl=600)
def get_user(user_id: int) -> User:
    return db.users.find_by_id(user_id)
```

> **VIP EXPERT NOTE:** Decorators extend function behavior without modification.
> Always use `@wraps` to preserve metadata. Use `ParamSpec` (Python 3.10+) for
> type-safe decorators that preserve argument signatures. For class decorators,
> consider metaclasses or `__init_subclass__` for cleaner patterns.

---

## 🔷 DATA CLASS PATTERNS

```python
from dataclasses import dataclass, field, asdict
from datetime import datetime

@dataclass
class User:
    id: int
    name: str
    email: str
    created_at: datetime = field(default_factory=datetime.now)
    tags: list[str] = field(default_factory=list)
    
    def to_dict(self) -> dict:
        return asdict(self)
    
    @classmethod
    def from_dict(cls, data: dict) -> 'User':
        return cls(**data)

# Frozen dataclass for immutability
@dataclass(frozen=True)
class Point:
    x: float
    y: float
    
    def distance_to(self, other: 'Point') -> float:
        return ((self.x - other.x) ** 2 + (self.y - other.y) ** 2) ** 0.5

# Slotted dataclass for memory efficiency
@dataclass(slots=True)
class Config:
    debug: bool = False
    host: str = "localhost"
    port: int = 8080

# Post-init validation
@dataclass
class Product:
    name: str
    price: float
    quantity: int
    
    def __post_init__(self):
        if self.price < 0:
            raise ValueError("Price cannot be negative")
        if self.quantity < 0:
            raise ValueError("Quantity cannot be negative")
```

> **VIP EXPERT NOTE:** Dataclasses reduce boilerplate for data containers. Use
> `frozen=True` for immutable value objects. Use `slots=True` (Python 3.10+) for
> memory efficiency and faster attribute access. Use `__post_init__` for validation.
> For complex validation, prefer Pydantic models. Use `asdict()` and `astuple()`
> for serialization.

---

## 🔷 VIP EXPERT NOTE: PYTHON PROJECT STRUCTURE

```
my-project/
├── src/
│   └── my_project/
│       ├── __init__.py
│       ├── models/          # Pydantic/dataclass models
│       ├── services/        # Business logic
│       ├── repositories/    # Data access
│       ├── api/             # FastAPI routes or Flask blueprints
│       └── utils/           # Pure utility functions
├── tests/
│   ├── conftest.py      # Shared fixtures
│   ├── test_models.py
│   ├── test_services.py
│   └── test_api.py
├── pyproject.toml           # Modern Python project config (PEP 621)
├── .env.example             # Document required env vars
└── README.md

Key rules:
• Use pyproject.toml (not setup.py) for new projects
• Use src/ layout to prevent import confusion
• Co-locate tests with same naming: services/user.py → tests/test_user.py
• Pin dependencies in pyproject.toml or requirements.txt
```

> **VIP EXPERT NOTE:** The `src/` layout is now the Python packaging best practice.
> It prevents accidental imports from the current working directory and ensures
> your tests run against the installed package, not the source. Use `hatch`,
> `pdm`, or `poetry` for modern packaging. Always include `__init__.py` files
> (even though Python 3.3+ supports implicit namespaces, explicit is clearer).

---

## 🔷 VIP EXPERT NOTE: ASYNC CONTEXT MANAGERS

```python
# For async resources (DB connections, HTTP sessions, etc.)
from contextlib import asynccontextmanager
from typing import AsyncGenerator

@asynccontextmanager
async def get_db_session() -> AsyncGenerator[AsyncSession, None]:
    session = AsyncSession(engine)
    try:
        yield session
        await session.commit()
    except Exception:
        await session.rollback()
        raise
    finally:
        await session.close()

# Usage:
async def create_user(data: CreateUserInput) -> User:
    async with get_db_session() as session:
        user = User(**data.model_dump())
        session.add(user)
        # commit happens automatically on exit
        return user

# FastAPI lifespan pattern (replaces @app.on_event):
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: initialize resources
    db_pool = await create_db_pool()
    redis = await connect_redis()
    app.state.db = db_pool
    app.state.redis = redis
    
    yield  # Application runs here
    
    # Shutdown: cleanup resources
    await db_pool.close()
    await redis.close()

app = FastAPI(lifespan=lifespan)
```

> **VIP EXPERT NOTE:** Async context managers manage async resources. The
> `@asynccontextmanager` decorator works like `@contextmanager` but for async
> functions. FastAPI's lifespan is the modern pattern for startup/shutdown
> logic (replaces the deprecated @app.on_event). Always clean up resources
> in `finally` blocks to prevent leaks.

---

## 🔷 VIP EXPERT NOTE: TESTING PATTERNS FOR PYTHON

```python
# Golden rule: Use fixtures for setup, parametrize for variations, factories for data.

import pytest
from unittest.mock import AsyncMock, patch, MagicMock
from datetime import datetime

# Async test support
@pytest.mark.asyncio
async def test_create_user_sends_welcome_email():
    mock_mailer = AsyncMock()
    service = UserService(mailer=mock_mailer)
    
    user = await service.create({"name": "Alice", "email": "alice@test.com"})
    
    mock_mailer.send.assert_called_once_with(
        to="alice@test.com",
        subject="Welcome!",
    )

# Testing exceptions with context
def test_negative_amount_raises_value_error():
    with pytest.raises(ValueError, match="Amount must be positive"):
        transfer_funds(from_id="a", to_id="b", amount=-100)

# Freezing time for deterministic tests
from freezegun import freeze_time

@freeze_time("2024-01-15 12:00:00")
def test_subscription_expires_after_30_days():
    sub = create_subscription()
    assert sub.expires_at == datetime(2024, 2, 14, 12, 0, 0)

# Parameterized tests
@pytest.mark.parametrize("input,expected", [
    ("hello", 5),
    ("world", 5),
    ("", 0),
])
def test_string_length(input: str, expected: int):
    assert len(input) == expected

# Fixtures for reusable setup
@pytest.fixture
def sample_user():
    return User(id=1, name="Test User", email="test@example.com")

def test_user_has_valid_email(sample_user: User):
    assert "@" in sample_user.email

# Mocking external APIs
@patch("my_module.requests.get")
def test_fetch_data_handles_error(mock_get: MagicMock):
    mock_get.return_value.status_code = 500
    result = fetch_data()
    assert result is None
```

> **VIP EXPERT NOTE:** Use pytest as the test runner (industry standard). Use
> fixtures for dependency injection and setup. Use `unittest.mock` for mocking
> (or `pytest-mock` for convenience). Use `freezegun` for time-dependent tests.
> Aim for 80%+ coverage on business logic. Test behavior, not implementation.
> Use factories (factory_boy or custom) for test data generation.

---

## 🔷 VIP EXPERT NOTE: PERFORMANCE OPTIMIZATION PATTERNS

```python
# Use __slots__ for memory efficiency
class Point:
    __slots__ = ['x', 'y']
    
    def __init__(self, x: float, y: float):
        self.x = x
        self.y = y

# Use lru_cache for expensive computations
from functools import lru_cache

@lru_cache(maxsize=1024)
def fibonacci(n: int) -> int:
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

# Use list comprehensions over map/filter
# Good
squares = [x**2 for x in range(1000) if x % 2 == 0]

# Use generators for large data
# Good
def process_large_file(path: str) -> Iterator[str]:
    with open(path) as f:
        for line in f:
            yield line.strip().upper()

# Use multiprocessing for CPU-bound tasks
from concurrent.futures import ProcessPoolExecutor

def cpu_intensive_task(data: list[int]) -> int:
    return sum(x ** 2 for x in data)

with ProcessPoolExecutor() as executor:
    results = list(executor.map(cpu_intensive_task, chunks))
```

> **VIP EXPERT NOTE:** Profile before optimizing. Use `cProfile` or `py-spy` to
> find bottlenecks. Python is slow for CPU-bound work — use C extensions, Cython,
> or multiprocessing. For I/O-bound work, use asyncio. Avoid premature optimization,
> but know these patterns for when they're needed. Memory is often the bottleneck
> — use `__slots__` and generators to reduce footprint.

---

*This module provides Python-specific patterns and best practices for production applications.*
