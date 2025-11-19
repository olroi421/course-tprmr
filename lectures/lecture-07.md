# Лекція 7. Message Queues, Celery та мікросервіси

## Вступ

Сучасні вебдодатки часто потребують виконання довготривалих або ресурсоємних операцій, які не можна виконувати безпосередньо під час обробки HTTP-запиту. Також складні системи часто розділяються на окремі незалежні сервіси, які потребують ефективної комунікації між собою. У цій лекції ми розглянемо підходи до асинхронної обробки задач через черги повідомлень, роботу з Celery для фонових задач та архітектурні принципи побудови мікросервісних систем.

## 1. Background tasks у FastAPI: можливості та обмеження

FastAPI надає вбудований механізм для виконання фонових задач через клас `BackgroundTasks`. Цей механізм дозволяє виконувати певні операції після відправки відповіді клієнту, не змушуючи його чекати на завершення цих операцій.

```python
from fastapi import BackgroundTasks, FastAPI

app = FastAPI()

def write_log(message: str):
    with open("log.txt", mode="a") as log_file:
        log_file.write(f"{message}\n")

@app.post("/send-notification/")
async def send_notification(
    email: str,
    background_tasks: BackgroundTasks
):
    background_tasks.add_task(write_log, f"Notification sent to {email}")
    return {"message": "Notification sent"}
```

Вбудовані фонові задачі FastAPI мають свої переваги та обмеження. До переваг належить простота використання, відсутність потреби в додатковій інфраструктурі та автоматична інтеграція з життєвим циклом запиту. Однак існують істотні обмеження, які роблять цей механізм непридатним для складних сценаріїв.

По-перше, фонові задачі виконуються в тому самому процесі, що й основний додаток. Це означає, що вони споживають ресурси сервера та можуть впливати на продуктивність обробки інших запитів. По-друге, якщо сервер перезапускається або падає, всі невиконані фонові задачі втрачаються, оскільки немає механізму персистентності. По-третє, відсутня можливість масштабування, адже неможливо розподілити виконання задач між кількома воркерами або серверами.

Черезці обмеження вбудовані фонові задачі FastAPI підходять лише для простих операцій, таких як логування, відправка простих повідомлень або очищення тимчасових файлів. Для складніших сценаріїв потрібні спеціалізовані системи черг повідомлень.

## 2. Концепція черг повідомлень

Черги повідомлень є фундаментальним компонентом розподілених систем, що забезпечує асинхронну комунікацію між різними частинами додатку. Основна ідея полягає у тому, що відправник розміщує повідомлення в черзі, а один або кілька обробників забирають ці повідомлення та виконують необхідні операції.

Архітектура системи з чергою повідомлень складається з кількох ключових компонентів. Продюсер (producer) створює та відправляє повідомлення до черги. Брокер повідомлень (message broker) зберігає повідомлення та забезпечує їх доставку до консьюмерів. Консьюмер (consumer) забирає повідомлення з черги та обробляє їх. Саме повідомлення містить дані, необхідні для виконання задачі.

```mermaid
graph LR
    A[Продюсер 1] -->|Відправка| B[Message Broker]
    C[Продюсер 2] -->|Відправка| B
    B -->|Черга 1| D[Консьюмер 1]
    B -->|Черга 2| E[Консьюмер 2]
    B -->|Черга 2| F[Консьюмер 3]
```

Існує кілька основних патернів роботи з чергами повідомлень. Патерн точка-точка (point-to-point) означає, що кожне повідомлення обробляється лише одним консьюмером. Патерн публікація-підписка (publish-subscribe) дозволяє одному повідомленню бути доставленим кільком консьюмерам одночасно. Патерн топіків (topics) надає можливість консьюмерам підписуватися на повідомлення певних категорій.

Використання черг повідомлень надає системі низку переваг. По-перше, відбувається розв'язка компонентів системи, оскільки продюсери та консьюмери не знають один про одного безпосередньо. По-друге, забезпечується надійність через персистентність повідомлень та можливість повторної обробки. По-третє, система стає масштабованою, адже можна додавати консьюмерів для збільшення пропускної здатності. По-четверте, досягається збалансування навантаження між кількома обробниками.

## 3. Redis як message broker

Redis є популярним інструментом, який може виконувати роль брокера повідомлень завдяки своїм структурам даних та функціям публікації-підписки. Розглянемо основні способи використання Redis для організації черг повідомлень.

### 3.1. Redis Lists для черг

Списки Redis можуть використовуватися як прості черги повідомлень. Операції `LPUSH` та `RPOP` дозволяють реалізувати класичну чергу FIFO (First In, First Out).

```python
import redis
import json
from typing import Dict, Any

redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)

def enqueue_task(queue_name: str, task_data: Dict[str, Any]):
    """Додає задачу до черги."""
    task_json = json.dumps(task_data)
    redis_client.lpush(queue_name, task_json)

def dequeue_task(queue_name: str, timeout: int = 0) -> Dict[str, Any]:
    """Забирає задачу з черги з можливістю блокування."""
    result = redis_client.brpop(queue_name, timeout=timeout)
    if result:
        queue, task_json = result
        return json.loads(task_json)
    return None

# Використання
enqueue_task("email_queue", {
    "type": "email",
    "to": "user@example.com",
    "subject": "Welcome",
    "body": "Thank you for registration"
})

# Обробник задач
while True:
    task = dequeue_task("email_queue", timeout=5)
    if task:
        process_email(task)
```

Операція `BRPOP` є блокуючою версією `RPOP`, яка чекає на появу елементів у черзі протягом заданого таймауту. Це дозволяє уникнути активного опитування черги та зменшити навантаження на Redis.

### 3.2. Redis Pub/Sub

Механізм публікації-підписки Redis дозволяє реалізувати патерн, коли одне повідомлення може бути доставлене кільком підписникам одночасно. Це корисно для сценаріїв, коли потрібно сповістити кілька сервісів про певну подію.

```python
import aioredis
import asyncio
import json

async def publisher():
    redis = await aioredis.from_url("redis://localhost")

    event_data = {
        "event": "user_registered",
        "user_id": 12345,
        "timestamp": "2024-01-15T10:30:00"
    }

    await redis.publish("events", json.dumps(event_data))
    await redis.close()

async def subscriber():
    redis = await aioredis.from_url("redis://localhost")
    pubsub = redis.pubsub()

    await pubsub.subscribe("events")

    async for message in pubsub.listen():
        if message["type"] == "message":
            event_data = json.loads(message["data"])
            print(f"Отримано подію: {event_data}")
            # Обробка події
            await process_event(event_data)

    await redis.close()
```

Важливо розуміти, що Redis Pub/Sub не гарантує доставку повідомлень. Якщо підписник не активний на момент публікації повідомлення, він його не отримає. Також повідомлення не зберігаються та не можуть бути перечитані пізніше. Ці особливості роблять Pub/Sub придатним для сценаріїв реального часу, де втрата окремих повідомлень допустима.

### 3.3. Redis Streams

Redis Streams є більш досконалим механізмом для роботи з потоками подій. На відміну від Pub/Sub, Streams зберігають повідомлення, підтримують групи консьюмерів та забезпечують гарантії доставки.

```python
import redis
import time

redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)

def add_to_stream(stream_name: str, data: dict):
    """Додає повідомлення до потоку."""
    redis_client.xadd(stream_name, data)

def create_consumer_group(stream_name: str, group_name: str):
    """Створює групу консьюмерів."""
    try:
        redis_client.xgroup_create(stream_name, group_name, id='0', mkstream=True)
    except redis.ResponseError:
        pass  # Група вже існує

def consume_from_stream(stream_name: str, group_name: str, consumer_name: str):
    """Читає повідомлення з потоку як член групи консьюмерів."""
    while True:
        messages = redis_client.xreadgroup(
            group_name,
            consumer_name,
            {stream_name: '>'},
            count=10,
            block=5000
        )

        if messages:
            for stream, msg_list in messages:
                for msg_id, msg_data in msg_list:
                    try:
                        process_message(msg_data)
                        redis_client.xack(stream_name, group_name, msg_id)
                    except Exception as e:
                        print(f"Помилка обробки: {e}")
```

Redis Streams підтримують групи консьюмерів, що дозволяє розподіляти обробку повідомлень між кількома воркерами. Кожне повідомлення обробляється лише одним консьюмером у групі. Механізм підтвердження (acknowledgment) через команду `XACK` забезпечує надійну обробку повідомлень, адже неопрацьовані повідомлення можуть бути повторно призначені іншому консьюмеру.

## 4. Celery для фонових задач

Celery є потужною системою розподіленої обробки задач для Python. Вона підтримує різні брокери повідомлень, забезпечує надійне виконання задач та надає багато можливостей для моніторингу та управління.

### 4.1. Базова конфігурація Celery

Для роботи з Celery потрібно налаштувати додаток, визначити задачі та запустити воркери.

```python
# celery_app.py
from celery import Celery

celery_app = Celery(
    "tasks",
    broker="redis://localhost:6379/0",
    backend="redis://localhost:6379/1"
)

celery_app.conf.update(
    task_serializer='json',
    accept_content=['json'],
    result_serializer='json',
    timezone='Europe/Kiev',
    enable_utc=True,
    task_track_started=True,
    task_time_limit=3600,
    worker_prefetch_multiplier=4,
)

@celery_app.task(bind=True, max_retries=3)
def send_email(self, email: str, subject: str, body: str):
    """Відправка email з повторними спробами при помилці."""
    try:
        # Логіка відправки email
        email_service.send(email, subject, body)
        return {"status": "sent", "email": email}
    except Exception as exc:
        # Повторна спроба через 60 секунд
        raise self.retry(exc=exc, countdown=60)
```

Параметр `bind=True` дозволяє отримати доступ до самої задачі через аргумент `self`, що необхідно для використання функції `retry`. Параметр `max_retries` визначає максимальну кількість повторних спроб виконання задачі при помилці.

### 4.2. Інтеграція Celery з FastAPI

Для використання Celery у FastAPI додатку потрібно правильно організувати виклик задач та перевірку їх статусу.

```python
# main.py
from fastapi import FastAPI, BackgroundTasks, HTTPException
from celery.result import AsyncResult
from celery_app import celery_app, send_email
from pydantic import BaseModel, EmailStr

app = FastAPI()

class EmailRequest(BaseModel):
    email: EmailStr
    subject: str
    body: str

@app.post("/send-email/")
async def create_email_task(email_request: EmailRequest):
    """Створює задачу на відправку email."""
    task = send_email.delay(
        email_request.email,
        email_request.subject,
        email_request.body
    )
    return {
        "task_id": task.id,
        "status": "queued"
    }

@app.get("/tasks/{task_id}")
async def get_task_status(task_id: str):
    """Перевіряє статус задачі."""
    task_result = AsyncResult(task_id, app=celery_app)

    if task_result.ready():
        if task_result.successful():
            return {
                "task_id": task_id,
                "status": "completed",
                "result": task_result.result
            }
        else:
            return {
                "task_id": task_id,
                "status": "failed",
                "error": str(task_result.info)
            }
    else:
        return {
            "task_id": task_id,
            "status": "pending"
        }
```

Метод `delay()` є скороченим варіантом методу `apply_async()` та використовується для асинхронного виклику задачі. Він повертає об'єкт `AsyncResult`, який містить ідентифікатор задачі та дозволяє перевірити її статус.

### 4.3. Періодичні задачі з Celery Beat

Celery Beat є планувальником задач, що дозволяє виконувати задачі за розкладом, подібно до cron у Unix-системах.

```python
# celery_app.py (продовження)
from celery.schedules import crontab

celery_app.conf.beat_schedule = {
    'cleanup-old-files-daily': {
        'task': 'tasks.cleanup_old_files',
        'schedule': crontab(hour=2, minute=0),
    },
    'send-report-weekly': {
        'task': 'tasks.send_weekly_report',
        'schedule': crontab(day_of_week=1, hour=9, minute=0),
    },
    'health-check-every-minute': {
        'task': 'tasks.health_check',
        'schedule': 60.0,  # кожні 60 секунд
    },
}

@celery_app.task
def cleanup_old_files():
    """Очищення старих файлів."""
    deleted_count = file_service.cleanup_old_files(days=30)
    return {"deleted": deleted_count}

@celery_app.task
def send_weekly_report():
    """Відправка тижневого звіту."""
    report_data = analytics_service.generate_weekly_report()
    email_service.send_report(report_data)
    return {"status": "sent"}
```

Для запуску Celery Beat потрібна окрема команда. Воркери Celery виконують задачі, а Beat планує їх запуск.

```bash
# Запуск воркера
celery -A celery_app worker --loglevel=info

# Запуск beat scheduler
celery -A celery_app beat --loglevel=info
```

### 4.4. Моніторинг задач з Flower

Flower є веббасованим інструментом для моніторингу та адміністрування Celery. Він надає інтерфейс для перегляду статусу задач, воркерів та черг.

```bash
# Встановлення
pip install flower

# Запуск
celery -A celery_app flower --port=5555
```

Після запуску Flower доступний за адресою `http://localhost:5555`, де можна переглядати активні та завершені задачі, статистику виконання, інформацію про воркери та керувати чергами.

## 5. Архітектура мікросервісів

Мікросервісна архітектура є підходом до розробки додатків, при якому великий монолітний додаток розділяється на набір невеликих незалежних сервісів, кожен з яких виконує специфічну бізнес-функцію.

### 5.1. Принципи мікросервісів

Кожен мікросервіс має свою чітко визначену відповідальність та домен. Сервіси є автономними та можуть розгортатися незалежно один від одного. Кожен сервіс має власну базу даних, що забезпечує повну незалежність даних. Комунікація між сервісами відбувається через чітко визначені інтерфейси, найчастіше через HTTP API або черги повідомлень.

```mermaid
graph TB
    subgraph "Клієнт"
        A[Web/Mobile App]
    end

    subgraph "API Gateway"
        B[Kong/Nginx]
    end

    subgraph "Мікросервіси"
        C[User Service]
        D[Order Service]
        E[Product Service]
        F[Payment Service]
    end

    subgraph "Бази даних"
        G[(Users DB)]
        H[(Orders DB)]
        I[(Products DB)]
        J[(Payments DB)]
    end

    subgraph "Message Queue"
        K[Redis/RabbitMQ]
    end

    A --> B
    B --> C
    B --> D
    B --> E
    B --> F
    C --> G
    D --> H
    E --> I
    F --> J
    D --> K
    F --> K
```

Переваги мікросервісної архітектури включають можливість незалежного масштабування окремих сервісів, використання різних технологій для різних сервісів, незалежне розгортання та оновлення, кращу відмовостійкість та простоту розуміння окремих компонентів. Однак є й недоліки, такі як складність розподіленої системи, необхідність забезпечення узгодженості даних, складніше тестування та відлагодження, додаткове навантаження на мережу та необхідність в інструментах для моніторингу.

### 5.2. Приклад структури мікросервісу на FastAPI

Кожен мікросервіс має власну структуру проєкту, подібну до монолітного додатку, але зосереджену на конкретній функціональності.

```python
# users_service/main.py
from fastapi import FastAPI, HTTPException, Depends
from sqlalchemy.ext.asyncio import AsyncSession
from typing import List
import httpx

app = FastAPI(title="Users Service")

@app.get("/health")
async def health_check():
    """Перевірка здоров'я сервісу."""
    return {"status": "healthy", "service": "users"}

@app.post("/users/", response_model=UserResponse)
async def create_user(
    user_data: UserCreate,
    db: AsyncSession = Depends(get_db)
):
    """Створення нового користувача."""
    user = await user_service.create_user(db, user_data)

    # Публікація події про створення користувача
    await event_publisher.publish("user.created", {
        "user_id": user.id,
        "email": user.email
    })

    return user

@app.get("/users/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: int,
    db: AsyncSession = Depends(get_db)
):
    """Отримання інформації про користувача."""
    user = await user_service.get_user(db, user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user
```

## 6. Комунікація між сервісами

Мікросервіси можуть взаємодіяти один з одним різними способами. Вибір способу комунікації залежить від конкретних вимог до системи.

### 6.1. Синхронна комунікація через HTTP

Синхронна комунікація означає, що сервіс-клієнт чекає на відповідь від сервісу-сервера. Це найпростіший спосіб, але він створює щільне зв'язування між сервісами.

```python
# orders_service/main.py
import httpx
from fastapi import FastAPI, HTTPException

app = FastAPI(title="Orders Service")

USER_SERVICE_URL = "http://users-service:8000"

async def verify_user(user_id: int) -> dict:
    """Перевірка існування користувача через User Service."""
    async with httpx.AsyncClient(timeout=5.0) as client:
        try:
            response = await client.get(f"{USER_SERVICE_URL}/users/{user_id}")
            response.raise_for_status()
            return response.json()
        except httpx.HTTPStatusError as e:
            if e.response.status_code == 404:
                raise HTTPException(status_code=404, detail="User not found")
            raise HTTPException(status_code=503, detail="User service unavailable")
        except httpx.RequestError:
            raise HTTPException(status_code=503, detail="User service unavailable")

@app.post("/orders/")
async def create_order(order_data: OrderCreate):
    """Створення нового замовлення."""
    # Перевірка існування користувача
    user = await verify_user(order_data.user_id)

    # Створення замовлення
    order = await order_service.create_order(order_data)

    return order
```

При синхронній комунікації важливо правильно обробляти помилки та таймаути. Якщо залежний сервіс недоступний, потрібно або повернути помилку клієнту, або використати fallback-логіку.

### 6.2. Асинхронна комунікація через черги повідомлень

Асинхронна комунікація через черги повідомлень дозволяє розв'язати сервіси та підвищити надійність системи. Сервіс-продюсер публікує подію, а сервіси-консьюмери незалежно її обробляють.

```python
# orders_service/events.py
import aioredis
import json
from typing import Dict, Any

class EventPublisher:
    def __init__(self, redis_url: str):
        self.redis_url = redis_url
        self.redis = None

    async def connect(self):
        self.redis = await aioredis.from_url(self.redis_url)

    async def publish(self, event_type: str, data: Dict[str, Any]):
        """Публікація події."""
        event = {
            "type": event_type,
            "data": data,
            "timestamp": datetime.utcnow().isoformat()
        }
        await self.redis.publish("events", json.dumps(event))

# orders_service/main.py
event_publisher = EventPublisher("redis://redis:6379")

@app.on_event("startup")
async def startup():
    await event_publisher.connect()

@app.post("/orders/")
async def create_order(order_data: OrderCreate):
    """Створення замовлення з публікацією події."""
    order = await order_service.create_order(order_data)

    # Публікація події - інші сервіси зможуть на неї відреагувати
    await event_publisher.publish("order.created", {
        "order_id": order.id,
        "user_id": order.user_id,
        "total_amount": float(order.total_amount)
    })

    return order
```

```python
# notifications_service/consumer.py
import aioredis
import json
import asyncio

async def event_consumer():
    """Консьюмер подій для сервісу повідомлень."""
    redis = await aioredis.from_url("redis://redis:6379")
    pubsub = redis.pubsub()
    await pubsub.subscribe("events")

    async for message in pubsub.listen():
        if message["type"] == "message":
            event = json.loads(message["data"])

            if event["type"] == "order.created":
                await send_order_confirmation(
                    event["data"]["user_id"],
                    event["data"]["order_id"]
                )
            elif event["type"] == "user.created":
                await send_welcome_email(event["data"]["email"])
```

Асинхронна комунікація через події особливо корисна, коли одна подія має викликати дії в кількох сервісах або коли немає необхідності в негайній відповіді.

## 7. API Gateway pattern

API Gateway є єдиною точкою входу для всіх клієнтських запитів до мікросервісної системи. Він виконує маршрутизацію запитів до відповідних сервісів, агрегацію відповідей та додаткові функції.

```mermaid
graph LR
    A[Клієнт] --> B[API Gateway]
    B --> C[Users Service]
    B --> D[Orders Service]
    B --> E[Products Service]

    subgraph "API Gateway функції"
        F[Маршрутизація]
        G[Автентифікація]
        H[Rate Limiting]
        I[Кешування]
    end
```

API Gateway надає низку переваг. По-перше, спрощується взаємодія клієнта з системою, адже він звертається до одного endpoint замість багатьох сервісів. По-друге, централізується автентифікація та авторизація. По-третє, можна реалізувати загальні функції, такі як rate limiting, логування та моніторинг. По-четверте, забезпечується додатковий рівень безпеки, приховуючи внутрішню структуру системи.

Для реалізації API Gateway можна використовувати готові рішення, такі як Kong, Ambassador або Traefik, або створити власний gateway на FastAPI.

```python
# api_gateway/main.py
from fastapi import FastAPI, HTTPException, Depends
import httpx
from typing import Optional

app = FastAPI(title="API Gateway")

SERVICES = {
    "users": "http://users-service:8000",
    "orders": "http://orders-service:8001",
    "products": "http://products-service:8002"
}

async def forward_request(
    service_name: str,
    path: str,
    method: str = "GET",
    **kwargs
) -> dict:
    """Переадресація запиту до відповідного сервісу."""
    if service_name not in SERVICES:
        raise HTTPException(status_code=404, detail="Service not found")

    url = f"{SERVICES[service_name]}{path}"

    async with httpx.AsyncClient(timeout=10.0) as client:
        try:
            if method == "GET":
                response = await client.get(url, **kwargs)
            elif method == "POST":
                response = await client.post(url, **kwargs)
            elif method == "PUT":
                response = await client.put(url, **kwargs)
            elif method == "DELETE":
                response = await client.delete(url, **kwargs)

            response.raise_for_status()
            return response.json()
        except httpx.RequestError:
            raise HTTPException(status_code=503, detail=f"{service_name} unavailable")

@app.get("/api/users/{user_id}")
async def get_user(user_id: int):
    """Проксування запиту до Users Service."""
    return await forward_request("users", f"/users/{user_id}")

@app.post("/api/orders/")
async def create_order(order_data: dict):
    """Проксування запиту до Orders Service."""
    return await forward_request(
        "orders",
        "/orders/",
        method="POST",
        json=order_data
    )
```

## 8. Service Discovery та Health Checks

У мікросервісній архітектурі важливо знати, які сервіси доступні та як до них підключитися. Service Discovery автоматизує процес виявлення сервісів та їх адрес.

### 8.1. Health Checks

Кожен сервіс має надавати endpoint для перевірки свого стану. Це дозволяє системам оркестрації та балансування навантаження визначати, чи готовий сервіс обробляти запити.

```python
# Простий health check
@app.get("/health")
async def health_check():
    return {"status": "healthy"}

# Розширений health check з перевіркою залежностей
@app.get("/health")
async def health_check(db: AsyncSession = Depends(get_db)):
    health_status = {
        "status": "healthy",
        "checks": {}
    }

    # Перевірка бази даних
    try:
        await db.execute("SELECT 1")
        health_status["checks"]["database"] = "healthy"
    except Exception as e:
        health_status["checks"]["database"] = "unhealthy"
        health_status["status"] = "unhealthy"

    # Перевірка Redis
    try:
        await redis_client.ping()
        health_status["checks"]["redis"] = "healthy"
    except Exception:
        health_status["checks"]["redis"] = "unhealthy"
        health_status["status"] = "degraded"

    return health_status
```

### 8.2. Docker Compose для локальної розробки

Docker Compose дозволяє легко запускати всі мікросервіси разом для локальної розробки та тестування.

```yaml
# docker-compose.yml
version: '3.8'

services:
  users-service:
    build: ./users_service
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql+asyncpg://user:pass@users-db:5432/users
      REDIS_URL: redis://redis:6379
    depends_on:
      - users-db
      - redis
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  orders-service:
    build: ./orders_service
    ports:
      - "8001:8000"
    environment:
      DATABASE_URL: postgresql+asyncpg://user:pass@orders-db:5432/orders
      REDIS_URL: redis://redis:6379
      USER_SERVICE_URL: http://users-service:8000
    depends_on:
      - orders-db
      - redis

  users-db:
    image: postgres:15
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: users
    volumes:
      - users-data:/var/lib/postgresql/data

  orders-db:
    image: postgres:15
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: orders
    volumes:
      - orders-data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  users-data:
  orders-data:
```

## 9. Розподілені транзакції та Saga Pattern

Однією з найскладніших проблем у мікросервісній архітектурі є забезпечення узгодженості даних при виконанні операцій, що охоплюють кілька сервісів. Традиційні ACID транзакції не працюють у розподіленому середовищі.

Saga Pattern є підходом до управління розподіленими транзакціями шляхом розбиття їх на послідовність локальних транзакцій. Кожна локальна транзакція оновлює дані в одному сервісі та публікує подію або повідомлення, що запускає наступну транзакцію. Якщо одна з транзакцій не вдається, Saga виконує компенсаційні транзакції для відміни попередніх змін.

```mermaid
sequenceDiagram
    participant O as Order Service
    participant P as Payment Service
    participant I as Inventory Service
    participant S as Shipping Service

    O->>O: Створити замовлення
    O->>P: Обробити платіж
    P->>P: Резервувати кошти
    P->>I: Зарезервувати товар
    I->>I: Зменшити кількість
    I->>S: Створити відправлення
    S->>S: Створити задачу доставки

    Note over O,S: Успішний сценарій

    O->>O: Створити замовлення
    O->>P: Обробити платіж
    P->>P: Резервувати кошти
    P->>I: Зарезервувати товар
    I-->>P: Помилка: немає в наявності
    P->>P: Компенсація: скасувати резервацію коштів
    P-->>O: Компенсація: скасувати замовлення

    Note over O,S: Сценарій з компенсацією
```

Приклад реалізації простої Saga для процесу створення замовлення.

```python
# orders_service/saga.py
from typing import List, Callable
from dataclasses import dataclass

@dataclass
class SagaStep:
    action: Callable
    compensation: Callable

class Saga:
    def __init__(self):
        self.steps: List[SagaStep] = []
        self.executed_steps: List[SagaStep] = []

    def add_step(self, action: Callable, compensation: Callable):
        """Додає крок до саги."""
        self.steps.append(SagaStep(action, compensation))

    async def execute(self):
        """Виконує всі кроки саги."""
        try:
            for step in self.steps:
                await step.action()
                self.executed_steps.append(step)
        except Exception as e:
            await self.compensate()
            raise e

    async def compensate(self):
        """Виконує компенсаційні дії для всіх виконаних кроків."""
        for step in reversed(self.executed_steps):
            try:
                await step.compensation()
            except Exception as e:
                # Логування помилки компенсації
                logger.error(f"Compensation failed: {e}")

# Використання
async def create_order_saga(order_data: OrderCreate):
    saga = Saga()

    # Крок 1: Створити замовлення
    order_id = None
    async def create_order():
        nonlocal order_id
        order = await order_repository.create(order_data)
        order_id = order.id

    async def cancel_order():
        if order_id:
            await order_repository.delete(order_id)

    saga.add_step(create_order, cancel_order)

    # Крок 2: Обробити платіж
    payment_id = None
    async def process_payment():
        nonlocal payment_id
        result = await payment_service.charge(
            order_data.user_id,
            order_data.total_amount
        )
        payment_id = result["payment_id"]

    async def refund_payment():
        if payment_id:
            await payment_service.refund(payment_id)

    saga.add_step(process_payment, refund_payment)

    # Крок 3: Зарезервувати товари
    async def reserve_inventory():
        await inventory_service.reserve(order_data.items)

    async def release_inventory():
        await inventory_service.release(order_data.items)

    saga.add_step(reserve_inventory, release_inventory)

    # Виконати сагу
    await saga.execute()
    return order_id
```

## 10. Обробка помилок та стійкість системи

У розподіленій системі помилки є нормою, а не винятком. Важливо проєктувати сервіси з урахуванням можливих збоїв.

### 10.1. Circuit Breaker Pattern

Circuit Breaker запобігає каскадним збоям, блокуючи виклики до сервісу, який не відповідає. Він має три стани: закритий (closed), відкритий (open) та напіввідкритий (half-open).

```python
# circuit_breaker.py
import time
from enum import Enum
from typing import Callable

class CircuitState(Enum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"

class CircuitBreaker:
    def __init__(
        self,
        failure_threshold: int = 5,
        timeout: int = 60,
        expected_exception: Exception = Exception
    ):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.expected_exception = expected_exception

        self.failure_count = 0
        self.last_failure_time = None
        self.state = CircuitState.CLOSED

    async def call(self, func: Callable, *args, **kwargs):
        """Виконує виклик через circuit breaker."""
        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time >= self.timeout:
                self.state = CircuitState.HALF_OPEN
            else:
                raise Exception("Circuit breaker is OPEN")

        try:
            result = await func(*args, **kwargs)
            self._on_success()
            return result
        except self.expected_exception as e:
            self._on_failure()
            raise e

    def _on_success(self):
        """Скидає лічильник при успішному виклику."""
        self.failure_count = 0
        if self.state == CircuitState.HALF_OPEN:
            self.state = CircuitState.CLOSED

    def _on_failure(self):
        """Збільшує лічильник помилок."""
        self.failure_count += 1
        self.last_failure_time = time.time()

        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN

# Використання
user_service_breaker = CircuitBreaker(failure_threshold=3, timeout=30)

async def get_user_with_breaker(user_id: int):
    """Виклик user service через circuit breaker."""
    return await user_service_breaker.call(
        httpx_client.get,
        f"{USER_SERVICE_URL}/users/{user_id}"
    )
```

### 10.2. Retry з експоненційною затримкою

Для тимчасових помилок корисно використовувати повторні спроби з експоненційною затримкою між ними.

```python
import asyncio
import random

async def retry_with_backoff(
    func: Callable,
    max_retries: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 60.0,
    *args,
    **kwargs
):
    """Виконує функцію з повторними спробами та експоненційною затримкою."""
    for attempt in range(max_retries):
        try:
            return await func(*args, **kwargs)
        except Exception as e:
            if attempt == max_retries - 1:
                raise e

            # Експоненційна затримка з jitter
            delay = min(base_delay * (2 ** attempt), max_delay)
            jitter = random.uniform(0, delay * 0.1)
            await asyncio.sleep(delay + jitter)

            logger.warning(f"Retry {attempt + 1}/{max_retries} after {delay}s")
```

## Висновки

У цій лекції ми розглянули ключові концепції асинхронної обробки задач та побудови розподілених систем. Черги повідомлень дозволяють розв'язати компоненти системи та забезпечити надійну обробку задач. Celery надає потужні можливості для виконання фонових задач, включаючи планування, моніторинг та обробку помилок. Мікросервісна архітектура дозволяє будувати масштабовані та гнучкі системи, хоча й вимагає врахування складностей розподілених систем, таких як узгодженість даних, обробка помилок та моніторинг.

Розуміння цих концепцій та патернів є критично важливим для розробки сучасних вебдодатків, здатних обробляти великі навантаження та забезпечувати високу доступність.
