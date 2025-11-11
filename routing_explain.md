
## 1. Основы роутинга в FastAPI

### Что такое роутинг?
Роутинг - это процесс определения того, какой код выполнится при обращении к определенному URL (эндпоинту).

### Базовый синтаксис:
```python
from fastapi import FastAPI

app = FastAPI()

@app.http_method("path")
def function_name(parameters):
    return response
```

## 2. Создание приложения и базовые роуты

### Основное приложение:
```python
from fastapi import FastAPI

app = FastAPI(
    title="My API",
    description="API description",
    version="1.0.0"
)

# Простейший GET роут
@app.get("/")
def read_root():
    return {"message": "Hello World"}

# GET роут с параметром в пути
@app.get("/items/{item_id}")
def read_item(item_id: int):
    return {"item_id": item_id}
```

## 3. HTTP методы (декораторы)

FastAPI поддерживает все основные HTTP методы:

```python
@app.get("/items")        # GET - получение данных
@app.post("/items")       # POST - создание данных
@app.put("/items/{id}")   # PUT - полное обновление
@app.patch("/items/{id}") # PATCH - частичное обновление
@app.delete("/items/{id}")# DELETE - удаление данных
@app.head("/items")       # HEAD - заголовки
@app.options("/items")    # OPTIONS - опции
```

## 4. Параметры путей (Path Parameters)

### Простые параметры:
```python
@app.get("/items/{item_id}")
def get_item(item_id: int):  # FastAPI автоматически конвертирует тип
    return {"item_id": item_id}
```

### Несколько параметров:
```python
@app.get("/users/{user_id}/orders/{order_id}")
def get_user_order(user_id: int, order_id: str):
    return {"user_id": user_id, "order_id": order_id}
```

### Параметры с валидацией:
```python
from fastapi import Path

@app.get("/items/{item_id}")
def get_item(
    item_id: int = Path(..., gt=0, le=1000, description="ID товара от 1 до 1000"),
    category: str = Path(..., min_length=2, max_length=50)
):
    return {"item_id": item_id, "category": category}
```

## 5. Query параметры

### Базовые query параметры:
```python
@app.get("/items/")
def read_items(
    skip: int = 0,      # /items/?skip=0
    limit: int = 100    # /items/?skip=0&limit=100
):
    return {"skip": skip, "limit": limit}
```

### Query параметры с валидацией:
```python
from fastapi import Query

@app.get("/items/")
def read_items(
    q: str = Query(None, min_length=3, max_length=50),  # опциональный параметр
    skip: int = Query(0, ge=0),                         # >= 0
    limit: int = Query(100, ge=1, le=1000)              # от 1 до 1000
):
    results = {"items": [], "skip": skip, "limit": limit}
    if q:
        results["q"] = q
    return results
```

### Множественные query параметры:
```python
@app.get("/items/")
def read_items(
    tags: list[str] = Query(["default"], description="Список тегов")
):
    return {"tags": tags}

# Доступ: /items/?tags=python&tags=fastapi&tags=web
```

## 6. Тело запроса (Request Body)

### Pydantic модели для тела запроса:
```python
from pydantic import BaseModel
from typing import Optional

class Item(BaseModel):
    name: str
    description: Optional[str] = None
    price: float
    tax: Optional[float] = None

@app.post("/items/")
def create_item(item: Item):  # FastAPI автоматически валидирует тело запроса
    item_dict = item.dict()
    if item.tax:
        total = item.price + item.tax
        item_dict["total"] = total
    return item_dict
```

### Комбинация параметров:
```python
@app.put("/items/{item_id}")
def update_item(
    item_id: int,           # path параметр
    item: Item,             # тело запроса
    q: str = None,          # query параметр
    importance: int = Query(1, ge=1, le=10)  # query с валидацией
):
    result = {"item_id": item_id, **item.dict()}
    if q:
        result["q"] = q
    result["importance"] = importance
    return result
```

## 7. Роутеры (APIRouter) - организация кода

### Создание роутера:
```python
from fastapi import APIRouter

router = APIRouter(
    prefix="/items",      # автоматический префикс для всех путей
    tags=["items"],       # группировка в документации
    responses={404: {"description": "Not found"}}  # общие ответы
)

@router.get("/")
def read_items():
    return [{"item": "Foo"}, {"item": "Bar"}]

@router.get("/{item_id}")
def read_item(item_id: int):
    return {"item_id": item_id}

@router.post("/")
def create_item(item: Item):
    return item
```

### Подключение роутеров:
```python
from fastapi import FastAPI
from .routers import items, users

app = FastAPI()

app.include_router(items.router)
app.include_router(users.router, prefix="/api/v1")  # кастомный префикс
```

## 8. Зависимости (Dependencies) в роутах

### Функции-зависимости:
```python
from fastapi import Depends, Header, HTTPException

async def verify_token(x_token: str = Header(...)):
    if x_token != "secret-token":
        raise HTTPException(status_code=400, detail="Invalid token")
    return x_token

async def get_db():
    db = "database_connection"
    try:
        yield db
    finally:
        db.close()

@router.get("/items/", dependencies=[Depends(verify_token)])
def read_items(db: str = Depends(get_db)):
    return {"db": db, "items": []}
```

## 9. Статические файлы и HTML

```python
from fastapi.staticfiles import StaticFiles
from fastapi.responses import HTMLResponse

app.mount("/static", StaticFiles(directory="static"), name="static")

@app.get("/", response_class=HTMLResponse)
def read_root():
    return """
    <html>
        <head><title>My API</title></head>
        <body>
            <h1>Welcome to FastAPI</h1>
        </body>
    </html>
    """
```

## 10. Продвинутые возможности роутинга

### Кастомные статус-коды:
```python
from fastapi import status

@app.post("/items/", status_code=status.HTTP_201_CREATED)
def create_item(item: Item):
    return item
```

### Кастомные ответы:
```python
from fastapi.responses import JSONResponse, FileResponse

@app.get("/json")
def get_json():
    return JSONResponse(
        content={"message": "Hello"},
        status_code=200,
        headers={"X-Custom-Header": "value"}
    )

@app.get("/download")
def download_file():
    return FileResponse("file.pdf", filename="custom_name.pdf")
```

### Фоновые задачи:
```python
from fastapi import BackgroundTasks

def write_log(message: str):
    with open("log.txt", "a") as log:
        log.write(message + "\n")

@app.post("/items/")
def create_item(
    item: Item, 
    background_tasks: BackgroundTasks
):
    background_tasks.add_task(write_log, f"Created item: {item.name}")
    return item
```

## 11. Валидация и документация

### Автоматическая документация:
```python
@app.post(
    "/items/",
    response_model=Item,
    summary="Создать товар",
    description="Создает новый товар в системе",
    response_description="Созданный товар",
    tags=["items", "admin"],
    deprecated=False
)
def create_item(item: Item):
    """
    Создать новый товар:
    
    - **name**: Название товара (обязательно)
    - **description**: Описание товара
    - **price**: Цена товара
    - **tax**: Налог на товар
    """
    return item
```

## 12. Обработка ошибок

### Кастомные исключения:
```python
from fastapi import HTTPException, Request
from fastapi.responses import JSONResponse

@app.exception_handler(HTTPException)
async def http_exception_handler(request: Request, exc: HTTPException):
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": exc.detail, "path": request.url.path}
    )

@app.get("/items/{item_id}")
def read_item(item_id: int):
    if item_id > 100:
        raise HTTPException(
            status_code=404, 
            detail="Item not found",
            headers={"X-Error": "Item ID too large"}
        )
    return {"item_id": item_id}
```

## 13. Полный пример структуры проекта

```
my_project/
├── main.py
├── routers/
│   ├── __init__.py
│   ├── items.py
│   ├── users.py
│   └── admin.py
├── models/
│   ├── __init__.py
│   └── schemas.py
├── dependencies.py
└── config.py
```

### Пример `main.py`:
```python
from fastapi import FastAPI, Depends
from .routers import items, users, admin
from .dependencies import get_db

app = FastAPI()

# Подключаем роутеры
app.include_router(items.router)
app.include_router(users.router)
app.include_router(
    admin.router,
    prefix="/admin",
    tags=["admin"],
    dependencies=[Depends(get_db)],
    responses={418: {"description": "I'm a teapot"}}
)

@app.get("/")
def read_root():
    return {"message": "API is running"}
```

## 14. Правила написания качественных роутов

### ✅ ХОРОШО:
```python
@router.get(
    "/users/{user_id}",
    response_model=UserResponse,
    summary="Получить пользователя",
    tags=["users"]
)
def get_user(
    user_id: int = Path(..., description="ID пользователя"),
    db: Session = Depends(get_db)
) -> UserResponse:
    """
    Получить информацию о пользователе по ID.
    
    - **user_id**: Уникальный идентификатор пользователя
    - **returns**: Объект пользователя
    """
    user = crud.get_user(db, user_id)
    if not user:
        raise HTTPException(404, "User not found")
    return user
```

### ❌ ПЛОХО:
```python
@app.get("/get_user")  # ❌ GET для получения данных не должен менять состояние
def get_user(user_id: int, db: str):  # ❌ нет типизации, нет зависимостей
    # ❌ нет обработки ошибок, нет документации
    return some_function(user_id)
```

## 15. Лучшие практики

1. **Используйте APIRouter** для модульной организации
2. **Валидируйте все входные данные** через Pydantic
3. **Используйте зависимости** для повторяющейся логики
4. **Документируйте эндпоинты** через docstrings
5. **Разделяйте ответственность** между роутерами
6. **Обрабатывайте ошибки** грамотно
7. **Используйте правильные HTTP статус-коды**
8. **Тестируйте роуты** автоматически

## 16. Автоматическая документация

После запуска приложения доступно:
- **Swagger UI**: http://localhost:8000/docs
- **ReDoc**: http://localhost:8000/redoc

FastAPI автоматически генерирует документацию на основе типов, валидации и docstrings!
