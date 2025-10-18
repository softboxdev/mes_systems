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

class ModelInfo(BaseModel):
    id: str
    name: str
    latest_version_id: Optional[str] = None
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
                "name": model.name
                
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

# Подробная инструкция по тестированию API для начинающих

## Способ 1: Тестирование через документацию Swagger UI (Самый простой)

### 1. Откройте документацию API
После запуска сервера откройте в браузере:
```
http://127.0.0.1:8000/docs
```
или если используете порт 8001:
```
http://127.0.0.1:8001/docs
```

### 2. Тестирование GET /api/v1/models

**В документации Swagger:**
1. Найдите раздел "simulations"
2. Найдите метод "GET /api/v1/models"
3. Нажмите кнопку "Try it out"
4. Нажмите "Execute"
5. Посмотрите результат в разделе "Responses"



### 3. Тестирование POST /api/v1/simulations/run

**В документации Swagger:**
1. Найдите метод "POST /api/v1/simulations/run"
2. Нажмите "Try it out"
3. Измените параметры в JSON (или оставьте значения по умолчанию):
```json
{
  "server_capacity": 10,
  "model_name": "Service System Demo",
  "experiment_name": "Baseline"
}
```
4. Нажмите "Execute"
5. Посмотрите результат

## Способ 2: Тестирование с помощью Python скриптов

### Создайте тестовый скрипт `test_api.py`:

```python
import requests
import json

# Базовый URL вашего API
BASE_URL = "http://127.0.0.1:8000/api/v1"

def test_get_models():
    """
    Тестирование GET запроса для получения списка моделей
    """
    print("=== Тестирование GET /api/v1/models ===")
    
    # Формируем полный URL для запроса
    url = f"{BASE_URL}/models"
    
    try:
        # Выполняем GET запрос
        response = requests.get(url)
        
        # Проверяем статус ответа
        print(f"Статус код: {response.status_code}")
        
        # Если запрос успешен (статус 200)
        if response.status_code == 200:
            # Преобразуем JSON ответ в словарь Python
            data = response.json()
            print("✅ Успешный ответ!")
            print(f"Получено моделей: {len(data.get('models', []))}")
            
            # Выводим информацию о каждой модели
            for model in data.get('models', []):
                print(f"  - Модель: {model.get('name')} (ID: {model.get('id')})")
                
        else:
            print("❌ Ошибка при запросе")
            print(f"Ответ: {response.text}")
            
    except Exception as e:
        print(f"❌ Произошла ошибка: {e}")

def test_post_simulation(server_capacity=8):
    """
    Тестирование POST запроса для запуска симуляции
    
    Args:
        server_capacity (int): Количество серверов для симуляции
    """
    print(f"\n=== Тестирование POST /api/v1/simulations/run ===")
    print(f"Параметр server_capacity: {server_capacity}")
    
    # Формируем полный URL для запроса
    url = f"{BASE_URL}/simulations/run"
    
    # Подготавливаем данные для отправки (тело запроса)
    payload = {
        "server_capacity": server_capacity,
        "model_name": "Service System Demo", 
        "experiment_name": "Baseline"
    }
    
    # Указываем заголовки (Content-Type для JSON)
    headers = {
        "Content-Type": "application/json"
    }
    
    try:
        # Выполняем POST запрос
        # json=payload автоматически преобразует словарь в JSON и устанавливает заголовки
        response = requests.post(url, json=payload, headers=headers)
        
        print(f"Статус код: {response.status_code}")
        
        if response.status_code == 200:
            data = response.json()
            print("✅ Симуляция успешно выполнена!")
            print(f"ID симуляции: {data.get('simulation_id')}")
            print(f"Размер очереди: {data.get('mean_queue_size')}")
            print(f"Загрузка серверов: {data.get('server_utilization')}")
            
            # Дополнительная информация
            print("\n📊 Детали результатов:")
            raw_outputs = data.get('raw_outputs', {})
            for key, value in list(raw_outputs.items())[:5]:  # Покажем первые 5 результатов
                print(f"  {key}: {value}")
                
        else:
            print("❌ Ошибка при выполнении симуляции")
            print(f"Ответ сервера: {response.text}")
            
    except Exception as e:
        print(f"❌ Произошла ошибка: {e}")

def test_multiple_simulations():
    """
    Тестирование нескольких симуляций с разными параметрами
    """
    print("\n=== Тестирование нескольких симуляций ===")
    
    # Тестируем с разным количеством серверов
    for capacity in [5, 8, 12, 15]:
        test_post_simulation(server_capacity=capacity)
        print("-" * 50)

if __name__ == "__main__":
    """
    Главная функция - точка входа в программу
    """
    print("🚀 Начало тестирования AnyLogic FastAPI")
    print("=" * 60)
    
    # Тест 1: Получение списка моделей
    test_get_models()
    
    # Тест 2: Одиночная симуляция
    test_post_simulation(server_capacity=10)
    
    # Тест 3: Несколько симуляций (раскомментируйте для теста)
    # test_multiple_simulations()
    
    print("\n" + "=" * 60)
    print("✅ Тестирование завершено!")
```

### Как запустить тестовый скрипт:

1. **Установите библиотеку requests** (если еще не установлена):
```bash
pip install requests
```

2. **Запустите скрипт**:
```bash
python test_api.py
```

## Способ 3: Тестирование с помощью curl (командная строка)

### GET запрос для получения моделей:
```bash
curl -X 'GET' \
  'http://127.0.0.1:8000/api/v1/models' \
  -H 'accept: application/json'
```

### POST запрос для запуска симуляции:
```bash
curl -X 'POST' \
  'http://127.0.0.1:8000/api/v1/simulations/run' \
  -H 'Content-Type: application/json' \
  -d '{
  "server_capacity": 10,
  "model_name": "Service System Demo",
  "experiment_name": "Baseline"
}'
```

## Способ 4: Тестирование через Postman

### Настройка запроса GET:
1. **Метод**: GET
2. **URL**: `http://127.0.0.1:8000/api/v1/models`
3. **Headers**: 
   - `Content-Type: application/json`

### Настройка запроса POST:
1. **Метод**: POST
2. **URL**: `http://127.0.0.1:8000/api/v1/simulations/run`
3. **Headers**:
   - `Content-Type: application/json`
4. **Body** (raw JSON):
```json
{
  "server_capacity": 10,
  "model_name": "Service System Demo",
  "experiment_name": "Baseline"
}
```

## Полный пример с обработкой ошибок

```python
import requests
import time

def advanced_api_test():
    """
    Продвинутое тестирование с обработкой ошибок и повторами
    """
    BASE_URL = "http://127.0.0.1:8000/api/v1"
    
    # Ждем пока сервер запустится
    print("⏳ Ожидание запуска сервера...")
    time.sleep(2)
    
    # Тестируем доступность сервера
    try:
        health_response = requests.get("http://127.0.0.1:8000/health", timeout=5)
        if health_response.status_code == 200:
            print("✅ Сервер доступен")
        else:
            print("⚠️ Сервер отвечает, но с ошибкой")
    except:
        print("❌ Сервер не доступен. Убедитесь, что он запущен на порту 8000")
        return
    
    # Тест получения моделей
    print("\n1. Тестируем получение списка моделей...")
    try:
        response = requests.get(f"{BASE_URL}/models", timeout=10)
        
        if response.status_code == 200:
            models = response.json().get('models', [])
            if models:
                print(f"✅ Найдено {len(models)} моделей:")
                for model in models:
                    print(f"   📁 {model['name']}")
            else:
                print("⚠️ Модели не найдены")
        else:
            print(f"❌ Ошибка HTTP {response.status_code}: {response.text}")
            
    except requests.exceptions.Timeout:
        print("❌ Таймаут запроса")
    except requests.exceptions.ConnectionError:
        print("❌ Ошибка подключения")
    except Exception as e:
        print(f"❌ Неожиданная ошибка: {e}")
    
    # Тест запуска симуляции
    print("\n2. Тестируем запуск симуляции...")
    test_data = [
        {"capacity": 5, "expected_queue": "high"},
        {"capacity": 8, "expected_queue": "medium"}, 
        {"capacity": 12, "expected_queue": "low"}
    ]
    
    for test in test_data:
        print(f"\n   🧪 Тест с {test['capacity']} серверами:")
        
        payload = {
            "server_capacity": test["capacity"],
            "model_name": "Service System Demo",
            "experiment_name": "Baseline"
        }
        
        try:
            response = requests.post(
                f"{BASE_URL}/simulations/run", 
                json=payload, 
                timeout=30  # Даем больше времени для симуляции
            )
            
            if response.status_code == 200:
                result = response.json()
                queue_size = result.get('mean_queue_size', 0)
                utilization = result.get('server_utilization', 0)
                
                print(f"      ✅ Успех! Очередь: {queue_size:.2f}, Загрузка: {utilization:.1%}")
            else:
                print(f"      ❌ Ошибка {response.status_code}: {response.text}")
                
        except requests.exceptions.Timeout:
            print("      ❌ Таймаут - симуляция заняла слишком много времени")
        except Exception as e:
            print(f"      ❌ Ошибка: {e}")

if __name__ == "__main__":
    advanced_api_test()
```

## Ожидаемые результаты при успешной работе:

### Для GET /api/v1/models:
```json
{
  "models": [
    {
      "id": "model-id-1",
      "name": "Service System Demo",
      "latest_version_id": "version-id-1"
    }
  ]
}
```

### Для POST /api/v1/simulations/run:
```json
{
  "simulation_id": "sim-12345",
  "server_capacity": 10,
  "mean_queue_size": 2.5,
  "server_utilization": 0.75,
  "raw_outputs": {
    "Mean queue size|Mean queue size": 2.5,
    "Utilization|Server utilization": 0.75
  },
  "status": "completed"
}
```

**Рекомендация для начинающих:** Начните с тестирования через Swagger UI (способ 1), затем переходите к Python скриптам для автоматизации тестирования.