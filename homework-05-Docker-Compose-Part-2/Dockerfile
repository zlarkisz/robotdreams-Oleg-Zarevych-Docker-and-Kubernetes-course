# Базовий образ — Python 3.13 на Alpine Linux (мінімалістичний, ~50 MB)
FROM python:3.13-alpine

# Встановлюємо curl — знадобиться для healthcheck у docker-compose.yaml
# --no-cache — не зберігати індекс пакетів у образі (економія місця)
RUN apk add --no-cache curl

# Робоча директорія всередині контейнера
# Усі наступні COPY/RUN/CMD виконуються відносно /app
WORKDIR /app

# Копіюємо requirements.txt ОКРЕМО від коду — це оптимізація кешу
# Якщо requirements.txt не змінився, Docker використає кеш для pip install
COPY requirements.txt .

# Встановлюємо Python-залежності
# --no-cache-dir — не зберігати завантажені .whl файли в ~/.cache/pip
RUN pip install --no-cache-dir -r requirements.txt

# Копіюємо код застосунку ПІСЛЯ залежностей
# Зміна коду не інвалідує кеш pip install
COPY src/ .

# Документуємо порт на якому застосунок слухає
# Це метадані — фактичне відкриття порту робиться через -p або ports: в Compose
EXPOSE 8080

# Команда запуску при старті контейнера
# Exec-форма (JSON-масив) — щоб uvicorn отримав SIGTERM напряму при docker stop
# --host 0.0.0.0 — слухати на всіх інтерфейсах, щоб приймати запити ззовні контейнера
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]