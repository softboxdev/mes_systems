Вот пошаговое руководство по созданию типового CRUD проекта на FastAPI.

## 1. Установка зависимостей

```bash
pip install fastapi uvicorn sqlalchemy python-multipart
```

## 2. Структура проекта

```
my_fastapi_project/
├── main.py
├── models.py
├── schemas.py
├── crud.py
├── database.py
└── requirements.txt
```

## 3. Базовые файлы проекта

### `database.py` - настройка базы данных
```python
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

SQLALCHEMY_DATABASE_URL = "sqlite:///./test.db"
# Для PostgreSQL используйте:
# SQLALCHEMY_DATABASE_URL = "postgresql://user:password@localhost/dbname"

engine = create_engine(
    SQLALCHEMY_DATABASE_URL, connect_args={"check_same_thread": False}
)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

Base = declarative_base()

# Dependency для получения сессии БД
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

### `models.py` - модели базы данных
```python
from sqlalchemy import Column, Integer, String, Text
from database import Base

class Item(Base):
    __tablename__ = "items"
    
    id = Column(Integer, primary_key=True, index=True)
    title = Column(String(100), index=True)
    description = Column(Text, nullable=True)
    price = Column(Integer, nullable=True)
```

### `schemas.py` - Pydantic схемы
```python
from pydantic import BaseModel
from typing import Optional

# Схема для создания элемента
class ItemCreate(BaseModel):
    title: str
    description: Optional[str] = None
    price: Optional[int] = None

# Схема для обновления элемента
class ItemUpdate(BaseModel):
    title: Optional[str] = None
    description: Optional[str] = None
    price: Optional[int] = None

# Схема для ответа (чтения)
class ItemResponse(BaseModel):
    id: int
    title: str
    description: Optional[str]
    price: Optional[int]
    
    class Config:
        orm_mode = True
```

### `crud.py` - CRUD операции
```python
from sqlalchemy.orm import Session
from models import Item
from schemas import ItemCreate, ItemUpdate

def get_item(db: Session, item_id: int):
    return db.query(Item).filter(Item.id == item_id).first()

def get_items(db: Session, skip: int = 0, limit: int = 100):
    return db.query(Item).offset(skip).limit(limit).all()

def create_item(db: Session, item: ItemCreate):
    db_item = Item(**item.dict())
    db.add(db_item)
    db.commit()
    db.refresh(db_item)
    return db_item

def update_item(db: Session, item_id: int, item: ItemUpdate):
    db_item = db.query(Item).filter(Item.id == item_id).first()
    if db_item:
        update_data = item.dict(exclude_unset=True)
        for field, value in update_data.items():
            setattr(db_item, field, value)
        db.commit()
        db.refresh(db_item)
    return db_item

def delete_item(db: Session, item_id: int):
    db_item = db.query(Item).filter(Item.id == item_id).first()
    if db_item:
        db.delete(db_item)
        db.commit()
        return True
    return False
```

### `main.py` - основной файл приложения
```python
from fastapi import FastAPI, Depends, HTTPException, status
from sqlalchemy.orm import Session
from typing import List

from database import get_db, engine
from models import Base
from schemas import ItemCreate, ItemUpdate, ItemResponse
import crud

# Создаем таблицы
Base.metadata.create_all(bind=engine)

app = FastAPI(
    title="My CRUD API",
    description="Простое CRUD приложение на FastAPI",
    version="1.0.0"
)

@app.get("/")
def read_root():
    return {"message": "Welcome to FastAPI CRUD API"}

# CREATE - Создать элемент
@app.post("/items/", response_model=ItemResponse, status_code=status.HTTP_201_CREATED)
def create_item(item: ItemCreate, db: Session = Depends(get_db)):
    return crud.create_item(db=db, item=item)

# READ - Получить все элементы
@app.get("/items/", response_model=List[ItemResponse])
def read_items(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    items = crud.get_items(db, skip=skip, limit=limit)
    return items

# READ - Получить один элемент
@app.get("/items/{item_id}", response_model=ItemResponse)
def read_item(item_id: int, db: Session = Depends(get_db)):
    db_item = crud.get_item(db, item_id=item_id)
    if db_item is None:
        raise HTTPException(status_code=404, detail="Item not found")
    return db_item

# UPDATE - Обновить элемент
@app.put("/items/{item_id}", response_model=ItemResponse)
def update_item(item_id: int, item: ItemUpdate, db: Session = Depends(get_db)):
    db_item = crud.update_item(db, item_id=item_id, item=item)
    if db_item is None:
        raise HTTPException(status_code=404, detail="Item not found")
    return db_item

# DELETE - Удалить элемент
@app.delete("/items/{item_id}")
def delete_item(item_id: int, db: Session = Depends(get_db)):
    success = crud.delete_item(db, item_id=item_id)
    if not success:
        raise HTTPException(status_code=404, detail="Item not found")
    return {"message": "Item deleted successfully"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## 4. Запуск приложения

```bash
# Запуск сервера
uvicorn main:app --reload

# Или через Python
python main.py
```

## 5. Тестирование API

После запуска откройте в браузере:
- Документация Swagger: http://127.0.0.1:8000/docs
- Альтернативная документация: http://127.0.0.1:8000/redoc

### Примеры запросов:

**Создать элемент:**
```bash
curl -X POST "http://127.0.0.1:8000/items/" \
     -H "Content-Type: application/json" \
     -d '{"title": "Test Item", "description": "Test Description", "price": 100}'
```

**Получить все элементы:**
```bash
curl -X GET "http://127.0.0.1:8000/items/"
```

**Обновить элемент:**
```bash
curl -X PUT "http://127.0.0.1:8000/items/1" \
     -H "Content-Type: application/json" \
     -d '{"title": "Updated Title"}'
```

**Удалить элемент:**
```bash
curl -X DELETE "http://127.0.0.1:8000/items/1"
```

## 6. requirements.txt

```txt
fastapi==0.104.1
uvicorn==0.24.0
sqlalchemy==2.0.23
python-multipart==0.0.6
```

Этот шаблон можно расширять, добавляя аутентификацию, валидацию, дополнительные модели и более сложную бизнес-логику.

Подробно разберем каждый компонент и их взаимодействие:

## 1. Архитектура проекта и взаимодействие файлов

```
Client (HTTP) 
    ↓
main.py (FastAPI routes) 
    ↓  
crud.py (Business logic) 
    ↓  
schemas.py (Data validation) 
    ↓  
models.py (Database models) 
    ↓  
database.py (DB connection)
```

## 2. Подробное описание каждого файла

### `database.py` - Слой работы с базой данных

```python
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

# Подключение к БД (SQLite для примера)
SQLALCHEMY_DATABASE_URL = "sqlite:///./test.db"

# Создание движка SQLAlchemy
engine = create_engine(
    SQLALCHEMY_DATABASE_URL, 
    connect_args={"check_same_thread": False}  # Только для SQLite
)

# Создание фабрики сессий
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

# Базовый класс для моделей
Base = declarative_base()

# Dependency Injection для FastAPI
def get_db():
    """
    Генератор сессий БД.
    FastAPI будет вызывать эту функцию для каждого запроса.
    """
    db = SessionLocal()
    try:
        yield db  # ← сессия передается в route функцию
    finally:
        db.close()  # ← сессия закрывается после обработки запроса
```

**Что делает:**
- Создает соединение с БД
- Управляет жизненным циклом сессий
- Предоставляет базовый класс для моделей

**Вход:** ничего
**Выход:** сессия БД через генератор

### `models.py` - Модели данных (ORM)

```python
from sqlalchemy import Column, Integer, String, Text
from database import Base

class Item(Base):
    """
    Модель предмета в БД.
    Каждый атрибут - колонка в таблице.
    """
    __tablename__ = "items"  # имя таблицы в БД
    
    id = Column(Integer, primary_key=True, index=True)
    title = Column(String(100), index=True)  # строка max 100 символов
    description = Column(Text, nullable=True)  # текст, может быть пустым
    price = Column(Integer, nullable=True)  # число, может быть пустым
    
    def __repr__(self):
        return f"<Item(id={self.id}, title='{self.title}')>"
```

**Что делает:**
- Описывает структуру таблицы в БД
- Преобразует Python объекты в SQL записи и обратно
- Создает связи между таблицами

**Вход:** данные из БД
**Выход:** Python объекты

### `schemas.py` - Схемы валидации (Pydantic)

```python
from pydantic import BaseModel, Field
from typing import Optional

class ItemCreate(BaseModel):
    """
    Схема для СОЗДАНИЯ нового предмета.
    Используется в POST запросах.
    """
    title: str = Field(..., min_length=1, max_length=100, example="Новый предмет")
    description: Optional[str] = Field(None, max_length=500, example="Описание предмета")
    price: Optional[int] = Field(None, ge=0, example=1000)
    
    class Config:
        schema_extra = {
            "example": {
                "title": "Мой предмет",
                "description": "Отличный предмет",
                "price": 1500
            }
        }

class ItemUpdate(BaseModel):
    """
    Схема для ОБНОВЛЕНИЯ предмета.
    Все поля опциональны - можно обновлять только часть полей.
    """
    title: Optional[str] = Field(None, min_length=1, max_length=100)
    description: Optional[str] = Field(None, max_length=500)
    price: Optional[int] = Field(None, ge=0)

class ItemResponse(BaseModel):
    """
    Схема для ОТВЕТА клиенту.
    Включает id, который генерируется в БД.
    """
    id: int
    title: str
    description: Optional[str]
    price: Optional[int]
    
    class Config:
        orm_mode = True  # ← позволяет преобразовывать ORM объекты в Pydantic
```

**Что делает:**
- Валидирует входящие данные
- Преобразует типы данных
- Определяет структуру ответов API
- Генерирует документацию автоматически

**Вход:** сырые данные от клиента
**Выход:** валидированные данные или ошибки валидации

### `crud.py` - Data Access Object (DAO) слой

```python
from sqlalchemy.orm import Session
from sqlalchemy import and_
from models import Item
from schemas import ItemCreate, ItemUpdate

def get_item(db: Session, item_id: int) -> Item:
    """
    Получить один предмет по ID.
    
    Args:
        db: Сессия БД
        item_id: ID предмета
        
    Returns:
        Item или None если не найден
    """
    return db.query(Item).filter(Item.id == item_id).first()

def get_items(
    db: Session, 
    skip: int = 0, 
    limit: int = 100,
    title_filter: str = None
) -> list[Item]:
    """
    Получить список предметов с пагинацией и фильтрацией.
    
    Args:
        db: Сессия БД
        skip: Сколько записей пропустить (для пагинации)
        limit: Максимальное количество записей
        title_filter: Фильтр по названию
        
    Returns:
        Список предметов
    """
    query = db.query(Item)
    
    if title_filter:
        query = query.filter(Item.title.contains(title_filter))
    
    return query.offset(skip).limit(limit).all()

def create_item(db: Session, item: ItemCreate) -> Item:
    """
    Создать новый предмет.
    
    Args:
        db: Сессия БД
        item: Данные для создания (валидированные схемой)
        
    Returns:
        Созданный предмет с ID
    """
    # Преобразуем Pydantic модель в словарь
    item_data = item.dict()
    
    # Создаем объект модели БД
    db_item = Item(**item_data)
    
    # Добавляем в сессию
    db.add(db_item)
    
    # Сохраняем в БД
    db.commit()
    
    # Обновляем объект данными из БД (получаем ID)
    db.refresh(db_item)
    
    return db_item

def update_item(db: Session, item_id: int, item: ItemUpdate) -> Item:
    """
    Обновить существующий предмет.
    
    Args:
        db: Сессия БД
        item_id: ID обновляемого предмета
        item: Данные для обновления (только измененные поля)
        
    Returns:
        Обновленный предмет или None если не найден
    """
    # Ищем предмет в БД
    db_item = db.query(Item).filter(Item.id == item_id).first()
    
    if not db_item:
        return None
    
    # Получаем только переданные поля (исключаем None)
    update_data = item.dict(exclude_unset=True)
    
    # Обновляем каждое поле
    for field, value in update_data.items():
        setattr(db_item, field, value)
    
    # Сохраняем изменения
    db.commit()
    db.refresh(db_item)
    
    return db_item

def delete_item(db: Session, item_id: int) -> bool:
    """
    Удалить предмет по ID.
    
    Args:
        db: Сессия БД
        item_id: ID предмета для удаления
        
    Returns:
        True если удален, False если не найден
    """
    db_item = db.query(Item).filter(Item.id == item_id).first()
    
    if not db_item:
        return False
    
    db.delete(db_item)
    db.commit()
    
    return True
```

**Что делает (DAO слой):**
- Инкапсулирует всю работу с БД
- Выполняет CRUD операции
- Преобразует бизнес-объекты в SQL запросы
- Обрабатывает ошибки БД

**Вход:** 
- Сессия БД
- Валидированные данные
- Параметры запроса

**Выход:**
- ORM объекты
- Статус операции

### `main.py` - Маршруты API (Controller слой)

```python
from fastapi import FastAPI, Depends, HTTPException, status, Query
from sqlalchemy.orm import Session
from typing import List, Optional

from database import get_db, engine
from models import Base
from schemas import ItemCreate, ItemUpdate, ItemResponse
import crud

# Создаем таблицы в БД (в продакшене используйте миграции)
Base.metadata.create_all(bind=engine)

app = FastAPI(
    title="My CRUD API",
    description="Простое CRUD приложение на FastAPI",
    version="1.0.0"
)

@app.get("/")
def read_root():
    return {"message": "Welcome to FastAPI CRUD API"}

# CREATE - Создать новый предмет
@app.post("/items/", 
          response_model=ItemResponse, 
          status_code=status.HTTP_201_CREATED,
          summary="Создать предмет",
          description="Создает новый предмет в базе данных")
def create_item(
    item: ItemCreate,  # ← автоматическая валидация через Pydantic
    db: Session = Depends(get_db)  # ← автоматическое получение сессии БД
):
    """
    Создать новый предмет.
    
    - **title**: Название предмета (обязательно)
    - **description**: Описание предмета
    - **price**: Цена предмета
    """
    # Просто передаем данные в CRUD слой
    return crud.create_item(db=db, item=item)

# READ - Получить все предметы
@app.get("/items/", 
         response_model=List[ItemResponse],
         summary="Получить все предметы",
         description="Возвращает список предметов с пагинацией")
def read_items(
    skip: int = Query(0, ge=0, description="Сколько записей пропустить"),
    limit: int = Query(100, ge=1, le=1000, description="Лимит записей"),
    title: Optional[str] = Query(None, description="Фильтр по названию"),
    db: Session = Depends(get_db)
):
    """
    Получить список предметов.
    
    - **skip**: Пагинация - сколько записей пропустить
    - **limit**: Пагинация - макс. количество записей (1-1000)
    - **title**: Фильтрация по названию
    """
    items = crud.get_items(db, skip=skip, limit=limit, title_filter=title)
    return items

# READ - Получить один предмет
@app.get("/items/{item_id}", 
         response_model=ItemResponse,
         summary="Получить предмет по ID",
         description="Возвращает один предмет по его идентификатору")
def read_item(
    item_id: int = Query(..., ge=1, description="ID предмета"),
    db: Session = Depends(get_db)
):
    db_item = crud.get_item(db, item_id=item_id)
    if db_item is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND, 
            detail="Item not found"
        )
    return db_item

# UPDATE - Обновить предмет
@app.put("/items/{item_id}", 
         response_model=ItemResponse,
         summary="Обновить предмет",
         description="Обновляет данные предмета по ID")
def update_item(
    item_id: int,
    item: ItemUpdate,
    db: Session = Depends(get_db)
):
    db_item = crud.update_item(db, item_id=item_id, item=item)
    if db_item is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND, 
            detail="Item not found"
        )
    return db_item

# DELETE - Удалить предмет
@app.delete("/items/{item_id}",
            summary="Удалить предмет",
            description="Удаляет предмет по ID из базы данных")
def delete_item(
    item_id: int,
    db: Session = Depends(get_db)
):
    success = crud.delete_item(db, item_id=item_id)
    if not success:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND, 
            detail="Item not found"
        )
    return {"message": "Item deleted successfully"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

**Что делает (Controller слой):**
- Принимает HTTP запросы
- Валидирует входные параметры
- Вызывает бизнес-логику (CRUD слой)
- Обрабатывает ошибки
- Формирует HTTP ответы

## 3. Полный цикл обработки запроса

**Пример: POST /items/**

1. **Клиент** отправляет JSON:
```json
{
  "title": "Новый предмет",
  "description": "Описание",
  "price": 1000
}
```

2. **FastAPI** (main.py):
   - Валидирует данные через `ItemCreate` схему
   - Получает сессию БД через `get_db()`
   - Вызывает `crud.create_item(db, item)`

3. **CRUD слой** (crud.py):
   - Преобразует Pydantic модель в словарь
   - Создает ORM объект `Item`
   - Сохраняет в БД через SQLAlchemy session
   - Возвращает ORM объект с ID

4. **FastAPI** преобразует ORM объект в JSON через `ItemResponse` схему (благодаря `orm_mode=True`)

5. **Клиент** получает ответ:
```json
{
  "id": 1,
  "title": "Новый предмет",
  "description": "Описание",
  "price": 1000
}
```

## 4. Преимущества такой архитектуры

- **Разделение ответственности**: каждый слой отвечает за свою задачу
- **Тестируемость**: можно тестировать каждый слой отдельно
- **Гибкость**: легко заменить БД или добавить новую функциональность
- **Безопасность**: валидация на всех уровнях
- **Документирование**: автоматическая генерация документации

Этот подход следует принципам чистой архитектуры и легко масштабируется для сложных приложений.