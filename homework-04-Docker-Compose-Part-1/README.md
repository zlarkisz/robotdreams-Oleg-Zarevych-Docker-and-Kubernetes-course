# Домашнє завдання #04 — Docker Compose (Частина 1)

## Зміст

- [Середовище](#середовище)
- [Завдання 1 — Описати Compose файл для course-app](#завдання-1--описати-compose-файл-для-course-app)
- [Завдання 2 — Перевірити доступність застосунку](#завдання-2--перевірити-доступність-застосунку)
- [Завдання 3 — Додати Redis як зовнішнє сховище](#завдання-3--додати-redis-як-зовнішнє-сховище)
- [Завдання 4 — Додати змінні середовища](#завдання-4--додати-змінні-середовища)
- [Висновки](#висновки)

---

## Середовище

| Параметр       | Значення                     |
| -------------- | ---------------------------- |
| OS             | macOS (Apple Silicon, arm64) |
| Docker         | Docker Desktop v4.69.0       |
| Shell          | zsh                          |
| Дата виконання | 15 квітня 2026               |

---

## Завдання 1 — Описати Compose файл для course-app

### Що таке Docker Compose?

Docker Compose — інструмент для опису та запуску **багатоконтейнерних застосунків**. Замість запускати кожен контейнер окремою командою — описуємо всю інфраструктуру в одному `docker-compose.yaml` файлі.

> 💡 **Аналогія:** Якщо `docker run` — це запуск одного музиканта, то Docker Compose — це диригент який одночасно запускає весь оркестр за єдиною партитурою.

### Структура проєкту

```
course-app/
├── Dockerfile          # інструкція для збірки образу застосунку
├── docker-compose.yaml # опис всієї інфраструктури
├── requirements.txt    # Python залежності
└── src/
    └── main.py         # FastAPI застосунок
```

### Dockerfile

```dockerfile
# Базовий образ — Python 3.13 на Alpine Linux
# Alpine — мінімалістичний Linux (~5MB), завдяки чому образ важить ~50MB замість ~900MB
FROM python:3.13-alpine

# Встановлюємо робочу директорію всередині контейнера
WORKDIR /app

# Спочатку копіюємо ТІЛЬКИ requirements.txt (без коду)
# Це дозволяє Docker кешувати шар із залежностями окремо від коду
COPY requirements.txt .

# Встановлюємо Python залежності
# --no-cache-dir — не зберігати кеш pip всередині образу, зменшує розмір
RUN pip install --no-cache-dir -r requirements.txt

# Копіюємо код застосунку ПІСЛЯ залежностей
# Зміни в коді не інвалідують кеш pip
COPY src/ .

# Документуємо що застосунок слухає порт 8080
EXPOSE 8080

# Команда запуску застосунку при старті контейнера
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

### docker-compose.yaml

```yaml
services:
  course-app:
    build: .
    ports:
      - "8080:8080"
    environment:
      APP_STORE: redis
      APP_REDIS_URL: redis://redis:6379/0
    depends_on:
      - redis

  redis:
    image: redis:alpine
    volumes:
      - redis_data:/data

volumes:
  redis_data:
```

### Запуск

```bash
docker compose up -d
```

![docker compose up](screenshots/Screenshot%202026-04-15%20at%2021.42.48.png)

Docker виконав наступні кроки:

| Крок | Що відбулось |
|---|---|
| `Image redis:alpine Pulled` | Завантажив готовий образ Redis з Docker Hub |
| `[1/5] FROM python:3.13-alpine` | Завантажив базовий образ Python |
| `[4/5] RUN pip install` | Встановив Python залежності |
| `Network course-app_default Created` | Створив внутрішню мережу між контейнерами |
| `Volume course-app_redis_data Created` | Створив volume для збереження даних Redis |
| `Container course-app-redis-1 Started` | Redis запустився першим (завдяки `depends_on`) |
| `Container course-app-course-app-1 Started` | course-app запустився після Redis |

![docker desktop](screenshots/Screenshot%202026-04-15%20at%2021.43.24.png)

✅ Обидва контейнери запущені та працюють.

---

## Завдання 2 — Перевірити доступність застосунку

### http://localhost:8080

```bash
open http://localhost:8080
```

![localhost:8080](screenshots/Screenshot%202026-04-15%20at%2021.44.41.png)

### /healthz

```bash
curl http://localhost:8080/healthz
```

![healthz](screenshots/Screenshot%202026-04-15%20at%2021.45.40.png)

✅ Застосунок доступний на `http://localhost:8080`, `/healthz` повертає `{"status":"ok"}`.

---

## Завдання 3 — Додати Redis як зовнішнє сховище

### Що таке Redis і навіщо він тут?

За замовчуванням `course-app` зберігає дані в SQLite (локальний файл). Але в реальних проєктах потрібне **зовнішнє сховище** — щоб дані зберігались незалежно від контейнера і були доступні для кількох інстанцій застосунку.

> 💡 **Аналогія:** SQLite — це нотатки в телефоні, доступні тільки тобі. Redis — це Google Docs, до якого можуть підключитись одночасно кілька користувачів.

### Сервіс redis в Compose файлі

```yaml
redis:
  image: redis:alpine        # готовий образ з Docker Hub
  volumes:
    - redis_data:/data       # зберігаємо дані між перезапусками
```

### Як контейнери спілкуються між собою?

Docker Compose автоматично створює внутрішню мережу і резолвить імена сервісів як DNS:

```
course-app  →  "redis"  →  Docker DNS  →  IP контейнера redis
```

Тому в `APP_REDIS_URL` використовуємо `redis` як хост, а не `localhost`.

| URL | Пояснення |
|---|---|
| `redis://redis:6379/0` | ✅ правильно — ім'я сервісу в Compose |
| `redis://localhost:6379/0` | ❌ неправильно — localhost це сам контейнер course-app |

### Named Volume vs Bind Mount

| Тип | Синтаксис | Де зберігається |
|---|---|---|
| Named volume | `redis_data:/data` | Docker керує сам |
| Bind mount | `./redis_data:/data` | Конкретна папка на хості |

Використовуємо named volume — рекомендований підхід для баз даних.

✅ Redis сервіс додано, дані зберігаються між перезапусками завдяки `redis_data` volume.

---

## Завдання 4 — Додати змінні середовища

### Навіщо змінні середовища?

Змінні середовища дозволяють **налаштовувати застосунок без зміни коду**. Застосунок читає їх через `os.getenv()`.

> 💡 **Аналогія:** Як `.env` файл у Vue/Nuxt проєкті — один і той самий код, різна поведінка залежно від середовища.

### Додані змінні

```yaml
environment:
  APP_STORE: redis              # перемикає бекенд з sqlite на redis
  APP_REDIS_URL: redis://redis:6379/0  # URL підключення до Redis
```

### Перевірка через /api/info

```bash
curl http://localhost:8080/api/info
```

![api/info](screenshots/Screenshot%202026-04-15%20at%2021.46.51.png)

Відповідь підтверджує:

| Поле | Значення | Статус |
|---|---|---|
| `store` | `redis` | ✅ |
| `redis_url` | `redis://redis:6379/0` | ✅ |
| `APP_STORE` | `redis` | ✅ |
| `APP_REDIS_URL` | `redis://redis:6379/0` | ✅ |

✅ Змінні середовища передані коректно, застосунок використовує Redis як сховище.

---

## Висновки

### Статус завдань

| # | Завдання | Статус |
|---|---|---|
| 1 | Описати Compose файл для course-app | ✅ |
| 2 | Перевірити `http://localhost:8080` і `/healthz` | ✅ |
| 3 | Додати Redis як зовнішнє сховище | ✅ |
| 4 | Додати змінні `APP_STORE` і `APP_REDIS_URL` | ✅ |

### Корисні команди

```bash
# Запуск
docker compose up -d              # запустити всі сервіси у фоні
docker compose up -d --build      # запустити з перебілдом образів

# Перегляд
docker compose ps                 # статус сервісів
docker compose logs               # логи всіх сервісів
docker compose logs -f course-app # логи конкретного сервісу в реальному часі

# Зупинка
docker compose down               # зупинити і видалити контейнери (volumes залишаються)
docker compose down -v            # зупинити і видалити разом з volumes

# Перевірка застосунку
curl http://localhost:8080/healthz  # перевірка здоров'я
curl http://localhost:8080/api/info # інформація про застосунок
```
