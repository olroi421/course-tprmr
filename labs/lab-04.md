# Лабораторна робота 4. Service Layer та бізнес-логіка

## Мета

Навчитися проектувати та реалізовувати service layer для винесення бізнес-логіки з роутерів, застосовувати принципи Dependency Injection та створювати масштабовані архітектурні рішення у FastAPI додатках.

## Завдання

Розробити повноцінний service layer для FastAPI додатка з бізнес-логікою, валідаціями та dependency injection.

## Теоретичні відомості

### Service Layer у вебдодатках

Service layer є важливою частиною архітектури вебдодатків, що відповідає за інкапсуляцію бізнес-логіки. Цей шар розміщується між presentation layer (роутерами) та data access layer (репозиторіями чи прямою роботою з базою даних). Основна мета service layer полягає у забезпеченні чіткого розділення відповідальностей та створенні переісного коду.

Основні принципи організації service layer включають відокремлення бізнес-логіки від технічних деталей API, забезпечення можливості повторного використання логіки та спрощення тестування. Сервіси інкапсулюють складні бізнес-операції, координують роботу з різними джерелами даних та обробляють бізнес-правила.

### Dependency Injection у FastAPI

Dependency Injection є патерном проектування, який дозволяє передавати залежності об'єктам ззовні замість їх створення всередині класу. FastAPI має вбудовану підтримку dependency injection через систему Depends, що забезпечує елегантний спосіб управління залежностями.

Система залежностей FastAPI підтримує автоматичне створення та впровадження залежностей, кешування залежностей в межах одного запиту та вкладені залежності. Залежності можуть бути функціями, класами або генераторами, що надає гнучкість у виборі способу їх реалізації.

### Організація бізнес-логіки

Бізнес-логіка у service layer організовується навколо бізнес-сутностей та операцій. Кожен сервіс зазвичай відповідає за певну доменну область та містить методи для виконання бізнес-операцій над відповідними сутностями.

Важливим аспектом організації бізнес-логіки є валідація даних на рівні бізнес-правил, яка відрізняється від валідації структури даних. Бізнес-валідація перевіряє семантичну коректність операцій, наприклад чи можна виконати певну дію з урахуванням поточного стану системи.

### Структура проєкту з service layer

Типова структура FastAPI проєкту з service layer включає окремі модулі для роутерів, сервісів, моделей даних та конфігурації. Роутери відповідають за HTTP-взаємодію, сервіси містять бізнес-логіку, моделі описують структури даних, а модуль залежностей керує створенням та впровадженням сервісів.

Така організація забезпечує високу підтримуваність коду, оскільки кожен компонент має чітко визначену відповідальність. Зміни в бізнес-логіці локалізуються в сервісах, не зачіпаючи роутери, а зміни в API не впливають на бізнес-логіку.

## Хід роботи

### Крок 1. Створення базової структури проєкту

Створіть нову директорію для проєкту та ініціалізуйте віртуальне середовище:

```bash
mkdir fastapi-service-layer
cd fastapi-service-layer
python -m venv venv
source venv/bin/activate  # для Linux/Mac
# або venv\Scripts\activate для Windows
```

Встановіть необхідні пакети:

```bash
pip install fastapi uvicorn sqlalchemy asyncpg pydantic-settings
```

Створіть структуру директорій:

```bash
mkdir -p app/api app/services app/models app/database app/schemas app/core
touch app/__init__.py app/api/__init__.py app/services/__init__.py
touch app/models/__init__.py app/database/__init__.py app/schemas/__init__.py
touch app/core/__init__.py app/main.py
```

### Крок 2. Налаштування конфігурації та підключення до бази даних

Створіть файл конфігурації `app/core/config.py`:

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str = "postgresql+asyncpg://user:password@localhost/dbname"

    class Config:
        env_file = ".env"

settings = Settings()
```

Створіть файл `app/database/session.py` для налаштування з'єднання з базою даних:

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.ext.asyncio import async_sessionmaker
from sqlalchemy.orm import declarative_base
from app.core.config import settings

engine = create_async_engine(settings.database_url, echo=True)
async_session_maker = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)
Base = declarative_base()

async def get_session() -> AsyncSession:
    async with async_session_maker() as session:
        yield session
```

### Крок 3. Створення моделей даних

Створіть файл `app/models/product.py` з моделлю товару:

```python
from sqlalchemy import Column, Integer, String, Float, DateTime
from sqlalchemy.sql import func
from app.database.session import Base

class Product(Base):
    __tablename__ = "products"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String, nullable=False)
    description = Column(String)
    price = Column(Float, nullable=False)
    stock_quantity = Column(Integer, default=0)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())
```

### Крок 4. Створення Pydantic схем

Створіть файл `app/schemas/product.py` зі схемами для валідації:

```python
from pydantic import BaseModel, Field
from datetime import datetime
from typing import Optional

class ProductBase(BaseModel):
    name: str = Field(..., min_length=1, max_length=200)
    description: Optional[str] = None
    price: float = Field(..., gt=0)
    stock_quantity: int = Field(default=0, ge=0)

class ProductCreate(ProductBase):
    pass

class ProductUpdate(BaseModel):
    name: Optional[str] = Field(None, min_length=1, max_length=200)
    description: Optional[str] = None
    price: Optional[float] = Field(None, gt=0)
    stock_quantity: Optional[int] = Field(None, ge=0)

class ProductResponse(ProductBase):
    id: int
    created_at: datetime
    updated_at: Optional[datetime]

    class Config:
        from_attributes = True
```

### Крок 5. Реалізація service layer

Створіть файл `app/services/product_service.py` з бізнес-логікою:

```python
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, update, delete
from app.models.product import Product
from app.schemas.product import ProductCreate, ProductUpdate
from typing import List, Optional
from fastapi import HTTPException, status

class ProductService:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def create_product(self, product_data: ProductCreate) -> Product:
        existing_product = await self._get_product_by_name(product_data.name)
        if existing_product:
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail=f"Товар з назвою '{product_data.name}' вже існує"
            )

        new_product = Product(**product_data.model_dump())
        self.session.add(new_product)
        await self.session.commit()
        await self.session.refresh(new_product)
        return new_product

    async def get_product(self, product_id: int) -> Product:
        result = await self.session.execute(
            select(Product).where(Product.id == product_id)
        )
        product = result.scalar_one_or_none()
        if not product:
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail=f"Товар з ID {product_id} не знайдено"
            )
        return product

    async def get_all_products(self, skip: int = 0, limit: int = 100) -> List[Product]:
        result = await self.session.execute(
            select(Product).offset(skip).limit(limit)
        )
        return list(result.scalars().all())

    async def update_product(self, product_id: int, product_data: ProductUpdate) -> Product:
        product = await self.get_product(product_id)

        update_data = product_data.model_dump(exclude_unset=True)

        if "name" in update_data:
            existing_product = await self._get_product_by_name(update_data["name"])
            if existing_product and existing_product.id != product_id:
                raise HTTPException(
                    status_code=status.HTTP_400_BAD_REQUEST,
                    detail=f"Товар з назвою '{update_data['name']}' вже існує"
                )

        for field, value in update_data.items():
            setattr(product, field, value)

        await self.session.commit()
        await self.session.refresh(product)
        return product

    async def delete_product(self, product_id: int) -> None:
        product = await self.get_product(product_id)
        await self.session.delete(product)
        await self.session.commit()

    async def reduce_stock(self, product_id: int, quantity: int) -> Product:
        if quantity <= 0:
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail="Кількість має бути більше нуля"
            )

        product = await self.get_product(product_id)

        if product.stock_quantity < quantity:
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail=f"Недостатня кількість товару на складі. Доступно: {product.stock_quantity}"
            )

        product.stock_quantity -= quantity
        await self.session.commit()
        await self.session.refresh(product)
        return product

    async def _get_product_by_name(self, name: str) -> Optional[Product]:
        result = await self.session.execute(
            select(Product).where(Product.name == name)
        )
        return result.scalar_one_or_none()
```

### Крок 6. Налаштування dependency injection

Створіть файл `app/core/dependencies.py`:

```python
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession
from app.database.session import get_session
from app.services.product_service import ProductService

async def get_product_service(
    session: AsyncSession = Depends(get_session)
) -> ProductService:
    return ProductService(session)
```

### Крок 7. Створення роутерів з використанням сервісів

Створіть файл `app/api/products.py`:

```python
from fastapi import APIRouter, Depends, status
from typing import List
from app.schemas.product import ProductCreate, ProductUpdate, ProductResponse
from app.services.product_service import ProductService
from app.core.dependencies import get_product_service

router = APIRouter(prefix="/products", tags=["products"])

@router.post("/", response_model=ProductResponse, status_code=status.HTTP_201_CREATED)
async def create_product(
    product: ProductCreate,
    service: ProductService = Depends(get_product_service)
):
    return await service.create_product(product)

@router.get("/", response_model=List[ProductResponse])
async def get_products(
    skip: int = 0,
    limit: int = 100,
    service: ProductService = Depends(get_product_service)
):
    return await service.get_all_products(skip, limit)

@router.get("/{product_id}", response_model=ProductResponse)
async def get_product(
    product_id: int,
    service: ProductService = Depends(get_product_service)
):
    return await service.get_product(product_id)

@router.put("/{product_id}", response_model=ProductResponse)
async def update_product(
    product_id: int,
    product: ProductUpdate,
    service: ProductService = Depends(get_product_service)
):
    return await service.update_product(product_id, product)

@router.delete("/{product_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_product(
    product_id: int,
    service: ProductService = Depends(get_product_service)
):
    await service.delete_product(product_id)

@router.post("/{product_id}/reduce-stock", response_model=ProductResponse)
async def reduce_product_stock(
    product_id: int,
    quantity: int,
    service: ProductService = Depends(get_product_service)
):
    return await service.reduce_stock(product_id, quantity)
```

### Крок 8. Налаштування головного файлу додатка

Створіть файл `app/main.py`:

```python
from fastapi import FastAPI
from contextlib import asynccontextmanager
from app.api import products
from app.database.session import engine, Base

@asynccontextmanager
async def lifespan(app: FastAPI):
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    await engine.dispose()

app = FastAPI(
    title="Product Management API",
    description="API для управління товарами з service layer",
    version="1.0.0",
    lifespan=lifespan
)

app.include_router(products.router, prefix="/api")

@app.get("/")
async def root():
    return {"message": "Product Management API"}
```

### Крок 9. Створення файлу змінних середовища

Створіть файл `.env` у кореневій директорії проєкту:

```
DATABASE_URL=postgresql+asyncpg://postgres:password@localhost:5432/products_db
```

### Крок 10. Запуск та тестування додатка

Запустіть додаток командою:

```bash
uvicorn app.main:app --reload
```

Перейдіть за адресою http://localhost:8000/docs для доступу до автоматичної документації та тестування endpoints.

### Крок 11. Розширення функціоналу з додатковими бізнес-правилами

Для демонстрації складнішої бізнес-логіки додайте до `ProductService` методи з комплексними валідаціями. Наприклад, створіть метод для масового оновлення цін з урахуванням бізнес-правил.

### Крок 12. Підготовка звіту

Створіть файл `README.md` у кореневій директорії проєкту з детальним описом реалізованого функціоналу, архітектурних рішень та прикладами використання API.

На Moodle в якості відповіді вставте посилання на репозиторій.


## Критерії оцінювання

Максимальна кількість балів за лабораторну роботу становить 7 балів. Оцінювання проводиться за наступними критеріями:

**1-2 бали** - створено базову структуру проєкту з окремими модулями для роутерів, сервісів та моделей. Реалізовано підключення до бази даних з використанням SQLAlchemy. Створено Pydantic схеми для валідації даних.

**3-4 бали** - реалізовано service layer з базовими CRUD операціями. Бізнес-логіка винесена з роутерів у сервіси. Налаштовано dependency injection для сервісів через систему Depends у FastAPI. Створено роутери з використанням впроваджених залежностей.

**5-6 балів** - реалізовано складні бізнес-правила у service layer, такі як перевірка унікальності, контроль залишків та комплексні валідації. Додано обробку помилок з коректними HTTP статусами та інформативними повідомленнями. Код демонструє розуміння принципів організації бізнес-логіки.

**7 балів** - повністю функціональний додаток з чіткою архітектурою та розділенням відповідальностей. Реалізовано розширений функціонал з комплексними бізнес-правилами та валідаціями. Створено детальний README файл з описом архітектури, прикладами використання API, поясненням прийнятих рішень та інструкціями з розгортання. Код відповідає принципам SOLID та best practices FastAPI, демонструє глибоке розуміння service layer pattern.

## Контрольні запитання

1. Яку роль відіграє service layer у архітектурі вебдодатків та які переваги надає його використання?
2. Як працює система dependency injection у FastAPI та які основні способи визначення залежностей ви знаєте?
3. У чому полягає різниця між валідацією даних на рівні Pydantic схем та бізнес-валідацією у service layer?
4. Чому важливо відокремлювати бізнес-логіку від роутерів та як це впливає на підтримуваність коду?
5. Які принципи проектування варто враховувати при організації service layer для забезпечення масштабованості додатка?
6. Як dependency injection спрощує тестування компонентів додатка та які переваги це надає при розробці?
7. Яким чином організація коду з service layer сприяє дотриманню принципу Single Responsibility Principle?
