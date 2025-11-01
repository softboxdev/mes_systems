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