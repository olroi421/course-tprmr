# Лекція 1. Архітектура сучасних вебдодатків

## Анотація

Перша лекція курсу закладає фундаментальні знання про архітектуру сучасних вебдодатків, незалежно від конкретних технологій та фреймворків. Студенти ознайомляться з принципами взаємодії клієнта та сервера, глибоко вивчать протокол HTTP, архітектурний стиль REST та базові принципи проєктування API. Особлива увага приділяється шаруватій архітектурі, патернам проєктування та підходам до структурування коду, які будуть застосовуватися протягом усього курсу.

---

## Частина 1. Основи веброзробки

### 1.1. Архітектура клієнт-сервер

Сучасний інтернет побудований на фундаментальній концепції розподіленої обчислювальної системи, де функції та відповідальність розділені між постачальниками послуг (серверами) та споживачами послуг (клієнтами).

#### Роль клієнта та сервера

**Клієнт** є ініціатором взаємодії. Це може бути веббраузер, мобільний додаток, настільний застосунок або навіть інший сервер, що виступає у ролі клієнта. Основні обов'язки клієнта включають:

- Ініціювання запитів до сервера з метою отримання або зміни даних.
- Відображення отриманої інформації користувачеві у зручному форматі.
- Обробка введення користувача та формування відповідних запитів.
- Управління локальним станом застосунку.

**Сервер** відповідає на запити клієнтів, обробляє бізнес-логіку та забезпечує доступ до ресурсів. Ключові функції сервера:

- Прийом та обробка вхідних запитів від множини клієнтів.
- Виконання бізнес-логіки та застосування правил предметної області.
- Взаємодія з базами даних та іншими системами зберігання.
- Забезпечення безпеки, автентифікації та авторизації.
- Повернення відповідей клієнтам у стандартизованому форматі.

Така архітектура забезпечує централізоване управління даними та бізнес-логікою, що спрощує підтримку та масштабування системи. Водночас, вона створює єдину точку відмови та потребує надійного мережевого з'єднання.

#### Цикл запит-відповідь

Взаємодія між клієнтом та сервером відбувається у формі циклу запит-відповідь. Цей процес є синхронним за своєю природою: клієнт надсилає запит і очікує на відповідь від сервера.

Типовий цикл виглядає наступним чином:

1. **Ініціація запиту.** Клієнт формує HTTP-запит, який містить метод, URI ресурсу, заголовки та опціональне тіло повідомлення.

2. **Передача через мережу.** Запит передається через мережу, проходячи через різні мережеві рівні та можливі проміжні вузли.

3. **Обробка на сервері.** Сервер отримує запит, валідує його, виконує необхідну бізнес-логіку, взаємодіє з базою даних чи іншими сервісами.

4. **Формування відповіді.** Сервер створює HTTP-відповідь з відповідним статус-кодом, заголовками та тілом повідомлення.

5. **Повернення відповіді.** Відповідь передається назад клієнту через мережу.

6. **Обробка на клієнті.** Клієнт отримує відповідь, обробляє дані та оновлює інтерфейс користувача.

Важливо розуміти, що кожен запит є незалежним, і сервер не зберігає стан між запитами. Це фундаментальний принцип, який має суттєві наслідки для проєктування систем.

#### Підходи без збереження стану та зі збереженням стану

**Підхід без збереження стану** означає, що сервер не зберігає інформацію про попередні запити від клієнта. Кожен запит містить всю необхідну інформацію для його обробки. Переваги цього підходу:

- Простіше масштабування, оскільки будь-який сервер може обробити будь-який запит.
- Вища надійність через відсутність стану, який може бути втрачений.
- Спрощена архітектура серверної частини.
- Легше балансування навантаження між серверами.

Недоліки підходу без збереження стану:

- Збільшення обсягу даних у запитах, оскільки контекст передається кожного разу.
- Додаткові витрати на повторну автентифікацію та валідацію.
- Складніше реалізувати деякі сценарії, які природно вимагають збереження стану.

**Підхід зі збереженням стану** передбачає, що сервер зберігає інформацію про сесію клієнта між запитами. Це може бути зручно для деяких застосунків, але створює складнощі:

- Прив'язка клієнта до конкретного серверу (прив'язка сесії).
- Складніше масштабування через необхідність синхронізації стану.
- Вища вразливість до відмов, оскільки втрата стану впливає на користувачів.
- Додаткові витрати пам'яті на сервері для зберігання сесій.

У сучасній веброзробці переважає підхід без збереження стану, особливо для RESTful API. Стан управляється на стороні клієнта або в розподілених системах кешування, а не на серверах застосунків.

---

### 1.2. HTTP протокол

HTTP є протоколом прикладного рівня, який визначає формат повідомлень та правила взаємодії між клієнтами та серверами у вебі. Розуміння внутрішньої структури HTTP є критично важливим для розробки ефективних вебдодатків.

#### Структура HTTP запиту

HTTP запит складається з декількох компонентів:

**Стартовий рядок (стартова рядок)** містить три елементи:

```
GET /api/users/123 HTTP/1.1
```

- HTTP метод (GET, POST, PUT тощо).
- URI ресурсу, до якого звертається запит.
- Версія протоколу HTTP.

**Заголовки (заголовки)** надають додаткову інформацію про запит:

```
Host: api.example.com
User-Agent: Mozilla/5.0
Accept: application/json
Authorization: Bearer eyJhbGc...
Content-Type: application/json
Content-Length: 348
```

Заголовки можуть містити інформацію про:

- Хост та порт призначення.
- Формати даних, які клієнт може обробити.
- Параметри автентифікації.
- Тип та довжину вмісту тіла запиту.
- Параметри кешування.
- Cookies та іншу інформацію про сесію.

**Тіло (тіло запиту)** містить дані, які надсилаються серверу. Воно присутнє у запитах POST, PUT, PATCH:

```json
{
  "name": "Іван Петренко",
  "email": "ivan@example.com",
  "role": "user"
}
```

#### Структура HTTP відповіді

HTTP відповідь має схожу структуру:

**Рядок статусу (рядок статусу):**

```
HTTP/1.1 200 OK
```

Містить версію протоколу, код статусу та текстовий опис статусу.

**Response Заголовки (заголовки відповіді):**

```
Content-Type: application/json
Content-Length: 256
Cache-Control: max-age=3600
ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"
```

**Response Тіло (тіло відповіді):**

```json
{
  "id": "123",
  "name": "Іван Петренко",
  "email": "ivan@example.com",
  "createdAt": "2025-01-15T10:30:00Z"
}
```

#### HTTP методи

HTTP визначає набір методів, кожен з яких має специфічну семантику:

**GET** використовується для отримання ресурсу. Запити GET повинні бути безпечними (не змінювати стан сервера) та ідемпотентними. Параметри передаються у рядок запиту:

```
GET /api/users?page=2&limit=20&sort=name
```

**POST** створює новий ресурс або виконує операцію, яка змінює стан. POST не є ідемпотентним. Дані передаються у тілі запиту:

```
POST /api/users
Content-Type: application/json

{
  "name": "Марія",
  "email": "maria@example.com"
}
```

**PUT** оновлює існуючий ресурс або створює його за вказаним URI, якщо він не існує. PUT є ідемпотентним - повторні виклики з тими самими даними не змінюють результат:

```
PUT /api/users/123
Content-Type: application/json

{
  "name": "Іван Петренко",
  "email": "newemail@example.com"
}
```

**PATCH** частково оновлює ресурс. На відміну від PUT, який замінює ресурс повністю, PATCH модифікує лише вказані поля:

```
PATCH /api/users/123
Content-Type: application/json

{
  "email": "updated@example.com"
}
```

**DELETE** видаляє ресурс. Метод є ідемпотентним - повторне видалення того самого ресурсу не змінює результат:

```
DELETE /api/users/123
```

**OPTIONS** повертає інформацію про дозволені методи для ресурсу. Часто використовується у механізмі CORS:

```
OPTIONS /api/users
```

**HEAD** аналогічний до GET, але повертає лише заголовки без тіла. Корисний для перевірки існування ресурсу або отримання метаданих:

```
HEAD /api/users/123
```

#### Ідемпотентність операцій

Ідемпотентність є важливою властивістю HTTP методів. Операція вважається ідемпотентною, якщо її повторне виконання з тими самими параметрами дає той самий результат, що й одноразове виконання.

**Ідемпотентні методи:** GET, PUT, DELETE, HEAD, OPTIONS.

**Не ідемпотентні методи:** POST, PATCH (хоча PATCH може бути спроєктований як ідемпотентний).

Ідемпотентність важлива для:

- Автоматичного повторення запитів при мережевих збоях.
- Проєктування надійних розподілених систем.
- Кешування відповідей.

#### HTTP заголовки

Заголовки відіграють критичну роль у HTTP комунікації. Розглянемо найважливіші:

**Content-Type** вказує формат даних у тілі повідомлення:

```
Content-Type: application/json; charset=utf-8
Content-Type: application/xml
Content-Type: multipart/form-data
Content-Type: text/html
```

**Authorization** містить облікові дані для автентифікації:

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=
```

**Cache-Control** управляє поведінкою кешування:

```
Cache-Control: no-cache
Cache-Control: max-age=3600
Cache-Control: private, must-revalidate
```

**Accept** вказує, які типи контенту клієнт може обробити:

```
Accept: application/json
Accept: application/json, application/xml;q=0.9
```

**ETag** надає ідентифікатор версії ресурсу для оптимістичної конкурентності:

```
ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"
If-None-Match: "33a64df551425fcc55e4d42a148795d9f25f89d4"
```

**CORS заголовки** управляють міжджерельні запити:

```
Access-Control-Allow-Origin: https://example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Заголовки: Content-Type, Authorization
```

#### Status codes

HTTP статус-коди інформують про результат обробки запиту. Вони групуються за категоріями:

**2xx Success** - запит успішно оброблено:

- **200 OK** - стандартна успішна відповідь для GET, PUT, PATCH.
- **201 Created** - ресурс успішно створено (POST).
- **204 No Content** - запит успішно оброблено, але немає контенту для відправки (часто для DELETE).
- **202 Accepted** - запит прийнято до обробки, але обробка ще не завершена.

**3xx Redirection** - потрібні додаткові дії для завершення запиту:

- **301 Moved Permanently** - ресурс переміщено назавжди.
- **302 Found** - тимчасове перенаправлення.
- **304 Not Modified** - ресурс не змінився з останнього запиту (кешування).

**4xx Client Error** - помилка у запиті клієнта:

- **400 Bad Request** - некоректний синтаксис запиту або валідаційна помилка.
- **401 Unauthorized** - необхідна автентифікація.
- **403 Forbidden** - сервер зрозумів запит, але відмовляє у доступі.
- **404 Not Found** - ресурс не знайдено.
- **409 Conflict** - конфлікт зі станом ресурсу (наприклад, duplicate).
- **422 Unprocessable Entity** - семантичні помилки у даних.
- **429 Too Many Requests** - клієнт надіслав забагато запитів (rate limiting).

**5xx Server Error** - помилка на сервері:

- **500 Internal Server Error** - загальна серверна помилка.
- **502 Bad Gateway** - неправильна відповідь від upstream сервера.
- **503 Service Unavailable** - сервер тимчасово недоступний.
- **504 Gateway Timeout** - upstream сервер не відповів вчасно.

Правильний вибір статус-кодів є важливою частиною проєктування API. Він дозволяє клієнтам коректно інтерпретувати результат операцій та реагувати відповідно.

---

### 1.3. REST архітектурний стиль

REST є архітектурним стилем для розподілених гіпермедійних систем, вперше описаним Roy Fielding у його дисертації 2000 року. Хоча REST часто плутають з HTTP API, насправді це набір архітектурних обмежень.

#### Принципи REST

REST визначає шість фундаментальних принципів:

**клієнт-сервер.** Розділення інтерфейсу користувача від зберігання даних покращує портативність інтерфейсу та масштабованість серверних компонентів.

**Stateless.** Кожен запит від клієнта до сервера має містити всю інформацію, необхідну для розуміння запиту. Стан сесії повністю тримається на клієнті.

**Cacheable.** Відповіді повинні явно позначати себе як придатні або непридатні для кешування. Кешування може частково або повністю усунути деякі взаємодії, покращуючи ефективність та масштабованість.

**Uniform Interface.** Єдиний інтерфейс між компонентами спрощує архітектуру та покращує видимість взаємодій. Це досягається через:

- Ідентифікацію ресурсів через URI.
- Маніпулювання ресурсами через представлення.
- Самоописувані повідомлення.
- Hypermedia as the Engine of Application State (HATEOAS).

**Layered System.** Архітектура може складатися з ієрархічних шарів, обмежуючи поведінку компонентів. Клієнт не може визначити, чи підключений безпосередньо до кінцевого сервера, чи до проміжного.

**Code-On-Demand (опціонально).** Сервери можуть тимчасово розширювати функціональність клієнта, передаючи виконуваний код.

#### Ресурсо-орієнтоване проєктування

У REST центральною концепцією є ресурс. Ресурс - це будь-яка іменована інформація, яка може бути представлена: документ, зображення, колекція інших ресурсів, нефізичний об'єкт.

Ключові принципи resource-oriented дизайну:

**Ідентифікація ресурсів.** Кожен ресурс має унікальний ідентифікатор (URI). Наприклад:

```
/api/users          - колекція користувачів
/api/users/123      - конкретний користувач
/api/users/123/posts - пости користувача
```

**Представлення ресурсів.** Клієнти не працюють безпосередньо з ресурсами, а з їх представленнями (JSON, XML, HTML). Те саме ресурс може мати різні представлення:

```
GET /api/users/123
Accept: application/json

GET /api/users/123
Accept: application/xml
```

**Маніпулювання через HTTP методи.** Дії над ресурсами виконуються через стандартні HTTP методи:

```
GET    /api/users      - отримати список
POST   /api/users      - створити нового
GET    /api/users/123  - отримати конкретного
PUT    /api/users/123  - оновити повністю
PATCH  /api/users/123  - оновити частково
DELETE /api/users/123  - видалити
```

#### Найкращі практики проєктування URI

Правильне проєктування URI робить API інтуїтивним та зручним:

**Використовуйте іменники, а не дієслова:**

```
✅ GET /api/users
❌ GET /api/getUsers

✅ POST /api/orders
❌ POST /api/createOrder
```

**Використовуйте множину для колекцій:**

```
✅ /api/users
✅ /api/products
❌ /api/user
```

**Використовуйте вкладені ресурси для відношень:**

```
/api/users/123/orders
/api/orders/456/items
```

**Уникайте глибокої вкладеності (максимум 2-3 рівні):**

```
✅ /api/users/123/orders
❌ /api/organizations/1/departments/2/teams/3/users/4/orders
```

**Використовуйте query parameters для фільтрації та пагінації:**

```
/api/users?role=admin&status=active
/api/products?category=electronics&page=2&limit=20
```

**Використовуйте kebab-case для multi-word ресурсів:**

```
✅ /api/user-profiles
✅ /api/order-items
❌ /api/userProfiles
❌ /api/order_items
```

**Версіонуйте API:**

```
/api/v1/users
/api/v2/users
```

#### Концепція HATEOAS

HATEOAS є найбільш складною та рідко реалізованою частиною REST. Концепція полягає у тому, що клієнт взаємодіє з застосунком виключно через гіпермедіа, динамічно надану сервером.

Приклад HATEOAS відповіді:

```json
{
  "id": "123",
  "name": "Іван Петренко",
  "email": "ivan@example.com",
  "_links": {
    "self": {
      "href": "/api/users/123"
    },
    "orders": {
      "href": "/api/users/123/orders"
    },
    "edit": {
      "href": "/api/users/123",
      "method": "PUT"
    },
    "delete": {
      "href": "/api/users/123",
      "method": "DELETE"
    }
  }
}
```

Клієнт не має знати структуру URI заздалегідь, а отримує її від сервера. Це дозволяє серверу змінювати структуру без порушення клієнтів.

На практиці HATEOAS рідко реалізується повністю через складність та додаткові витрати. Більшість API обмежуються рівнями 2-3 Richardson Maturity Model.

#### Порівняння REST, RPC та GraphQL

**RPC (Remote Procedure Call)** - підхід, при якому API проєктується як набір процедур або функцій:

```
POST /api/calculateOrderTotal
POST /api/sendEmail
POST /api/validateUser
```

Переваги RPC: простота, природність для розробників. Недоліки: менша стандартизація, складніше кешування.

**REST** - resource-oriented підхід з використанням HTTP методів та статус-кодів:

```
GET    /api/orders/123
POST   /api/orders
DELETE /api/orders/123
```

Переваги REST: стандартизація, кешування, stateless. Недоліки: over-fetching/under-fetching, множинні запити.

**GraphQL** - мова запитів, що дозволяє клієнту точно визначити, які дані потрібні:

```graphql
query {
  user(id: "123") {
    name
    email
    orders {
      id
      total
    }
  }
}
```

Переваги GraphQL: гнучкість, один endpoint, сильна типізація. Недоліки: складність, проблеми з кешуванням, N+1 запити.

Вибір між підходами залежить від специфіки проєкту, команди та вимог до системи.

---

## Частина 2. Проєктування API

### 2.1. Проєктування API

Якісне проєктування API є критично важливим для успіху продукту. API має бути інтуїтивним, послідовним та зручним у використанні.

#### Стратегії версіонування

Версіонування дозволяє вносити breaking changes без порушення існуючих клієнтів. Існує кілька підходів:

**URI versioning** - найпопулярніший підхід:

```
/api/v1/users
/api/v2/users
```

Переваги: простота, очевидність. Недоліки: розмноження endpoints, порушення принципу єдиного URI для ресурсу.

**Header versioning** - версія передається у заголовку:

```
GET /api/users
API-Version: 2
```

Переваги: чистіші URI. Недоліки: менш очевидно, складніше тестувати у браузері.

**Content negotiation** - версія є частиною Accept заголовка:

```
GET /api/users
Accept: application/vnd.myapi.v2+json
```

Переваги: RESTful підхід. Недоліки: складність, менша підтримка інструментів.

**Query parameter versioning:**

```
/api/users?version=2
```

Переваги: простота. Недоліки: може конфліктувати з іншими параметрами, проблеми з кешуванням.

Рекомендація: URI versioning для публічних API, header versioning для внутрішніх систем.

#### Патерни посторінкової навігації

Для колекцій з великою кількістю елементів необхідна пагінація:

**Offset-based pagination:**

```
GET /api/users?offset=20&limit=10
```

Переваги: простота реалізації, можливість переходу на довільну сторінку. Недоліки: проблеми при вставці/видаленні елементів, погана продуктивність для великих offset.

Відповідь:

```json
{
  "data": [...],
  "pagination": {
    "offset": 20,
    "limit": 10,
    "total": 150
  }
}
```

**Cursor-based pagination:**

```
GET /api/users?cursor=eyJpZCI6MTIzfQ==&limit=10
```

Переваги: стабільність при змінах даних, краща продуктивність. Недоліки: неможливість переходу на довільну сторінку.

Відповідь:

```json
{
  "data": [...],
  "pagination": {
    "nextCursor": "eyJpZCI6MTMzfQ==",
    "prevCursor": "eyJpZCI6MTEzfQ==",
    "hasMore": true
  }
}
```

**Page-based pagination:**

```
GET /api/users?page=3&pageSize=20
```

Переваги: зрозуміло для користувачів. Недоліки: ті самі що й у offset-based.

Рекомендація: cursor-based для великих датасетів та feeds, offset-based для адміністративних інтерфейсів.

#### Фільтрація, сортування, пошук

**Filtering** дозволяє клієнту отримувати підмножину ресурсів:

```
GET /api/users?role=admin&status=active
GET /api/products?category=electronics&price_min=100&price_max=500
```

Підтримка операторів:

```
GET /api/products?price[gte]=100&price[lte]=500
GET /api/users?createdAt[gt]=2025-01-01
```

**Sorting:**

```
GET /api/users?sort=name
GET /api/users?sort=-createdAt           # descending
GET /api/users?sort=lastName,firstName    # multiple fields
```

**Searching:**

```
GET /api/users?search=іван
GET /api/products?q=laptop
```

**Комбінування:**

```
GET /api/products?category=electronics&price[gte]=500&sort=-price&page=2
```

#### Обробка помилок та стандартизація відповідей

Послідовна обробка помилок критично важлива для якості API.

**Стандартизована структура помилки:**

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed for the request",
    "details": [
      {
        "field": "email",
        "message": "Invalid email format",
        "code": "INVALID_FORMAT"
      },
      {
        "field": "age",
        "message": "Must be at least 18",
        "code": "MIN_VALUE"
      }
    ],
    "timestamp": "2025-10-21T10:30:00Z",
    "path": "/api/users",
    "requestId": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

**Типи помилок та коди:**

- **400 Bad Request** - код: VALIDATION_ERROR, INVALID_REQUEST
- **401 Unauthorized** - код: AUTHENTICATION_REQUIRED, INVALID_TOKEN
- **403 Forbidden** - код: INSUFFICIENT_PERMISSIONS
- **404 Not Found** - код: RESOURCE_NOT_FOUND
- **409 Conflict** - код: RESOURCE_ALREADY_EXISTS, CONCURRENT_MODIFICATION
- **422 Unprocessable Entity** - код: BUSINESS_RULE_VIOLATION
- **429 Too Many Requests** - код: RATE_LIMIT_EXCEEDED
- **500 Internal Server Error** - код: INTERNAL_ERROR

**Стандартизована структура успішної відповіді:**

```json
{
  "data": {
    "id": "123",
    "name": "Іван"
  },
  "meta": {
    "timestamp": "2025-10-21T10:30:00Z",
    "version": "1.0"
  }
}
```

Для колекцій:

```json
{
  "data": [...],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "total": 150
  },
  "meta": {
    "timestamp": "2025-10-21T10:30:00Z"
  }
}
```

#### Практики документування API

Документація API має бути:

- **Повною** - описувати всі endpoints, параметри, відповіді.
- **Актуальною** - автоматично генеруватися з коду.
- **Інтерактивною** - дозволяти тестувати запити.
- **З прикладами** - містити реальні приклади запитів та відповідей.

**OpenAPI (Swagger)** є стандартом для документування REST API:

```yaml
openapi: 3.0.0
info:
  title: User Management API
  version: 1.0.0
paths:
  /api/users:
    get:
      summary: Get all users
      parameters:
        - name: page
          in: query
          schema:
            type: integer
            default: 1
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/User'
```

Інструменти автогенерації документації інтегруються безпосередньо у код та забезпечують актуальність документації.

---

### 2.2. Об'єкти передачі даних

DTO є патерном проєктування, який використовується для передачі даних між різними шарами застосунку або між клієнтом та сервером.

#### Чому потрібні DTO

**Відокремлення внутрішньої моделі від API contract.** Domain model може змінюватися, але API має залишатися стабільним для зворотної сумісності.

**Безпека.** DTO дозволяє контролювати, які поля доступні зовні. Наприклад, не відправляти пароль або внутрішні ідентифікатори:

```python
# Domain Model
class User:
    id: int
    username: str
    email: str
    password_hash: str  # ніколи не відправляємо
    internal_notes: str  # внутрішня інформація

# DTO для відповіді
class UserResponseDTO:
    id: str  # UUID замість внутрішнього int
    username: str
    email: str
    created_at: datetime
```

**Валідація та трансформація.** DTO забезпечує валідацію вхідних даних до того, як вони потраплять у бізнес-логіку:

```python
class CreateUserDTO:
    username: str  # 3-20 символів
    email: str     # валідний email
    password: str  # мінімум 8 символів
    age: int       # мінімум 18
```

**Оптимізація трафіку.** DTO може містити лише необхідні поля, зменшуючи обсяг даних:

```python
# Повна модель
class Article:
    id, title, content, author, tags, comments, metadata...

# DTO для списку (мінімум даних)
class ArticleListItemDTO:
    id, title, author_name, created_at

# DTO для детального перегляду
class ArticleDetailDTO:
    id, title, content, author, tags, comments_count
```

#### Доменна модель проти DTO

**Domain Model** представляє бізнес-логіку та правила:

```python
class Order:
    def __init__(self):
        self.items = []
        self.status = OrderStatus.DRAFT

    def add_item(self, item):
        if self.status != OrderStatus.DRAFT:
            raise InvalidOperationError("Cannot modify confirmed order")
        self.items.append(item)

    def calculate_total(self):
        return sum(item.price * item.quantity for item in self.items)

    def confirm(self):
        if not self.items:
            raise ValidationError("Cannot confirm empty order")
        self.status = OrderStatus.CONFIRMED
```

**DTO** є простими data containers без логіки:

```python
class CreateOrderDTO:
    items: List[OrderItemDTO]
    shipping_address: AddressDTO

class OrderResponseDTO:
    id: str
    items: List[OrderItemDTO]
    total: Decimal
    status: str
    created_at: datetime
```

Розділення domain model та DTO дозволяє змінювати внутрішню реалізацію без впливу на API.

#### Принципи валідації введення

Валідація вхідних даних має відбуватися на рівні DTO:

**Типізація** - перевірка типів даних:

```python
class CreateUserDTO:
    username: str       # має бути строкою
    age: int           # має бути цілим числом
    email: str         # має бути строкою
    is_active: bool    # має бути булевим значенням
```

**Обмеження значень:**

```python
class CreateUserDTO:
    username: str  # min: 3, max: 20, pattern: ^[a-zA-Z0-9_]+$
    age: int       # min: 18, max: 120
    email: str     # format: email
    password: str  # min: 8, має містити букви та цифри
```

**Бізнес-правила:**

```python
class CreateOrderDTO:
    items: List[OrderItemDTO]  # min items: 1
    shipping_address: AddressDTO  # required
    billing_address: Optional[AddressDTO]  # optional

    def validate(self):
        if not self.items:
            raise ValidationError("Order must contain at least one item")
        if any(item.quantity <= 0 for item in self.items):
            raise ValidationError("Item quantity must be positive")
```

**Fail fast principle** - валідація має відбуватися якомога раніше, до виконання бізнес-логіки.

#### Серіалізація та десеріалізація

Процес перетворення об'єктів у формат для передачі (зазвичай JSON) та назад.

**Serialization** (об'єкт → JSON):

```python
user = User(id=123, name="Іван", email="ivan@example.com")
json_data = serialize(user)
# {"id": 123, "name": "Іван", "email": "ivan@example.com"}
```

**Deserialization** (JSON → об'єкт):

```python
json_data = '{"username": "ivan", "email": "ivan@example.com"}'
dto = deserialize(json_data, CreateUserDTO)
# dto.username == "ivan"
```

Сучасні бібліотеки (Pydantic у Python, Jackson у Java, System.Text.Json у .NET) автоматично виконують серіалізація та десеріалізація з валідацією.

---

### 2.3. Патерн впровадження залежностей

впровадження залежностей є фундаментальним патерном для створення loosely coupled та testable коду.

#### Інверсія управління

Традиційний підхід - клас створює свої залежності:

```python
class UserService:
    def __init__(self):
        self.repository = UserRepository()  # tight coupling
        self.email_service = EmailService()
```

Проблеми: складно тестувати, неможливо замінити реалізацію, порушення Single Responsibility.

**Інверсія управління** - залежності надаються zzovні:

```python
class UserService:
    def __init__(self, repository: IUserRepository, email_service: IEmailService):
        self.repository = repository
        self.email_service = email_service
```

Тепер UserService не знає про конкретні реалізації, залежить лише від інтерфейсів.

#### Переваги впровадження залежностей для тестування

DI дозволяє легко замінювати реальні залежності на mock об'єкти у тестах:

```python
def test_create_user():
    # Створюємо mock залежності
    mock_repo = MockUserRepository()
    mock_email = MockEmailService()

    # Інжектимо їх у сервіс
    service = UserService(mock_repo, mock_email)

    # Тестуємо без реальної БД та email
    result = service.create_user("ivan", "ivan@example.com")

    assert mock_repo.save_called
    assert mock_email.send_called
```

Без DI тестування вимагало б реальної бази даних та email сервера.

#### Життєвий цикл залежностей

DI контейнери керують життєвим циклом об'єктів:

**Transient** - новий екземпляр створюється кожного разу:

```python
# Кожен запит отримує новий UserService
container.register(UserService, lifetime=Transient)
```

Використовується для легковагових stateless сервісів.

**Scoped** - один екземпляр на scope (зазвичай HTTP request):

```python
# Один DatabaseContext на HTTP запит
container.register(DatabaseContext, lifetime=Scoped)
```

Використовується для об'єктів, які мають стан протягом запиту.

**Singleton** - один екземпляр на весь час життя застосунку:

```python
# Один Configuration об'єкт на весь застосунок
container.register(Configuration, lifetime=Singleton)
```

Використовується для дорогих у створенні об'єктів або глобальних ресурсів.

Правильний вибір lifetime критично важливий для продуктивності та коректності роботи.

---

## Частина 3. Архітектурні шари

### 3.1. Шарувата архітектура

Шарувата архітектура організує код у горизонтальні шари, кожен з яких має специфічну відповідальність.

#### Шар представлення

Відповідає за взаємодію з зовнішнім світом - приймає HTTP запити, валідує вхідні дані, викликає бізнес-логіку, форматує відповіді.

```python
class UserController:
    def __init__(self, user_service: UserService):
        self.user_service = user_service

    def create_user(self, request: CreateUserRequest):
        # 1. Валідація вхідних даних
        dto = CreateUserDTO.from_request(request)

        # 2. Виклик бізнес-логіки
        user = self.user_service.create_user(dto)

        # 3. Форматування відповіді
        return UserResponse.from_domain(user), 201
```

Presentation layer не містить бізнес-логіки, лише маршрутизацію та перетворення даних.

#### Шар бізнес-логіки

Містить основну логіку застосунку, бізнес-правила та операції над даними.

```python
class UserService:
    def __init__(self, user_repository: IUserRepository,
                 email_service: IEmailService):
        self.user_repository = user_repository
        self.email_service = email_service

    def create_user(self, dto: CreateUserDTO):
        # Бізнес-правила
        if self.user_repository.exists_by_email(dto.email):
            raise DuplicateEmailError("Email already registered")

        # Створення domain об'єкта
        user = User(
            username=dto.username,
            email=dto.email,
            password_hash=hash_password(dto.password)
        )

        # Валідація domain правил
        user.validate()

        # Збереження
        self.user_repository.save(user)

        # Side effects
        self.email_service.send_welcome_email(user.email)

        return user
```

Business layer не знає про HTTP, JSON чи базу даних - працює з domain об'єктами.

#### Шар доступу до даних

Відповідає за взаємодію з базою даних та іншими системами зберігання.

```python
class UserRepository(IUserRepository):
    def __init__(self, db_context: DatabaseContext):
        self.db = db_context

    def save(self, user: User):
        # Перетворення domain model в database entity
        entity = UserEntity.from_domain(user)
        self.db.users.add(entity)
        self.db.commit()

    def get_by_id(self, user_id: str) -> Optional[User]:
        entity = self.db.users.filter_by(id=user_id).first()
        if not entity:
            return None
        # Перетворення database entity в domain model
        return entity.to_domain()

    def exists_by_email(self, email: str) -> bool:
        return self.db.users.filter_by(email=email).count() > 0
```

Шар доступу до даних ізолює решту застосунку від деталей бази даних. Можна змінити БД без впливу на бізнес-логіку.

#### Чому розділяти?

Розділення на шари надає численні переваги:

**Maintainability** - легше знайти та змінити код, коли він організований за відповідальностями.

**Testability** - кожен шар можна тестувати незалежно:

```python
# Тестування business layer без БД та HTTP
def test_create_user_duplicate_email():
    mock_repo = MockUserRepository()
    mock_repo.exists_by_email = lambda email: True
    service = UserService(mock_repo, MockEmailService())

    with pytest.raises(DuplicateEmailError):
        service.create_user(dto)
```

**Reusability** - бізнес-логіка може використовуватися різними шар представленняs (REST API, GraphQL, CLI, message queues).

**Team collaboration** - різні команди можуть працювати над різними шарами паралельно.

**Technology independence** - можна змінити технології в одному шарі без впливу на інші.

---

### 3.2. Separation of Concerns

Separation of Concerns є фундаментальним принципом, який лежить в основі якісної архітектури.

#### Single Responsibility Principle

Кожен клас має одну причину для зміни. Класи мають бути focused та cohesive.

**Погано:**

```python
class UserService:
    def create_user(self, data):
        # Валідація
        if not validate_email(data['email']):
            raise ValidationError()

        # Хешування паролю
        password_hash = bcrypt.hash(data['password'])

        # SQL запит
        cursor.execute("INSERT INTO users ...")

        # Відправка email
        smtp.send_email(...)

        # Логування
        logger.info(...)
```

Цей клас має занадто багато відповідальностей.

**Добре:**

```python
class UserService:
    def __init__(self, repository, email_service, logger):
        self.repository = repository
        self.email_service = email_service
        self.logger = logger

    def create_user(self, dto: CreateUserDTO):
        # dto вже провалідоване
        user = User.from_dto(dto)
        self.repository.save(user)
        self.email_service.send_welcome_email(user.email)
        self.logger.info(f"User created: {user.id}")
        return user
```

Тепер кожен компонент має одну відповідальність: валідація у DTO, хешування у User, SQL у Repository, email у EmailService.

#### Boundaries між шарами

Шари мають взаємодіяти лише через чітко визначені інтерфейси:

```python
# Presentation → Business
controller.create_user(dto) → service.create_user(dto)

# Business → Data Access
service.create_user() → repository.save(user)

# Business ← Data Access
user = repository.get_by_id(id) ← service.get_user()

# Presentation ← Business
response = UserResponse.from_domain(user) ← controller
```

**Правило залежностей:** зовнішні шари залежать від внутрішніх, а не навпаки:

```
Presentation → Business Logic → Domain Model
                ↓
           Data Access
```

Domain Model не має залежностей, Business Logic залежить від Domain, Presentation залежить від Business.

#### Domain-centric design

У domain-centric підході domain model є центром застосунку. Всі інші шари обертаються навколо нього та служать йому.

```python
# Domain Model - незалежний
class Order:
    def add_item(self, item):
        if self.is_confirmed:
            raise InvalidOperationError()
        self.items.append(item)

    def calculate_total(self):
        return sum(item.subtotal for item in self.items)

# Business Logic - залежить від Domain
class OrderService:
    def create_order(self, items):
        order = Order()
        for item_dto in items:
            item = OrderItem.from_dto(item_dto)
            order.add_item(item)  # використовує domain логіку
        return order

# Data Access - залежить від Domain
class OrderRepository:
    def save(self, order: Order):  # приймає domain об'єкт
        entity = OrderEntity.from_domain(order)
        self.db.save(entity)
```

Domain model містить бізнес-правила та інваріанти. Інфраструктурні деталі (БД, API) ізольовані в зовнішніх шарах.

---

## Висновки

Ця лекція заклала фундамент для розуміння архітектури сучасних вебдодатків. Ми розглянули базові концепції клієнт-серверної взаємодії, детально вивчили HTTP протокол та REST архітектурний стиль, ознайомилися з принципами проєктування API та важливістю використання DTO для розділення concerns.

Ключові takeaways:

- HTTP є фундаментом веб-комунікації, розуміння його структури критично важливе.
- REST надає принципи для проєктування масштабованих та зручних API.
- DTO забезпечують чисте розділення між domain model та API contract.
- впровадження залежностей робить код testable та maintainable.
- Шарувата архітектура організує код за відповідальностями.
- Separation of Concerns є ключем до якісної архітектури.

У наступних лекціях ми поглибимо ці знання, вивчаючи конкретні технології та інструменти для реалізації описаних принципів на практиці.

---

## Питання для самоперевірки

1. У чому полягає різниця між stateless та stateful підходами? Які переваги має stateless архітектура?

2. Чому важлива ідемпотентність HTTP методів? Які методи є ідемпотентними?

3. Які основні принципи REST? Чому HATEOAS рідко реалізується на практиці?

4. Як правильно проєктувати URI для REST API? Наведіть приклади добрих та поганих практик.

5. Навіщо потрібні DTO? Чому не можна використовувати domain model безпосередньо в API?

6. Що таке впровадження залежностей? Які переваги він надає для тестування?

7. Які шари включає Шарувата архітектура? Яка відповідальність кожного шару?

8. Що означає Single Responsibility Principle? Наведіть приклад його порушення та виправлення.

9. Які стратегії версіонування API ви знаєте? Яка найпопулярніша?

10. Як правильно обробляти помилки в API? Яку структуру має мати error response?

---

## Додаткові матеріали

1. Roy Fielding - Architectural Styles and the Design of Network-based Software Architectures (Дисертація, 2000)
2. Martin Fowler - Patterns of Enterprise Application Architecture
3. RFC 7231 - Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content
4. OpenAPI Specification - https://swagger.io/specification/
5. Richardson Maturity Model - https://martinfowler.com/articles/richardsonMaturityModel.html
