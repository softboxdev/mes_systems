

## 🚀 Полное руководство по созданию CRUD API на FastAPI

## 1. Установка зависимостей

```bash
# fastapi - основной фреймворк
# uvicorn - ASGI сервер для запуска
# sqlalchemy - ORM для работы с БД
# python-multipart - для обработки form-data
pip install fastapi uvicorn sqlalchemy python-multipart
```

## 2. Структура проекта

```
my_fastapi_project/
├── main.py          # Точка входа, маршруты API
├── models.py        # Модели базы данных (SQLAlchemy)
├── schemas.py       # Схемы Pydantic для валидации
├── crud.py          # Бизнес-логика (CRUD операции)
├── database.py      # Настройка подключения к БД
└── requirements.txt # Зависимости проекта
```

## 3. Детальное описание каждого файла

### `database.py` - Настройка подключения к базе данных

```python
# Импортируем необходимые компоненты из SQLAlchemy
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

# URL для подключения к базе данных
# SQLite - файловая БД, хороша для разработки
SQLALCHEMY_DATABASE_URL = "sqlite:///./test.db"

# Для PostgreSQL раскомментируйте следующую строку:
# SQLALCHEMY_DATABASE_URL = "postgresql://username:password@localhost/dbname"

# Создаем "движок" - основной интерфейс к базе данных
# connect_args нужен только для SQLite
engine = create_engine(
    SQLALCHEMY_DATABASE_URL, 
    connect_args={"check_same_thread": False}  # Разрешаем использовать один поток
)

# SessionLocal - фабрика для создания сессий БД
# autocommit=False - отключаем автоматическое сохранение
# autoflush=False - отключаем автоматическую синхронизацию
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

# Base - базовый класс для всех моделей
# Все модели будут наследоваться от этого класса
Base = declarative_base()

# Функция-генератор для dependency injection в FastAPI
def get_db():
    """
    Эта функция будет вызываться для каждого запроса.
    Она создает новую сессию БД и закрывает ее после обработки запроса.
    """
    # Создаем новую сессию
    db = SessionLocal()
    try:
        # Возвращаем сессию в route функцию
        yield db
    finally:
        # Всегда закрываем сессию, даже если произошла ошибка
        db.close()
```

### `models.py` - Модели данных (SQLAlchemy)

```python
# Импортируем типы данных и базовый класс
from sqlalchemy import Column, Integer, String, Text
from database import Base

class Item(Base):
    """
    Модель предмета (аналог таблицы в БД)
    Каждый атрибут класса - колонка в таблице
    """
    
    # Имя таблицы в базе данных
    __tablename__ = "items"
    
    # ID - первичный ключ, автоинкремент
    # index=True - создает индекс для ускорения поиска
    id = Column(Integer, primary_key=True, index=True)
    
    # Название предмета, строка максимум 100 символов
    # index=True - индекс для поиска по названию
    title = Column(String(100), index=True)
    
    # Описание, текстовое поле, может быть пустым
    description = Column(Text, nullable=True)
    
    # Цена, целое число, может быть пустым
    price = Column(Integer, nullable=True)
    
    def __repr__(self):
        """Строковое представление объекта для отладки"""
        return f"<Item(id={self.id}, title='{self.title}', price={self.price})>"
```

### `schemas.py` - Схемы валидации (Pydantic)

```python
from pydantic import BaseModel, Field
from typing import Optional

class ItemCreate(BaseModel):
    """
    Схема для СОЗДАНИЯ нового предмета.
    Используется когда клиент отправляет POST запрос.
    """
    
    # Обязательное поле, минимум 1 символ, максимум 100
    title: str = Field(
        ...,  # ... означает что поле обязательное
        min_length=1, 
        max_length=100,
        example="Новый предмет"  # Пример для документации
    )
    
    # Необязательное поле
    description: Optional[str] = Field(
        None,  # Значение по умолчанию
        max_length=500,
        example="Описание предмета"
    )
    
    # Необязательное поле, должно быть >= 0 если указано
    price: Optional[int] = Field(
        None,
        ge=0,  # greater or equal - больше или равно 0
        example=1000
    )

class ItemUpdate(BaseModel):
    """
    Схема для ОБНОВЛЕНИЯ предмета.
    Все поля необязательные - можно обновить только часть данных.
    """
    title: Optional[str] = Field(None, min_length=1, max_length=100)
    description: Optional[str] = Field(None, max_length=500)
    price: Optional[int] = Field(None, ge=0)

class ItemResponse(BaseModel):
    """
    Схема для ОТВЕТА клиенту.
    Включает ID, который генерируется базой данных.
    """
    id: int
    title: str
    description: Optional[str]
    price: Optional[int]
    
    class Config:
        # Включаем режим ORM для работы с SQLAlchemy объектами
        orm_mode = True
        # Это позволяет Pydantic читать данные из ORM объектов,
        # а не только из словарей
```

### `crud.py` - Бизнес-логика (CRUD операции)

```python
from sqlalchemy.orm import Session
from models import Item
from schemas import ItemCreate, ItemUpdate

def get_item(db: Session, item_id: int):
    """
    Получить один предмет по ID из базы данных.
    
    Args:
        db: Сессия базы данных
        item_id: ID искомого предмета
        
    Returns:
        Item object или None если не найден
    """
    # Создаем запрос: SELECT * FROM items WHERE id = item_id
    # .first() - берем первую найденную запись или None
    return db.query(Item).filter(Item.id == item_id).first()

def get_items(db: Session, skip: int = 0, limit: int = 100):
    """
    Получить список предметов с пагинацией.
    
    Args:
        db: Сессия базы данных
        skip: Сколько записей пропустить (для пагинации)
        limit: Максимальное количество записей
        
    Returns:
        Список предметов
    """
    # Создаем запрос с пагинацией:
    # SELECT * FROM items LIMIT limit OFFSET skip
    return db.query(Item).offset(skip).limit(limit).all()

def create_item(db: Session, item: ItemCreate):
    """
    Создать новый предмет в базе данных.
    
    Args:
        db: Сессия базы данных
        item: Данные для создания (валидированные схемой ItemCreate)
        
    Returns:
        Созданный предмет с присвоенным ID
    """
    # Преобразуем Pydantic модель в словарь
    item_data = item.dict()
    
    # Создаем объект модели SQLAlchemy
    # **item_data - распаковываем словарь в аргументы: Item(title=..., description=...)
    db_item = Item(**item_data)
    
    # Добавляем объект в сессию
    db.add(db_item)
    
    # Сохраняем изменения в базе данных
    db.commit()
    
    # Обновляем объект данными из БД (получаем сгенерированный ID)
    db.refresh(db_item)
    
    return db_item

def update_item(db: Session, item_id: int, item: ItemUpdate):
    """
    Обновить существующий предмет в базе данных.
    
    Args:
        db: Сессия базы данных
        item_id: ID обновляемого предмета
        item: Новые данные (только измененные поля)
        
    Returns:
        Обновленный предмет или None если не найден
    """
    # Ищем предмет в базе данных
    db_item = db.query(Item).filter(Item.id == item_id).first()
    
    # Если предмет не найден, возвращаем None
    if db_item is None:
        return None
    
    # Получаем только переданные поля (исключаем None значения)
    # exclude_unset=True - берем только те поля, которые действительно были переданы
    update_data = item.dict(exclude_unset=True)
    
    # Обновляем каждое переданное поле
    for field, value in update_data.items():
        # setattr устанавливает значение атрибута объекта
        setattr(db_item, field, value)
    
    # Сохраняем изменения
    db.commit()
    
    # Обновляем объект из БД
    db.refresh(db_item)
    
    return db_item

def delete_item(db: Session, item_id: int):
    """
    Удалить предмет из базы данных.
    
    Args:
        db: Сессия базы данных
        item_id: ID удаляемого предмета
        
    Returns:
        True если удален, False если не найден
    """
    # Ищем предмет
    db_item = db.query(Item).filter(Item.id == item_id).first()
    
    # Если не найден, возвращаем False
    if db_item is None:
        return False
    
    # Удаляем предмет из сессии
    db.delete(db_item)
    
    # Сохраняем изменения
    db.commit()
    
    return True
```

### `main.py` - Основное приложение FastAPI

```python
# Импортируем необходимые компоненты FastAPI
from fastapi import FastAPI, Depends, HTTPException, status, Query
from sqlalchemy.orm import Session
from typing import List, Optional

# Импортируем наши модули
from database import get_db, engine
from models import Base
from schemas import ItemCreate, ItemUpdate, ItemResponse
import crud

# 🔨 СОЗДАЕМ ТАБЛИЦЫ В БАЗЕ ДАННЫХ
# ВНИМАНИЕ: В продакшене используйте миграции (Alembic) вместо этого!
Base.metadata.create_all(bind=engine)

# 🚀 СОЗДАЕМ ПРИЛОЖЕНИЕ FASTAPI
app = FastAPI(
    title="My CRUD API",           # Название API
    description="Простое CRUD приложение на FastAPI",  # Описание
    version="1.0.0"                # Версия API
)

# 🌟 КОРНЕВОЙ ЭНДПОИНТ
@app.get("/")
def read_root():
    """
    Корневой endpoint для проверки работы API.
    """
    return {"message": "Welcome to FastAPI CRUD API"}

# ✅ CREATE - СОЗДАТЬ НОВЫЙ ПРЕДМЕТ
@app.post("/items/", 
          response_model=ItemResponse, 
          status_code=status.HTTP_201_CREATED,
          summary="Создать предмет",
          description="Создает новый предмет в базе данных")
def create_item(
    item: ItemCreate,  # 📝 Данные из тела запроса (валидируются схемой ItemCreate)
    db: Session = Depends(get_db)  # 🗄️ Сессия БД (автоматически создается и закрывается)
):
    """
    Создать новый предмет в системе.
    
    - **title**: Название предмета (обязательное поле, 1-100 символов)
    - **description**: Описание предмета (необязательное)
    - **price**: Цена предмета (необязательное, должно быть >= 0)
    """
    # Просто передаем данные в CRUD слой
    return crud.create_item(db=db, item=item)

# 📖 READ - ПОЛУЧИТЬ ВСЕ ПРЕДМЕТЫ
@app.get("/items/", 
         response_model=List[ItemResponse],
         summary="Получить все предметы",
         description="Возвращает список предметов с пагинацией и фильтрацией")
def read_items(
    skip: int = Query(0, ge=0, description="Сколько записей пропустить"),  # 🎯 Параметр запроса с валидацией
    limit: int = Query(100, ge=1, le=1000, description="Лимит записей"),   # 🎯 Максимум 1000 записей
    title: Optional[str] = Query(None, description="Фильтр по названию"),  # 🎯 Необязательный фильтр
    db: Session = Depends(get_db)  # 🗄️ Сессия БД
):
    """
    Получить список предметов с поддержкой пагинации.
    
    - **skip**: Сколько записей пропустить (для пагинации)
    - **limit**: Максимальное количество записей (1-1000)
    - **title**: Фильтр по названию (необязательный)
    """
    # В реальном проекте можно добавить фильтрацию в CRUD функцию
    items = crud.get_items(db, skip=skip, limit=limit)
    
    # Если передан фильтр по названию, фильтруем результаты
    if title:
        items = [item for item in items if title.lower() in item.title.lower()]
    
    return items

# 🔍 READ - ПОЛУЧИТЬ ОДИН ПРЕДМЕТ ПО ID
@app.get("/items/{item_id}", 
         response_model=ItemResponse,
         summary="Получить предмет по ID",
         description="Возвращает один предмет по его идентификатору")
def read_item(
    item_id: int,  # 🎯 Параметр пути из URL
    db: Session = Depends(get_db)  # 🗄️ Сессия БД
):
    """
    Получить предмет по его уникальному идентификатору.
    
    - **item_id**: ID предмета (целое число > 0)
    """
    # Получаем предмет через CRUD слой
    db_item = crud.get_item(db, item_id=item_id)
    
    # Если предмет не найден, возвращаем 404 ошибку
    if db_item is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND, 
            detail="Item not found"  # 📝 Сообщение об ошибке
        )
    
    return db_item

# ✏️ UPDATE - ОБНОВИТЬ ПРЕДМЕТ
@app.put("/items/{item_id}", 
         response_model=ItemResponse,
         summary="Обновить предмет",
         description="Обновляет данные предмета по ID")
def update_item(
    item_id: int,  # 🎯 ID предмета из URL пути
    item: ItemUpdate,  # 📝 Данные для обновления (только измененные поля)
    db: Session = Depends(get_db)  # 🗄️ Сессия БД
):
    """
    Обновить данные существующего предмета.
    
    - **item_id**: ID обновляемого предмета
    - Можно передать любое количество полей для обновления
    - Только переданные поля будут обновлены
    """
    # Вызываем CRUD операцию обновления
    db_item = crud.update_item(db, item_id=item_id, item=item)
    
    # Если предмет не найден, возвращаем 404
    if db_item is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND, 
            detail="Item not found"
        )
    
    return db_item

# 🗑️ DELETE - УДАЛИТЬ ПРЕДМЕТ
@app.delete("/items/{item_id}",
            summary="Удалить предмет",
            description="Удаляет предмет по ID из базы данных")
def delete_item(
    item_id: int,  # 🎯 ID предмета из URL пути
    db: Session = Depends(get_db)  # 🗄️ Сессия БД
):
    """
    Удалить предмет из системы по его ID.
    
    - **item_id**: ID удаляемого предмета
    """
    # Вызываем CRUD операцию удаления
    success = crud.delete_item(db, item_id=item_id)
    
    # Если предмет не найден, возвращаем 404
    if not success:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND, 
            detail="Item not found"
        )
    
    return {"message": "Item deleted successfully"}

# 🚀 ЗАПУСК СЕРВЕРА
if __name__ == "__main__":
    import uvicorn
    # Запускаем сервер на всех интерфейсах (0.0.0.0) порт 8000
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## 4. Запуск приложения

```bash
# Запуск с автоматической перезагрузкой при изменениях
uvicorn main:app --reload

# Запуск без перезагрузки (для продакшена)
uvicorn main:app --host 0.0.0.0 --port 8000
```

## 5. Тестирование API

После запуска откройте в браузере:
- **Swagger UI**: http://127.0.0.1:8000/docs
- **ReDoc**: http://127.0.0.1:8000/redoc

### Примеры тестирования через curl:

**Создать предмет:**
```bash
curl -X POST "http://127.0.0.1:8000/items/" \
     -H "Content-Type: application/json" \
     -d '{
       "title": "MacBook Pro",
       "description": "Ноутбук Apple",
       "price": 200000
     }'
```

**Получить все предметы:**
```bash
curl -X GET "http://127.0.0.1:8000/items/?skip=0&limit=10"
```

**Получить один предмет:**
```bash
curl -X GET "http://127.0.0.1:8000/items/1"
```

**Обновить предмет:**
```bash
curl -X PUT "http://127.0.0.1:8000/items/1" \
     -H "Content-Type: application/json" \
     -d '{"title": "MacBook Pro M2", "price": 220000}'
```

**Удалить предмет:**
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

## 🎯 Ключевые концепции, объясненные в руководстве:

1. **Dependency Injection** - автоматическое управление сессиями БД
2. **Pydantic Schemas** - валидация данных на входе и выходе
3. **ORM (Object-Relational Mapping)** - работа с БД через Python объекты
4. **CRUD Operations** - Create, Read, Update, Delete
5. **HTTP Status Codes** - правильные коды ответов
6. **API Documentation** - автоматическая генерация документации
7. **Error Handling** - обработка ошибок и исключений

Давайте адаптируем CRUD проект к AnyLogic Cloud API проекту и дополним роутинг. Задача - интегрировать CRUD операции для управления симуляциями и результатами.

## 🚀 Адаптация CRUD проекта к AnyLogic Cloud API

### 1. Обновленная структура проекта

```
anylogic-fastapi-project/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── models.py           # SQLAlchemy + Pydantic модели
│   ├── schemas.py          # Pydantic схемы
│   ├── database.py         # Настройка БД
│   ├── dependencies.py     # Зависимости
│   ├── crud.py             # CRUD операции для симуляций
│   └── routers/
│       ├── __init__.py
│       ├── simulations.py  # Роутер для симуляций AnyLogic
│       └── results.py      # Роутер для управления результатами (CRUD)
├── requirements.txt
├── .env
└── README.md
```

### 2. Обновленные зависимости

**requirements.txt:**
```txt
fastapi
uvicorn
python-dotenv
requests
pydantic
anylogiccloudclient
sqlalchemy
python-multipart
```

### 3. База данных для хранения результатов симуляций

**app/database.py:**
```python
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
import os

# Используем SQLite для хранения результатов симуляций
SQLALCHEMY_DATABASE_URL = "sqlite:///./simulation_results.db"

engine = create_engine(
    SQLALCHEMY_DATABASE_URL, 
    connect_args={"check_same_thread": False}
)

SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

Base = declarative_base()

def get_db():
    """
    Dependency для получения сессии БД.
    Используется для хранения результатов симуляций локально.
    """
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

### 4. Модели базы данных для хранения результатов

**app/models.py:**
```python
from sqlalchemy import Column, Integer, String, Float, DateTime, Text, JSON
from sqlalchemy.sql import func
from database import Base

class SimulationResult(Base):
    """
    Модель для хранения результатов симуляций в локальной БД
    """
    __tablename__ = "simulation_results"
    
    id = Column(Integer, primary_key=True, index=True)
    simulation_id = Column(String(100), unique=True, index=True)  # ID из AnyLogic Cloud
    model_name = Column(String(200), index=True)
    experiment_name = Column(String(200))
    server_capacity = Column(Integer)
    mean_queue_size = Column(Float)
    server_utilization = Column(Float)
    raw_outputs = Column(JSON)  # Храним все выходные данные как JSON
    status = Column(String(50), default="completed")
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    
    def __repr__(self):
        return f"<SimulationResult(id={self.id}, model='{self.model_name}', capacity={self.server_capacity})>"

# Pydantic схемы для валидации
from pydantic import BaseModel, Field
from typing import Dict, Any, Optional, List
from datetime import datetime

class SimulationRequest(BaseModel):
    """Схема для запроса запуска симуляции"""
    server_capacity: int = Field(ge=1, le=50, example=8)
    model_name: str = Field(default="Service System Demo")
    experiment_name: str = Field(default="Baseline")
    save_to_db: bool = Field(default=True, description="Сохранить результат в локальную БД")

class SimulationResponse(BaseModel):
    """Схема ответа с результатами симуляции"""
    simulation_id: str
    server_capacity: int
    mean_queue_size: float
    server_utilization: float
    raw_outputs: Dict[str, Any]
    status: str
    db_record_id: Optional[int] = None

class SimulationResultCreate(BaseModel):
    """Схема для создания записи результата в БД"""
    simulation_id: str
    model_name: str
    experiment_name: str
    server_capacity: int
    mean_queue_size: float
    server_utilization: float
    raw_outputs: Dict[str, Any]
    status: str = "completed"

class SimulationResultUpdate(BaseModel):
    """Схема для обновления записи результата"""
    server_capacity: Optional[int] = None
    mean_queue_size: Optional[float] = None
    server_utilization: Optional[float] = None
    status: Optional[str] = None

class SimulationResultResponse(BaseModel):
    """Схема ответа для записей из БД"""
    id: int
    simulation_id: str
    model_name: str
    experiment_name: str
    server_capacity: int
    mean_queue_size: float
    server_utilization: float
    raw_outputs: Dict[str, Any]
    status: str
    created_at: datetime
    
    class Config:
        orm_mode = True

class ErrorResponse(BaseModel):
    error: str
    detail: Optional[str] = None

class ModelInfo(BaseModel):
    id: str
    name: str
    latest_version_id: Optional[str] = None
```

### 5. CRUD операции для управления результатами симуляций

**app/crud.py:**
```python
from sqlalchemy.orm import Session
from models import SimulationResult
from schemas import SimulationResultCreate, SimulationResultUpdate

def get_simulation_result(db: Session, result_id: int):
    """Получить результат симуляции по ID"""
    return db.query(SimulationResult).filter(SimulationResult.id == result_id).first()

def get_simulation_results(db: Session, skip: int = 0, limit: int = 100, model_name: str = None):
    """Получить список результатов симуляций с фильтрацией"""
    query = db.query(SimulationResult)
    
    if model_name:
        query = query.filter(SimulationResult.model_name.contains(model_name))
    
    return query.order_by(SimulationResult.created_at.desc()).offset(skip).limit(limit).all()

def get_simulation_by_external_id(db: Session, simulation_id: str):
    """Найти результат по ID симуляции из AnyLogic Cloud"""
    return db.query(SimulationResult).filter(SimulationResult.simulation_id == simulation_id).first()

def create_simulation_result(db: Session, result: SimulationResultCreate):
    """Создать запись результата симуляции в БД"""
    db_result = SimulationResult(**result.dict())
    db.add(db_result)
    db.commit()
    db.refresh(db_result)
    return db_result

def update_simulation_result(db: Session, result_id: int, result: SimulationResultUpdate):
    """Обновить запись результата симуляции"""
    db_result = db.query(SimulationResult).filter(SimulationResult.id == result_id).first()
    
    if db_result is None:
        return None
    
    update_data = result.dict(exclude_unset=True)
    for field, value in update_data.items():
        setattr(db_result, field, value)
    
    db.commit()
    db.refresh(db_result)
    return db_result

def delete_simulation_result(db: Session, result_id: int):
    """Удалить запись результата симуляции"""
    db_result = db.query(SimulationResult).filter(SimulationResult.id == result_id).first()
    
    if db_result is None:
        return False
    
    db.delete(db_result)
    db.commit()
    return True

def get_simulation_statistics(db: Session):
    """Получить статистику по всем симуляциям"""
    from sqlalchemy import func
    
    stats = db.query(
        SimulationResult.model_name,
        func.count(SimulationResult.id).label('total_simulations'),
        func.avg(SimulationResult.mean_queue_size).label('avg_queue_size'),
        func.avg(SimulationResult.server_utilization).label('avg_utilization'),
        func.min(SimulationResult.created_at).label('first_simulation'),
        func.max(SimulationResult.created_at).label('last_simulation')
    ).group_by(SimulationResult.model_name).all()
    
    return stats
```

### 6. Обновленный роутер для симуляций AnyLogic

**app/routers/simulations.py:**
```python
from fastapi import APIRouter, HTTPException, Depends, Query
from sqlalchemy.orm import Session
from anylogiccloudclient.client.cloud_client import CloudClient
import logging
from typing import List, Optional

from app.models import SimulationRequest, SimulationResponse, ErrorResponse, ModelInfo
from app.dependencies import get_cloud_client
from app.database import get_db
from app import crud
from app.schemas import SimulationResultCreate

logger = logging.getLogger(__name__)

router = APIRouter()

@router.post(
    "/simulations/run",
    response_model=SimulationResponse,
    responses={500: {"model": ErrorResponse}},
    summary="Запуск симуляции",
    description="Запускает симуляцию в AnyLogic Cloud и сохраняет результаты в локальную БД"
)
async def run_simulation(
    request: SimulationRequest,
    client: CloudClient = Depends(get_cloud_client),
    db: Session = Depends(get_db)
):
    """
    Запуск симуляции демо-модели Service System Demo с сохранением результатов
    """
    try:
        logger.info(f"Запуск симуляции с параметрами: {request.dict()}")
        
        # Получение последней версии модели
        version = client.get_latest_model_version(request.model_name)
        logger.info(f"Найдена версия модели: {version.id}")
        
        # Создание входных параметров
        inputs = client.create_inputs_from_experiment(version, request.experiment_name)
        
        # Установка параметров
        inputs.set_input("Server capacity", request.server_capacity)
        
        # Создание и запуск симуляции
        simulation = client.create_simulation(inputs)
        logger.info(f"Создана симуляция с ID: {simulation.id}")
        
        # Получение результатов
        outputs = simulation.get_outputs_and_run_if_absent()
        logger.info("Симуляция завершена, получены результаты")
        
        # Извлечение данных
        mean_queue_size = outputs.value("Mean queue size|Mean queue size")
        server_utilization = outputs.value("Utilization|Server utilization")
        raw_outputs = outputs.get_raw_outputs()
        
        db_record_id = None
        
        # Сохранение в локальную БД если требуется
        if request.save_to_db:
            result_data = SimulationResultCreate(
                simulation_id=simulation.id,
                model_name=request.model_name,
                experiment_name=request.experiment_name,
                server_capacity=request.server_capacity,
                mean_queue_size=mean_queue_size,
                server_utilization=server_utilization,
                raw_outputs=raw_outputs
            )
            
            db_result = crud.create_simulation_result(db, result_data)
            db_record_id = db_result.id
            logger.info(f"Результат сохранен в БД с ID: {db_record_id}")
        
        return SimulationResponse(
            simulation_id=simulation.id,
            server_capacity=request.server_capacity,
            mean_queue_size=mean_queue_size,
            server_utilization=server_utilization,
            raw_outputs=raw_outputs,
            status="completed",
            db_record_id=db_record_id
        )
        
    except Exception as e:
        logger.error(f"Ошибка при выполнении симуляции: {str(e)}")
        raise HTTPException(
            status_code=500,
            detail=f"Ошибка моделирования: {str(e)}"
        )

@router.get("/models", response_model=List[ModelInfo])
async def get_models(client: CloudClient = Depends(get_cloud_client)):
    """
    Получение списка доступных моделей из AnyLogic Cloud
    """
    try:
        models = client.get_models()
        models_list = []
        
        for model in models:
            models_list.append(ModelInfo(
                id=model.id,
                name=model.name
            ))
        
        return models_list
        
    except Exception as e:
        raise HTTPException(
            status_code=500,
            detail=f"Ошибка получения списка моделей: {str(e)}"
        )

@router.get("/simulations/statistics")
async def get_simulation_statistics(db: Session = Depends(get_db)):
    """
    Получение статистики по выполненным симуляциям
    """
    try:
        stats = crud.get_simulation_statistics(db)
        return {"statistics": stats}
    except Exception as e:
        raise HTTPException(
            status_code=500,
            detail=f"Ошибка получения статистики: {str(e)}"
        )
```

### 7. Новый роутер для управления результатами (CRUD)

**app/routers/results.py:**
```python
from fastapi import APIRouter, HTTPException, Depends, Query
from sqlalchemy.orm import Session
from typing import List, Optional

from app.database import get_db
from app import crud
from app.models import SimulationResultResponse, SimulationResultUpdate, ErrorResponse

router = APIRouter(prefix="/results", tags=["results"])

@router.get("/", response_model=List[SimulationResultResponse])
def read_results(
    skip: int = Query(0, ge=0, description="Сколько записей пропустить"),
    limit: int = Query(100, ge=1, le=1000, description="Лимит записей"),
    model_name: Optional[str] = Query(None, description="Фильтр по названию модели"),
    db: Session = Depends(get_db)
):
    """
    Получить список всех результатов симуляций с пагинацией
    """
    results = crud.get_simulation_results(db, skip=skip, limit=limit, model_name=model_name)
    return results

@router.get("/{result_id}", response_model=SimulationResultResponse)
def read_result(result_id: int, db: Session = Depends(get_db)):
    """
    Получить результат симуляции по ID
    """
    db_result = crud.get_simulation_result(db, result_id=result_id)
    if db_result is None:
        raise HTTPException(status_code=404, detail="Result not found")
    return db_result

@router.get("/external/{simulation_id}", response_model=SimulationResultResponse)
def read_result_by_external_id(simulation_id: str, db: Session = Depends(get_db)):
    """
    Получить результат симуляции по ID из AnyLogic Cloud
    """
    db_result = crud.get_simulation_by_external_id(db, simulation_id=simulation_id)
    if db_result is None:
        raise HTTPException(status_code=404, detail="Result not found")
    return db_result

@router.put("/{result_id}", response_model=SimulationResultResponse)
def update_result(
    result_id: int, 
    result: SimulationResultUpdate,
    db: Session = Depends(get_db)
):
    """
    Обновить результат симуляции
    """
    db_result = crud.update_simulation_result(db, result_id=result_id, result=result)
    if db_result is None:
        raise HTTPException(status_code=404, detail="Result not found")
    return db_result

@router.delete("/{result_id}")
def delete_result(result_id: int, db: Session = Depends(get_db)):
    """
    Удалить результат симуляции
    """
    success = crud.delete_simulation_result(db, result_id=result_id)
    if not success:
        raise HTTPException(status_code=404, detail="Result not found")
    return {"message": "Result deleted successfully"}

@router.get("/analysis/comparison")
def compare_simulations(
    result_ids: str = Query(..., description="ID результатов для сравнения (через запятую)"),
    db: Session = Depends(get_db)
):
    """
    Сравнить несколько результатов симуляций
    """
    try:
        ids = [int(id.strip()) for id in result_ids.split(",")]
        results = []
        
        for result_id in ids:
            result = crud.get_simulation_result(db, result_id)
            if result:
                results.append({
                    "id": result.id,
                    "simulation_id": result.simulation_id,
                    "model_name": result.model_name,
                    "server_capacity": result.server_capacity,
                    "mean_queue_size": result.mean_queue_size,
                    "server_utilization": result.server_utilization,
                    "created_at": result.created_at
                })
        
        if not results:
            raise HTTPException(status_code=404, detail="No results found")
        
        # Анализ сравнения
        comparison = {
            "results": results,
            "summary": {
                "min_queue_size": min(r["mean_queue_size"] for r in results),
                "max_queue_size": max(r["mean_queue_size"] for r in results),
                "min_utilization": min(r["server_utilization"] for r in results),
                "max_utilization": max(r["server_utilization"] for r in results),
                "best_performance": min(results, key=lambda x: x["mean_queue_size"])
            }
        }
        
        return comparison
        
    except ValueError:
        raise HTTPException(status_code=400, detail="Invalid result IDs format")
```

### 8. Обновленный главный файл приложения

**app/main.py:**
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
import os
from dotenv import load_dotenv

from app.routers import simulations, results
from app.database import Base, engine

# Создаем таблицы в БД
Base.metadata.create_all(bind=engine)

# Загрузка переменных окружения
load_dotenv()

app = FastAPI(
    title="AnyLogic Cloud API Integration",
    description="FastAPI приложение для работы с AnyLogic Cloud и управления результатами симуляций",
    version="2.0.0"
)

# Настройка CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Подключение роутеров
app.include_router(simulations.router, prefix="/api/v1", tags=["simulations"])
app.include_router(results.router, prefix="/api/v1", tags=["results"])

@app.get("/")
async def root():
    return {
        "message": "AnyLogic Cloud API Integration Service with CRUD",
        "version": "2.0.0",
        "features": [
            "AnyLogic Cloud integration",
            "CRUD operations for simulation results", 
            "Local database storage",
            "Simulation comparison and analysis"
        ]
    }

@app.get("/health")
async def health_check():
    return {"status": "healthy", "database": "connected"}
```

### 9. Запуск и тестирование

```bash
# Запуск сервера
uvicorn app.main:app --reload --port 8000

# Или с fastapi dev
fastapi dev app/main.py --port 8000
```

### 10. Примеры тестирования нового функционала

**Запуск симуляции с сохранением в БД:**
```bash
curl -X POST "http://127.0.0.1:8000/api/v1/simulations/run" \
  -H "Content-Type: application/json" \
  -d '{
    "server_capacity": 10,
    "model_name": "Service System Demo",
    "experiment_name": "Baseline",
    "save_to_db": true
  }'
```

**Получить все результаты:**
```bash
curl -X GET "http://127.0.0.1:8000/api/v1/results/"
```

**Получить статистику:**
```bash
curl -X GET "http://127.0.0.1:8000/api/v1/simulations/statistics"
```

**Сравнить результаты:**
```bash
curl -X GET "http://127.0.0.1:8000/api/v1/results/analysis/comparison?result_ids=1,2,3"
```

## 🎯 Ключевые улучшения:

1. **Интеграция CRUD**: Теперь есть полный CRUD для управления результатами симуляций
2. **Локальное хранение**: Все результаты сохраняются в SQLite БД
3. **Расширенный роутинг**: Добавлены новые endpoints для анализа и сравнения
4. **Статистика**: Возможность просмотра статистики по всем симуляциям
5. **Гибкость**: Можно запускать симуляции с сохранением в БД или без

Теперь ваш проект сочетает мощь AnyLogic Cloud API с гибкостью локального хранения данных и полноценным CRUD функционалом!