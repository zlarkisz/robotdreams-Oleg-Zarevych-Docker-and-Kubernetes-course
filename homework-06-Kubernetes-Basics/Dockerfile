# Базовий образ — Python 3.13 на Alpine Linux (мінімалістичний, ~50 MB)
FROM python:3.13-alpine

# Робоча директорія всередині контейнера
# Усі наступні COPY/RUN/CMD виконуються відносно /app
WORKDIR /app

# Створюємо non-root користувача для безпеки
# -D — без пароля, -H — без home директорії, -u 1000 — стандартний UID для non-root
# Якщо застосунок зламають, атакуючий не отримає root всередині контейнера
RUN adduser -D -H -u 1000 appuser

# Копіюємо requirements.txt ОКРЕМО від коду — це оптимізація кешу
# Якщо requirements.txt не змінився, Docker використає кеш для pip install
COPY requirements.txt .

# Встановлюємо Python-залежності
# --no-cache-dir — не зберігати завантажені .whl файли в ~/.cache/pip
# Економить ~30-50 MB у фінальному образі
RUN pip install --no-cache-dir -r requirements.txt

# Копіюємо код застосунку ПІСЛЯ залежностей
# Зміна коду не інвалідує кеш pip install
COPY src/ .

# Створюємо папку data/ і даємо права appuser на запис
# Застосунок створює тут data.sql (SQLite) при старті
# Без цього — PermissionError при запуску, бо /app належить root
# Точкова видача прав (тільки на data/) — principle of least privilege
RUN mkdir -p /app/data && chown appuser:appuser /app/data

# Перемикаємось на non-root користувача
# Усі наступні команди (включно з CMD) виконуються від appuser
USER appuser

# Документуємо порт на якому застосунок слухає
# Це метадані — фактичне відкриття порту робиться через -p, ports: в Compose
# або containerPort у K8s deployment.yaml
EXPOSE 8080

# Команда запуску при старті контейнера
# Exec-форма (JSON-масив) — щоб uvicorn отримав SIGTERM напряму при docker stop / kubectl delete
# --host 0.0.0.0 — слухати на всіх інтерфейсах, щоб приймати запити ззовні контейнера
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
