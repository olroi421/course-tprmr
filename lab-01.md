# Лабораторна робота 1. Проєктування REST API

**Дисципліна:** Технології проектування та розробки мережевих ресурсів  
**Тема:** Проєктування REST API для системи управління задачами  
**Тривалість:** 2 години

---

## Мета роботи

Навчитися проєктувати REST API, застосовуючи принципи resource-oriented design, правильно використовуючи HTTP методи та статус-коди. Студенти мають продемонструвати базове розуміння принципів REST.

---

## Теоретичні відомості

### REST API Principles

REST API проєктується навколо концепції ресурсів:
- Кожен ресурс має унікальний URI
- Операції виконуються через HTTP методи
- Використовуються стандартні статус-коди

### HTTP Methods

- **GET** - отримання ресурсу (безпечний, ідемпотентний)
- **POST** - створення ресурсу (не ідемпотентний)
- **PUT** - повне оновлення ресурсу (ідемпотентний)
- **PATCH** - часткове оновлення ресурсу
- **DELETE** - видалення ресурсу (ідемпотентний)

### HTTP Status Codes

**2xx Success:**
- 200 OK - успішний GET, PUT, PATCH
- 201 Created - успішний POST
- 204 No Content - успішний DELETE

**4xx Client Errors:**
- 400 Bad Request - невалідні дані
- 401 Unauthorized - потрібна автентифікація
- 404 Not Found - ресурс не існує
- 409 Conflict - конфлікт стану

**5xx Server Errors:**
- 500 Internal Server Error - помилка сервера

### Data Transfer Objects (DTO)

DTO відокремлюють внутрішню модель від API:
- **CreateDTO** - дані для створення (без id)
- **UpdateDTO** - дані для оновлення (опціональні поля)
- **ResponseDTO** - дані у відповіді (включає readonly поля)

---

## Завдання

Спроєктуйте REST API для простої системи управління задачами.

### Функціональні вимоги

**Задачі (Tasks):**
- Створення задачі
- Перегляд списку задач
- Перегляд конкретної задачі
- Оновлення задачі
- Видалення задачі
- Фільтрація за статусом

**Користувачі (Users):**
- Перегляд списку користувачів
- Перегляд конкретного користувача

### Атрибути

**Task:**
- Назва (обов'язково, 3-200 символів)
- Опис (опціонально, max 1000 символів)
- Статус: todo, in_progress, done
- Пріоритет: low, medium, high
- Дата створення (автоматично)

**User:**
- Ім'я
- Email

---

## Хід роботи

### Частина 1. Аналіз предметної області (15 хв)

Заповніть таблицю:

| Ресурс | Основні атрибути |
|--------|------------------|
| Task | id, title, description, status, priority, createdAt |
| User | id, name, email |

---

### Частина 2. Дизайн API Endpoints (30 хв)

Створіть таблицю endpoints (мінімум 7):

| HTTP Method | URI Path | Опис | Status Codes |
|-------------|----------|------|--------------|
| GET | /api/tasks | Список задач | 200, 401 |
| GET | /api/tasks/{id} | Конкретна задача | 200, 404, 401 |
| POST | /api/tasks | Створити задачу | 201, 400, 401 |
| PUT | /api/tasks/{id} | Оновити задачу | 200, 400, 404, 401 |
| DELETE | /api/tasks/{id} | Видалити задачу | 204, 404, 401 |
| GET | /api/users | Список користувачів | 200, 401 |
| GET | /api/users/{id} | Конкретний користувач | 200, 404, 401 |

**Фільтрація:**
```
GET /api/tasks?status=done
GET /api/tasks?priority=high&status=in_progress
```

**Питання для роздумів:**
1. Чому GET /api/tasks/{id} повертає 404, коли задача не існує?
2. Чому DELETE повертає 204, а не 200?
3. Яка різниця між PUT та PATCH?

---

### Частина 3. Структури даних (40 хв)

Опишіть JSON структури:

**Task Response:**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "title": "Завершити лабораторну роботу",
  "description": "Спроєктувати REST API",
  "status": "in_progress",
  "priority": "high",
  "createdAt": "2025-10-21T10:30:00Z"
}
```

**CreateTaskDTO:**
```json
{
  "title": "string (required, 3-200 chars)",
  "description": "string (optional, max 1000 chars)",
  "priority": "low | medium | high (default: medium)"
}
```

**UpdateTaskDTO:**
```json
{
  "title": "string (optional, 3-200 chars)",
  "description": "string (optional, max 1000 chars)",
  "status": "todo | in_progress | done (optional)",
  "priority": "low | medium | high (optional)"
}
```

**User Response:**
```json
{
  "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "name": "Іван Петренко",
  "email": "ivan@example.com"
}
```

**Список задач:**
```json
{
  "data": [
    { /* Task object */ },
    { /* Task object */ }
  ],
  "total": 42
}
```

---

### Частина 4. Error Handling (25 хв)

**Стандартна структура помилки:**
```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Зрозуміле повідомлення",
    "timestamp": "2025-10-21T10:30:00Z"
  }
}
```

**Основні error codes:**

| HTTP Status | Error Code | Приклад |
|-------------|------------|---------|
| 400 | VALIDATION_ERROR | "Title is required" |
| 401 | AUTHENTICATION_REQUIRED | "Token is missing" |
| 404 | RESOURCE_NOT_FOUND | "Task not found" |
| 409 | CONFLICT | "Task already exists" |
| 500 | INTERNAL_ERROR | "Server error" |

**Приклад validation error:**
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": [
      {
        "field": "title",
        "message": "Title must be at least 3 characters"
      }
    ],
    "timestamp": "2025-10-21T10:30:00Z"
  }
}
```

**Приклад not found error:**
```json
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "Task with id '123' not found",
    "timestamp": "2025-10-21T10:30:00Z"
  }
}
```

---

### Частина 5. Додатково (Опціонально, +1 бал)

**Опція А:** Опишіть 2-3 endpoints у форматі OpenAPI 3.0

**Опція Б:** Додайте pagination та розширені фільтри

---

## Форма здачі

Створіть файл `lab-01-[прізвище].md` з наступною структурою:

```markdown
# Лабораторна робота 1: Проєктування REST API

**ПІБ:** [Ваше ПІБ]
**Група:** [Група]
**Дата:** [Дата]

## 1. Аналіз предметної області

| Ресурс | Атрибути |
|--------|----------|
| Task | ... |
| User | ... |

## 2. API Endpoints

| HTTP Method | URI Path | Опис | Status Codes |
|-------------|----------|------|--------------|
| ... | ... | ... | ... |

**Пояснення:** [Чому обрали такі endpoints]

## 3. Структури даних

### Task Response
\```json
{...}
\```

### CreateTaskDTO
\```json
{...}
\```

### UpdateTaskDTO
\```json
{...}
\```

### User Response
\```json
{...}
\```

### Error Response
\```json
{...}
\```

## 4. Відповіді на питання

1. [Відповідь]
2. [Відповідь]
3. [Відповідь]

## 5. Висновки

[Ваші висновки]
```

**Здача:** Moodle курс дисципліни

---

## Критерії оцінювання

Максимальна оцінка: **4 бали** (+ **1 бонусний**)

| Критерій | Бали | Що оцінюється |
|----------|------|---------------|
| Аналіз | 0.5 | Ресурси та атрибути |
| Endpoints | 1.5 | HTTP методи, URI, статус-коди |
| Структури | 1.0 | JSON для Task, User, DTO, Error |
| Error handling | 0.5 | Структура та codes |
| Відповіді | 0.5 | Розуміння REST |
| **Бонус** | +1.0 | OpenAPI або розширення |

### Як отримати максимум:

✅ **0.5 за аналіз:**
- Task та User описані з атрибутами

✅ **1.5 за endpoints:**
- Мінімум 7 правильних endpoints
- Правильні HTTP методи
- REST URI (іменники, множина)
- Правильні статус-коди

✅ **1.0 за структури:**
- Task Response
- CreateTaskDTO та UpdateTaskDTO (різні!)
- User Response
- Error Response

✅ **0.5 за errors:**
- Стандартна структура
- Мінімум 5 error codes
- 2 приклади

✅ **0.5 за питання:**
- Обґрунтовані відповіді

---

## Контрольні питання

1. У чому різниця між PUT та PATCH?
2. Чому важливо розділяти CreateDTO і UpdateDTO?
3. Який статус-код для успішного DELETE: 200 чи 204?
4. Як правильно називати URI: іменники чи дієслова?
5. Що означає "ідемпотентний" метод?
6. Коли використовувати 404 vs 400?

---

## Поширені помилки

❌ **Не робіть так:**

1. Дієслова в URI:
   ```
   ❌ POST /api/createTask
   ✅ POST /api/tasks
   ```

2. Неправильні статус-коди:
   ```
   ❌ 200 OK при створенні
   ✅ 201 Created
   ```

3. Однина для колекцій:
   ```
   ❌ /api/task
   ✅ /api/tasks
   ```

4. Те саме DTO для всього:
   ```
   ❌ Один TaskDTO
   ✅ CreateTaskDTO, UpdateTaskDTO, TaskResponse
   ```

✅ **Робіть так:**

- Іменники у множині: `/api/tasks`
- Правильні HTTP методи для операцій
- Різні DTO для create/update/response
- Стандартизовані помилки з codes
- ISO 8601 для дат: `2025-10-21T10:30:00Z`

---

## Приклад виконання (фрагмент)

### 1. Аналіз предметної області

| Ресурс | Атрибути |
|--------|----------|
| Task | id (UUID), title (string), description (string), status (enum), priority (enum), createdAt (datetime) |
| User | id (UUID), name (string), email (string) |

### 2. API Endpoints

| HTTP Method | URI Path | Опис | Status Codes |
|-------------|----------|------|--------------|
| GET | /api/tasks | Отримати список всіх задач | 200, 401 |
| GET | /api/tasks?status=done | Фільтр за статусом | 200, 401 |
| GET | /api/tasks/{id} | Отримати задачу за ID | 200, 404, 401 |
| POST | /api/tasks | Створити нову задачу | 201, 400, 401 |
| PUT | /api/tasks/{id} | Повністю оновити задачу | 200, 400, 404, 401 |
| PATCH | /api/tasks/{id} | Частково оновити задачу | 200, 400, 404, 401 |
| DELETE | /api/tasks/{id} | Видалити задачу | 204, 404, 401 |
| GET | /api/users | Отримати список користувачів | 200, 401 |
| GET | /api/users/{id} | Отримати користувача за ID | 200, 404, 401 |

**Пояснення:** Використовуємо REST conventions - іменники у множині, стандартні HTTP методи. GET для читання, POST для створення, PUT для повного оновлення, DELETE для видалення. Фільтрація через query parameters.

---

## Додаткові ресурси

1. **RFC 7231** - HTTP/1.1 Semantics
2. **REST API Tutorial** - https://restfulapi.net/
3. **OpenAPI Specification** - https://swagger.io/specification/
4. **Richardson Maturity Model** - https://martinfowler.com/articles/richardsonMaturityModel.html

---

**Успіхів у виконанні лабораторної роботи!**

Час на проєктування - це інвестиція у майбутнє проєкту.
