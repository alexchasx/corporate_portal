# corporate_portal

Корпоративный веб-портал производственного предприятия — единая точка доступа сотрудников (10 000+ активных пользователей) к корпоративным сервисам, документам, коммуникациям и бизнес-процессам. Пиковая нагрузка — до 3 000 одновременных сессий. Развёртывание on-premise / приватное облако, доступ из LAN и через VPN/mTLS.

## Документация проекта

| Документ | Назначение |
|---|---|
| [`portal_requirements.md`](./portal_requirements.md) | Требования (функциональные и нефункциональные) |
| [`DEVELOPMENT_PLAN.md`](./DEVELOPMENT_PLAN.md) | План разработки: архитектура, стек, этапы, критерии приёмки |
| [`ROADMAP_OPENSPEC.md`](./ROADMAP_OPENSPEC.md) | Детальная дорожная карта из ~65 OpenSpec-изменений (этапы 0–10) |
| [`openspec/`](./openspec/) | OpenSpec-спецификации, изменения (`changes/`) и capabilities (`specs/`) |

## Технологический стек

| Слой | Технология |
|---|---|
| Backend | Laravel 13 (PHP 8.4+) — модульный монолит, API, очереди, события |
| Frontend | Nuxt 4 + Vue 3 + Vuetify 3 — SSR/SPA-гибрид per-route (Nitro `routeRules`) |
| БД | PostgreSQL 18+ (FTS, партиционирование, read-replicas) |
| Кэш / очереди / сессии | Redis 7+ |
| WebSocket | Laravel Reverb (уведомления, real-time; резерв — Soketi) |
| Файлы | MinIO (S3-совместимое) |
| Поиск | PostgreSQL FTS + `pg_trgm` (MVP) → OpenSearch (опционально) |
| Инфраструктура | Docker, Docker Compose; Nginx (TLS, HTTP-кэш, rate limiting) |
| CI/CD | GitLab CI / GitHub Actions |
| Мониторинг | Prometheus + Grafana + Sentry |
| E2E / нагрузка | Playwright / k6 |

Подробное обоснование решений — раздел 2 [`DEVELOPMENT_PLAN.md`](./DEVELOPMENT_PLAN.md:60).

## Структура репозитория (монорепозиторий)

```
corporate_portal/
├── backend/               # Laravel 13: API, миграции, доменные модули (app/Modules/)
├── frontend/              # Nuxt 4: SSR/SPA, компоненты Vuetify
├── docker/                # Dockerfile, docker-compose, entrypoint-скрипты
├── docs/                  # ADR, OpenAPI-спецификация, runbook
├── deploy/                # Helm-чарты или Ansible-плейбуки (prod)
├── openspec/              # OpenSpec: specs/ и changes/
├── .gitlab-ci.yml         # CI/CD (или .github/)
├── DEVELOPMENT_PLAN.md    # план разработки
├── portal_requirements.md # требования
└── README.md              # настоящий файл
```

## Быстрый старт (dev/staging)

Для развёртывания staging-окружения достаточно одного контейнерного стенда:

```bash
cp .env.example .env   # заполнить плейсхолдеры (секреты никогда не хранятся в репозитории)
docker compose up -d   # nginx, backend, worker, scheduler, reverb, frontend, postgres, redis, minio, mailpit
```

- Приложение: `https://localhost` (dev — self-signed TLS).
- Почта в dev перехватывается Mailpit.
- Health-check: `GET /health/live` (процесс) и `GET /health/ready` (БД, Redis, S3, LDAP, очереди).
- Критерий приёмки окружения: `docker compose up` поднимает пустой портал (раздел 9 плана).

## Краткая архитектура

```
Клиенты (Browser LAN/VPN) → Nginx/LB (TLS 1.3, HSTS, mTLS, HTTP-кэш, rate limiting)
                             ├── Nuxt 4 (Nitro): SSR-контент / SPA-модули per-route
                             └── Laravel 13 (API /api/v1): модульный монолит
                                  ├── PostgreSQL 18 (primary + read-replicas)
                                  ├── Redis 7 (кэш, очереди, сессии, pub/sub)
                                  ├── MinIO (файлы, медиа, документы)
                                  └── Reverb (WebSocket: уведомления, канбан)
Интеграции: LDAP/AD, СЭД (REST/SOAP), ERP (REST/OData), SMTP/IMAP, FCM/APNs, Prometheus
```

Модули backend (Bounded Context): Auth, HR, News, Documents, Tasks, Surveys, Directory, Search, Admin. Межмодульная связь — только через `Contracts` и события; тяжёлая работа — в Jobs (Redis-очереди).

## Ключевые показатели успеха

| Метрика | Цель |
|---|---|
| Время ответа API (P95) | ≤ 300 мс штатно; ≤ 500 мс при 3 000 пользователей |
| Первая отрисовка (FCP) | ≤ 1,5 с |
| Lighthouse Performance / Accessibility | ≥ 85 / ≥ 90 |
| Доступность | 99,5 % |
| Покрытие тестами backend | ≥ 70 % (unit + feature) |

## Дорожная карта (кратко)

Оценка для команды 7–8 человек; backend и frontend идут параллельно. Детализация до ~65 OpenSpec-изменений — в [`ROADMAP_OPENSPEC.md`](./ROADMAP_OPENSPEC.md).

| # | Этап | Длительность | Ключевой результат |
|---|---|---|---|
| 0 | Фундамент и инфраструктура | 2–3 нед | Репозитории, docker-compose, каркасы Laravel/Nuxt, CI-скелет, health-check |
| 1 | Аутентификация и авторизация | 3–4 нед | LDAP-вход, 2FA, Sanctum, RBAC+ABAC, SSO-заглушка, аудит |
| 2 | Личный кабинет + оргструктура | 3–4 нед | Профиль, дашборд, дерево подразделений, импорт XLSX/CSV |
| 3 | Новостной портал | 3–4 нед | Ленты, редактор, модерация, публикация по расписанию, комментарии |
| 4 | Документооборот | 4–5 нед | Библиотека, версии, превью, ЖЦ документа, подписи |
| 5 | Задачи и поручения | 3–4 нед | Задачи, канбан, минимальный Гант, уведомления о дедлайнах |
| 6 | Опросы и голосования | 2–3 нед | Конструктор, анонимность, квоты, live-результаты |
| 7 | Справочники + Поиск | 3–4 нед | Телефонный справочник, нормативные документы, глобальный поиск |
| 8 | Администрирование и аудит | 2–3 нед | Пользователи/роли, аудит-журнал, конфигурация портала |
| 9 | Интеграции | 3–4 нед | СЭД, ERP, SMTP, FCM/APNs, Prometheus-метрики |
| 10 | Стабилизация и запуск | 3–4 нед | Нагрузочный тест, аудит безопасности, Lighthouse, релиз |

Итого ~29–38 недель. MVP (аутентификация + новости + справочники + задачи) — к концу этапа 5 (~17–20 недель).

## Процесс разработки

- **Git-флоу:** `main` — production, `staging` — предрелиз, feature-ветки; code review обязателен (минимум 1 аппрув).
- **CI/CD:** lint → static analysis (PHPStan 8, `vue-tsc`) → security (`composer audit`, `npm audit`, Trivy) → тесты (PHPUnit, Vitest) → build → деплой на staging + smoke-тесты → ручное подтверждение → rolling-деплой в prod (zero-downtime).
- **OpenSpec:** каждый пункт дорожной карты реализуется как изменение в [`openspec/changes/`](./openspec/changes/): propose → apply → archive. Сквозные требования (безопасность, i18n ru/en, WCAG AA, OpenAPI/TS-типы, аудит) — через Definition of Done (раздел 18 [`DEVELOPMENT_PLAN.md`](./DEVELOPMENT_PLAN.md:606)).
- **DoD:** линтеры, PHPStan level 8, покрытие новых строк ≥ 80 %, миграции с reverse, локализация, аудит-события, зелёные E2E.

## Тестирование и качество

Пирамида тестов: PHPUnit (unit + feature) и Vitest — база; Playwright E2E — критические пути (логин, новости, задачи, документы, опросы); k6 — нагрузочные прогоны (3 000 пользователей, P95 < 500 мс); Lighthouse CI — Performance ≥ 85, Accessibility ≥ 90.

## Мониторинг и эксплуатация

- Метрики Prometheus (latency P50/P95/P99, throughput, глубина очередей, WebSocket-соединения), дашборды Grafana, алерты (error rate, P95, очереди, SSL).
- Логи — структурированные JSON (request id, trace id), PII-редакция.
- Ошибки — Sentry (backend + frontend, source maps для прода).
- Бэкапы — ежедневные инкрементальные, еженедельные полные (pgBackRest/WAL-G), хранение 30 дней, регулярные тесты восстановления.
- SLO: аптайм 99,5 %.

## Безопасность

TLS 1.3 + HSTS + CSP, mTLS для внешнего доступа; защита по OWASP Top 10 (CSRF, XSS-санитизация, XXE, rate limiting); LDAP/AD + SSO + 2FA TOTP; авторизация RBAC + ABAC (запрет по умолчанию); PII-redaction и шифрование чувствительных полей; append-only аудит; SAST/SCA в пайплайне и пентест перед релизом.

## Критерии приёмки проекта

- [ ] Развёртывание из одного `docker compose up` (staging).
- [ ] Все E2E-тесты (Playwright) проходят в свежем окружении.
- [ ] Нагрузочный тест: 3 000 пользователей, P95 < 500 мс.
- [ ] Аудит безопасности без критических уязвимостей.
- [ ] Lighthouse: Performance ≥ 85, Accessibility ≥ 90, FCP ≤ 1,5 с.
- [ ] Документация: README, OpenAPI, схема БД (в `docs/`).
- [ ] Бэкапы и протестированное восстановление.
- [ ] Покрытие backend-тестами ≥ 70 %.

Полный перечень и статус — раздел 17 [`DEVELOPMENT_PLAN.md`](./DEVELOPMENT_PLAN.md:591).
