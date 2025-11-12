# Лабораторна робота 5 Repository Pattern та тестування

## Мета

Навчитися застосовувати Repository Pattern для абстрагування доступу до даних, організовувати проєкт згідно з принципами Clean Architecture та впроваджувати автоматизоване тестування компонентів додатка.

## Завдання

Реструктурувати існуючий FastAPI проєкт згідно з принципами Clean Architecture, реалізувати Repository Pattern та створити набір тестів для перевірки функціональності.

## Теоретичні відомості

### Repository Pattern

Repository Pattern є архітектурним патерном, який забезпечує абстракцію для роботи з даними та відокремлює бізнес-логіку від деталей доступу до джерел даних. Цей патерн створює проміжний шар між доменним шаром додатка та шаром відображення даних, надаючи колекціє-подібний інтерфейс для доступу до доменних об'єктів.

Основна ідея Repository Pattern полягає у створенні класів-репозиторіїв, які інкапсулюють логіку отримання та збереження даних. Репозиторії надають методи для виконання типових операцій з даними, таких як створення, читання, оновлення та видалення записів, приховуючи деталі реалізації від бізнес-логіки.

Переваги використання Repository Pattern включають відокремлення бізнес-логіки від механізмів зберігання даних, спрощення тестування через можливість підміни репозиторіїв моками, централізацію логіки доступу до даних та полегшення зміни технології зберігання даних без впливу на бізнес-логіку.

### Clean Architecture

Clean Architecture є підходом до організації коду, який наголошує на незалежності бізнес-логіки від зовнішніх деталей реалізації. Ця архітектура організована у вигляді концентричних шарів, де внутрішні шари не залежать від зовнішніх.

Основні шари Clean Architecture включають Domain layer, який містить бізнес-логіку та доменні моделі, Application layer з use cases та сервісами, Infrastructure layer з репозиторіями та технічними деталями, та Presentation layer з API endpoints та контролерами.

Принципи Clean Architecture передбачають залежність напрямку всередину, де зовнішні шари залежать від внутрішніх але не навпаки, незалежність від фреймворків та інструментів, тестованість кожного шару окремо та незалежність від користувацького інтерфейсу та баз даних.

### Організація проєкту за принципами Clean Architecture

Типова структура FastAPI проєкту з Clean Architecture включає окремі пакети для кожного шару. Пакет domain містить доменні моделі та інтерфейси репозиторіїв, application включає бізнес-логіку та use cases, infrastructure містить реалізації репозиторіїв та підключення до БД, а api пакет відповідає за HTTP endpoints.

Така організація забезпечує чітке розділення відповідальностей та дозволяє легко орієнтуватися в кодовій базі. Залежності між шарами визначаються через інтерфейси, що дозволяє легко замінювати реалізації без впливу на інші частини системи.

### Тестування FastAPI додатків

Тестування є критично важливою частиною розробки надійних додатків. FastAPI надає відмінні інструменти для тестування через інтеграцію з pytest та бібліотекою httpx для тестування HTTP endpoints.

Основні типи тестів включають unit тести для перевірки окремих компонентів в ізоляції, інтеграційні тести для перевірки взаємодії компонентів та end-to-end тести для перевірки повних сценаріїв використання. Кожен тип тестів має своє призначення та рівень деталізації.

Використання Repository Pattern значно спрощує тестування, оскільки дозволяє легко створювати моки репозиторіїв для unit тестів сервісів. Це дозволяє тестувати бізнес-логіку незалежно від бази даних.

### Dependency Injection для тестування

FastAPI надає потужний механізм перевизначення залежностей для тестування через метод app.dependency_overrides. Це дозволяє замінювати реальні залежності тестовими реалізаціями без зміни коду додатка.

При тестуванні можна створювати тестову базу даних або використовувати моки репозиторіїв залежно від типу тесту. Для інтеграційних тестів зазвичай використовується окрема тестова база даних, тоді як для unit тестів достатньо моків.

## Хід роботи

### Крок 1. Підготовка середовища для реструктуризації

Створіть нову структуру директорій для Clean Architecture:

```bash
mkdir -p app/domain app/application app/infrastructure/database
mkdir -p app/infrastructure/repositories tests/unit tests/integration
touch app/domain/__init__.py app/application/__init__.py
touch app/infrastructure/__init__.py tests/__init__.py
touch tests/unit/__init__.py tests/integration/__init__.py
```

Встановіть додаткові пакети для тестування:

```bash
pip install pytest pytest-asyncio httpx
```

### Крок 2. Створення доменного шару

Перенесіть моделі у доменний шар. Створіть файл `app/domain/entities.py`:

```python
from datetime import datetime
from typing import Optional

class ProductEntity:
    def __init__(
        self,
        id: Optional[int],
        name: str,
        description: Optional[str],
        price: float,
        stock_quantity: int,
        created_at: Optional[datetime] = None,
        updated_at: Optional[datetime] = None
    ):
        self.id = id
        self.name = name
        self.description = description
        self.price = price
        self.stock_quantity = stock_quantity
        self.created_at = created_at
        self.updated_at = updated_at

    def reduce_stock(self, quantity: int) -> None:
        if quantity <= 0:
            raise ValueError("Кількість має бути більше нуля")
        if self.stock_quantity < quantity:
            raise ValueError(
                f"Недостатня кількість товару. Доступно: {self.stock_quantity}"
            )
        self.stock_quantity -= quantity
```

Створіть інтерфейс репозиторію `app/domain/repositories.py`:

```python
from abc import ABC, abstractmethod
from typing import List, Optional
from app.domain.entities import ProductEntity

class IProductRepository(ABC):
    @abstractmethod
    async def create(self, product: ProductEntity) -> ProductEntity:
        pass

    @abstractmethod
    async def get_by_id(self, product_id: int) -> Optional[ProductEntity]:
        pass

    @abstractmethod
    async def get_by_name(self, name: str) -> Optional[ProductEntity]:
        pass

    @abstractmethod
    async def get_all(self, skip: int = 0, limit: int = 100) -> List[ProductEntity]:
        pass

    @abstractmethod
    async def update(self, product: ProductEntity) -> ProductEntity:
        pass

    @abstractmethod
    async def delete(self, product_id: int) -> None:
        pass
```

### Крок 3. Перенесення моделей бази даних в infrastructure

Створіть файл `app/infrastructure/database/models.py`:

```python
from sqlalchemy import Column, Integer, String, Float, DateTime
from sqlalchemy.sql import func
from app.database.session import Base

class ProductModel(Base):
    __tablename__ = "products"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String, nullable=False, unique=True)
    description = Column(String)
    price = Column(Float, nullable=False)
    stock_quantity = Column(Integer, default=0)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())
```

### Крок 4. Реалізація Repository Pattern

Створіть файл `app/infrastructure/repositories/product_repository.py`:

```python
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from typing import List, Optional
from app.domain.repositories import IProductRepository
from app.domain.entities import ProductEntity
from app.infrastructure.database.models import ProductModel

class ProductRepository(IProductRepository):
    def __init__(self, session: AsyncSession):
        self.session = session

    async def create(self, product: ProductEntity) -> ProductEntity:
        db_product = ProductModel(
            name=product.name,
            description=product.description,
            price=product.price,
            stock_quantity=product.stock_quantity
        )
        self.session.add(db_product)
        await self.session.commit()
        await self.session.refresh(db_product)
        return self._to_entity(db_product)

    async def get_by_id(self, product_id: int) -> Optional[ProductEntity]:
        result = await self.session.execute(
            select(ProductModel).where(ProductModel.id == product_id)
        )
        db_product = result.scalar_one_or_none()
        return self._to_entity(db_product) if db_product else None

    async def get_by_name(self, name: str) -> Optional[ProductEntity]:
        result = await self.session.execute(
            select(ProductModel).where(ProductModel.name == name)
        )
        db_product = result.scalar_one_or_none()
        return self._to_entity(db_product) if db_product else None

    async def get_all(self, skip: int = 0, limit: int = 100) -> List[ProductEntity]:
        result = await self.session.execute(
            select(ProductModel).offset(skip).limit(limit)
        )
        db_products = result.scalars().all()
        return [self._to_entity(p) for p in db_products]

    async def update(self, product: ProductEntity) -> ProductEntity:
        result = await self.session.execute(
            select(ProductModel).where(ProductModel.id == product.id)
        )
        db_product = result.scalar_one_or_none()
        if not db_product:
            raise ValueError(f"Товар з ID {product.id} не знайдено")

        db_product.name = product.name
        db_product.description = product.description
        db_product.price = product.price
        db_product.stock_quantity = product.stock_quantity

        await self.session.commit()
        await self.session.refresh(db_product)
        return self._to_entity(db_product)

    async def delete(self, product_id: int) -> None:
        result = await self.session.execute(
            select(ProductModel).where(ProductModel.id == product_id)
        )
        db_product = result.scalar_one_or_none()
        if db_product:
            await self.session.delete(db_product)
            await self.session.commit()

    @staticmethod
    def _to_entity(model: ProductModel) -> ProductEntity:
        return ProductEntity(
            id=model.id,
            name=model.name,
            description=model.description,
            price=model.price,
            stock_quantity=model.stock_quantity,
            created_at=model.created_at,
            updated_at=model.updated_at
        )
```

### Крок 5. Оновлення application layer

Оновіть сервіс для використання репозиторію. Створіть файл `app/application/product_service.py`:

```python
from typing import List
from fastapi import HTTPException, status
from app.domain.repositories import IProductRepository
from app.domain.entities import ProductEntity
from app.schemas.product import ProductCreate, ProductUpdate

class ProductService:
    def __init__(self, repository: IProductRepository):
        self.repository = repository

    async def create_product(self, product_data: ProductCreate) -> ProductEntity:
        existing_product = await self.repository.get_by_name(product_data.name)
        if existing_product:
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail=f"Товар з назвою '{product_data.name}' вже існує"
            )

        product = ProductEntity(
            id=None,
            name=product_data.name,
            description=product_data.description,
            price=product_data.price,
            stock_quantity=product_data.stock_quantity
        )
        return await self.repository.create(product)

    async def get_product(self, product_id: int) -> ProductEntity:
        product = await self.repository.get_by_id(product_id)
        if not product:
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail=f"Товар з ID {product_id} не знайдено"
            )
        return product

    async def get_all_products(self, skip: int = 0, limit: int = 100) -> List[ProductEntity]:
        return await self.repository.get_all(skip, limit)

    async def update_product(
        self,
        product_id: int,
        product_data: ProductUpdate
    ) -> ProductEntity:
        product = await self.get_product(product_id)

        if product_data.name is not None:
            existing_product = await self.repository.get_by_name(product_data.name)
            if existing_product and existing_product.id != product_id:
                raise HTTPException(
                    status_code=status.HTTP_400_BAD_REQUEST,
                    detail=f"Товар з назвою '{product_data.name}' вже існує"
                )
            product.name = product_data.name

        if product_data.description is not None:
            product.description = product_data.description
        if product_data.price is not None:
            product.price = product_data.price
        if product_data.stock_quantity is not None:
            product.stock_quantity = product_data.stock_quantity

        return await self.repository.update(product)

    async def delete_product(self, product_id: int) -> None:
        await self.get_product(product_id)
        await self.repository.delete(product_id)

    async def reduce_stock(self, product_id: int, quantity: int) -> ProductEntity:
        product = await self.get_product(product_id)

        try:
            product.reduce_stock(quantity)
        except ValueError as e:
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail=str(e)
            )

        return await self.repository.update(product)
```

### Крок 6. Оновлення dependency injection

Оновіть файл `app/core/dependencies.py`:

```python
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession
from app.database.session import get_session
from app.infrastructure.repositories.product_repository import ProductRepository
from app.application.product_service import ProductService

async def get_product_repository(
    session: AsyncSession = Depends(get_session)
) -> ProductRepository:
    return ProductRepository(session)

async def get_product_service(
    repository: ProductRepository = Depends(get_product_repository)
) -> ProductService:
    return ProductService(repository)
```

### Крок 7. Створення unit тестів для сервісу

Створіть файл `tests/unit/test_product_service.py`:

```python
import pytest
from unittest.mock import AsyncMock
from fastapi import HTTPException
from app.application.product_service import ProductService
from app.domain.entities import ProductEntity
from app.schemas.product import ProductCreate, ProductUpdate

@pytest.fixture
def mock_repository():
    return AsyncMock()

@pytest.fixture
def product_service(mock_repository):
    return ProductService(mock_repository)

@pytest.fixture
def sample_product():
    return ProductEntity(
        id=1,
        name="Test Product",
        description="Test Description",
        price=100.0,
        stock_quantity=10
    )

@pytest.mark.asyncio
async def test_create_product_success(product_service, mock_repository):
    mock_repository.get_by_name.return_value = None
    mock_repository.create.return_value = ProductEntity(
        id=1,
        name="New Product",
        description="Description",
        price=50.0,
        stock_quantity=5
    )

    product_data = ProductCreate(
        name="New Product",
        description="Description",
        price=50.0,
        stock_quantity=5
    )

    result = await product_service.create_product(product_data)

    assert result.name == "New Product"
    assert result.price == 50.0
    mock_repository.create.assert_called_once()

@pytest.mark.asyncio
async def test_create_product_duplicate_name(product_service, mock_repository, sample_product):
    mock_repository.get_by_name.return_value = sample_product

    product_data = ProductCreate(
        name="Test Product",
        description="Description",
        price=50.0,
        stock_quantity=5
    )

    with pytest.raises(HTTPException) as exc_info:
        await product_service.create_product(product_data)

    assert exc_info.value.status_code == 400
    assert "вже існує" in exc_info.value.detail

@pytest.mark.asyncio
async def test_get_product_success(product_service, mock_repository, sample_product):
    mock_repository.get_by_id.return_value = sample_product

    result = await product_service.get_product(1)

    assert result.id == 1
    assert result.name == "Test Product"
    mock_repository.get_by_id.assert_called_once_with(1)

@pytest.mark.asyncio
async def test_get_product_not_found(product_service, mock_repository):
    mock_repository.get_by_id.return_value = None

    with pytest.raises(HTTPException) as exc_info:
        await product_service.get_product(999)

    assert exc_info.value.status_code == 404

@pytest.mark.asyncio
async def test_reduce_stock_success(product_service, mock_repository, sample_product):
    mock_repository.get_by_id.return_value = sample_product
    mock_repository.update.return_value = sample_product

    result = await product_service.reduce_stock(1, 5)

    assert result.stock_quantity == 5
    mock_repository.update.assert_called_once()

@pytest.mark.asyncio
async def test_reduce_stock_insufficient(product_service, mock_repository, sample_product):
    mock_repository.get_by_id.return_value = sample_product

    with pytest.raises(HTTPException) as exc_info:
        await product_service.reduce_stock(1, 20)

    assert exc_info.value.status_code == 400
    assert "Недостатня кількість" in exc_info.value.detail
```

### Крок 8. Створення інтеграційних тестів

Створіть файл `tests/integration/test_products_api.py`:

```python
import pytest
from httpx import AsyncClient
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.ext.asyncio import async_sessionmaker
from app.main import app
from app.infrastructure.database.models import ProductModel
from app.database.session import Base, get_session

TEST_DATABASE_URL = "sqlite+aiosqlite:///./test.db"

@pytest.fixture
async def test_engine():
    engine = create_async_engine(TEST_DATABASE_URL, echo=True)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await engine.dispose()

@pytest.fixture
async def test_session_maker(test_engine):
    return async_sessionmaker(test_engine, class_=AsyncSession, expire_on_commit=False)

@pytest.fixture
async def override_get_session(test_session_maker):
    async def _get_session():
        async with test_session_maker() as session:
            yield session

    app.dependency_overrides[get_session] = _get_session
    yield
    app.dependency_overrides.clear()

@pytest.fixture
async def client(override_get_session):
    async with AsyncClient(app=app, base_url="http://test") as ac:
        yield ac

@pytest.mark.asyncio
async def test_create_product(client):
    response = await client.post(
        "/api/products/",
        json={
            "name": "Test Product",
            "description": "Test Description",
            "price": 99.99,
            "stock_quantity": 10
        }
    )
    assert response.status_code == 201
    data = response.json()
    assert data["name"] == "Test Product"
    assert data["price"] == 99.99

@pytest.mark.asyncio
async def test_get_product(client):
    create_response = await client.post(
        "/api/products/",
        json={
            "name": "Product to Get",
            "description": "Description",
            "price": 50.0,
            "stock_quantity": 5
        }
    )
    product_id = create_response.json()["id"]

    response = await client.get(f"/api/products/{product_id}")
    assert response.status_code == 200
    data = response.json()
    assert data["id"] == product_id
    assert data["name"] == "Product to Get"

@pytest.mark.asyncio
async def test_update_product(client):
    create_response = await client.post(
        "/api/products/",
        json={
            "name": "Original Name",
            "description": "Description",
            "price": 100.0,
            "stock_quantity": 10
        }
    )
    product_id = create_response.json()["id"]

    response = await client.put(
        f"/api/products/{product_id}",
        json={"name": "Updated Name", "price": 150.0}
    )
    assert response.status_code == 200
    data = response.json()
    assert data["name"] == "Updated Name"
    assert data["price"] == 150.0

@pytest.mark.asyncio
async def test_delete_product(client):
    create_response = await client.post(
        "/api/products/",
        json={
            "name": "Product to Delete",
            "description": "Description",
            "price": 50.0,
            "stock_quantity": 5
        }
    )
    product_id = create_response.json()["id"]

    response = await client.delete(f"/api/products/{product_id}")
    assert response.status_code == 204

    get_response = await client.get(f"/api/products/{product_id}")
    assert get_response.status_code == 404

@pytest.mark.asyncio
async def test_reduce_stock(client):
    create_response = await client.post(
        "/api/products/",
        json={
            "name": "Product with Stock",
            "description": "Description",
            "price": 50.0,
            "stock_quantity": 20
        }
    )
    product_id = create_response.json()["id"]

    response = await client.post(
        f"/api/products/{product_id}/reduce-stock?quantity=5"
    )
    assert response.status_code == 200
    data = response.json()
    assert data["stock_quantity"] == 15
```

### Крок 9. Налаштування конфігурації pytest

Створіть файл `pytest.ini` у кореневій директорії:

```ini
[pytest]
asyncio_mode = auto
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
```

### Крок 10. Запуск тестів

Запустіть тести командою:

```bash
pytest -v
```

Для запуску тільки unit тестів:

```bash
pytest tests/unit -v
```

Для запуску з покриттям коду:

```bash
pip install pytest-cov
pytest --cov=app --cov-report=html
```

### Крок 11. Оновлення структури проєкту

Переконайтеся що структура вашого проєкту виглядає наступним чином:

```
app/
├── api/
│   └── products.py
├── application/
│   └── product_service.py
├── core/
│   ├── config.py
│   └── dependencies.py
├── database/
│   └── session.py
├── domain/
│   ├── entities.py
│   └── repositories.py
├── infrastructure/
│   ├── database/
│   │   └── models.py
│   └── repositories/
│       └── product_repository.py
├── schemas/
│   └── product.py
└── main.py
tests/
├── unit/
│   └── test_product_service.py
└── integration/
    └── test_products_api.py
```

### Крок 12. Підготовка документації

Створіть детальний `README.md` з описом архітектури проєкту, поясненням кожного шару, інструкціями з запуску тестів та прикладами використання API. Додайте діаграму архітектури у форматі Mermaid.

На Moodle в якості відповіді вставте посилання на репозиторій.

## Критерії оцінювання

Максимальна кількість балів за лабораторну роботу становить 7 балів. Оцінювання проводиться за наступними критеріями:

**1-2 бали** - проєкт реструктуровано з виділенням основних шарів Clean Architecture. Створено базову структуру директорій з domain, application та infrastructure пакетами. Реалізовано Repository Pattern з абстрактним інтерфейсом та конкретною реалізацією.

**3-4 бали** - створено доменні сутності та перенесено моделі бази даних у infrastructure шар. Оновлено service layer для використання репозиторіїв через інтерфейси. Налаштовано dependency injection для репозиторіїв та сервісів. Проєкт функціонує з новою архітектурою.

**5-6 балів** - створено unit тести для сервісного шару з використанням моків репозиторіїв. Реалізовано базові інтеграційні тести для API endpoints з тестовою базою даних. Тести успішно виконуються та покривають основний функціонал. Код демонструє розуміння принципів тестування.

**7 балів** - повний набір unit та інтеграційних тестів з високим покриттям коду. Створено детальну документацію з поясненням архітектурних рішень, діаграмами та прикладами. Код відповідає принципам SOLID та Clean Architecture, демонструє глибоке розуміння патернів проектування. Реалізовано розширене тестування з перевіркою граничних випадків та обробки помилок. Документація включає інструкції з розгортання та підтримки проєкту.

## Контрольні запитання

1. Які переваги надає використання Repository Pattern та як він сприяє відокремленню бізнес-логіки від деталей зберігання даних?
2. Що таке Clean Architecture та які основні принципи організації коду вона передбачає?
3. Як Repository Pattern спрощує тестування додатка та які види тестів стають можливими завдяки цьому патерну?
4. У чому полягає різниця між unit тестами та інтеграційними тестами, і коли доцільно використовувати кожен тип?
5. Яким чином dependency injection у FastAPI допомагає при написанні тестів та підміні залежностей?
6. Які шари включає Clean Architecture у контексті FastAPI додатків та які відповідальності має кожен шар?
7. Чому важливо визначати інтерфейси репозиторіїв у domain layer замість безпосереднього використання конкретних реалізацій?
