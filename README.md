# Paltus Banking — агрегатор банковских счетов

Веб‑приложение для агрегирования счетов из разных банков (совместимых со стандартами OpenAPI), удобного контроля расходов и прогнозов, а также отслеживания доступных продуктов и создания платежей.

Главная особенность — режим для самозанятых и ИП (premium‑статус): можно помечать счета и транзакции типами personal/business, что помогает разделять личные и деловые операции. Для удобства доступны 2 ML‑модели:
- прогноз расходов на ближайшие 2 месяца (CatBoost);
- автоматическая классификация транзакций personal/business (ONNX) для premium‑пользователей.

## Возможности
- Агрегатор счетов из разных банков, совместимых с OpenAPI.
- Просмотр и фильтрация транзакций, расчет расходов за период.
- Прогноз расходов на текущий и следующий месяц.
- Просмотр доступных банковских продуктов.
- Создание платежей (single/multi consent + платежи).
- Premium: пометка счетов и транзакций типами personal/business, авто‑классификация для истории.
- Swagger UI для изучения API.

## Технологии
- Frontend: React + Vite + TypeScript, Tailwind, Redux Toolkit.
- Backend: Java 21, Spring Boot (Web, Security, Data JPA), springdoc‑openapi.
- БД: PostgreSQL 16.
- ML: CatBoost (прогноз), ONNX Runtime (классификация).
- Контейнеризация: Docker/Docker Compose.

## Структура
- `frontend/` — SPA на React (Vite), локальная разработка на `5173`.
- `backend/` — Spring Boot API (порт `8080`).
- `docker-compose.yml` — общий (dev) compose для фронта/бэка/БД.
- `backend/docker-compose.yml` — compose только для бэка и БД.
- `.env` — секреты и ключи (см. ниже).

## Быстрый старт (локальная разработка)
Ниже — самый надежный сценарий разработки: бэкенд+БД в Docker, фронтенд локально.

1) Предустановки: Docker, Docker Compose, Node.js 18+, npm.

2) Заполните корневой `.env` (примеры):
```
TEAM_CODE=teamXXX
TEAM_SECRET=changeme-team-secret
SECRET_KEY=change-me-long-random-secret
```
`SECRET_KEY` используется как ключ HMAC для JWT/шифрования (длина ≥ 32 байта).

3) Поднимите бэкенд и БД в Docker:
```
cd backend
docker compose up --build
```
Бэкенд поднимется на `http://localhost:8080`, PostgreSQL — на `5432`.

4) Запустите фронтенд локально:
```
cd frontend
npm ci
npm run dev -- --host
```
Фронтенд будет доступен на `http://localhost:5173`.

5) Swagger UI: `http://localhost:8080/swagger-ui/index.html`

## Альтернативы запуска (контейнеризация фронтенда)
- Производительная сборка фронта в контейнере:
  - `frontend/Dockerfile` собирает статику и отдает её через `serve` на порту `3000`.
  - Можно добавить сервис фронтенда в общий compose или запустить как отдельный контейнер.

- Корневой `docker-compose.yml` подготовлен под dev c Vite (`5173`), но ожидает `frontend/Dockerfile.dev` и volume `../multibank-frontend:/app`.
  - Если используете текущую папку `frontend/`, обновите сервис фронтенда в compose на `dockerfile: Dockerfile` и уберите volume, либо просто запускайте фронтенд локально как описано выше.

## Переменные окружения
Бэкенд читает значения из `application.properties` и `.env`:
- `DB_URL` (по умолчанию `db:5432/pbanking` через Compose)
- `DB_USERNAME` (по умолчанию `pbanking`)
- `DB_PASSWORD` (по умолчанию `pbanking`)
- `SECRET_KEY` — секрет для JWT/шифрования
- `TEAM_CODE`, `TEAM_SECRET` — параметры интеграции (TPP)

Compose‑файлы уже прокидывают дефолты, но для JWT/шифрования задайте свой `SECRET_KEY` в корневом `.env`.

## Добавление банков
Есть два способа:
1) Через конфиг: `backend/src/main/resources/banks.yml` (id, имя, базовый URL OpenAPI). При первом обращении к банку он автоматически добавляется в БД, если присутствует в этом файле.
2) Через административный эндпоинт addBank: доступен пользователю с ролью ADMIN, который инициализируется при старте приложения. Точный путь и схема запроса доступны в Swagger UI.
```
POST /api/banks/addBank
Authorization: Bearer <admin-token>
Content-Type: application/json

{
  "id": "abank",
  "name": "A Bank",
  "url": "https://abank.open.bankingapi.ru"
}
```


## Аутентификация и роли
- Регистрация: `POST /api/user/register { username, password }`
- Логин: `POST /api/user/login { username, password }`
- Ответ содержит `token` и `expiresIn`. Все защищенные запросы выполняйте с заголовком:
```
Authorization: Bearer <token>
```
- Активация premium: `POST /api/user/activatePremium/{days}` — станет доступен функционал разметки и авто‑классификации.

## Основные эндпоинты (кратко)
Ниже — удобная выжимка. Точные схемы и модели смотрите в Swagger UI.

- Пользователь (`/api/user`):
  - `POST /register`, `POST /login`, `GET /generalData`, `POST /activatePremium/{days}`, `POST /setNameSurname?name=..&surname=..`

- Банки (`/api/banks`):
  - `GET /` — список доступных банков (из `banks.yml`)
  - `GET /availableProducts/{bank_id}` — продукты банка

- Счета (`/api/account`):
  - `GET /getAccounts/{bank_id}/{client_id}` — счета клиента
  - `POST /setType` — установить тип счета `{ bankId, id, type }`
  - `POST /setDescription` — описание счета `{ bankId, id, text }`

- Транзакции и данные (`/api/data`):
  - `GET /transactions/{bank_id}/{account_id}?from_booking_date_time=..&to_booking_date_time=..&page=1&limit=50&predict=true`
    - `predict=true` применит авто‑классификацию для premium
  - `GET /expens?from_booking_date_time=..&to_booking_date_time=..[&bank_id=..&account_id=..]` — сумма расходов
  - `GET /statistic[?type=PERSONAL|BUSINESS]` — статистика + прогноз (PERSONAL и общий)

- Разметка транзакций (`/api/transaction`) — только для premium:
  - `POST /setType` — установить тип транзакции `{ id, type }`

- Согласия/платежи (`/api/consent`, `/api/payment`):
  - `POST /consent/account { bank_id, client_id }` — чтение аккаунтов
  - `POST /consent/singlePayment`, `POST /consent/multiplePayment` — платежные согласия
  - `POST /payment/single` — создать платеж

## Примеры cURL
- Регистрация:
```
curl -X POST http://localhost:8080/api/user/register \
  -H 'Content-Type: application/json' \
  -d '{"username":"demo","password":"demo"}'
```
- Логин:
```
curl -X POST http://localhost:8080/api/user/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"demo","password":"demo"}'
```
- Получить банки:
```
curl http://localhost:8080/api/banks \
  -H 'Authorization: Bearer <token>'
```
- Транзакции с авто‑классификацией (premium):
```
curl "http://localhost:8080/api/data/transactions/{bank_id}/{account_id}?from_booking_date_time=2024-01-01T00:00:00Z&to_booking_date_time=2024-12-31T23:59:59Z&predict=true" \
  -H 'Authorization: Bearer <token>'
```

## Состояние проекта и планы
Проект находится в активной разработке. Ближайшие задачи:
- Улучшить визуальную часть фронтенда и UX.
- Повысить надежность серверной части.
- Поднять собственный OpenAPI‑банк и подготовить обогащенную базу транзакций для улучшения моделей.
- Возможная интеграция MLflow и расширение набора фичей.
- Больше интеграций с банками (карты, открытие продуктов и т. п.).
- Расширение функций для premium‑пользователей, облегчающих финансовую деятельность самозанятых.

## Лицензия
Без лицензии. Используйте для ознакомления и внутренних испытаний.
