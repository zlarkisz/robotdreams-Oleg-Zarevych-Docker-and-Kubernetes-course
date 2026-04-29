# Домашнє завдання #07 — Масштабування та оркестрація у Kubernetes

## Зміст

- [Середовище](#середовище)
- [Завдання 1 — Deployment з 10 репліками](#завдання-1--deployment-з-10-репліками)
- [Завдання 2 — Зміна ConfigMap і оновлення Pods](#завдання-2--зміна-configmap-і-оновлення-pods)
- [Завдання 3 — Оновлення образу контейнера та rollout](#завдання-3--оновлення-образу-контейнера-та-rollout)
- [Завдання 4 — RollingUpdate з різними `maxUnavailable` / `maxSurge`](#завдання-4--rollingupdate-з-різними-maxunavailable--maxsurge)
- [Завдання 5 — Стратегія Recreate](#завдання-5--стратегія-recreate)
- [Завдання 6 — Порівняння стратегій](#завдання-6--порівняння-стратегій)
- [Висновки](#висновки)

---

## Середовище

| Параметр   | Значення                     |
| ---------- | ---------------------------- |
| OS         | macOS (Apple Silicon, arm64) |
| Kubernetes | Docker Desktop, v1.34.1      |
| kubectl    | v1.34.1                      |
| Контекст   | `docker-desktop`             |
| Образ      | `nginx:1.25` / `nginx:1.27`  |

### Перевірка кластера

```bash
kubectl version --client
kubectl config current-context
kubectl get nodes
kubectl cluster-info
```

![cluster check](screenshots/Screenshot%202026-04-29%20at%2018.00.43.png)

✅ Один node `docker-desktop` у статусі `Ready`, control plane відповідає на `https://kubernetes.docker.internal:6443`.

---

## Завдання 1 — Deployment з 10 репліками

### Що таке Deployment, ReplicaSet, Pod

Це три рівні абстракції, які працюють разом:

- **Pod** — найменша одиниця, обгортка навколо контейнера. Сам по собі не самовідновлюється.
- **ReplicaSet** — контролер, який підтримує задану кількість Pods. Помер один — створює новий.
- **Deployment** — керує ReplicaSet. Дозволяє оновлювати, відкочуватись, тримає історію змін.

> 💡 **Аналогія:** Deployment — головний офіс мережі ресторанів, ReplicaSet — регіональний менеджер ("у мене має бути 10 закладів"), Pod — окремий заклад. Ти ніколи не створюєш Pod або ReplicaSet напряму — тільки Deployment.

### Маніфест

`manifests/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 10
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
```

| Поле                        | Значення                                                                   |
| --------------------------- | -------------------------------------------------------------------------- |
| `apiVersion: apps/v1`       | API-група для Deployment                                                   |
| `kind: Deployment`          | Тип ресурсу                                                                |
| `spec.replicas: 10`         | Бажана кількість Pods                                                      |
| `spec.selector.matchLabels` | Правило: "Deployment керує тими Pods, у яких є label `app: nginx`"         |
| `template.metadata.labels`  | Labels, які отримає кожен створений Pod. **Мусять збігатися з `selector`** |
| `containers[].image`        | Образ для запуску                                                          |

### Команди

```bash
kubectl apply -f manifests/deployment.yaml
kubectl get deployments
kubectl get replicasets
kubectl get pods
```

![deployment created](screenshots/Screenshot%202026-04-29%20at%2018.30.55.png)

✅ Створено Deployment `nginx-deployment` з 10 Running Pods. ReplicaSet з'явився сам — Deployment створив його автоматично, ім'я закінчується хешем `77bf8679f9` (це хеш від template Pod).

### Ієрархія в одному виводі

```bash
kubectl get all -l app=nginx
```

![get all](screenshots/Screenshot%202026-04-29%20at%2019.50.06.png)

✅ Видно ієрархію: 1 Deployment → 1 ReplicaSet → 10 Pods. Усі імена Pods починаються з `nginx-deployment-77bf8679f9-...` — тобто всі належать одному ReplicaSet.

---

## Завдання 2 — Зміна ConfigMap і оновлення Pods

### Що таке ConfigMap

**ConfigMap** — це key-value сховище для конфігурації, окреме від образу контейнера. Дозволяє один image використовувати в різних оточеннях (dev/staging/prod) з різними налаштуваннями.

> 💡 **Аналогія для frontend:** як `.env` файл, тільки керується Kubernetes-ом.

### Два способи підключення — і КРИТИЧНА різниця

| Спосіб                | Як працює                                 | Реакція на зміну ConfigMap                         |
| --------------------- | ----------------------------------------- | -------------------------------------------------- |
| **`env` / `envFrom`** | Значення копіюються в env vars при старті | ❌ **НЕ оновлюється** — потрібен `rollout restart` |
| **Volume mount**      | ConfigMap монтується як файл              | ✅ Оновлюється автоматично (~1 хв)                 |

### Маніфест ConfigMap

`manifests/configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  APP_VERSION: "v1"
  WELCOME_MESSAGE: "Hello from ConfigMap v1"
  index.html: |
    <!DOCTYPE html>
    <html>
      <head><title>ConfigMap demo</title></head>
      <body style="font-family: sans-serif; padding: 40px;">
        <h1>Версія: v1</h1>
        <p>Це сторінка з ConfigMap (volume mount).</p>
      </body>
    </html>
```

### Оновлений Deployment

У `deployment.yaml` додано **обидва** способи підключення — щоб на одному прикладі побачити різницю:

```yaml
env:
  - name: APP_VERSION
    valueFrom:
      configMapKeyRef:
        name: nginx-config
        key: APP_VERSION
  - name: WELCOME_MESSAGE
    valueFrom:
      configMapKeyRef:
        name: nginx-config
        key: WELCOME_MESSAGE
volumeMounts:
  - name: html-volume
    mountPath: /usr/share/nginx/html
# ... volumes на рівні Pod
volumes:
  - name: html-volume
    configMap:
      name: nginx-config
      items:
        - key: index.html
          path: index.html
```

### Застосування

```bash
kubectl apply -f manifests/configmap.yaml
kubectl apply -f manifests/deployment.yaml
```

![configmap applied](screenshots/Screenshot%202026-04-29%20at%2020.04.06.png)

✅ Створено ConfigMap і Deployment, який його використовує.

### Перевірка стартового стану (v1)

```bash
POD=$(kubectl get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -- env | grep -E "APP_VERSION|WELCOME_MESSAGE"
kubectl exec $POD -- cat /usr/share/nginx/html/index.html
```

![env and file v1](screenshots/Screenshot%202026-04-29%20at%2020.06.24.png)

✅ Усе синхронізовано на `v1`: ConfigMap v1 → env v1 → файл v1.

### Зміна ConfigMap на v2 — момент розбіжності

Замінив усі `v1` на `v2` у `configmap.yaml` і застосував:

```bash
kubectl apply -f manifests/configmap.yaml
kubectl exec $POD -- env | grep -E "APP_VERSION|WELCOME_MESSAGE"   # одразу
kubectl exec $POD -- cat /usr/share/nginx/html/index.html          # одразу
# ... зачекати ~1 хв
kubectl exec $POD -- cat /usr/share/nginx/html/index.html          # ще раз
```

![env stale, file updated](screenshots/Screenshot%202026-04-29%20at%2020.20.30.png)

⚠️ **Ключове спостереження** — у тому самому Pod:

| Що                | Стан після зміни ConfigMap  | Чому                                                                          |
| ----------------- | --------------------------- | ----------------------------------------------------------------------------- |
| `APP_VERSION` env | **`v1`** (стара!)           | Env скопіювався при старті, kubelet не може змінити змінні працюючого процесу |
| `index.html`      | **`v2`** (нова) через ~1 хв | kubelet періодично перезаписує файл у tmpfs                                   |

Це **не баг**, це фундаментальна властивість Linux-процесів: env vars живуть у пам'яті процесу, а файл — на диску.

### Як примусово оновити env vars

```bash
kubectl rollout restart deployment/nginx-deployment
kubectl rollout status deployment/nginx-deployment
POD=$(kubectl get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -- env | grep APP_VERSION
```

![rollout restart](screenshots/Screenshot%202026-04-29%20at%2020.22.18.png)

✅ Після `rollout restart` створюється новий ReplicaSet, Pods замінюються поступово (rolling restart без даунтайму), нові Pods читають актуальний ConfigMap → `APP_VERSION=v2`.

> 💡 `rollout restart` — це той самий rolling update, але без зміни image. Найчастіше використовується саме після зміни ConfigMap або Secret.

---

## Завдання 3 — Оновлення образу контейнера та rollout

### Команда

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.27
kubectl rollout status deployment/nginx-deployment
```

`set image` — швидкий imperative-спосіб оновити image без редагування YAML. У продакшні краще міняти image у YAML і робити `apply` (declarative, GitOps-friendly), але для експериментів `set image` зручніший.

### Що сталося — три ReplicaSets

```bash
kubectl get rs
kubectl rollout history deployment/nginx-deployment
```

![three replicasets](screenshots/Screenshot%202026-04-29%20at%2020.26.08.png)

| ReplicaSet                    | DESIRED | Що це                                  |
| ----------------------------- | ------- | -------------------------------------- |
| `nginx-deployment-7c9f477945` | 10      | Поточний (`nginx:1.27`)                |
| `nginx-deployment-58b9d4d4cf` | 0       | Попередній (`nginx:1.25` + ConfigMap)  |
| `nginx-deployment-74b86dfc79` | 0       | Найперший (`nginx:1.25` без ConfigMap) |

3 revisions у `rollout history` = 3 ReplicaSets. Кожне оновлення Deployment створює новий ReplicaSet, старі залишаються з 0 Pods для можливості швидкого rollback.

### AGE Pods показує що rollout йшов хвилями

`AGE` 10 нових Pods: **4 по 37s + 6 по 28-30s**. Це не випадковість — дефолтні `maxSurge=25%, maxUnavailable=25%` обмежують скільки Pods можна заміняти одночасно. Ми спеціально гратимемось з цим у завданні 4.

### Rollback — суперсила Deployment

```bash
kubectl rollout undo deployment/nginx-deployment
kubectl rollout status deployment/nginx-deployment
kubectl get rs
POD=$(kubectl get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -- nginx -v
```

![rollout undo](screenshots/Screenshot%202026-04-29%20at%2020.34.36.png)

✅ `kubectl rollout undo` за секунду повернуло на попередню revision (`nginx:1.25`). Зверни увагу: **новий ReplicaSet НЕ створюється** — Kubernetes "оживляє" попередній (`58b9d4d4cf`), масштабуючи його з 0 до 10.

> ⚠️ Класична пастка з `$POD`: після rollback змінна тримала ім'я померлого Pod (`nginx-deployment-7c9f477945-7p2s7 not found`). Довелось переприсвоїти. Імена Pods ефемерні — їх не можна кешувати.

Після перевірки повернув на `1.27` для подальших експериментів.

### Imperative vs Declarative

| Підхід          | Команда                                    | Коли                         |
| --------------- | ------------------------------------------ | ---------------------------- |
| **Imperative**  | `kubectl set image deployment/X nginx=...` | Швидкі експерименти, hot-fix |
| **Declarative** | Змінити YAML → `kubectl apply -f`          | Production, GitOps           |

`set image` змінює стан у кластері, але **не змінює YAML на диску** — призводить до drift між Git і кластером.

---

## Завдання 4 — RollingUpdate з різними `maxUnavailable` / `maxSurge`

### Параметри стратегії

| Параметр         | Що означає                                            | За замовчуванням |
| ---------------- | ----------------------------------------------------- | ---------------- |
| `maxUnavailable` | Скільки Pods можуть бути НЕдоступні під час оновлення | 25%              |
| `maxSurge`       | Скільки Pods можуть бути СТВОРЕНІ понад `replicas`    | 25%              |

⚠️ Обидва одночасно `0` заборонено — Kubernetes не може зрушити з місця.

### Експеримент A — `maxSurge=1, maxUnavailable=0` (повільно і безпечно)

Очікування: Pods оновлюються **по одному**, у кластері постійно щонайменше 10 Ready.

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

```bash
kubectl apply -f manifests/deployment.yaml
kubectl get pods -l app=nginx -w
```

![experiment A](screenshots/Screenshot%202026-04-29%20at%2020.44.07.png)

✅ Видно класичний "by-one" patterning у `STATUS`:

- `Pending → ContainerCreating → Running` для одного нового Pod
- `Terminating → Completed` для одного старого
- Повторюється 10 разів

Тривалість rollout: **~30 секунд**. Пік Pods: **11** (10 старих + 1 новий створюється). Min Ready: **10** — деградації немає.

### Експеримент B — `maxSurge=10, maxUnavailable=0` (blue-green подібний)

Очікування: усі 10 нових створяться **паралельно**, на кілька секунд буде 20 Pods, потім старі помруть разом.

```yaml
rollingUpdate:
  maxSurge: 10
  maxUnavailable: 0
```

![experiment B](screenshots/Screenshot%202026-04-29%20at%2020.49.33.png)

🔥 **Той самий пік 20 Pods зловлено!** Розбір timeline (правий термінал):

```
20:49:18  OLD: 10/10/10  NEW:  0/0/0     ← старт
20:49:19  OLD: 10/10/10  NEW: 10/10/0    ← БУМ! 10 нових створились одразу. У кластері 20 Pods!
20:49:20  OLD:  1/2/2    NEW: 10/10/9    ← старі починають вмирати, нові майже готові
20:49:21  OLD:  0/0/0    NEW: 10/10/10   ← все, готово
```

Тривалість: **~4 секунди**. Пік: **20 Pods**. Min Ready: **10**. Ціна — потрібно 2× ресурсів на момент rollout.

Лівий термінал показує шквал `Pending` для нових Pods — десять одночасно, без черг. Це і є `maxSurge=10` в дії.

### Експеримент C — `maxSurge=0, maxUnavailable=5` (агресивний)

Очікування: половина Pods (5) **зразу помре**, capacity тимчасово впаде до 50%.

```yaml
rollingUpdate:
  maxSurge: 0
  maxUnavailable: 5
```

![experiment C](screenshots/Screenshot%202026-04-29%20at%2020.54.38.png)

⚠️ **Видно деградацію 50%:**

```
20:54:09  OLD: 10/10/10  NEW:  0/0/0     ← старт
20:54:11  OLD:  5/5/5    NEW:  5/5/0     ← вбили 5 старих, створили 5 нових (READY=0!)
20:54:13  OLD:  0/0/0    NEW: 10/9/5     ← вбили решту, нові ще не Ready
20:54:14  OLD:  0/0/0    NEW: 10/10/5    ← 5 з 10 нових Ready
20:54:16  OLD:  0/0/0    NEW: 10/10/10   ← все ок
```

**~3-4 секунди кластер обслуговував лише 5 з 10 Pods.** Якщо це продакшн з трафіком — латенсі стрибнула б у 2 рази, могли б початись 503.

Лівий термінал показує **протилежний патерн B**: спочатку шквал `Terminating`, тільки потім `Pending`. Це і є `maxUnavailable` в дії — Kubernetes "вбиває не чекаючи нових".

### Зведена таблиця експериментів

| Експ. | `maxSurge` | `maxUnavailable` | Тривалість | Пік Pods | Min Ready | Що бачимо                    |
| ----- | ---------- | ---------------- | ---------- | -------- | --------- | ---------------------------- |
| A     | 1          | 0                | ~30 сек    | 11       | 10 ✅     | by-one, найбезпечніше        |
| B     | 10 (100%)  | 0                | ~4 сек     | **20**   | 10 ✅     | швидко, потрібно 2× ресурсів |
| C     | 0          | 5 (50%)          | ~5 сек     | 10       | **5** ⚠️  | швидко, але деградація 50%   |

---

## Завдання 5 — Стратегія Recreate

### Ідея

`Recreate` — антипод RollingUpdate:

1. Вбиває **ВСІ** старі Pods одночасно
2. Чекає поки всі помруть
3. Створює **ВСІ** нові Pods одночасно

Між кроками 2 і 3 у кластері **0 Pods** — повний даунтайм.

> 💡 **Аналогія:** RollingUpdate — ремонт магазину по секторах (один закритий, інші працюють). Recreate — "магазин закритий на санітарний день".

### Коли потрібен Recreate

- Stateful застосунок з єдиною копією БД (несумісні міграції schema)
- Backwards-incompatible зміни API (стара і нова версії не вміють спілкуватись)
- Singleton-додатки
- Volumes з режимом ReadWriteOnce (2 Pods не можуть тримати такий volume одночасно)

### Маніфест

```yaml
strategy:
  type: Recreate
```

Параметри `maxSurge`/`maxUnavailable` тут **заборонені** — API їх відхилить.

### Фаза вмирання

```bash
kubectl apply -f manifests/deployment.yaml
kubectl get pods -l app=nginx -w
```

![recreate killing](screenshots/Screenshot%202026-04-29%20at%2021.02.17.png)

✅ Усі 10 Pods переходять у `Terminating` **одночасно**. У RollingUpdate такого не було ніколи — там процес був хвильовим.

Правий термінал — момент тиші:

```
21:01:48  OLD: 10/10/10  NEW: 0/0/0    ← старі живі
21:01:50  OLD:  0/0/0    NEW: 0/0/0    ← 🔴 ВСІ МЕРТВІ! ZERO PODS!
21:01:51  OLD:  0/0/0    NEW: 10/0/0   ← створюємо 10 нових (READY=0)
21:01:52  OLD:  0/0/0    NEW: 10/10/0  ← створились, але READY=0
21:01:53  OLD:  0/0/0    NEW: 10/10/7  ← перші 7 готові
21:01:56  OLD:  0/0/0    NEW: 10/10/10 ← повне відновлення
```

**Даунтайм ≈ 3 секунди** (з 21:01:50 до 21:01:53), коли жодного Ready Pod не було.

### Фаза народження

![recreate creating](screenshots/Screenshot%202026-04-29%20at%2021.02.36.png)

Усі 10 нових Pods з'являються паралельно: `Pending → ContainerCreating → Running`. Зовнішньо схоже на експеримент B, але з принциповою відмінністю — до цього моменту в кластері **було 0 Pods**.

### Events підтверджують короткий і чіткий ланцюжок

```bash
kubectl describe deployment nginx-deployment | grep -A 30 "Events"
```

![recreate events](screenshots/Screenshot%202026-04-29%20at%2021.04.00.png)

✅ Видно весь життєвий цикл deployment-а — від першого створення (`74b86dfc79 from 0 to 10`) через RollingUpdate експерименти (хвильові `from 10 to 8`, `from 3 to 5`) до завершальних подій. У RollingUpdate експериментах події хвильові по 1-3 Pods за раз, у Recreate — пакетні (`to 0` → `to 10`).

---

## Завдання 6 — Порівняння стратегій

### Таблиця стратегій

| Аспект                                | RollingUpdate                    | Recreate                         |
| ------------------------------------- | -------------------------------- | -------------------------------- |
| **Як працює**                         | Поступово замінює Pods           | Вбиває всі → створює всі         |
| **Даунтайм**                          | Немає                            | **Є** (поки нові не Ready)       |
| **Швидкість**                         | Повільніше (чекає на готовність) | Швидше (без проміжних кроків)    |
| **Додаткові ресурси під час rollout** | Так (через `maxSurge`)           | Не потрібні                      |
| **Дві версії одночасно?**             | Так (під час rollout)            | Ні                               |
| **За замовчуванням**                  | ✅                               | ❌ (треба явно вказати)          |
| **Підходить для**                     | Stateless, веб-сервіси, API      | Stateful, БД-міграції, singleton |

### Переваги і недоліки

**RollingUpdate:**

✅ Zero downtime, гнучкість через параметри, безпечно для більшості веб-додатків, легкий rollback.

❌ Тимчасово паралельно існують 2 версії — потрібна **backwards-compatibility**. Потребує ресурси під час rollout. Повільніше за Recreate.

**Recreate:**

✅ Простота — ніколи немає 2 версій одночасно. Не потрібні додаткові ресурси. Обов'язковий для несумісних версій.

❌ **Даунтайм** — від секунд до хвилин. Не підходить для production user-facing сервісів.

### Як обрати — decision tree

```
Чи можуть стара і нова версія працювати одночасно?
├─ ТАК (95% веб-додатків) → RollingUpdate
│   ├─ Багато ресурсів?         → maxSurge=25-100%, maxUnavailable=0
│   ├─ Обмежені ресурси?        → maxSurge=1, maxUnavailable=1
│   └─ Можна терпіти просідання → maxSurge=0, maxUnavailable=25%
│
└─ НІ (БД-міграції, singleton, RWO volumes) → Recreate
```

### Спостережувані метрики (10 replicas)

| Стратегія     | Параметри                       | Час     | Пік Pods | Min Ready | Даунтайм   |
| ------------- | ------------------------------- | ------- | -------- | --------- | ---------- |
| RollingUpdate | `maxSurge=1, maxUnavailable=0`  | ~30 сек | 11       | 10        | немає      |
| RollingUpdate | `maxSurge=10, maxUnavailable=0` | ~4 сек  | **20**   | 10        | немає      |
| RollingUpdate | `maxSurge=0, maxUnavailable=5`  | ~5 сек  | 10       | **5**     | деградація |
| Recreate      | (без параметрів)                | ~6 сек  | 10       | **0**     | **~3 сек** |

---

## Висновки

### Статус завдань

| #   | Завдання                                                                  | Статус |
| --- | ------------------------------------------------------------------------- | ------ |
| 1   | Описати Deployment з мінімум 10 репліками                                 | ✅     |
| 2   | Змінити значення у ConfigMap і перевірити оновлення Pods                  | ✅     |
| 3   | Оновити образ контейнера та простежити rollout                            | ✅     |
| 4   | Дослідити RollingUpdate з різними `maxUnavailable` / `maxSurge` (3 експ.) | ✅     |
| 5   | Спробувати Replace (Recreate) стратегію                                   | ✅     |
| 6   | Пояснити переваги, недоліки та відмінність                                | ✅     |

### Що засвоєно (ключові інсайти)

1. **Ієрархія** — Deployment керує ReplicaSet, ReplicaSet керує Pods. Ти створюєш лише Deployment.
2. **ReplicaSets зберігаються як історія** — кожне оновлення = новий ReplicaSet, старі лежать з 0 Pods для миттєвого rollback.
3. **ConfigMap і env — підступна пара**: env НЕ оновлюється при зміні ConfigMap, потрібен `rollout restart`. Volume mount — оновлюється сам.
4. **`rollout undo` не створює новий ReplicaSet** — оживляє попередній (масштабує з 0 до N).
5. **Імена Pods ефемерні** — після rollout/rollback потрібно переприсвоювати `$POD`.
6. **`maxSurge` платить ресурсами за безпеку**, **`maxUnavailable` платить деградацією за швидкість**. У продакшні дефолт — `maxSurge>0, maxUnavailable=0`.
7. **Recreate існує для випадків, де "5 хв даунтайму краще ніж 5 хв незрозумілого стану"** — БД-міграції, RWO volumes, несумісний API.

### Корисні команди

```bash
# Перегляд
kubectl get deployments
kubectl get rs                                       # ReplicaSets — для розуміння історії
kubectl get pods -l app=nginx                        # фільтр по label
kubectl get pods -l app=nginx -w                     # watch — у реальному часі
kubectl get all -l app=nginx                         # вся ієрархія разом

# Створення/оновлення
kubectl apply -f manifests/                          # уся папка
kubectl set image deployment/X container=image:tag   # швидке оновлення (imperative)
kubectl rollout restart deployment/X                 # перезапуск без зміни image
kubectl annotate deployment/X kubernetes.io/change-cause="..." --overwrite  # підпис до revision

# Rollout
kubectl rollout status deployment/X                  # стежити за прогресом
kubectl rollout history deployment/X                 # список revisions
kubectl rollout history deployment/X --revision=N    # деталі конкретної revision
kubectl rollout undo deployment/X                    # rollback на попередню
kubectl rollout undo deployment/X --to-revision=N    # rollback на конкретну

# Дебаг
kubectl describe deployment X                        # повна інформація + Events
kubectl exec $POD -- <команда>                       # виконати команду в Pod
kubectl exec -it $POD -- sh                          # інтерактивний shell

# Видалення
kubectl delete -f manifests/                         # видалити те, що описано в маніфестах
kubectl delete deployment X                          # по імені (каскадно вб'є RS і Pods)

# Контекст
kubectl config current-context                       # де я зараз
kubectl config use-context <name>                    # перемкнутись
```
