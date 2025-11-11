
## Пошаговая инструкция по добавлению роутинга

### Шаг 1: Изменяем структуру проекта

Создаем новую структуру с папкой для роутеров:

```
my_fastapi_project/
├── main.py
├── models.py
├── schemas.py
├── crud.py
├── database.py
├── requirements.txt
└── routers/          # ← НОВАЯ ПАПКА ДЛЯ РОУТЕРОВ
    ├── __init__.py
    └── items.py      # ← роутер для items
```

### Шаг 2: Создаем роутер для items

**`routers/items.py`** - выносим все эндпоинты связанные с items в отдельный файл:

```python
from fastapi import APIRouter, Depends, HTTPException, status, Query
from sqlalchemy.orm import Session
from typing import List, Optional

from database import get_db
from schemas import ItemCreate, ItemUpdate, ItemResponse
import crud

# Создаем роутер - это как "мини-приложение" FastAPI
router = APIRouter(
    prefix="/items",      # автоматически добавляет префикс /items ко всем путям
    tags=["items"],       # группирует эндпоинты в документации
    responses={404: {"description": "Not found"}}  # общие ответы для всех эндпоинтов
)

# CREATE - Создать элемент
@router.post(
    "/", 
    response_model=ItemResponse, 
    status_code=status.HTTP_201_CREATED,
    summary="Создать предмет",
    description="Создает новый предмет в базе данных"
)
def create_item(item: ItemCreate, db: Session = Depends(get_db)):
    """
    Создать новый предмет.
    
    - **title**: Название предмета (обязательно)
    - **description**: Описание предмета
    - **price**: Цена предмета
    """
    return crud.create_item(db=db, item=item)

# READ - Получить все элементы
@router.get(
    "/", 
    response_model=List[ItemResponse],
    summary="Получить все предметы",
    description="Возвращает список предметов с пагинацией"
)
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

# READ - Получить один элемент
@router.get(
    "/{item_id}", 
    response_model=ItemResponse,
    summary="Получить предмет по ID",
    description="Возвращает один предмет по его идентификатору"
)
def read_item(
    item_id: int,
    db: Session = Depends(get_db)
):
    db_item = crud.get_item(db, item_id=item_id)
    if db_item is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND, 
            detail="Item not found"
        )
    return db_item

# UPDATE - Обновить элемент
@router.put(
    "/{item_id}", 
    response_model=ItemResponse,
    summary="Обновить предмет",
    description="Обновляет данные предмета по ID"
)
def update_item(item_id: int, item: ItemUpdate, db: Session = Depends(get_db)):
    db_item = crud.update_item(db, item_id=item_id, item=item)
    if db_item is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND, 
            detail="Item not found"
        )
    return db_item

# DELETE - Удалить элемент
@router.delete(
    "/{item_id}",
    summary="Удалить предмет",
    description="Удаляет предмет по ID из базы данных"
)
def delete_item(item_id: int, db: Session = Depends(get_db)):
    success = crud.delete_item(db, item_id=item_id)
    if not success:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND, 
            detail="Item not found"
        )
    return {"message": "Item deleted successfully"}
```

### Шаг 3: Обновляем основной файл приложения

**`main.py`** - теперь он становится намного чище:

```python
from fastapi import FastAPI
from database import engine
from models import Base

# Импортируем роутеры
from routers import items

# Создаем таблицы
Base.metadata.create_all(bind=engine)

app = FastAPI(
    title="My CRUD API",
    description="Простое CRUD приложение на FastAPI с роутингом",
    version="1.0.0"
)

# Подключаем роутеры к приложению
app.include_router(items.router)

@app.get("/")
def read_root():
    return {"message": "Welcome to FastAPI CRUD API with routing"}

@app.get("/health")
def health_check():
    return {"status": "healthy", "version": "1.0.0"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Шаг 4: Добавляем дополнительные роутеры (опционально)

Давайте создадим еще один роутер для пользователей, чтобы показать масштабируемость:

**`routers/users.py`**:

```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from database import get_db

router = APIRouter(
    prefix="/users",
    tags=["users"],
    responses={404: {"description": "Not found"}}
)

# Простые примеры эндпоинтов для пользователей
@router.get("/")
def get_users(db: Session = Depends(get_db)):
    return {"message": "List of users"}

@router.get("/{user_id}")
def get_user(user_id: int, db: Session = Depends(get_db)):
    return {"message": f"User {user_id}"}

@router.post("/")
def create_user(db: Session = Depends(get_db)):
    return {"message": "User created"}
```

Обновляем **`main.py`** чтобы подключить новый роутер:

```python
from fastapi import FastAPI
from database import engine
from models import Base

# Импортируем все роутеры
from routers import items, users  # ← добавляем импорт users

# Создаем таблицы
Base.metadata.create_all(bind=engine)

app = FastAPI(
    title="My CRUD API",
    description="Простое CRUD приложение на FastAPI с роутингом",
    version="1.0.0"
)

# Подключаем все роутеры
app.include_router(items.router)
app.include_router(users.router)  # ← подключаем роутер пользователей

@app.get("/")
def read_root():
    return {"message": "Welcome to FastAPI CRUD API with routing"}

@app.get("/health")
def health_check():
    return {"status": "healthy", "version": "1.0.0"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Шаг 5: Создаем файл для объединения роутеров (опционально)

**`routers/__init__.py`**:

```python
# Этот файл делает папку routers Python пакетом
# Можно использовать для импорта всех роутеров сразу

from .items import router as items_router
from .users import router as users_router

# Список всех роутеров для удобного импорта
__all__ = ["items_router", "users_router"]
```

Теперь в `main.py` можно импортировать так:

```python
from routers import items_router, users_router

app.include_router(items_router)
app.include_router(users_router)
```

## Результат после рефакторинга

### Новая структура API:

```
GET    /                 # корневой эндпоинт
GET    /health          # проверка здоровья

POST   /items/          # создать item
GET    /items/          # получить все items  
GET    /items/{id}      # получить item по ID
PUT    /items/{id}      # обновить item
DELETE /items/{id}      # удалить item

GET    /users/          # получить всех users
GET    /users/{id}      # получить user по ID
POST   /users/          # создать user
```

### Преимущества нового подхода:

1. **Модульность**: Каждая функциональность в отдельном файле
2. **Масштабируемость**: Легко добавлять новые роутеры
3. **Читаемость**: Код лучше организован
4. **Поддержка**: Легче находить и исправлять ошибки
5. **Тестирование**: Можно тестировать роутеры по отдельности

### Запуск и тестирование:

```bash
uvicorn main:app --reload
```

Перейдите по адресу http://127.0.0.1:8000/docs чтобы увидеть:

- Все эндпоинты сгруппированы по тегам (items, users)
- Четкую структуру API
- Автоматическую документацию

### Дальнейшее развитие:

- Добавьте аутентификацию как отдельный роутер
- Создайте роутеры для других сущностей
- Добавьте middleware в конкретные роутеры
- Используйте под-роутеры для вложенных путей

Добавим интеграцию с ChatGPT API к вашему проекту.

## Шаг 1: Установка дополнительных зависимостей

```bash
pip install openai python-multipart httpx
```

Обновите `requirements.txt`:
```txt
fastapi
uvicorn
sqlalchemy
python-multipart
openai
httpx
```

## Шаг 2: Добавляем конфигурацию и утилиты

### `config.py` - настройки приложения
```python
import os
from dotenv import load_dotenv

load_dotenv()

class Settings:
    # OpenAI API настройки
    OPENAI_API_KEY: str = os.getenv("OPENAI_API_KEY", "")
    OPENAI_MODEL: str = os.getenv("OPENAI_MODEL", "gpt-3.5-turbo")
    
    # Настройки приложения
    APP_NAME: str = "FastAPI ChatGPT Integration"
    DEBUG: bool = os.getenv("DEBUG", "False").lower() == "true"

settings = Settings()
```

### `services/chatgpt_service.py` - сервис для работы с ChatGPT
```python
import logging
from openai import OpenAI
from config import settings
from typing import List, Dict, Optional

logger = logging.getLogger(__name__)

class ChatGPTService:
    def __init__(self):
        self.client = OpenAI(api_key=settings.OPENAI_API_KEY)
        self.model = settings.OPENAI_MODEL
    
    async def get_chat_completion(
        self, 
        messages: List[Dict[str, str]],
        temperature: float = 0.7,
        max_tokens: int = 1000
    ) -> Optional[str]:
        """
        Получить ответ от ChatGPT
        
        Args:
            messages: Список сообщений в формате [{"role": "user", "content": "текст"}]
            temperature: Креативность ответа (0-1)
            max_tokens: Максимальное количество токенов в ответе
            
        Returns:
            Текст ответа или None в случае ошибки
        """
        try:
            response = self.client.chat.completions.create(
                model=self.model,
                messages=messages,
                temperature=temperature,
                max_tokens=max_tokens
            )
            
            return response.choices[0].message.content
            
        except Exception as e:
            logger.error(f"Error calling ChatGPT API: {str(e)}")
            return None
    
    async def analyze_item_description(self, description: str) -> str:
        """Анализировать описание товара с помощью ChatGPT"""
        prompt = f"""
        Проанализируй описание товара и предоставь краткий анализ:
        
        Описание: {description}
        
        Сделай анализ по следующим пунктам:
        1. Ключевые характеристики товара
        2. Потенциальная целевая аудитория  
        3. Рекомендации по улучшению описания
        4. Оценка убедительности описания (1-10)
        
        Ответ предоставь в структурированном виде.
        """
        
        messages = [
            {"role": "system", "content": "Ты помощник для анализа товаров в интернет-магазине."},
            {"role": "user", "content": prompt}
        ]
        
        return await self.get_chat_completion(messages)
    
    async def generate_item_title(self, description: str) -> str:
        """Сгенерировать заголовок для товара на основе описания"""
        prompt = f"""
        На основе этого описания товара придумай 3 привлекательных заголовка:
        
        Описание: {description}
        
        Требования к заголовкам:
        - Не более 60 символов
        - Привлекательные для покупателей
        - Соответствуют описанию
        - Разные стили (прямой, вопрос, преимущество)
        
        Верни только заголовки через запятую.
        """
        
        messages = [
            {"role": "system", "content": "Ты специалист по маркетингу и копирайтингу."},
            {"role": "user", "content": prompt}
        ]
        
        return await self.get_chat_completion(messages)

# Создаем экземпляр сервиса
chatgpt_service = ChatGPTService()
```

## Шаг 3: Добавляем схемы для ChatGPT

### `schemas/chatgpt_schemas.py` - схемы для работы с ChatGPT
```python
from pydantic import BaseModel, Field
from typing import Optional, List, Dict

class ChatGPTPrompt(BaseModel):
    messages: List[Dict[str, str]] = Field(
        ...,
        example=[
            {"role": "system", "content": "You are a helpful assistant."},
            {"role": "user", "content": "Hello!"}
        ]
    )
    temperature: float = Field(0.7, ge=0, le=1)
    max_tokens: int = Field(1000, ge=1, le=4000)

class ChatGPTResponse(BaseModel):
    success: bool
    response: Optional[str] = None
    error: Optional[str] = None

class ItemAnalysisRequest(BaseModel):
    description: str = Field(..., min_length=10, description="Описание товара для анализа")

class ItemAnalysisResponse(BaseModel):
    original_description: str
    analysis: str
    generated_titles: str

class TitleGenerationRequest(BaseModel):
    description: str = Field(..., min_length=10)
    style: Optional[str] = Field("marketing", description="Стиль заголовков")

class TitleGenerationResponse(BaseModel):
    original_description: str
    generated_titles: str
```

## Шаг 4: Создаем роутер для ChatGPT

### `routers/chatgpt.py` - роутер для ChatGPT API
```python
from fastapi import APIRouter, Depends, HTTPException, status
from typing import List

from schemas.chatgpt_schemas import (
    ChatGPTPrompt, 
    ChatGPTResponse,
    ItemAnalysisRequest,
    ItemAnalysisResponse,
    TitleGenerationRequest,
    TitleGenerationResponse
)
from services.chatgpt_service import chatgpt_service
from config import settings

router = APIRouter(
    prefix="/chatgpt",
    tags=["chatgpt"],
    responses={404: {"description": "Not found"}}
)

@router.post("/chat", response_model=ChatGPTResponse)
async def chat_with_gpt(prompt: ChatGPTPrompt):
    """
    Общий эндпоинт для общения с ChatGPT
    
    - **messages**: Список сообщений в формате OpenAI
    - **temperature**: Креативность ответа (0-1)
    - **max_tokens**: Максимальная длина ответа
    """
    if not settings.OPENAI_API_KEY:
        raise HTTPException(
            status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
            detail="ChatGPT service is not configured"
        )
    
    response = await chatgpt_service.get_chat_completion(
        messages=prompt.messages,
        temperature=prompt.temperature,
        max_tokens=prompt.max_tokens
    )
    
    if response is None:
        return ChatGPTResponse(
            success=False,
            error="Failed to get response from ChatGPT"
        )
    
    return ChatGPTResponse(success=True, response=response)

@router.post("/analyze-item", response_model=ItemAnalysisResponse)
async def analyze_item_description(request: ItemAnalysisRequest):
    """
    Проанализировать описание товара с помощью ChatGPT
    
    - **description**: Описание товара для анализа
    """
    if not settings.OPENAI_API_KEY:
        raise HTTPException(
            status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
            detail="ChatGPT service is not configured"
        )
    
    analysis = await chatgpt_service.analyze_item_description(request.description)
    titles = await chatgpt_service.generate_item_title(request.description)
    
    if analysis is None:
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail="Failed to analyze item description"
        )
    
    return ItemAnalysisResponse(
        original_description=request.description,
        analysis=analysis,
        generated_titles=titles or "Не удалось сгенерировать заголовки"
    )

@router.post("/generate-titles", response_model=TitleGenerationResponse)
async def generate_item_titles(request: TitleGenerationRequest):
    """
    Сгенерировать заголовки для товара
    
    - **description**: Описание товара
    - **style**: Стиль заголовков (marketing, professional, creative)
    """
    if not settings.OPENAI_API_KEY:
        raise HTTPException(
            status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
            detail="ChatGPT service is not configured"
        )
    
    generated_titles = await chatgpt_service.generate_item_title(request.description)
    
    if generated_titles is None:
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail="Failed to generate titles"
        )
    
    return TitleGenerationResponse(
        original_description=request.description,
        generated_titles=generated_titles
    )

@router.get("/health")
async def chatgpt_health_check():
    """Проверка доступности ChatGPT сервиса"""
    return {
        "service": "chatgpt",
        "status": "available" if settings.OPENAI_API_KEY else "not_configured",
        "model": settings.OPENAI_MODEL
    }
```

## Шаг 5: Обновляем роутер items с интеграцией ChatGPT

### `routers/items.py` - обновленная версия с ChatGPT
```python
from fastapi import APIRouter, Depends, HTTPException, status, Query, BackgroundTasks
from sqlalchemy.orm import Session
from typing import List, Optional

from database import get_db
from schemas import ItemCreate, ItemUpdate, ItemResponse
from schemas.chatgpt_schemas import ItemAnalysisResponse
import crud
from services.chatgpt_service import chatgpt_service
from config import settings

router = APIRouter(
    prefix="/items",
    tags=["items"],
    responses={404: {"description": "Not found"}}
)

# Добавляем новые эндпоинты с ChatGPT интеграцией
@router.post(
    "/{item_id}/analyze", 
    response_model=ItemAnalysisResponse,
    summary="Проанализировать товар с помощью AI",
    description="Использует ChatGPT для анализа описания товара"
)
async def analyze_item(
    item_id: int,
    db: Session = Depends(get_db)
):
    """
    Проанализировать описание товара с помощью ChatGPT
    
    - **item_id**: ID товара для анализа
    """
    if not settings.OPENAI_API_KEY:
        raise HTTPException(
            status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
            detail="ChatGPT service is not configured"
        )
    
    # Получаем товар из БД
    db_item = crud.get_item(db, item_id=item_id)
    if db_item is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND, 
            detail="Item not found"
        )
    
    # Если у товара нет описания
    if not db_item.description:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Item doesn't have description to analyze"
        )
    
    # Анализируем описание с помощью ChatGPT
    analysis = await chatgpt_service.analyze_item_description(db_item.description)
    titles = await chatgpt_service.generate_item_title(db_item.description)
    
    if analysis is None:
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail="Failed to analyze item description"
        )
    
    return ItemAnalysisResponse(
        original_description=db_item.description,
        analysis=analysis,
        generated_titles=titles or "Не удалось сгенерировать заголовки"
    )

@router.post(
    "/{item_id}/suggest-improvements",
    summary="Предложить улучшения для товара",
    description="Генерирует рекомендации по улучшению описания товара"
)
async def suggest_improvements(
    item_id: int,
    db: Session = Depends(get_db)
):
    """
    Сгенерировать рекомендации по улучшению товара
    
    - **item_id**: ID товара
    """
    db_item = crud.get_item(db, item_id=item_id)
    if db_item is None:
        raise HTTPException(status_code=404, detail="Item not found")
    
    if not db_item.description:
        raise HTTPException(status_code=400, detail="Item doesn't have description")
    
    prompt = f"""
    Проанализируй описание товара и предложи конкретные улучшения:
    
    Текущее описание: {db_item.description}
    
    Предложи:
    1. 3 варианта улучшенного описания
    2. Ключевые слова для SEO
    3. Структуру для лучшего восприятия
    4. Призывы к действию
    
    Ответ предоставь в виде структурированного JSON.
    """
    
    messages = [
        {"role": "system", "content": "Ты эксперт по интернет-маркетингу и копирайтингу."},
        {"role": "user", "content": prompt}
    ]
    
    improvements = await chatgpt_service.get_chat_completion(messages)
    
    return {
        "item_id": item_id,
        "original_description": db_item.description,
        "suggestions": improvements or "Не удалось сгенерировать рекомендации"
    }

# Оригинальные CRUD эндпоинты остаются без изменений
@router.post("/", response_model=ItemResponse, status_code=status.HTTP_201_CREATED)
def create_item(item: ItemCreate, db: Session = Depends(get_db)):
    return crud.create_item(db=db, item=item)

@router.get("/", response_model=List[ItemResponse])
def read_items(
    skip: int = Query(0, ge=0, description="Сколько записей пропустить"),
    limit: int = Query(100, ge=1, le=1000, description="Лимит записей"),
    title: Optional[str] = Query(None, description="Фильтр по названию"),
    db: Session = Depends(get_db)
):
    items = crud.get_items(db, skip=skip, limit=limit, title_filter=title)
    return items

@router.get("/{item_id}", response_model=ItemResponse)
def read_item(item_id: int, db: Session = Depends(get_db)):
    db_item = crud.get_item(db, item_id=item_id)
    if db_item is None:
        raise HTTPException(status_code=404, detail="Item not found")
    return db_item

@router.put("/{item_id}", response_model=ItemResponse)
def update_item(item_id: int, item: ItemUpdate, db: Session = Depends(get_db)):
    db_item = crud.update_item(db, item_id=item_id, item=item)
    if db_item is None:
        raise HTTPException(status_code=404, detail="Item not found")
    return db_item

@router.delete("/{item_id}")
def delete_item(item_id: int, db: Session = Depends(get_db)):
    success = crud.delete_item(db, item_id=item_id)
    if not success:
        raise HTTPException(status_code=404, detail="Item not found")
    return {"message": "Item deleted successfully"}
```

## Шаг 6: Обновляем основной файл приложения

### `main.py` - добавляем ChatGPT роутер
```python
from fastapi import FastAPI
from database import engine
from models import Base
from config import settings

# Импортируем все роутеры
from routers import items, users, chatgpt

# Создаем таблицы
Base.metadata.create_all(bind=engine)

app = FastAPI(
    title="FastAPI + ChatGPT Integration",
    description="CRUD приложение с интеграцией ChatGPT API",
    version="2.0.0"
)

# Подключаем все роутеры
app.include_router(items.router)
app.include_router(users.router)
app.include_router(chatgpt.router)  # ← добавляем ChatGPT роутер

@app.get("/")
def read_root():
    return {
        "message": "Welcome to FastAPI CRUD API with ChatGPT Integration",
        "chatgpt_available": bool(settings.OPENAI_API_KEY),
        "endpoints": {
            "items": "/items",
            "users": "/users", 
            "chatgpt": "/chatgpt",
            "docs": "/docs"
        }
    }

@app.get("/health")
def health_check():
    return {
        "status": "healthy", 
        "version": "2.0.0",
        "services": {
            "database": "available",
            "chatgpt": "available" if settings.OPENAI_API_KEY else "not_configured"
        }
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Шаг 7: Создаем файл окружения

### `.env` - настройки окружения
```env
# OpenAI API Settings
OPENAI_API_KEY=your_openai_api_key_here
OPENAI_MODEL=gpt-3.5-turbo

# App Settings
DEBUG=True
```

## Шаг 8: Тестирование интеграции

Запускаем приложение:
```bash
uvicorn main:app --reload
```

### Примеры запросов к ChatGPT API:

**1. Общий чат с ChatGPT:**
```bash
curl -X POST "http://127.0.0.1:8000/chatgpt/chat" \
     -H "Content-Type: application/json" \
     -d '{
       "messages": [
         {"role": "system", "content": "You are a helpful assistant."},
         {"role": "user", "content": "Расскажи о преимуществах FastAPI"}
       ],
       "temperature": 0.7
     }'
```

**2. Анализ товара:**
```bash
curl -X POST "http://127.0.0.1:8000/items/1/analyze"
```

**3. Генерация заголовков:**
```bash
curl -X POST "http://127.0.0.1:8000/chatgpt/generate-titles" \
     -H "Content-Type: application/json" \
     -d '{
       "description": "Современный смартфон с большим экраном, мощным процессором и отличной камерой для съемки фото и видео в высоком качестве"
     }'
```

## Итоговая структура проекта:

```
my_fastapi_project/
├── main.py
├── config.py
├── models.py
├── schemas.py
├── crud.py
├── database.py
├── requirements.txt
├── .env
├── services/
│   └── chatgpt_service.py
├── schemas/
│   └── chatgpt_schemas.py
└── routers/
    ├── __init__.py
    ├── items.py
    ├── users.py
    └── chatgpt.py
```

## Преимущества такой интеграции:

1. **Модульность**: ChatGPT функциональность выделена в отдельные сервисы и роутеры
2. **Масштабируемость**: Легко добавлять новые AI-функции
3. **Безопасность**: Ключи API в конфигурации
4. **Обработка ошибок**: Грамотная обработка сбоев API
5. **Интеграция с существующей логикой**: ChatGPT дополняет CRUD операции
