# Робота з базами даних у FastAPI

---

## Чому бази даних?

**Проблеми зберігання даних у пам'яті:**

- Дані втрачаються при перезапуску сервера
- Неможливість масштабування
- Відсутність узгодженості даних
- Обмежений обсяг пам'яті

**Рішення:** реляційні бази даних надають надійне, масштабоване та узгоджене зберігання даних з потужними можливостями запитів.

---

## SQLAlchemy: два підходи

### Core vs ORM

**SQLAlchemy Core:**

- Низькорівневий API
- Програмне створення SQL запитів
- Повний контроль над SQL
- Висока продуктивність

**SQLAlchemy ORM:**

- Високорівнева абстракція
- Робота з Python об'єктами
- Автоматичне відображення таблиць
- Швидка розробка CRUD операцій

---

## SQLAlchemy Core: приклад

```python
from sqlalchemy import select, insert

# Побудова запиту програмно
stmt = select(users_table).where(users_table.c.email == "user@example.com")

# Виконання
result = await session.execute(stmt)
user = result.scalar_one_or_none()

# Вставка даних
stmt = insert(users_table).values(
    email="new@example.com",
    first_name="John"
)
```

**Переваги:** точний контроль, оптимальний SQL, відсутність overhead ORM.

---

## SQLAlchemy ORM: приклад

```python
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    email = Column(String, unique=True)
    first_name = Column(String)

# Робота з об'єктами
user = await session.get(User, user_id)
user.first_name = "Jane"
await session.commit()
```

**Переваги:** природна робота з об'єктами, автоматичне відстеження змін.

---

## Коли використовувати Core vs ORM?

```mermaid
graph TD
    A[Потреба в роботі з БД] --> B{Тип операції?}
    B -->|Стандартні CRUD| C[ORM]
    B -->|Складні аналітичні запити| D[Core]
    B -->|Масові операції| D
    B -->|Високі вимоги до продуктивності| D

    C --> E[Швидка розробка]
    D --> F[Максимальний контроль]

    style C fill:#90EE90
    style D fill:#87CEEB
```

---

## Async SQLAlchemy

**Чому асинхронність важлива?**

- FastAPI є асинхронним фреймворком
- Блокуючі операції знижують продуктивність
- Очікування відповіді БД блокує потік

**Рішення:** AsyncEngine + AsyncSession

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession

engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/db",
    echo=True
)
```

---

## AsyncSession у FastAPI

```python
from sqlalchemy.ext.asyncio import async_sessionmaker

AsyncSessionLocal = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False
)

async def get_db():
    async with AsyncSessionLocal() as session:
        try:
            yield session
        finally:
            await session.close()
```

**Dependency Injection автоматично керує життєвим циклом сесії.**

---

## Async vs Sync: порівняння

```mermaid
sequenceDiagram
    participant Client
    participant FastAPI
    participant Database

    Note over FastAPI,Database: Синхронний підхід
    Client->>FastAPI: HTTP запит
    FastAPI->>Database: SQL запит (блокування)
    Database-->>FastAPI: Результат
    FastAPI-->>Client: HTTP відповідь

    Note over FastAPI,Database: Асинхронний підхід
    Client->>FastAPI: HTTP запит 1
    Client->>FastAPI: HTTP запит 2
    FastAPI->>Database: SQL запит 1 (без блокування)
    FastAPI->>Database: SQL запит 2 (паралельно)
    Database-->>FastAPI: Результат 1
    Database-->>FastAPI: Результат 2
    FastAPI-->>Client: HTTP відповідь 1
    FastAPI-->>Client: HTTP відповідь 2
```

---

## Alembic: міграції БД

**Що таке міграція?**

Версіонований набір змін у структурі бази даних, що дозволяє еволюціонувати схему разом з розвитком застосунку.

**Основні команди:**

```bash
alembic init alembic                          # Ініціалізація
alembic revision --autogenerate -m "message"  # Створення міграції
alembic upgrade head                          # Застосування міграцій
alembic downgrade -1                          # Відкат останньої міграції
```

---

## Структура міграції

```python
def upgrade() -> None:
    # Застосування змін
    op.create_table(
        'users',
        sa.Column('id', sa.Integer(), primary_key=True),
        sa.Column('email', sa.String(100), unique=True),
        sa.Column('first_name', sa.String(100))
    )
    op.create_index('ix_users_email', 'users', ['email'])

def downgrade() -> None:
    # Відкат змін
    op.drop_index('ix_users_email', 'users')
    op.drop_table('users')
```

**Кожна міграція має унікальний ID та зберігається у системі контролю версій.**

---

## Автогенерація міграцій

```mermaid
graph LR
    A[SQLAlchemy Models] --> B[Alembic]
    C[Поточна схема БД] --> B
    B --> D{Порівняння}
    D --> E[Генерація міграції]
    E --> F[Файл міграції]

    style E fill:#FFD700
```

**Alembic автоматично виявляє:**

- Нові таблиці та колонки
- Видалені таблиці та колонки
- Зміни типів даних
- Нові індекси та constraints

---

## Repository Pattern

**Архітектурний шаблон для інкапсуляції логіки доступу до даних.**

```mermaid
graph TD
    A[Controller/Router] --> B[Repository]
    B --> C[Database]

    style B fill:#87CEEB

    D[Переваги] --> E[Відокремлення логіки]
    D --> F[Легке тестування]
    D --> G[Централізація запитів]
    D --> H[Повторне використання]
```

---

## Базовий Repository

```python
from typing import Generic, TypeVar, Type, Optional, List

ModelType = TypeVar("ModelType", bound=Base)

class BaseRepository(Generic[ModelType]):
    def __init__(self, model: Type[ModelType], db: AsyncSession):
        self.model = model
        self.db = db

    async def get_by_id(self, id: int) -> Optional[ModelType]:
        result = await self.db.execute(
            select(self.model).where(self.model.id == id)
        )
        return result.scalar_one_or_none()

    async def get_all(self, skip: int = 0, limit: int = 100) -> List[ModelType]:
        result = await self.db.execute(
            select(self.model).offset(skip).limit(limit)
        )
        return list(result.scalars().all())
```

---

## Спеціалізований Repository

```python
class UserRepository(BaseRepository[User]):
    async def get_by_email(self, email: str) -> Optional[User]:
        result = await self.db.execute(
            select(User).where(User.email == email)
        )
        return result.scalar_one_or_none()

    async def get_active_users(self) -> List[User]:
        result = await self.db.execute(
            select(User).where(User.is_active == True)
        )
        return list(result.scalars().all())
```

**Репозиторій інкапсулює всю логіку роботи з конкретною сутністю.**

---

## Repository у FastAPI

```python
@router.get("/users/{user_id}")
async def get_user(
    user_id: int,
    db: AsyncSession = Depends(get_db)
):
    repo = UserRepository(User, db)
    user = await repo.get_by_id(user_id)

    if not user:
        raise HTTPException(status_code=404, detail="Користувача не знайдено")

    return user
```

**Чистий, читабельний код без SQL деталей.**

---

## Connection Pooling

**Пул з'єднань підтримує готові підключення до БД.**

```mermaid
graph TD
    A[FastAPI Requests] --> B[Connection Pool]
    B --> C[Connection 1]
    B --> D[Connection 2]
    B --> E[Connection 3]
    B --> F[Connection N]

    C --> G[PostgreSQL]
    D --> G
    E --> G
    F --> G

    style B fill:#90EE90
```

**Уникнення overhead на створення нових з'єднань для кожного запиту.**

---

## Налаштування пулу

```python
engine = create_async_engine(
    database_url,
    pool_size=10,           # Базова кількість з'єднань
    max_overflow=20,        # Додаткові з'єднання при навантаженні
    pool_timeout=30,        # Таймаут очікування з'єднання
    pool_recycle=3600,      # Переcтворення з'єднань кожну годину
    pool_pre_ping=True      # Перевірка з'єднань перед використанням
)
```

**Параметри залежать від навантаження застосунку та можливостей БД.**

---

## Моніторинг пулу

```python
# Отримання інформації про пул
pool = engine.pool

print(f"Розмір пулу: {pool.size()}")
print(f"Активні з'єднання: {pool.checkedin()}")
print(f"Використовувані: {pool.checkedout()}")
print(f"Overflow з'єднання: {pool.overflow()}")
```

**Важливо відстежувати метрики пулу у продакшені для виявлення проблем.**

---

## Оптимізація запитів

### N+1 Problem

```python
# Погано: N+1 запитів
books = await session.execute(select(Book))
for book in books.scalars():
    print(book.author.name)  # Додатковий запит для кожної книги!

# Добре: 1 запит з JOIN
books = await session.execute(
    select(Book).options(selectinload(Book.author))
)
for book in books.scalars():
    print(book.author.name)  # Дані вже завантажені
```

---

## Eager Loading стратегії

```python
from sqlalchemy.orm import selectinload, joinedload, subqueryload

# selectinload: окремий SELECT IN запит
stmt = select(Book).options(selectinload(Book.author))

# joinedload: LEFT OUTER JOIN
stmt = select(Book).options(joinedload(Book.author))

# subqueryload: додатковий підзапит
stmt = select(Book).options(subqueryload(Book.reviews))
```

**Вибір стратегії залежить від структури даних та кількості зв'язків.**

---

## Індекси для продуктивності

```python
class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True)
    email = Column(String, unique=True, index=True)  # Індекс для швидкого пошуку
    first_name = Column(String, index=True)
    last_name = Column(String, index=True)
    created_at = Column(DateTime, index=True)

    # Композитний індекс
    __table_args__ = (
        Index('ix_user_name', 'first_name', 'last_name'),
    )
```

**Індекси прискорюють SELECT, але уповільнюють INSERT/UPDATE.**

---

## Пагінація результатів

```python
async def get_books_paginated(
    page: int = 1,
    page_size: int = 20,
    db: AsyncSession = Depends(get_db)
):
    skip = (page - 1) * page_size

    # Отримання загальної кількості
    count_result = await db.execute(select(func.count(Book.id)))
    total = count_result.scalar()

    # Отримання сторінки даних
    result = await db.execute(
        select(Book).offset(skip).limit(page_size)
    )
    books = result.scalars().all()

    return {
        "items": books,
        "total": total,
        "page": page,
        "pages": (total + page_size - 1) // page_size
    }
```

---

## Транзакції та ACID

**ACID властивості гарантують надійність транзакцій:**

- **Atomicity (Атомарність):** всі операції або виконуються, або відкочуються
- **Consistency (Узгодженість):** дані залишаються валідними після транзакції
- **Isolation (Ізоляція):** паралельні транзакції не впливають одна на одну
- **Durability (Довговічність):** зафіксовані зміни зберігаються назавжди

---

## Робота з транзакціями

```python
async def transfer_money(from_id: int, to_id: int, amount: Decimal):
    async with AsyncSessionLocal() as session:
        try:
            # Всі операції в одній транзакції
            from_account = await session.get(Account, from_id)
            to_account = await session.get(Account, to_id)

            from_account.balance -= amount
            to_account.balance += amount

            await session.commit()  # Атомарне збереження
        except Exception:
            await session.rollback()  # Відкат при помилці
            raise
```

---

## Рівні ізоляції

```mermaid
graph TD
    A[Read Uncommitted] --> B[Найнижча ізоляція]
    C[Read Committed] --> D[За замовчуванням у PostgreSQL]
    E[Repeatable Read] --> F[Захист від non-repeatable reads]
    G[Serializable] --> H[Найвища ізоляція]

    B --> I[Висока продуктивність]
    H --> J[Низька продуктивність]

    style C fill:#90EE90
    style G fill:#FFB6C1
```

---

## Вибір рівня ізоляції

```python
from sqlalchemy import create_engine

# Встановлення рівня ізоляції
engine = create_async_engine(
    database_url,
    isolation_level="REPEATABLE READ"
)

# Або для конкретної сесії
async with AsyncSessionLocal() as session:
    await session.execute(
        text("SET TRANSACTION ISOLATION LEVEL SERIALIZABLE")
    )
```

**Вищий рівень = більша узгодженість, але менша продуктивність.**

---

## Керування сесіями

```mermaid
sequenceDiagram
    participant Request
    participant Dependency
    participant Session
    participant Database

    Request->>Dependency: HTTP запит
    Dependency->>Session: Створення сесії
    Session->>Database: Відкриття з'єднання

    Note over Session,Database: Виконання операцій

    Session->>Database: COMMIT
    Database-->>Session: Підтвердження
    Session-->>Dependency: Закриття сесії
    Dependency-->>Request: HTTP відповідь
```

---

## Best Practices

**Завжди закривайте сесії:**

```python
async def get_db():
    async with AsyncSessionLocal() as session:
        try:
            yield session
        finally:
            await session.close()  # Гарантоване закриття
```

**Використовуйте context managers для транзакцій:**

```python
async with session.begin():
    # Автоматичний commit або rollback
    user.balance += 100
```

---

## Типові помилки

**Забуття await:**

```python
# Погано
user = session.get(User, user_id)  # Повертає корутину, не результат!

# Добре
user = await session.get(User, user_id)
```

**Витік з'єднань:**

```python
# Погано: сесія не закривається
session = AsyncSessionLocal()
user = await session.get(User, user_id)

# Добре: автоматичне закриття
async with AsyncSessionLocal() as session:
    user = await session.get(User, user_id)
```

---

## Архітектура застосунку

```mermaid
graph TD
    A[FastAPI Router] --> B[Repository]
    B --> C[SQLAlchemy ORM]
    C --> D[AsyncSession]
    D --> E[Connection Pool]
    E --> F[PostgreSQL]

    G[Alembic] -.->|Міграції| F

    style B fill:#87CEEB
    style D fill:#90EE90
    style E fill:#FFD700
```

---

## Практичні рекомендації

**Використовуйте ORM для більшості випадків:**

- Швидка розробка CRUD
- Автоматичне відстеження змін
- Природна робота з об'єктами

**Переходьте до Core для:**

- Складних аналітичних запитів
- Масових операцій
- Критичних для продуктивності місць

**Завжди:**

- Використовуйте міграції для змін схеми
- Впроваджуйте Repository Pattern
- Налаштовуйте Connection Pool
- Створюйте індекси для частих запитів
