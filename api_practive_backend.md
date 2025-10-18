# Руководство по созданию FastAPI проекта для работы с AnyLogic Cloud API

## Создание проекта

### 1. Структура проекта
```
anylogic-fastapi-project/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── models.py
│   ├── dependencies.py
│   └── routers/
│       ├── __init__.py
│       └── simulations.py
├── requirements.txt
├── .env
└── README.md
```

### 2. Установка зависимостей
доустановите системные зависимости на всякий случай в корне проекта anylogic-fastapi-project/

```
# Установка FastAPI и стандартного набора для сервера (например, Uvicorn)
pip install "fastapi[standard]"

# Установка клиентской библиотеки AnyLogic Cloud для Python
pip install https://cloud.anylogic.com/files/api-8.5.0/clients/anylogiccloudclient-8.5.0-py3-none-any.whl
```

**requirements.txt:**
```txt
fastapi==0.104.1
uvicorn==0.24.0
python-dotenv==1.0.0
requests==2.31.0
pydantic==2.5.0
anylogiccloudclient==8.5.0
```

Альтернатива:

```
fastapi
uvicorn
python-dotenv
requests
pydantic
anylogiccloudclient
```

Установка:
```bash
pip install -r requirements.txt
```

### 3. Основной файл приложения

**app/main.py:**
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
import os
from dotenv import load_dotenv

from app.routers import simulations

# Загрузка переменных окружения
load_dotenv()

# Создание приложения FastAPI
app = FastAPI(
    title="AnyLogic Cloud API Integration",
    description="FastAPI приложение для работы с AnyLogic Cloud",
    version="1.0.0"
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

@app.get("/")
async def root():
    return {"message": "AnyLogic Cloud API Integration Service"}

@app.get("/health")
async def health_check():
    return {"status": "healthy"}
```

### 4. Модели данных

**app/models.py:**
```python
from pydantic import BaseModel
from typing import Dict, Any, Optional

class SimulationRequest(BaseModel):
    server_capacity: int = 8
    model_name: str = "Service System Demo"
    experiment_name: str = "Baseline"

class SimulationResponse(BaseModel):
    simulation_id: str
    server_capacity: int
    mean_queue_size: float
    server_utilization: float
    raw_outputs: Dict[str, Any]
    status: str

class ErrorResponse(BaseModel):
    error: str
    detail: Optional[str] = None
```

### 5. Зависимости и конфигурация

**app/dependencies.py:**
```python
import os
from anylogiccloudclient.client.cloud_client import CloudClient
from fastapi import HTTPException

def get_cloud_client():
    """Инициализация клиента AnyLogic Cloud"""
    api_key = os.getenv("ANYLOGIC_API_KEY", "e05a6efa-ea5f-4adf-b090-ae0ca7d16c20")
    try:
        return CloudClient(api_key)
    except Exception as e:
        raise HTTPException(
            status_code=500,
            detail=f"Ошибка инициализации клиента AnyLogic: {str(e)}"
        )
```

### 6. Роутер для симуляций

**app/routers/simulations.py:**
```python
from fastapi import APIRouter, HTTPException, Depends
from anylogiccloudclient.client.cloud_client import CloudClient
import logging

from app.models import SimulationRequest, SimulationResponse, ErrorResponse
from app.dependencies import get_cloud_client

# Настройка логирования
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

router = APIRouter()

@router.post(
    "/simulations/run",
    response_model=SimulationResponse,
    responses={500: {"model": ErrorResponse}}
)
async def run_simulation(
    request: SimulationRequest,
    client: CloudClient = Depends(get_cloud_client)
):
    """
    Запуск симуляции демо-модели Service System Demo
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
        
        return SimulationResponse(
            simulation_id=simulation.id,
            server_capacity=request.server_capacity,
            mean_queue_size=mean_queue_size,
            server_utilization=server_utilization,
            raw_outputs=outputs.get_raw_outputs(),
            status="completed"
        )
        
    except Exception as e:
        logger.error(f"Ошибка при выполнении симуляции: {str(e)}")
        raise HTTPException(
            status_code=500,
            detail=f"Ошибка моделирования: {str(e)}"
        )

@router.get("/models")
async def get_models(client: CloudClient = Depends(get_cloud_client)):
    """
    Получение списка доступных моделей
    """
    try:
        # Получение всех моделей
        models = client.get_models()
        models_list = []
        
        for model in models:
            models_list.append({
                "id": model.id,
                "name": model.name,
                "latest_version_id": model.latest_version.id if model.latest_version else None
            })
        
        return {"models": models_list}
        
    except Exception as e:
        raise HTTPException(
            status_code=500,
            detail=f"Ошибка получения списка моделей: {str(e)}"
        )
```

## Альтернативная реализация с прямыми REST API запросами

**app/services/anylogic_rest_client.py:**
```python
import requests
import os
from fastapi import HTTPException
import logging

logger = logging.getLogger(__name__)

class AnyLogicRESTClient:
    def __init__(self):
        self.api_key = os.getenv("ANYLOGIC_API_KEY", "e05a6efa-ea5f-4adf-b090-ae0ca7d16c20")
        self.base_url = "https://cloud.anylogic.com/api/open/8.5.0"
        self.headers = {
            "Authorization": self.api_key,
            "Content-Type": "application/json"
        }
    
    def get_models(self):
        """Получение списка моделей через REST API"""
        try:
            response = requests.get(
                f"{self.base_url}/models",
                headers=self.headers
            )
            response.raise_for_status()
            return response.json()
        except requests.RequestException as e:
            logger.error(f"Ошибка получения моделей: {str(e)}")
            raise HTTPException(status_code=500, detail="Ошибка подключения к AnyLogic Cloud")
    
    def run_simulation(self, version_id: str, inputs: dict):
        """Запуск симуляции через REST API"""
        try:
            # Создание прогона
            run_response = requests.post(
                f"{self.base_url}/versions/{version_id}/runs",
                headers=self.headers,
                json=inputs
            )
            run_response.raise_for_status()
            run_data = run_response.json()
            
            # Получение результатов
            results_response = requests.post(
                f"{self.base_url}/versions/{version_id}/results",
                headers=self.headers,
                json={"runId": run_data["id"]}
            )
            results_response.raise_for_status()
            
            return results_response.json()
            
        except requests.RequestException as e:
            logger.error(f"Ошибка выполнения симуляции: {str(e)}")
            raise HTTPException(status_code=500, detail="Ошибка выполнения симуляции")
```

## Запуск приложения

### 1. Создайте файл .env:
```env
ANYLOGIC_API_KEY=e05a6efa-ea5f-4adf-b090-ae0ca7d16c20
```

### 2. Запуск в development режиме:
```bash
# Из корневой директории проекта
fastapi dev app/main.py
```

### 3. Запуск в production режиме:
```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

## Тестирование API

После запуска откройте в браузере:
- Документация Swagger: http://127.0.0.1:8000/docs
- Альтернативная документация: http://127.0.0.1:8000/redoc

### Примеры запросов:

**GET /api/v1/models** - получение списка моделей

**POST /api/v1/simulations/run** - запуск симуляции
```json
{
  "server_capacity": 10,
  "model_name": "Service System Demo",
  "experiment_name": "Baseline"
}
```

## Дополнительные возможности

### 1. Добавление обработки ошибок
```python
@app.exception_handler(HTTPException)
async def http_exception_handler(request, exc):
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": exc.detail}
    )
```

### 2. Добавление middleware для логирования
```python
@app.middleware("http")
async def log_requests(request: Request, call_next):
    logger.info(f"Запрос: {request.method} {request.url}")
    response = await call_next(request)
    logger.info(f"Ответ: {response.status_code}")
    return response
```

### 3. Добавление rate limiting
```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@router.post("/simulations/run")
@limiter.limit("10/minute")
async def run_simulation(request: SimulationRequest, client: CloudClient = Depends(get_cloud_client)):
    # ваш код
```

Это руководство предоставляет полную структуру для создания FastAPI приложения, интегрированного с AnyLogic Cloud API, с поддержкой как клиентской библиотеки, так и прямых REST запросов.


## Что делать если порт занят:
Если порт 8000 уже занят другим процессом. Вот несколько способов решения этой проблемы:

## Способ 1: Освободить порт 8000

```bash
# Найти процесс, использующий порт 8000
sudo lsof -i :8000

# Завершить процесс (замените PID на фактический ID процесса)
kill -9 PID

# Или принудительно завершить все процессы на порту 8000
sudo fuser -k 8000/tcp
```

## Способ 2: Запустить на другом порту

```bash
# Запустить на порту 8001
fastapi dev app/main.py --port 8001

# Или с uvicorn напрямую
uvicorn app.main:app --host 0.0.0.0 --port 8001 --reload
```

## Способ 3: Использовать другой хост

```bash
# Запустить на другом интерфейсе
fastapi dev app/main.py --host 0.0.0.0 --port 8000
```

## Способ 4: Проверить запущенные Python процессы

```bash
# Найти все запущенные Python процессы
ps aux | grep python

# Завершить все процессы FastAPI/uvicorn
pkill -f uvicorn
pkill -f fastapi
```

## Быстрое решение для продолжения работы:

```bash
# Просто запустите на другом порту
fastapi dev app/main.py --port 8001
```

После этого документация будет доступна по адресу: http://127.0.0.1:8001/docs

## Дополнительные команды для диагностики:

```bash
# Проверить использование портов
netstat -tulpn | grep :8000

# Или используя ss
ss -tulpn | grep :8000

# Проверить использование портов конкретно Python
ss -tulpn | grep python
```

## Если проблема повторяется, создайте скрипт для автоматического освобождения порта:

**start_server.sh:**
```bash
#!/bin/bash
echo "Останавливаем существующие серверы на порту 8000..."
sudo fuser -k 8000/tcp 2>/dev/null
sleep 2
echo "Запускаем FastAPI сервер..."
fastapi dev app/main.py
```

Сделайте скрипт исполняемым и запустите:
```bash
chmod +x start_server.sh
./start_server.sh
```

**Рекомендую использовать порт 8001 для быстрого решения:**
```bash
fastapi dev app/main.py --port 8001
```

После успешного запуска вы сможете открыть документацию API по адресу http://127.0.0.1:8001/docs и протестировать эндпоинты.