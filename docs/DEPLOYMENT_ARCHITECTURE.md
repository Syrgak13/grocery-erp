# Архитектура развёртывания

**Фаза 3 · Grocery ERP** · 27.11.2026 · статус: черновик на утверждение

Как система собирается, проверяется, разворачивается и восстанавливается. Цели: пилот до 10 магазинов на одном сервере 2 vCPU / 4 ГБ RAM (NFR-061), бюджет ≤ 20 USD в месяц (CON-009), доступность ≥ 99% в часы работы магазинов (NFR-010), RPO ≤ 24 ч и RTO ≤ 4 ч (NFR-071). Диаграмма — [`diagrams/deployment.md`](diagrams/deployment.md).

---

## 1. Окружения

| Окружение | Где | Назначение | Данные |
|---|---|---|---|
| `dev` | Компьютер автора: `docker compose -f docker-compose.yml -f docker-compose.dev.yml up db`, backend из IDE, `npm run dev` (Vite proxy на `:8080`) | Разработка | Демо-данные двух магазинов (AR-010) |
| `ci` | GitHub Actions (ubuntu-latest) | Тесты, сборка образов | Testcontainers, одноразовые |
| `staging` | Тот же сервер, отдельный Compose-проект и поддомен (по желанию) | Проверка перед пилотом, ZAP-скан | Демо-данные |
| `prod` (пилот) | Один VPS | Пилот 2–3 магазина, защита диплома | Реальные данные пилота |

Выбор провайдера VPS — за автором. Требования к нему: 2 vCPU, 4 ГБ RAM, 40+ ГБ SSD, Ubuntu 24.04 LTS, стоимость ≤ 20 USD. Размещение данных (в КР или за рубежом) нужно сверить с законом КР «Об информации персонального характера» (CON-010, [Н]).

---

## 2. Контейнеры (Docker Compose)

| Сервис | Образ | Порты | Ресурсы (лимит) | Тома |
|---|---|---|---|---|
| `nginx` | Сборка `frontend/Dockerfile`: этап `node:24` собирает SPA, итог `nginx:1.30-alpine` | 80, 443 (публичные) | 128 МБ | `certs`, `certbot-webroot` |
| `backend` | Сборка `backend/Dockerfile`: этап `eclipse-temurin:21-jdk` + `./mvnw package`, итог `eclipse-temurin:21-jre`, пользователь `app` | 8080 (только сеть `internal`) | 1,5 ГБ (`-Xmx1g`, `-XX:+UseG1GC`) | нет (stateless), `read_only: true`, `tmpfs: /tmp` |
| `db` | `postgres:16.16` (точный тег, обновление патчей вручную) | нет (только `internal`) | 1,5 ГБ | `pgdata` |
| `backup` | Небольшой образ на `postgres:16.16` с `age` и `rclone` | нет | 256 МБ | `backups` (локальные копии 7 дней) |
| `certbot` | `certbot/certbot` | нет | — | `certs`, `certbot-webroot` |

Сети: `edge` (nginx) и `internal` (nginx, backend, db, backup). База и backend наружу не публикуются.

Проверки здоровья: `db` — `pg_isready`; `backend` — `GET /actuator/health/readiness`; `nginx` зависит от `backend: service_healthy`. Всем сервисам `restart: unless-stopped`, журналы `json-file` с `max-size: 10m`, `max-file: 5`.

### 2.1. Набросок `docker-compose.yml` (создаётся в Фазе 4)

```yaml
services:
  db:
    image: postgres:16.16
    environment:
      POSTGRES_DB: grocery
      POSTGRES_USER: ${DB_OWNER_USER}
      POSTGRES_PASSWORD: ${DB_OWNER_PASSWORD}
    volumes: [pgdata:/var/lib/postgresql/data, ./infra/postgres/init:/docker-entrypoint-initdb.d:ro]
    networks: [internal]
    healthcheck: { test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER}"], interval: 10s }
  backend:
    build: ./backend
    env_file: .env
    read_only: true
    tmpfs: [/tmp]
    depends_on: { db: { condition: service_healthy } }
    networks: [internal]
  nginx:
    build: ./frontend
    ports: ["80:80", "443:443"]
    depends_on: { backend: { condition: service_healthy } }
    volumes: [./infra/nginx:/etc/nginx/conf.d:ro, certs:/etc/letsencrypt:ro]
    networks: [edge, internal]
  backup:
    build: ./infra/backup
    env_file: .env
    networks: [internal, edge]   # edge нужен для выгрузки копий
volumes: { pgdata: {}, certs: {}, backups: {} }
networks: { edge: {}, internal: { internal: true } }
```

`internal: true` у сети `internal` запрещает контейнерам выход в интернет; `backup` дополнительно подключается к `edge` для выгрузки копий.

### 2.2. Переменные окружения (`.env.example`)

```
# База данных
DB_OWNER_USER=            # роль для Flyway (DDL)
DB_OWNER_PASSWORD=
DB_APP_USER=grocery_app   # роль приложения (DML)
DB_APP_PASSWORD=
DB_BACKUP_PASSWORD=
SPRING_DATASOURCE_URL=jdbc:postgresql://db:5432/grocery
SPRING_FLYWAY_USER=${DB_OWNER_USER}
SPRING_FLYWAY_PASSWORD=${DB_OWNER_PASSWORD}
# Безопасность
JWT_SECRET_CURRENT=       # 32+ байта, base64
JWT_SECRET_PREVIOUS=
REGISTRATION_INVITE_CODE= # пусто = открытая регистрация (D-016)
APP_PUBLIC_ORIGIN=https://example.kg
# Резервные копии
BACKUP_AGE_PUBLIC_KEY=
RCLONE_REMOTE=            # например s3:bucket/path
RCLONE_CONFIG_S3_ACCESS_KEY_ID=
RCLONE_CONFIG_S3_SECRET_ACCESS_KEY=
```

### 2.3. Настройка ресурсов под 4 ГБ RAM

| Компонент | Параметр | Значение |
|---|---|---|
| JVM | `-Xmx` / `-Xms` | 1 ГБ / 256 МБ |
| HikariCP | `maximum-pool-size` | 15 (3 пользователя × 10 магазинов + задачи) |
| PostgreSQL | `shared_buffers` / `effective_cache_size` / `work_mem` / `max_connections` | 512 МБ / 1,5 ГБ / 8 МБ / 40 |
| Nginx | `worker_processes` | auto |
| ОС | swap | 2 ГБ (страховка) |

---

## 3. Nginx

Роли: завершение TLS, раздача SPA, обратный прокси `/api/` на backend, заголовки безопасности и лимиты (`SECURITY.md`, разделы 6–7).

```nginx
limit_req_zone $binary_remote_addr zone=login:10m rate=10r/m;
server {
  listen 443 ssl; http2 on;
  ssl_protocols TLSv1.2 TLSv1.3;
  client_max_body_size 1m;
  location /api/v1/auth/login { limit_req zone=login burst=5 nodelay; proxy_pass http://backend:8080; }
  location /api/v1/products/import { client_max_body_size 5m; proxy_pass http://backend:8080; }
  location /api/ { proxy_pass http://backend:8080; proxy_set_header X-Request-Id $request_id; }
  location /actuator/ { return 404; }
  location / { root /usr/share/nginx/html; try_files $uri /index.html; }
}
```

`X-Forwarded-For` передаётся backend, и Spring берёт IP клиента только от доверенного прокси (`server.forward-headers-strategy=native`, сеть `internal`).

---

## 4. Конвейер CI/CD (GitHub Actions)

Диаграмма этапов — в [`diagrams/deployment.md`](diagrams/deployment.md).

### 4.1. `ci.yml` — на каждый push и pull request

| Задание | Шаги | Требования |
|---|---|---|
| `backend` | `actions/setup-java` (Temurin 21, кэш Maven) → `./mvnw -B verify` (юнит, интеграционные на Testcontainers, ArchUnit, JaCoCo с порогами 80% ядра / 60% проекта) | NFR-050, NFR-051 |
| `openapi` | Запуск backend на Testcontainers → выгрузка `/v3/api-docs` → `git diff --exit-code docs/api/openapi.json` | NFR-052 |
| `frontend` | `actions/setup-node` (24) → `npm ci` → `npm run typecheck` → `npm run lint` → `npm run test` (Vitest) → `npm run build` → проверка размера стартового JS ≤ 500 КБ gzip → проверка «нет строк вне ресурсов i18n» | NFR-006, NFR-051, FR-207 |
| `e2e` | После `backend` и `frontend`: `docker compose up` (профиль ci) → Playwright (Chromium, Firefox, WebKit) | NFR-090, NFR-012 |
| `security` | gitleaks; `npm audit --omit=dev --audit-level=high`; Trivy по собранным образам; CodeQL (отдельный workflow) | NFR-035 |

Docker на раннерах `ubuntu-latest` уже установлен, поэтому Testcontainers работают без дополнительной настройки. Все сторонние Actions закрепляются по SHA коммита.

Длинные прогоны (нагрузочные k6 и тест 10 000 операций) — отдельный workflow `nightly.yml` по расписанию и вручную, чтобы не задерживать каждый push.

### 4.2. `deploy.yml` — только вручную (`workflow_dispatch`)

```
1. Требует успешного ci.yml для выбранного коммита; запуск только с ветки main и тега vX.Y.Z.
2. Сборка образов backend и nginx, публикация в GHCR с тегом версии и digest.
3. SSH на сервер под пользователем deploy (ключ ограничен командой ./deploy.sh в authorized_keys).
4. На сервере: внеплановая резервная копия → docker compose pull → docker compose up -d
   (Flyway применяет миграции при старте backend) → ожидание health → smoke-тест /api/v1/auth/me = 401.
5. При неудаче — откат: предыдущий тег образа; миграции пишутся только «вперёд» и совместимыми
   с предыдущей версией кода (expand → contract), поэтому откат кода не требует отката схемы.
```

Развёртывание в прод — внешнее действие, поэтому всегда выполняется по явной команде автора. Ветки `main` в репозитории пока нет (D-024).

---

## 5. Миграции базы данных

| Правило | Пояснение |
|---|---|
| Только Flyway, только SQL | `V001__baseline.sql`, `V002__...`; повторяемые `R__grants.sql` для прав ролей |
| Миграция под `grocery_owner`, работа под `grocery_app` | Отдельные учётные данные `spring.flyway.user` (DATABASE, раздел 6) |
| Проведённые миграции не меняются | Исправление — новой миграцией; `validateOnMigrate=true` |
| Совместимость с предыдущей версией | Добавить колонку → код начинает писать → удалить старую в следующем выпуске |
| Тест | Интеграционные тесты применяют все миграции к пустому PostgreSQL 16 в Testcontainers |

---

## 6. Резервное копирование и восстановление (NFR-070–NFR-072)

### 6.1. Схема

| Что | Как | Когда | Хранение |
|---|---|---|---|
| Полная логическая копия | `pg_dump -Fc` под ролью `grocery_backup` | Ежедневно 02:30 Asia/Bishkek и перед каждым развёртыванием | Локально 7 дней; вне сервера 30 дней |
| Шифрование | `age -r $BACKUP_AGE_PUBLIC_KEY` | Сразу после дампа | Приватный ключ только у автора (SEC-012) |
| Выгрузка | `rclone copy` в S3-совместимое хранилище другого провайдера | Сразу после шифрования | Правило жизненного цикла 30 дней |
| Контроль | Размер > 0, `pg_restore --list` проходит; результат в журнал и на панель OWNER-оператора | После каждой копии | |

RPO по этой схеме — 24 ч (NFR-071 Must). Вариант Should (≤ 1 ч) — непрерывный архив WAL через `pgBackRest` или `wal-g` в то же хранилище; включается, если пилот покажет такую потребность (D-040).

### 6.2. Восстановление (RTO ≤ 4 ч)

```
1. Новый сервер: установить Docker, склонировать репозиторий, создать .env из менеджера паролей.
2. docker compose up -d db
3. rclone copy <последняя копия> . && age -d -i key.txt dump.age > dump.pgc
4. pg_restore -d grocery --role=grocery_owner --clean --if-exists dump.pgc
5. docker compose up -d   (Flyway проверит, что схема актуальна)
6. Смоук-тест: вход OWNER, остатки, последний чек совпадает с ожидаемым.
7. Переключить DNS (TTL заранее 300 с).
```

Учебное восстановление на чистом сервере — раз в месяц в пилоте и обязательно перед защитой (NFR-072); протокол — `docs/runbooks/restore.md` (Фаза 4).

---

## 7. Эксплуатация сервера

| Мера | Значение |
|---|---|
| Доступ | SSH только по ключу, вход root запрещён, `fail2ban` |
| Межсетевой экран | `ufw`: открыты 22 (по возможности ограничен IP автора), 80, 443 |
| Обновления | `unattended-upgrades` для пакетов безопасности; образы — по Dependabot |
| Часы | `chrony`; часовой пояс ОС — UTC (бизнес-время считает приложение) |
| Мониторинг доступности | Внешний бесплатный сервис проверяет `https://<домен>/` и `/api/v1/auth/me` (ожидается 401) каждые 5 мин (NFR-010) |
| Диск | Предупреждение при заполнении > 80% (cron + журнал) |
| Сертификаты | certbot renew дважды в день, перезагрузка Nginx |

---

## 8. Масштабирование в будущем (R3)

| Порог | Действие |
|---|---|
| > 10 магазинов или p95 выше NFR | Вертикально: 4 vCPU / 8 ГБ |
| > 50 магазинов | БД на отдельный сервер или управляемый PostgreSQL; backend в 2 экземпляра за Nginx; ограничение частоты и кэш прав — в Redis или PostgreSQL (ADR-0004); фоновые задачи — с ShedLock |
| Тяжёлые отчёты мешают кассе | Реплика для чтения, Reporting читает с неё |

Ни одно из этих действий не требует менять модульную структуру кода.
