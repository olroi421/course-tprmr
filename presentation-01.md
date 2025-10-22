# Лекція 1. Архітектура сучасних вебдодатків

**Технології проектування та розробки мережевих ресурсів**

Викладач: Ройко О.Ю.

---

## План лекції

1. Основи веброзробки
   - Архітектура клієнт-сервер
   - HTTP протокол
   - REST архітектурний стиль

2. Проєктування API
   - Проєктування API
   - Об'єкти передачі даних
   - Впровадження залежностей

3. Архітектурні шари
   - Шарувата архітектура
   - Розділення відповідальності

---

# Частина 1. Основи веброзробки

---

## Архітектура клієнт-сервер

```mermaid
graph LR
    A[Client<br/>Браузер/Мобільний додаток] -->|HTTP запит| B[Server<br/>Вебсервер/API]
    B -->|HTTP відповідь| A
    B --> C[(Database)]
```

**Client:**
- Ініціює запити
- Відображає дані
- Управляє локальним станом

**Server:**
- Обробляє запити
- Виконує бізнес-логіку
- Забезпечує безпеку

---

## Цикл запит-відповідь

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    participant DB as Database

    C->>+S: HTTP запит
    S->>S: Валідація
    S->>+DB: Запит
    DB-->>-S: Дані
    S->>S: Бізнес-логіка
    S-->>-C: HTTP відповідь
```

Кожен запит є **незалежним** та **самодостатнім**

---

## Без збереження стану vs Зі збереженням стану

### Stateless (Без збереження стану (рекомендовано))

- Сервер не зберігає стан між запитами
- Кожен запит містить всю необхідну інформацію
- Легко масштабується горизонтально
- Вища надійність

### Stateful (Зі збереженням стану)

- Сервер зберігає сесії
- Прив'язка сесії (закріплені сесії)
- Складніше масштабування

---

## HTTP протокол

**HTTP (HyperText Transfer Protocol)** - протокол прикладного рівня для передачі даних

Версії: HTTP/1.1, HTTP/2, HTTP/3

Основа всієї веб-комунікації

---

## Структура HTTP запиту

```http
POST /api/users HTTP/1.1
Host: api.example.com
Content-Type: application/json
Authorization: Bearer eyJhbGc...
Accept: application/json

{
  "name": "Іван Петренко",
  "email": "ivan@example.com"
}
```

1. **Стартовий рядок:** метод, URI, версія
2. **Заголовки:** додаткова інформація
3. **Тіло:** дані (опціонально)

---

## Структура HTTP відповіді

```http
HTTP/1.1 201 Created
Content-Type: application/json
Location: /api/users/123

{
  "id": "123",
  "name": "Іван Петренко",
  "email": "ivan@example.com",
  "createdAt": "2025-10-21T10:30:00Z"
}
```

1. **Рядок статусу:** версія, код, опис
2. **Заголовки:** метадані відповіді
3. **Тіло:** дані

---

## HTTP методи

| Метод | Призначення | Ідемпотентний | Безпечний |
|-------|-------------|---------------|------|
| **GET** | Отримати ресурс | ✅ | ✅ |
| **POST** | Створити ресурс | ❌ | ❌ |
| **PUT** | Оновити повністю | ✅ | ❌ |
| **PATCH** | Оновити частково | ❌ | ❌ |
| **DELETE** | Видалити | ✅ | ❌ |
| **HEAD** | Отримати headers | ✅ | ✅ |
| **OPTIONS** | Отримати методи | ✅ | ✅ |

---

## Ідемпотентність

**Ідемпотентна операція** - повторне виконання дає той самий результат

```python
# Ідемпотентно
DELETE /api/users/123  # перший раз: видалено
DELETE /api/users/123  # другий раз: вже видалено (404)

PUT /api/users/123     # оновлення з тими самими даними
PUT /api/users/123     # результат ідентичний

# Не ідемпотентно
POST /api/users        # створює нового користувача
POST /api/users        # створює ще одного
```

---

## HTTP Status Codes

### 2xx Успішно
- **200 OK** - успішний запит
- **201 Created** - ресурс створено
- **204 No Content** - успіх без контенту

### 4xx Помилка клієнта
- **400 Bad Request** - невалідні дані
- **401 Unauthorized** - потрібна автентифікація
- **403 Forbidden** - доступ заборонено
- **404 Not Found** - ресурс не знайдено
- **409 Conflict** - конфлікт стану
- **422 Unprocessable Entity** - семантичні помилки
- **429 Too Many Requests** - rate limit

### 5xx Помилка сервера
- **500 Internal Помилка сервера** - помилка сервера
- **502 Bad Gateway** - помилка upstream
- **503 Service Unavailable** - сервіс недоступний

**Правило:** завжди використовуйте правильний статус-код!

---

## Важливі HTTP заголовки

```http
# Автентифікація
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...

# Тип контенту
Content-Type: application/json; charset=utf-8
Accept: application/json

# Кешування
Cache-Control: max-age=3600, private
ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"

# CORS
Access-Control-Allow-Origin: https://example.com
Access-Control-Allow-Methods: GET, POST, PUT
```

---

## REST архітектурний стиль

**REST (Representational State Transfer)**

Архітектурний стиль для розподілених гіпермедійних систем

Не протокол, а набір **принципів та обмежень**

---

## 6 Принципів REST

1. **Клієнт-сервер** - розділення відповідальностей
2. **Stateless** - без збереження стану
3. **Cacheable** - можливість кешування
4. **Uniform Interface** - єдиний інтерфейс
5. **Layered System** - шарувата система
6. **Code-On-Demand** (опціонально)

---

## Resource-Oriented Design

У REST все є **ресурсом**:
- Документ
- Колекція
- Об'єкт предметної області
- Процес

Кожен ресурс має **унікальний URI**

```
/api/users              # колекція користувачів
/api/users/123          # конкретний користувач
/api/users/123/orders   # замовлення користувача
/api/orders/456         # конкретне замовлення
```

---

## CRUD через HTTP методи

```
Створити    POST   /api/users
Прочитати   GET    /api/users/123
Оновити     PUT    /api/users/123
Видалити    DELETE /api/users/123

Список      GET    /api/users
Часткове    PATCH  /api/users/123
```

**Правило:** використовуйте іменники, а не дієслова в URI

```
✅ POST /api/orders
❌ POST /api/createOrder
```

---

## URI Design Найкращі практики

### ✅ Добрі практики

```
/api/users                    # множина
/api/products/123             # ідентифікатор
/api/users/123/orders         # вкладені ресурси
/api/products?category=tech   # фільтри
/api/user-profiles            # kebab-case
```

### ❌ Погані практики

```
/api/getUsers                 # дієслово
/api/user                     # однина
/api/users/orders/items/...   # занадто глибоко
/api/userProfiles             # camelCase
```

---

## HATEOAS

**Hypermedia As The Engine Of Application State**

Відповідь містить **посилання** на пов'язані ресурси

```json
{
  "id": "123",
  "name": "Іван Петренко",
  "_links": {
    "self": { "href": "/api/users/123" },
    "orders": { "href": "/api/users/123/orders" },
    "edit": {
      "href": "/api/users/123",
      "method": "PUT"
    }
  }
}
```

На практиці рідко реалізується повністю

---

## Richardson Maturity Model

```mermaid
graph TD
    A[Level 0: Single URI, Single Method<br/>RPC Style] --> B[Level 1: Multiple URIs<br/>Resources]
    B --> C[Level 2: HTTP Methods<br/>REST]
    C --> D[Level 3: HATEOAS<br/>Hypermedia]
```

Більшість API досягають рівня 2


# Частина 2. Проєктування API

---

## Версіонування

### URI Версіонування (найпопулярніше)
```
/api/v1/users
/api/v2/users
```

### Header Версіонування
```
GET /api/users
API-Version: 2
```

### Content Negotiation
```
Accept: application/vnd.myapi.v2+json
```

**Рекомендація:** URI для публічних API, Header для internal

---

## Посторінкова навігація (пагінація)

### Offset-Based
```
GET /api/users?offset=20&limit=10

Response:
{
  "data": [...],
  "pagination": {
    "offset": 20,
    "limit": 10,
    "total": 150
  }
}
```

**Переваги:** простота, перехід на довільну сторінку
**Недоліки:** проблеми при змінах даних

---

### Cursor-Based (рекомендовано)
```
GET /api/users?cursor=eyJpZCI6MTIzfQ==&limit=10

Response:
{
  "data": [...],
  "pagination": {
    "nextCursor": "eyJpZCI6MTMzfQ==",
    "prevCursor": "eyJpZCI6MTEzfQ==",
    "hasMore": true
  }
}
```

**Переваги:** стабільність, продуктивність
**Недоліки:** неможливість довільної сторінки

---

## Фільтрація, Сортування, Пошук

### Фільтрація
```
GET /api/products?category=electronics&price[gte]=100
GET /api/users?status=active&role=admin
```

### Сортування
```
GET /api/users?sort=name
GET /api/users?sort=-createdAt        # descending
GET /api/users?sort=lastName,firstName
```

### Пошук
```
GET /api/products?search=laptop
GET /api/users?q=іван
```

---

## Обробка помилок

### Стандартизована структура

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": [
      {
        "field": "email",
        "message": "Invalid email format",
        "code": "INVALID_FORMAT"
      }
    ],
    "timestamp": "2025-10-21T10:30:00Z",
    "path": "/api/users",
    "requestId": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

---

## Error Codes Mapping

| Status | Code | Приклад |
|--------|------|---------|
| 400 | VALIDATION_ERROR | Невалідний email |
| 401 | AUTHENTICATION_REQUIRED | Токен відсутній |
| 403 | INSUFFICIENT_PERMISSIONS | Недостатньо прав |
| 404 | RESOURCE_NOT_FOUND | Користувач не знайдений |
| 409 | RESOURCE_ALREADY_EXISTS | Email вже зареєстрований |
| 422 | BUSINESS_RULE_VIOLATION | Вік менше 18 |
| 429 | RATE_LIMIT_EXCEEDED | Забагато запитів |

---

## API Documentation

### OpenAPI (Swagger)

```yaml
openapi: 3.0.0
paths:
  /api/users:
    get:
      summary: Get all users
      parameters:
        - name: page
          in: query
          schema:
            type: integer
      responses:
        '200':
          description: Успішно
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserList'
```

**Автогенерація** з коду!

---

## Об'єкти передачі даних (DTO)

**DTO** - об'єкти для передачі даних між шарами

### Навіщо?

1. **Безпека** - контроль над тим, що відправляємо
2. **Валідація** - перевірка вхідних даних
3. **Відокремлення** - API contract ≠ Domain model
4. **Оптимізація** - лише потрібні поля

---

## Доменна модель vs DTO

### Доменна модель (внутрішня)
```python
class User:
    id: int
    username: str
    email: str
    password_hash: str      # ніколи не відправляємо!
    internal_notes: str     # тільки для адмінів
    created_at: datetime

    def change_password(self, new_password):
        # бізнес-логіка
```

### DTO (зовнішня)
```python
class UserResponseDTO:
    id: str                 # UUID замість int
    username: str
    email: str
    created_at: datetime
    # без паролю та внутрішніх полів!
```

---

## Input vs Output DTO

### CreateUserDTO (input)
```python
class CreateUserDTO:
    username: str       # required, 3-20 chars
    email: str         # required, valid email
    password: str      # required, min 8 chars
    age: int          # optional, min 18
```

### UserResponseDTO (output)
```python
class UserResponseDTO:
    id: str
    username: str
    email: str
    created_at: datetime
    updated_at: datetime
```

**Різні DTO** для різних операцій!


---

# Частина 3. Архітектурні шари

---

## Шарувата архітектура

```mermaid
graph TD
    A[Шар представлення<br/>Controllers, API] --> B[Шар бізнес-логіки<br/>Services, Domain]
    B --> C[Шар доступу до даних<br/>Repositories, ORM]
    C --> D[(Database)]
```

Кожен шар має **чітку відповідальність**

---

## Шар представлення

**Відповідальність:**
- Приймає HTTP запити
- Валідує вхідні дані (DTO)
- Викликає бізнес-логіку
- Форматує відповіді

```python
class UserController:
    def __init__(self, service: UserService):
        self.service = service

    def create_user(self, request):
        dto = CreateUserDTO.from_request(request)
        user = self.service.create_user(dto)
        return UserResponse.from_domain(user), 201
```

**Без бізнес-логіки!**

---

## Шар бізнес-логіки

**Відповідальність:**
- Бізнес-правила
- Валідація domain правил
- Координація операцій
- Транзакції

```python
class UserService:
    def create_user(self, dto: CreateUserDTO):
        # Бізнес-правило
        if self.repository.exists_by_email(dto.email):
            raise DuplicateEmailError()

        user = User.from_dto(dto)
        user.validate()
        self.repository.save(user)
        self.email_service.send_welcome_email(user)
        return user
```

---

## Шар доступу до даних

**Відповідальність:**
- Взаємодія з БД
- Перетворення domain ↔ entity
- Запит optimization

```python
class UserRepository:
    def save(self, user: User):
        entity = UserEntity.from_domain(user)
        self.db.add(entity)
        self.db.commit()

    def get_by_id(self, user_id: str) -> User:
        entity = self.db.query(UserEntity).get(user_id)
        return entity.to_domain()
```

**Ізоляція від деталей БД**

---

## Чому розділяти шари?

### Переваги

1. **Maintainability** - легше знайти та змінити код
2. **Testability** - кожен шар тестується окремо
3. **Reusability** - бізнес-логіка використовується різними clients
4. **Team collaboration** - паралельна робота
5. **Technology independence** - легко змінити БД/UI

---

## Розділення відповідальності (Separation of Concerns (SoC))

**Кожен компонент має одну відповідальність**

### Single Responsibility Principle

❌ **Погано:**
```python
class UserService:
    def create_user(self, data):
        # Валідація, SQL, Email, Логування...
        # Занадто багато!
```

✅ **Добре:**
```python
class UserService:
    def create_user(self, dto: CreateUserDTO):
        user = User.from_dto(dto)
        self.repository.save(user)
        self.email_service.send_welcome(user)
```

---

## Межі між шарами

### Правило залежностей

```
Presentation → Бізнес-логіка → Доменна модель
                ↓
           Дані Access
```

**Зовнішні шари залежать від внутрішніх**

Доменна модель не має залежностей!

```python
# ✅ Presentation залежить від Business
controller = UserController(user_service)

# ❌ Business НЕ залежить від Presentation
service = UserService()  # не знає про HTTP
```

---

## Domain-Centric Design

Доменна модель - **центр застосунку**

```python
# Domain - незалежний
class Order:
    def add_item(self, item):
        if self.is_confirmed:
            raise InvalidOperationError()
        self.items.append(item)

# Business - використовує Domain
class OrderService:
    def create_order(self, items):
        order = Order()
        for item in items:
            order.add_item(item)  # domain логіка!
        return order
```

Інфраструктура служить Domain, а не навпаки

---

## Архітектурна діаграма

```mermaid
graph TB
    subgraph Presentation
        A[Controllers]
        B[DTOs]
    end

    subgraph Business
        C[Services]
        D[Доменна модельs]
    end

    subgraph Дані
        E[Repositories]
        F[Entities]
    end

    A --> C
    B --> C
    C --> D
    C --> E
    E --> F
    F --> G[(Database)]
```

---

## Ключові висновки

1. **HTTP** - фундамент веб-комунікації
2. **REST** - принципи для проєктування API
3. **DTO** - розділення domain та API contract
4. **Layers** - організація за відповідальностями
5. **SoC** - кожен компонент має одну мету
