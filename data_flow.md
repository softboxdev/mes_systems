# Полный разбор сборки FastAPI проекта 

## 🏗️ Архитектура проекта - как все собирается

```
anylogic-fastapi-project/
├── app/
│   ├── __init__.py              # Делает папку Python пакетом
│   ├── main.py                  # ⭐ ГЛАВНЫЙ ФАЙЛ - точка входа
│   ├── models.py                # ⭐ МОДЕЛИ ДАННЫХ - структуры request/response
│   ├── dependencies.py          # ⭐ ЗАВИСИМОСТИ - переиспользуемый код
│   └── routers/
│       ├── __init__.py          # Делает папку Python пакетом
│       └── simulations.py       # ⭐ РОУТЕРЫ - эндпоинты API
├── requirements.txt             # Библиотеки проекта
├── .env                         # Переменные окружения
└── README.md                    # Документация
```

## 🔄 Полный поток данных (от запроса до ответа)

### 1. **Запрос приходит в приложение**
```
Пользователь → HTTP запрос → FastAPI приложение
```

### 2. **main.py - Входная точка**
```python
# app/main.py
from fastapi import FastAPI
from app.routers import simulations  # ⭐ ИМПОРТ роутеров

app = FastAPI()  # Создаем ядро приложения

# ⭐ ПОДКЛЮЧАЕМ роутеры к главному приложению
app.include_router(simulations.router, prefix="/api/v1", tags=["simulations"])
```
**Что происходит:**
- FastAPI создает "контейнер" для всего приложения
- Роутеры подключаются с префиксом `/api/v1`
- Теперь все пути из `simulations.py` будут доступны по `/api/v1/...`

### 3. **models.py - Определяем структуры данных**
```python
# app/models.py
from pydantic import BaseModel

class SimulationRequest(BaseModel):
    server_capacity: int = 8
    model_name: str = "Service System Demo"
    # ⭐ ЭТИ ПОЛЯ будут автоматически ВАЛИДИРОВАТЬСЯ
    # когда придет JSON запрос

class SimulationResponse(BaseModel):
    simulation_id: str
    server_capacity: int  
    mean_queue_size: float
    # ⭐ ЭТИ ПОЛЯ будут автоматически ПРЕОБРАЗОВАНЫ в JSON
    # когда будем возвращать ответ
```

### 4. **dependencies.py - Переиспользуемый код**
```python
# app/dependencies.py
def get_cloud_client():
    # ⭐ ЭТА ФУНКЦИЯ будет вызываться ПЕРЕД каждым эндпоинтом
    # где указана зависимость Depends(get_cloud_client)
    return CloudClient(api_key)
```

### 5. **routers/simulations.py - Обработка запросов**
```python
# app/routers/simulations.py
@router.post("/simulations/run")
async def run_simulation(
    request: SimulationRequest,                    # ⭐ ВХОДНЫЕ ДАННЫЕ
    client: CloudClient = Depends(get_cloud_client) # ⭐ ЗАВИСИМОСТЬ
) -> SimulationResponse:                          # ⭐ ВЫХОДНЫЕ ДАННЫЕ
    # ЛОГИКА ОБРАБОТКИ...
    return SimulationResponse(...)                # ⭐ ВОЗВРАТ РЕЗУЛЬТАТА
```

## 🎯 Конкретный пример: POST /api/v1/simulations/run

### Шаг 1: Пользователь отправляет запрос
```bash
curl -X POST http://localhost:8000/api/v1/simulations/run \
  -H "Content-Type: application/json" \
  -d '{"server_capacity": 10}'
```

### Шаг 2: FastAPI получает запрос
```
HTTP POST /api/v1/simulations/run
Body: {"server_capacity": 10}
↓
FastAPI Application (main.py)
↓
Router: simulations.router
↓
Endpoint: run_simulation()
```

### Шаг 3: Автоматическая валидация входных данных
```python
# FastAPI автоматически делает это:
request_data = {"server_capacity": 10}
# ПРЕОБРАЗУЕТ в объект SimulationRequest:
simulation_request = SimulationRequest(
    server_capacity=10,
    model_name="Service System Demo",  # значение по умолчанию
    experiment_name="Baseline"         # значение по умолчанию
)
```

### Шаг 4: Выполнение зависимости
```python
# FastAPI вызывает:
client = get_cloud_client()  # из dependencies.py
# Возвращает готовый CloudClient для работы с AnyLogic
```

### Шаг 5: Выполнение бизнес-логики
```python
# Ваш код в эндпоинте:
version = client.get_latest_model_version(request.model_name)
inputs = client.create_inputs_from_experiment(version, request.experiment_name)
# ... и т.д.
```

### Шаг 6: Подготовка ответа
```python
# Создаем объект ответа:
response = SimulationResponse(
    simulation_id="sim-123",
    server_capacity=10,
    mean_queue_size=2.5,
    server_utilization=0.75,
    raw_outputs={...},
    status="completed"
)
```

### Шаг 7: Автоматическое преобразование в JSON
```python
# FastAPI автоматически конвертирует:
SimulationResponse → JSON
```
**Результат для пользователя:**
```json
{
  "simulation_id": "sim-123",
  "server_capacity": 10,
  "mean_queue_size": 2.5,
  "server_utilization": 0.75,
  "raw_outputs": {...},
  "status": "completed"
}
```

## 🔍 Детальный разбор полей - кто откуда берется

### Для POST /api/v1/simulations/run:

**ВХОДНЫЕ ПОЛЯ (SimulationRequest):**
```python
class SimulationRequest(BaseModel):
    server_capacity: int = 8                    # ← от пользователя или значение по умолчанию
    model_name: str = "Service System Demo"     # ← от пользователя или значение по умолчанию  
    experiment_name: str = "Baseline"           # ← от пользователя или значение по умолчанию
```

**ВЫХОДНЫЕ ПОЛЯ (SimulationResponse):**
```python
class SimulationResponse(BaseModel):
    simulation_id: str           # ← из simulation.id (AnyLogic)
    server_capacity: int         # ← из request.server_capacity (повторяем входной параметр)
    mean_queue_size: float       # ← из outputs.value("Mean queue size...") (AnyLogic)
    server_utilization: float    # ← из outputs.value("Utilization...") (AnyLogic)
    raw_outputs: Dict[str, Any]  # ← из outputs.get_raw_outputs() (AnyLogic)
    status: str                  # ← мы сами устанавливаем "completed"
```

### Для GET /api/v1/models:

**ВХОДНЫЕ ПОЛЯ:** нет (простой GET запрос без тела)

**ВЫХОДНЫЕ ПОЛЯ:**
```python
# В роутере мы возвращаем:
return {"models": models_list}

# Где models_list создается так:
models_list.append({
    "id": model.id,                          # ← из model.id (AnyLogic)
    "name": model.name,                      # ← из model.name (AnyLogic)
    "latest_version_id": model.latest_version.id  # ← из model.latest_version.id (AnyLogic)
})
```

## 🛠️ Процесс сборки проекта (по шагам)

### Шаг 1: Импорты и настройки
```python
# main.py импортирует роутеры
# simulations.py импортирует модели и зависимости
# dependencies.py импортирует CloudClient
```

### Шаг 2: Создание экземпляров
```python
# main.py: app = FastAPI() - создает ядро
# simulations.py: router = APIRouter() - создает роутер
# dependencies.py: CloudClient() - создает клиент при вызове
```

### Шаг 3: Регистрация компонентов
```python
# main.py: app.include_router() - "приклеивает" роутер к приложению
```

### Шаг 4: FastAPI анализирует ВСЕ эндпоинты
- Читает все `@router.get/post/put/delete`
- Анализирует типы параметров
- Создает документацию OpenAPI
- Настраивает валидацию

### Шаг 5: Готово к работе!
```python
# При запуске uvicorn/fastapi dev:
# 1. Загружаются все модули
# 2. Создается приложение
# 3. Запускается сервер
# 4. Документация доступна по /docs
```

## 🎓 Концепции для понимания:

### **Dependency Injection (Внедрение зависимостей)**
```python
# Вместо того чтобы создавать клиент в каждой функции:
def run_simulation(request: SimulationRequest):
    client = CloudClient(api_key)  # ❌ ПЛОХО - дублирование кода

# Мы используем зависимости:
def run_simulation(request: SimulationRequest, client = Depends(get_cloud_client)):
    # ✅ ХОРОШО - клиент создается автоматически
```

### **Pydantic Models - валидация и сериализация**
```python
# Автоматическая валидация:
request = SimulationRequest(server_capacity="not_a_number")  # ❌ ОШИБКА!

# Автоматическая сериализация:
response = SimulationResponse(simulation_id="123", ...)
return response  # ✅ Автоматически конвертируется в JSON
```

### **Router - модульность**
```python
# Разделяем эндпоинты по файлам:
# - simulations.py - все для симуляций
# - users.py - все для пользователей  
# - analytics.py - все для аналитики
```

## 🚀 Полный цикл запроса - визуализация:

```
[КЛИЕНТ]
    ↓ HTTP POST /api/v1/simulations/run + JSON
[FASTAPI APP - main.py]
    ↓ Маршрутизация
[ROUTER - simulations.py] 
    ↓ Валидация + Зависимости
[ENDPOINT - run_simulation()]
    ↓ Бизнес-логика + AnyLogic Cloud
[AnyLogic Cloud API]
    ↓ Результаты симуляции
[ENDPOINT - формирование ответа]
    ↓ Сериализация в JSON
[FASTAPI APP]
    ↓ HTTP Response + JSON
[КЛИЕНТ] ← Получает результат!
```

