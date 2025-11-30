# Лабораторна робота 2. Створення базового FastAPI проєкту

## Мета

Набути практичних навичок створення RESTful API з використанням FastAPI, освоїти основи валідації даних через Pydantic models, навчитися працювати з різними типами параметрів запитів та використовувати автоматичну документацію Swagger UI для тестування API.

## Завдання

Розробити повноцінний CRUD API для управління завданнями (Tasks) з наступною функціональністю:

- Створення нового завдання (CREATE)
- Отримання списку всіх завдань з можливістю фільтрації (READ)
- Отримання конкретного завдання за ідентифікатором (READ)
- Оновлення існуючого завдання (UPDATE)
- Видалення завдання (DELETE)
- Валідація вхідних даних через Pydantic models
- Інтерактивне тестування через Swagger UI

## Теоретичні відомості

### FastAPI основи

FastAPI є сучасним вебфреймворком для створення API з Python 3.7+ на основі стандартних Python type hints. Фреймворк забезпечує автоматичну валідацію даних, серіалізацію та генерацію документації.

Ключовою особливістю FastAPI є використання асинхронного програмування, що дозволяє обробляти велику кількість одночасних запитів ефективніше порівняно з традиційними синхронними фреймворками. Кожна функція-обробник може бути оголошена як async def, що дозволяє використовувати await для неблокуючих операцій.

### RESTful API принципи

REST (Representational State Transfer) є архітектурним стилем для розподілених систем. RESTful API використовує HTTP методи для виконання операцій над ресурсами:

GET використовується для отримання даних без внесення змін. POST створює нові ресурси. PUT оновлює існуючі ресурси повністю. PATCH оновлює частину ресурсу. DELETE видаляє ресурси.

Кожен ресурс ідентифікується унікальною URL-адресою. Відповіді мають відповідні HTTP статус коди: 200 для успішних операцій, 201 для створення ресурсу, 404 для відсутнього ресурсу, 422 для помилок валідації.

### Pydantic для валідації даних

Pydantic використовує Python type hints для валідації даних у runtime. При створенні екземпляра Pydantic моделі автоматично перевіряються типи всіх полів, застосовуються значення за замовчуванням та виконуються кастомні валідатори.

BaseModel є базовим класом для всіх Pydantic моделей. Класи, що наслідуються від BaseModel, автоматично отримують можливості валідації, серіалізації в JSON та десеріалізації з JSON.

### Path, Query та Body Parameters

FastAPI розпізнає типи параметрів автоматично на основі їх використання у функції. Path parameters є частиною URL шляху та оголошуються у фігурних дужках. Query parameters додаються після знаку питання в URL. Body parameters передаються у тілі HTTP запиту та автоматично парсяться з JSON.

### HTTP статус коди

Правильне використання HTTP статус кодів є важливою частиною проектування API. Код 200 означає успішне виконання запиту. Код 201 вказує на успішне створення ресурсу. Код 204 використовується для успішних операцій без повернення контенту. Код 400 вказує на невалідний запит. Код 404 означає, що ресурс не знайдено. Код 422 використовується для помилок валідації даних.

## Хід роботи

### Крок 1. Підготовка середовища розробки

Створіть нову папку для проєкту та відкрийте її у Visual Studio Code або іншому редакторі.

```bash
mkdir fastapi-tasks
cd fastapi-tasks
```

Створіть віртуальне середовище Python для ізоляції залежностей проєкту.

```bash
python -m venv venv
```

Активуйте віртуальне середовище. Для Windows використовуйте команду:

```bash
venv\Scripts\activate
```

Для Linux або macOS:

```bash
source venv/bin/activate
```

Встановіть необхідні бібліотеки FastAPI та Uvicorn.

```bash
pip install fastapi uvicorn[standard]
```

Створіть файл requirements.txt для збереження залежностей.

```bash
pip freeze > requirements.txt
```

### Крок 2. Створення структури проєкту

Створіть наступну структуру файлів та папок:

```
fastapi-tasks/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── models.py
│   └── schemas.py
├── venv/
├── requirements.txt
└── README.md
```

Файл __init__.py робить папку app Python пакетом. Файл main.py міститиме основний код застосунку. Файл models.py буде містити тимчасове сховище даних. Файл schemas.py міститиме Pydantic схеми.

### Крок 3. Створення Pydantic схем

Відкрийте файл app/schemas.py та створіть схеми для валідації даних завдань.

```python
from pydantic import BaseModel, Field
from typing import Optional

class TaskBase(BaseModel):
    title: str = Field(..., min_length=1, max_length=100, description="Назва завдання")
    description: str = Field(..., min_length=1, max_length=500, description="Опис завдання")
    completed: bool = Field(default=False, description="Статус виконання")

class TaskCreate(TaskBase):
    pass

class TaskUpdate(BaseModel):
    title: Optional[str] = Field(None, min_length=1, max_length=100)
    description: Optional[str] = Field(None, min_length=1, max_length=500)
    completed: Optional[bool] = None

class Task(TaskBase):
    id: int = Field(..., description="Унікальний ідентифікатор")

    class Config:
        json_schema_extra = {
            "example": {
                "id": 1,
                "title": "Вивчити FastAPI",
                "description": "Пройти базовий курс FastAPI",
                "completed": False
            }
        }
```

TaskBase містить спільні поля для всіх схем. TaskCreate використовується при створенні нових завдань. TaskUpdate дозволяє оновлювати окремі поля, всі поля опціональні. Task представляє повну модель з ідентифікатором для відповідей API.

### Крок 4. Створення тимчасового сховища даних

Відкрийте файл app/models.py та створіть тимчасове сховище для завдань.

```python
from typing import Dict
from app.schemas import Task

class TaskDatabase:
    def __init__(self):
        self.tasks: Dict[int, Task] = {}
        self.current_id: int = 1

    def create(self, title: str, description: str, completed: bool = False) -> Task:
        task = Task(
            id=self.current_id,
            title=title,
            description=description,
            completed=completed
        )
        self.tasks[self.current_id] = task
        self.current_id += 1
        return task

    def get(self, task_id: int) -> Task | None:
        return self.tasks.get(task_id)

    def get_all(self, skip: int = 0, limit: int = 100, completed: bool | None = None) -> list[Task]:
        tasks = list(self.tasks.values())

        if completed is not None:
            tasks = [task for task in tasks if task.completed == completed]

        return tasks[skip:skip + limit]

    def update(self, task_id: int, title: str | None = None,
               description: str | None = None, completed: bool | None = None) -> Task | None:
        task = self.tasks.get(task_id)
        if not task:
            return None

        if title is not None:
            task.title = title
        if description is not None:
            task.description = description
        if completed is not None:
            task.completed = completed

        return task

    def delete(self, task_id: int) -> bool:
        if task_id in self.tasks:
            del self.tasks[task_id]
            return True
        return False

db = TaskDatabase()
```

Клас TaskDatabase інкапсулює логіку роботи зі сховищем. Метод create додає нове завдання та повертає його. Метод get повертає завдання за ідентифікатором або None. Метод get_all повертає список завдань з підтримкою пагінації та фільтрації. Метод update оновлює поля завдання. Метод delete видаляє завдання.

### Крок 5. Створення API ендпоінтів

Відкрийте файл app/main.py та створіть FastAPI застосунок з усіма необхідними ендпоінтами.

```python
from fastapi import FastAPI, HTTPException, status
from app.schemas import Task, TaskCreate, TaskUpdate
from app.models import db

app = FastAPI(
    title="Task Manager API",
    description="API для управління завданнями",
    version="1.0.0"
)

@app.get("/", tags=["Root"])
async def root():
    return {
        "message": "Ласкаво просимо до Task Manager API",
        "docs": "/docs",
        "redoc": "/redoc"
    }

@app.post(
    "/tasks/",
    response_model=Task,
    status_code=status.HTTP_201_CREATED,
    tags=["Tasks"],
    summary="Створити нове завдання",
    description="Створює нове завдання з вказаними параметрами"
)
async def create_task(task: TaskCreate):
    new_task = db.create(
        title=task.title,
        description=task.description,
        completed=task.completed
    )
    return new_task

@app.get(
    "/tasks/",
    response_model=list[Task],
    tags=["Tasks"],
    summary="Отримати список завдань",
    description="Повертає список всіх завдань з можливістю фільтрації та пагінації"
)
async def get_tasks(
    skip: int = 0,
    limit: int = 100,
    completed: bool | None = None
):
    tasks = db.get_all(skip=skip, limit=limit, completed=completed)
    return tasks

@app.get(
    "/tasks/{task_id}",
    response_model=Task,
    tags=["Tasks"],
    summary="Отримати завдання за ID",
    description="Повертає конкретне завдання за його ідентифікатором"
)
async def get_task(task_id: int):
    task = db.get(task_id)
    if not task:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Завдання з ID {task_id} не знайдено"
        )
    return task

@app.put(
    "/tasks/{task_id}",
    response_model=Task,
    tags=["Tasks"],
    summary="Оновити завдання",
    description="Оновлює існуюче завдання повністю або частково"
)
async def update_task(task_id: int, task_update: TaskUpdate):
    task = db.update(
        task_id=task_id,
        title=task_update.title,
        description=task_update.description,
        completed=task_update.completed
    )
    if not task:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Завдання з ID {task_id} не знайдено"
        )
    return task

@app.delete(
    "/tasks/{task_id}",
    status_code=status.HTTP_204_NO_CONTENT,
    tags=["Tasks"],
    summary="Видалити завдання",
    description="Видаляє завдання за його ідентифікатором"
)
async def delete_task(task_id: int):
    success = db.delete(task_id)
    if not success:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Завдання з ID {task_id} не знайдено"
        )
    return None
```

Кожен ендпоінт має детальний опис для автоматичної документації. Використовуються правильні HTTP статус коди для різних ситуацій. HTTPException використовується для повернення помилок з відповідними кодами.

### Крок 6. Запуск застосунку

Запустіть FastAPI застосунок через Uvicorn.

```bash
uvicorn app.main:app --reload
```

Параметр --reload забезпечує автоматичне перезавантаження при зміні коду. Застосунок буде доступний за адресою http://localhost:8000.

### Крок 7. Тестування через Swagger UI

Відкрийте браузер та перейдіть за адресою http://localhost:8000/docs. Ви побачите автоматично згенеровану інтерактивну документацію Swagger UI.

Протестуйте створення завдання через POST /tasks/. Натисніть на ендпоінт, потім на кнопку Try it out. Введіть JSON з даними завдання:

```json
{
  "title": "Вивчити FastAPI",
  "description": "Пройти базовий курс та створити перший проєкт",
  "completed": false
}
```

Натисніть Execute та перевірте відповідь. Ви маєте отримати створене завдання з присвоєним ідентифікатором.

Протестуйте отримання списку завдань через GET /tasks/. Створіть кілька завдань та отримайте їх список. Спробуйте використати параметри фільтрації completed=true або completed=false.

Протестуйте отримання конкретного завдання через GET /tasks/{task_id}. Введіть ідентифікатор існуючого завдання та перевірте відповідь. Спробуйте ввести неіснуючий ідентифікатор та перевірте помилку 404.

Протестуйте оновлення завдання через PUT /tasks/{task_id}. Оновіть назву або статус виконання існуючого завдання. Перевірте, що зміни застосувалися, отримавши завдання знову.

Протестуйте видалення завдання через DELETE /tasks/{task_id}. Видаліть одне з завдань та перевірте, що воно більше не доступне через GET запит.

### Крок 8. Додаткові покращення

Додайте валідацію для пагінації, щоб уникнути від'ємних значень.

```python
from fastapi import Query

@app.get("/tasks/", response_model=list[Task], tags=["Tasks"])
async def get_tasks(
    skip: int = Query(default=0, ge=0, description="Кількість завдань для пропуску"),
    limit: int = Query(default=100, ge=1, le=100, description="Максимальна кількість завдань"),
    completed: bool | None = Query(default=None, description="Фільтр за статусом виконання")
):
    tasks = db.get_all(skip=skip, limit=limit, completed=completed)
    return tasks
```

Додайте ендпоінт для отримання статистики.

```python
@app.get("/tasks/stats/summary", tags=["Statistics"])
async def get_statistics():
    all_tasks = db.get_all()
    completed_tasks = [task for task in all_tasks if task.completed]

    return {
        "total": len(all_tasks),
        "completed": len(completed_tasks),
        "pending": len(all_tasks) - len(completed_tasks),
        "completion_rate": len(completed_tasks) / len(all_tasks) * 100 if all_tasks else 0
    }
```



### Крок 9. Створення звіту README.md

Створіть файл README.md у кореневій папці проєкту. Цей файл є вашим звітом по лабораторній роботі і має містити:

Обов'язкові розділи звіту:

- Опис проєкту - коротко опишіть, що робить ваш API
- Функціональність - перелічіть реалізовані можливості
- Технології - вкажіть використані технології та бібліотеки
- Структура проєкту - опишіть призначення кожного файлу
- Встановлення та запуск - детальні інструкції для запуску проєкту
- API Endpoints - таблиця або список всіх ендпоінтів з описом
- Приклади використання - скріншоти з Swagger UI або приклади запитів
- Висновки - що було реалізовано, які труднощі виникли, що навчилися

Приклад структури README.md:

```markdown
    # Task Manager API

    REST API для управління завданнями, створений з використанням FastAPI.

    ## Опис проєкту

    Цей проєкт реалізує повноцінний CRUD API для управління списком завдань.
    API дозволяє створювати, переглядати, оновлювати та видаляти завдання,
    а також фільтрувати їх за статусом виконання.

    ## Функціональність

    - Створення нових завдань
    - Перегляд списку завдань з фільтрацією та пагінацією
    - Отримання конкретного завдання за ID
    - Оновлення завдань
    - Видалення завдань
    - Валідація вхідних даних
    - Автоматична документація API
    - Статистика виконання завдань

    ## Технології

    - Python 3.11+
    - FastAPI - вебфреймворк для створення API
    - Pydantic - валідація даних
    - Uvicorn - ASGI сервер


    ## Встановлення

    1. Клонуйте репозиторій:
    ```bash
    git clone <посилання-на-ваш-репозиторій>
    cd fastapi-tasks
    ```

    2. Створіть віртуальне середовище:
    ```bash
    python -m venv venv
    ```

    3. Активуйте віртуальне середовище:
    - Windows: `venv\Scripts\activate`
    - Linux/Mac: `source venv/bin/activate`

    4. Встановіть залежності:
    ```bash
    pip install -r requirements.txt
    ```

    ## Запуск
    ```bash
    uvicorn app.main:app --reload
    ```

    Застосунок буде доступний за адресою: http://localhost:8000

    ## Документація API

    - Swagger UI: http://localhost:8000/docs
    - ReDoc: http://localhost:8000/redoc

    ## API Endpoints

    | Метод | Endpoint | Опис |
    |-------|----------|------|
    | GET | / | Коренева сторінка з інформацією про API |
    | POST | /tasks/ | Створити нове завдання |
    | GET | /tasks/ | Отримати список завдань (з фільтрацією) |
    | GET | /tasks/{task_id} | Отримати завдання за ID |
    | PUT | /tasks/{task_id} | Оновити завдання |
    | DELETE | /tasks/{task_id} | Видалити завдання |
    | GET | /tasks/stats/summary | Отримати статистику завдань |

    ### Приклади використання

    #### Створення завдання
    ```json
    POST /tasks/
    {
      "title": "Вивчити FastAPI",
      "description": "Пройти базовий курс",
      "completed": false
    }
    ```

    #### Фільтрація завдань
    ```
    GET /tasks/?completed=true&skip=0&limit=10
    ```

    ## Скріншоти

    [Тут додайте скріншоти з Swagger UI, які демонструють роботу вашого API]

    1. Створення завдання (POST /tasks/)
    2. Отримання списку завдань (GET /tasks/)
    3. Отримання завдання за ID (GET /tasks/{task_id})
    4. Оновлення завдання (PUT /tasks/{task_id})
    5. Видалення завдання (DELETE /tasks/{task_id})
    6. Помилка 404 при запиті неіснуючого завдання
    7. Статистика завдань (GET /tasks/stats/summary)

    ## Реалізовані особливості

    - **Валідація даних**: використано Pydantic Field з обмеженнями min_length, max_length
    - **Пагінація**: параметри skip та limit для GET /tasks/
    - **Фільтрація**: можливість фільтрувати за статусом completed
    - **Документація**: детальні описи всіх ендпоінтів
    - **Обробка помилок**: коректні HTTP статус коди та повідомлення

```

Зверніть увагу, що README.md є основним звітом по лабораторній роботі і має містити всю необхідну інформацію про ваш проєкт.


### Крок 10. Підготовка до здачі

Створіть GitHub репозиторій та завантажте код проєкту. Переконайтеся, що файл .gitignore містить `venv/` та `__pycache__/`.

```
venv/
__pycache__/
*.pyc
.env
.DS_Store
```



Зробіть скріншоти Swagger UI з результатами тестування всіх ендпоінтів. Збережіть їх окремо у папці screenshots/.


## Критерії оцінювання

Максимальна кількість балів: 7 балів.

Оцінювання здійснюється за накопичувальною системою з кроком 0.5 або 1 бал.

### 1 бал - Базове налаштування

- Створено проєкт з правильною структурою (app/, schemas.py, models.py, main.py)
- Встановлено FastAPI та Uvicorn
- Створено Pydantic schemas (TaskBase, TaskCreate, Task)
- Застосунок запускається без помилок
- Реалізовано мінімум 1-2 ендпоінти (наприклад, POST та GET list)

### 2-3 бали - Базовий CRUD (частковий)

**Все з попереднього рівня +**

- Реалізовано 3-4 CRUD ендпоінти:
  - POST /tasks/ (створення)
  - GET /tasks/ (список)
  - GET /tasks/{task_id} (отримання за ID)
  - PUT /tasks/{task_id} АБО DELETE /tasks/{task_id}
- Базова валідація через Pydantic
- Обробка помилки 404 для неіснуючих ресурсів

### 4-5 балів - Повний CRUD

**Все з попереднього рівня +**

- Реалізовано всі 5 CRUD ендпоінтів:
  - POST /tasks/ (створення)
  - GET /tasks/ (список)
  - GET /tasks/{task_id} (отримання за ID)
  - PUT /tasks/{task_id} (оновлення)
  - DELETE /tasks/{task_id} (видалення)
- Правильні HTTP статус коди (201, 200, 204, 404)
- Створено TaskUpdate schema для оновлення

### 6 балів - CRUD + Розширений функціонал

**Все з попереднього рівня +**

- Валідація з використанням Field (min_length, max_length, ge, le)
- Query parameters для фільтрації та пагінації (skip, limit, completed)
- Документація в коді (summary, description для ендпоінтів)
- Додано title та description до FastAPI app

### 7 балів - Повна реалізація

**Все з попереднього рівня +**

- Реалізовано додатковий ендпоінт (наприклад, статистика /tasks/stats/summary)
- GitHub репозиторій з правильно налаштованим .gitignore
- Скріншоти тестування через Swagger UI (мінімум 5 штук)
- Детальні описи та приклади у Config для Pydantic models

### Швидка довідка по балах

| Бали | Що реалізовано |
|------|----------------|
| 1 | Проєкт запускається, є 1-2 ендпоінти |
| 2-3 | 3-4 CRUD ендпоінти працюють |
| 4-5 | Всі 5 CRUD ендпоінтів працюють |
| 6 | CRUD + валідація + фільтрація + документація |
| 7 | Все + додатковий функціонал + README + GitHub + скріншоти |


## Контрольні запитання

1. Які основні переваги FastAPI порівняно з Flask та Django для створення API?
2. Як FastAPI використовує Python type hints для валідації даних?
3. Яка різниця між path parameters, query parameters та request body у FastAPI?
4. Що таке Pydantic BaseModel та які його основні можливості?
5. Які HTTP методи використовуються для операцій CRUD та які статус коди їм відповідають?
6. Як працює автоматична генерація документації в FastAPI?
7. Чому важливо використовувати правильні HTTP статус коди у відповідях API?
8. Як можна додати валідацію з обмеженнями на довжину рядків та діапазон чисел у Pydantic?
9. Що таке асинхронні функції (async def) та чим вони відрізняються від звичайних?
10. Як організувати структуру проєкту FastAPI для зручності підтримки та масштабування?
