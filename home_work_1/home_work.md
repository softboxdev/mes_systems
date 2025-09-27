Для тестирования AnyLogic Cloud API с демо-моделью через FastAPI вы можете использовать специальный тестовый ключ. Вот пошаговое руководство, которое поможет вам настроить подключение.

### 🧪 Использование тестового API-ключа

Для доступа к демо-модели вы можете использовать предоставленный AnyLogic тестовый ключ.

- **API-ключ**: `e05a6efa-ea5f-4adf-b090-ae0ca7d16c20`
- **Демо-модель**: Этот ключ предоставляет доступ к демонстрационной модели **"Service System Demo"**.

Имейте в виду, что полнофункциональный API-ключ для работы с вашими собственными моделями доступен только пользователям коммерческих версий AnyLogic Cloud.

### 🔧 Пример FastAPI приложения

Вот код простого FastAPI приложения, которое использует клиентскую библиотеку AnyLogic Cloud для запуска демо-модели и получения результатов.

```python
from fastapi import FastAPI, HTTPException
from anylogiccloudclient.client.cloud_client import CloudClient

# Инициализация FastAPI приложения и клиента AnyLogic Cloud
app = FastAPI()
client = CloudClient("e05a6efa-ea5f-4adf-b090-ae0ca7d16c20")

@app.get("/run-simulation/")
async def run_simulation(server_capacity: int = 8):
    """
    Запускает демо-модель 'Service System Demo' с заданным параметром 'Server capacity'.
    Возвращает ключевые результаты моделирования.
    """
    try:
        # Получение последней версии модели по имени
        version = client.get_latest_model_version("Service System Demo")
        
        # Создание входных параметров на основе эксперимента "Baseline"
        inputs = client.create_inputs_from_experiment(version, "Baseline")
        
        # Установка своего значения для параметра "Server capacity"
        inputs.set_input("Server capacity", server_capacity)
        
        # Создание и запуск симуляции
        simulation = client.create_simulation(inputs)
        
        # Ожидание завершения и получение результатов
        outputs = simulation.get_outputs_and_run_if_absent()
        
        # Извлечение конкретных выходных данных
        mean_queue_size = outputs.value("Mean queue size|Mean queue size")
        server_utilization = outputs.value("Utilization|Server utilization")
        
        return {
            "server_capacity": server_capacity,
            "mean_queue_size": mean_queue_size,
            "server_utilization": server_utilization,
            "raw_outputs": outputs.get_raw_outputs()  # Все сырые выходные данные
        }
        
    except Exception as e:
        raise HTTPException(status_code=500, detail=f"Ошибка моделирования: {str(e)}")
```

### 📦 Установка зависимостей

Перед запуском приложения установите необходимые библиотеки. Клиент AnyLogic Cloud нужно скачать по прямой ссылке.

```bash
# Установка FastAPI и стандартного набора для сервера (например, Uvicorn)
pip install "fastapi[standard]"

# Установка клиентской библиотеки AnyLogic Cloud для Python
pip install https://cloud.anylogic.com/files/api-8.5.0/clients/anylogiccloudclient-8.5.0-py3-none-any.whl
```

### 🚀 Запуск и проверка

1.  Сохраните код в файл, например, `main.py`.
2.  Запустите сервер с помощью команды:
    ```bash
    fastapi dev main.py
    ```
3.  Откройте браузере документацию API по адресу `http://127.0.0.1:8000/docs`. Вы сможете протестировать эндпоинт `/run-simulation/` прямо из интерфейса Swagger UI, меняя параметр `server_capacity`.

### 💡 Важные моменты при работе с API

- **Авторизация**: Во всех запросах к REST API необходимо передавать API-ключ в заголовке `Authorization`. В приведенном примере клиентская библиотека делает это автоматически.
- **Поиск модели**: Вместо жесткого прописывания ID модели или версии в коде удобно использовать методы вроде `get_latest_model_version("Model Name")`.
- **Асинхронность**: Для обработки нескольких одновременных запросов к вашего FastAPI приложению эффективно,сделать функцию эндпоинта асинхронной (используя `async def`), как показано в примере. Это предотвратит блокировку сервера на время выполнения симуляции.

### 🔄 Прямые REST API запросы

Пример выше использует готовую Python-библиотеку, которая упрощает работу. Если вам нужны именно прямые HTTP-запросы к REST API, вот ключевые моменты:

- **Базовый URL**: `https://cloud.anylogic.com/api/open/8.5.0`
- **Заголовок авторизации**: `Authorization: e05a6efa-ea5f-4adf-b090-ae0ca7d16c20`
- **Основные эндпоинты**:
    -   `GET /models` - для получения списка моделей.
    -   `POST /versions/{version-id}/runs` - для запуска прогона.
    -   `POST /versions/{version-id}/results` - для получения результатов.

Надеюсь, это руководство поможет вам начать работу.

https://www.anylogic.com/blog/python-api-for-simulations-in-anylogic-cloud

