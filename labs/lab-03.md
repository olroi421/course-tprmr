# Лабораторна робота 3. Інтеграція бази даних

## Мета

Набути практичних навичок роботи з реляційними базами даних у FastAPI додатках, навчитися створювати та застосовувати міграції бази даних, реалізовувати Repository Pattern для організації доступу до даних та виконувати асинхронні CRUD операції.

## Завдання

Розробити RESTful API для управління бібліотечною системою з повноцінною інтеграцією бази даних PostgreSQL. Система повинна підтримувати управління книгами, авторами та користувачами з можливістю видачі книг читачам.

## Теоретичні відомості

### Асинхронний SQLAlchemy

SQLAlchemy версії 2.0 надає повну підтримку асинхронного програмування через asyncio. Основними компонентами асинхронної роботи є AsyncEngine для керування підключеннями та AsyncSession для виконання запитів.

Асинхронний підхід дозволяє FastAPI обробляти інші запити під час очікування відповіді від бази даних, що значно підвищує пропускну здатність застосунку. Замість блокування потоку виконання на час виконання SQL запиту, асинхронна корутина передає керування назад до event loop, який може переключитися на обробку іншого запиту.

### Alembic для міграцій

Alembic є інструментом для версіонування схеми бази даних. Кожна міграція представляє собою Python скрипт, що містить функції upgrade для застосування змін та downgrade для їх відкату. Alembic відстежує, які міграції вже застосовані до бази даних, зберігаючи цю інформацію у спеціальній таблиці alembic_version.

Автоматична генерація міграцій відбувається через порівняння поточного стану моделей SQLAlchemy з актуальною схемою бази даних. Alembic аналізує різницю та генерує код міграції, що синхронізує схему з моделями.

### Repository Pattern

Repository Pattern є архітектурним шаблоном, що інкапсулює логіку доступу до даних. Репозиторій надає методи для роботи з конкретною сутністю, приховуючи деталі реалізації запитів до бази даних. Це забезпечує відокремлення бізнес-логіки від деталей зберігання даних та спрощує тестування.

Основні переваги Repository Pattern включають централізацію логіки доступу до даних, можливість легкої заміни реалізації, спрощення тестування через mock об'єкти та покращення читабельності коду.

### Dependency Injection у FastAPI

FastAPI використовує механізм Depends для впровадження залежностей у функції-обробники. Це дозволяє автоматично створювати та передавати об'єкти, такі як сесії бази даних або репозиторії, без необхідності явного створення їх у кожному обробнику.

## Хід роботи

### Крок 1. Підготовка середовища

Створіть новий проєкт та встановіть необхідні залежності.

Створіть директорію проєкту та віртуальне середовище:

```bash
mkdir library-api
cd library-api
python -m venv venv
source venv/bin/activate  # для Linux/Mac
venv\Scripts\activate     # для Windows
```

Встановіть необхідні пакети:

```bash
pip install fastapi uvicorn sqlalchemy asyncpg alembic pydantic-settings
```

Створіть файл requirements.txt:

```
fastapi==0.104.1
uvicorn[standard]==0.24.0
sqlalchemy==2.0.23
asyncpg==0.29.0
alembic==1.12.1
pydantic-settings==2.1.0
```

### Крок 2. Налаштування PostgreSQL

Встановіть PostgreSQL локально або запустіть через Docker.

Варіант з Docker (рекомендовано):

```bash
docker run --name library-postgres -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=library -p 5432:5432 -d postgres:15
```

Перевірте підключення до бази даних:

```bash
docker exec -it library-postgres psql -U postgres -d library
```

### Крок 3. Структура проєкту

Створіть наступну структуру директорій та файлів:

```
library-api/
├── alembic/
│   ├── versions/
│   └── env.py
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── config.py
│   ├── database.py
│   ├── models/
│   │   ├── __init__.py
│   │   ├── author.py
│   │   ├── book.py
│   │   └── user.py
│   ├── schemas/
│   │   ├── __init__.py
│   │   ├── author.py
│   │   ├── book.py
│   │   └── user.py
│   ├── repositories/
│   │   ├── __init__.py
│   │   ├── base.py
│   │   ├── author.py
│   │   ├── book.py
│   │   └── user.py
│   └── routers/
│       ├── __init__.py
│       ├── authors.py
│       ├── books.py
│       └── users.py
├── alembic.ini
├── .env
└── requirements.txt
```

### Крок 4. Конфігурація застосунку

Створіть файл .env з налаштуваннями:

```
DATABASE_URL=postgresql+asyncpg://postgres:postgres@localhost:5432/library
```

Створіть app/config.py для керування налаштуваннями:

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str

    class Config:
        env_file = ".env"

settings = Settings()
```

### Крок 5. Налаштування підключення до бази даних

Створіть app/database.py для конфігурації SQLAlchemy:

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from sqlalchemy.orm import declarative_base

from app.config import settings

engine = create_async_engine(
    settings.database_url,
    echo=True,
    future=True,
    pool_size=10,
    max_overflow=20
)

AsyncSessionLocal = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,
    autocommit=False,
    autoflush=False
)

Base = declarative_base()

async def get_db() -> AsyncSession:
    async with AsyncSessionLocal() as session:
        try:
            yield session
        finally:
            await session.close()
```

### Крок 6. Створення моделей

Створіть app/models/author.py:

```python
from sqlalchemy import Column, Integer, String, Date
from sqlalchemy.orm import relationship
from app.database import Base

class Author(Base):
    __tablename__ = "authors"

    id = Column(Integer, primary_key=True, index=True)
    first_name = Column(String(100), nullable=False)
    last_name = Column(String(100), nullable=False)
    birth_date = Column(Date, nullable=True)
    biography = Column(String(1000), nullable=True)

    books = relationship("Book", back_populates="author")
```

Створіть app/models/book.py:

```python
from sqlalchemy import Column, Integer, String, ForeignKey, Boolean
from sqlalchemy.orm import relationship
from app.database import Base

class Book(Base):
    __tablename__ = "books"

    id = Column(Integer, primary_key=True, index=True)
    title = Column(String(200), nullable=False, index=True)
    isbn = Column(String(13), unique=True, nullable=False, index=True)
    publication_year = Column(Integer, nullable=False)
    author_id = Column(Integer, ForeignKey("authors.id"), nullable=False)
    is_available = Column(Boolean, default=True, nullable=False)

    author = relationship("Author", back_populates="books")
```

Створіть app/models/user.py:

```python
from sqlalchemy import Column, Integer, String, Boolean
from app.database import Base

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    email = Column(String(100), unique=True, nullable=False, index=True)
    first_name = Column(String(100), nullable=False)
    last_name = Column(String(100), nullable=False)
    is_active = Column(Boolean, default=True, nullable=False)
```

Створіть app/models/__init__.py:

```python
from app.models.author import Author
from app.models.book import Book
from app.models.user import User
```

### Крок 7. Створення Pydantic схем

Створіть app/schemas/author.py:

```python
from datetime import date
from typing import Optional
from pydantic import BaseModel, Field

class AuthorBase(BaseModel):
    first_name: str = Field(..., min_length=1, max_length=100)
    last_name: str = Field(..., min_length=1, max_length=100)
    birth_date: Optional[date] = None
    biography: Optional[str] = Field(None, max_length=1000)

class AuthorCreate(AuthorBase):
    pass

class AuthorUpdate(AuthorBase):
    first_name: Optional[str] = Field(None, min_length=1, max_length=100)
    last_name: Optional[str] = Field(None, min_length=1, max_length=100)

class AuthorResponse(AuthorBase):
    id: int

    class Config:
        from_attributes = True
```

Створіть app/schemas/book.py:

```python
from typing import Optional
from pydantic import BaseModel, Field

class BookBase(BaseModel):
    title: str = Field(..., min_length=1, max_length=200)
    isbn: str = Field(..., min_length=13, max_length=13)
    publication_year: int = Field(..., ge=1000, le=9999)
    author_id: int

class BookCreate(BookBase):
    pass

class BookUpdate(BaseModel):
    title: Optional[str] = Field(None, min_length=1, max_length=200)
    isbn: Optional[str] = Field(None, min_length=13, max_length=13)
    publication_year: Optional[int] = Field(None, ge=1000, le=9999)
    author_id: Optional[int] = None
    is_available: Optional[bool] = None

class BookResponse(BookBase):
    id: int
    is_available: bool

    class Config:
        from_attributes = True
```

Створіть app/schemas/user.py:

```python
from typing import Optional
from pydantic import BaseModel, EmailStr, Field

class UserBase(BaseModel):
    email: EmailStr
    first_name: str = Field(..., min_length=1, max_length=100)
    last_name: str = Field(..., min_length=1, max_length=100)

class UserCreate(UserBase):
    pass

class UserUpdate(BaseModel):
    email: Optional[EmailStr] = None
    first_name: Optional[str] = Field(None, min_length=1, max_length=100)
    last_name: Optional[str] = Field(None, min_length=1, max_length=100)
    is_active: Optional[bool] = None

class UserResponse(UserBase):
    id: int
    is_active: bool

    class Config:
        from_attributes = True
```

### Крок 8. Ініціалізація Alembic

Ініціалізуйте Alembic у проєкті:

```bash
alembic init alembic
```

Відредагуйте alembic.ini, змінивши рядок sqlalchemy.url на:

```ini
sqlalchemy.url = postgresql+asyncpg://postgres:postgres@localhost:5432/library
```

Відредагуйте alembic/env.py для підтримки асинхронних міграцій:

```python
from logging.config import fileConfig
from sqlalchemy import pool
from sqlalchemy.engine import Connection
from sqlalchemy.ext.asyncio import async_engine_from_config
from alembic import context

from app.database import Base
from app.models import Author, Book, User
from app.config import settings

config = context.config
config.set_main_option("sqlalchemy.url", settings.database_url)

if config.config_file_name is not None:
    fileConfig(config.config_file_name)

target_metadata = Base.metadata

def run_migrations_offline() -> None:
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )

    with context.begin_transaction():
        context.run_migrations()

def do_run_migrations(connection: Connection) -> None:
    context.configure(connection=connection, target_metadata=target_metadata)

    with context.begin_transaction():
        context.run_migrations()

async def run_async_migrations() -> None:
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)

    await connectable.dispose()

def run_migrations_online() -> None:
    import asyncio
    asyncio.run(run_async_migrations())

if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

Створіть першу міграцію:

```bash
alembic revision --autogenerate -m "Create initial tables"
```

Застосуйте міграцію:

```bash
alembic upgrade head
```

### Крок 9. Реалізація Repository Pattern

Створіть app/repositories/base.py з базовим репозиторієм:

```python
from typing import Generic, TypeVar, Type, Optional, List
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from app.database import Base

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

    async def create(self, obj: ModelType) -> ModelType:
        self.db.add(obj)
        await self.db.commit()
        await self.db.refresh(obj)
        return obj

    async def update(self, obj: ModelType) -> ModelType:
        await self.db.commit()
        await self.db.refresh(obj)
        return obj

    async def delete(self, id: int) -> bool:
        obj = await self.get_by_id(id)
        if obj:
            await self.db.delete(obj)
            await self.db.commit()
            return True
        return False
```

Створіть app/repositories/author.py:

```python
from typing import Optional
from sqlalchemy import select
from app.models.author import Author
from app.repositories.base import BaseRepository

class AuthorRepository(BaseRepository[Author]):
    async def get_by_name(self, first_name: str, last_name: str) -> Optional[Author]:
        result = await self.db.execute(
            select(Author).where(
                Author.first_name == first_name,
                Author.last_name == last_name
            )
        )
        return result.scalar_one_or_none()
```

Створіть app/repositories/book.py:

```python
from typing import Optional, List
from sqlalchemy import select
from sqlalchemy.orm import selectinload
from app.models.book import Book
from app.repositories.base import BaseRepository

class BookRepository(BaseRepository[Book]):
    async def get_by_isbn(self, isbn: str) -> Optional[Book]:
        result = await self.db.execute(
            select(Book).where(Book.isbn == isbn)
        )
        return result.scalar_one_or_none()

    async def get_available_books(self, skip: int = 0, limit: int = 100) -> List[Book]:
        result = await self.db.execute(
            select(Book)
            .where(Book.is_available == True)
            .offset(skip)
            .limit(limit)
        )
        return list(result.scalars().all())

    async def get_books_by_author(self, author_id: int) -> List[Book]:
        result = await self.db.execute(
            select(Book)
            .where(Book.author_id == author_id)
            .options(selectinload(Book.author))
        )
        return list(result.scalars().all())
```

Створіть app/repositories/user.py:

```python
from typing import Optional
from sqlalchemy import select
from app.models.user import User
from app.repositories.base import BaseRepository

class UserRepository(BaseRepository[User]):
    async def get_by_email(self, email: str) -> Optional[User]:
        result = await self.db.execute(
            select(User).where(User.email == email)
        )
        return result.scalar_one_or_none()
```

### Крок 10. Створення роутерів

Створіть app/routers/authors.py:

```python
from typing import List
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.ext.asyncio import AsyncSession

from app.database import get_db
from app.models.author import Author
from app.repositories.author import AuthorRepository
from app.schemas.author import AuthorCreate, AuthorResponse, AuthorUpdate

router = APIRouter(prefix="/authors", tags=["authors"])

@router.post("/", response_model=AuthorResponse, status_code=status.HTTP_201_CREATED)
async def create_author(
    author_data: AuthorCreate,
    db: AsyncSession = Depends(get_db)
):
    repo = AuthorRepository(Author, db)

    existing = await repo.get_by_name(author_data.first_name, author_data.last_name)
    if existing:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Автор з таким іменем вже існує"
        )

    author = Author(**author_data.model_dump())
    return await repo.create(author)

@router.get("/", response_model=List[AuthorResponse])
async def get_authors(
    skip: int = 0,
    limit: int = 100,
    db: AsyncSession = Depends(get_db)
):
    repo = AuthorRepository(Author, db)
    return await repo.get_all(skip=skip, limit=limit)

@router.get("/{author_id}", response_model=AuthorResponse)
async def get_author(
    author_id: int,
    db: AsyncSession = Depends(get_db)
):
    repo = AuthorRepository(Author, db)
    author = await repo.get_by_id(author_id)
    if not author:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Автора не знайдено"
        )
    return author

@router.put("/{author_id}", response_model=AuthorResponse)
async def update_author(
    author_id: int,
    author_data: AuthorUpdate,
    db: AsyncSession = Depends(get_db)
):
    repo = AuthorRepository(Author, db)
    author = await repo.get_by_id(author_id)
    if not author:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Автора не знайдено"
        )

    update_data = author_data.model_dump(exclude_unset=True)
    for field, value in update_data.items():
        setattr(author, field, value)

    return await repo.update(author)

@router.delete("/{author_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_author(
    author_id: int,
    db: AsyncSession = Depends(get_db)
):
    repo = AuthorRepository(Author, db)
    deleted = await repo.delete(author_id)
    if not deleted:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Автора не знайдено"
        )
```

Створіть app/routers/books.py:

```python
from typing import List
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.ext.asyncio import AsyncSession

from app.database import get_db
from app.models.book import Book
from app.repositories.book import BookRepository
from app.schemas.book import BookCreate, BookResponse, BookUpdate

router = APIRouter(prefix="/books", tags=["books"])

@router.post("/", response_model=BookResponse, status_code=status.HTTP_201_CREATED)
async def create_book(
    book_data: BookCreate,
    db: AsyncSession = Depends(get_db)
):
    repo = BookRepository(Book, db)

    existing = await repo.get_by_isbn(book_data.isbn)
    if existing:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Книга з таким ISBN вже існує"
        )

    book = Book(**book_data.model_dump())
    return await repo.create(book)

@router.get("/", response_model=List[BookResponse])
async def get_books(
    skip: int = 0,
    limit: int = 100,
    available_only: bool = False,
    db: AsyncSession = Depends(get_db)
):
    repo = BookRepository(Book, db)
    if available_only:
        return await repo.get_available_books(skip=skip, limit=limit)
    return await repo.get_all(skip=skip, limit=limit)

@router.get("/author/{author_id}", response_model=List[BookResponse])
async def get_books_by_author(
    author_id: int,
    db: AsyncSession = Depends(get_db)
):
    repo = BookRepository(Book, db)
    return await repo.get_books_by_author(author_id)

@router.get("/{book_id}", response_model=BookResponse)
async def get_book(
    book_id: int,
    db: AsyncSession = Depends(get_db)
):
    repo = BookRepository(Book, db)
    book = await repo.get_by_id(book_id)
    if not book:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Книгу не знайдено"
        )
    return book

@router.put("/{book_id}", response_model=BookResponse)
async def update_book(
    book_id: int,
    book_data: BookUpdate,
    db: AsyncSession = Depends(get_db)
):
    repo = BookRepository(Book, db)
    book = await repo.get_by_id(book_id)
    if not book:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Книгу не знайдено"
        )

    update_data = book_data.model_dump(exclude_unset=True)
    for field, value in update_data.items():
        setattr(book, field, value)

    return await repo.update(book)

@router.delete("/{book_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_book(
    book_id: int,
    db: AsyncSession = Depends(get_db)
):
    repo = BookRepository(Book, db)
    deleted = await repo.delete(book_id)
    if not deleted:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Книгу не знайдено"
        )
```

Створіть app/routers/users.py аналогічно до попередніх роутерів.

### Крок 11. Головний файл застосунку

Створіть app/main.py:

```python
from fastapi import FastAPI
from app.routers import authors, books, users

app = FastAPI(
    title="Library API",
    description="API для управління бібліотечною системою",
    version="1.0.0"
)

app.include_router(authors.router)
app.include_router(books.router)
app.include_router(users.router)

@app.get("/")
async def root():
    return {"message": "Library API"}

@app.get("/health")
async def health_check():
    return {"status": "healthy"}
```

### Крок 12. Запуск та тестування

Запустіть сервер:

```bash
uvicorn app.main:app --reload
```

Відкрийте браузер та перейдіть за адресою http://localhost:8000/docs для доступу до автоматично згенерованої документації Swagger UI.

Протестуйте ендпоінти через Swagger UI або за допомогою curl.



### Крок 13. Створення звіту README.md

Створіть файл README.md у кореневій папці проєкту. Цей файл є вашим звітом по лабораторній роботі і має містити всю інформацію про результати виконання роботи, включаючи скріншоти.

На Moodle в якості відповіді вставте посилання на репозиторій.


## Контрольні запитання

1. У чому полягає різниця між SQLAlchemy Core та ORM? Коли доцільно використовувати кожен з підходів?
2. Що таке AsyncEngine та AsyncSession? Чому важлива асинхронність при роботі з базами даних у FastAPI?
3. Як працює Alembic? Що містять функції upgrade та downgrade у міграціях?
4. Які переваги надає Repository Pattern? Як він спрощує тестування застосунку?
5. Що таке Connection Pooling та як він впливає на продуктивність застосунку?
6. Як працює механізм Dependency Injection у FastAPI? Чому він корисний для керування сесіями бази даних?
7. Що таке транзакції та властивості ACID? Як SQLAlchemy забезпечує атомарність операцій?
