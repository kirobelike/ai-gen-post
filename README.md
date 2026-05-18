# Туц Ыскшзе

Автономный сервис для мониторинга Telegram-каналов-доноров, парсинга карточек товаров с Яндекс Маркета, генерации партнёрских ссылок, сборки финального изображения (товар + блок «Детали цены») и публикации поста в целевой Telegram-канал. Тексты постов формируются строго по шаблону, **без AI-перезаписи**.

## Возможности

* **Два бэкенда сбора постов из каналов-доноров** (переключаются через `DONOR_BACKEND` в `.env`):
  * `http` (по умолчанию) — опрос публичных страниц `https://t.me/s/<channel>` каждые N секунд. Никакого Telegram-логина не требует. Подходит для любых публичных каналов с username.
  * `telethon` — MTProto user-bot через Telethon. Требует `TG_API_ID`/`TG_API_HASH` и одноразовый интерактивный логин по номеру. Даёт push-уведомления (быстрее реакция) и доступ к приватным каналам.
* Парсер Я.Маркета: каскад из мета-тегов, JSON-LD и `window.__NEXT_DATA__`.
* Генератор реф-ссылок через приватный endpoint `resolveShorLink` (с поддержкой `sk`, `x-market-front-glue`, `Session_id`, прокси).
* Композитор изображения **1280×844 px**: товар слева (≈80% площади) + HTML-блок «Детали цены» справа (рендер Playwright + Chromium).
* Публикация в целевой Telegram-канал через Bot API.
* Антидубль за окно 60 минут (по `product_id` + URL).
* FastAPI-дашборд (auth, разделы Каналы / Шаблоны / API & Cookie / Чёрный список / Логи и статистика).
* Кнопка «Check Auth» проверяет жив ли `Session_id`.
* Конфигурация и секреты — в `.env`; то, что нужно менять часто (`Session_id`, токены, проксь) — редактируется через веб-панель и хранится в БД.

## Стек

Python 3.10+, FastAPI, BeautifulSoup4 (HTTP-поллер), Telethon (опциональный бэкенд), Pillow, Playwright (Chromium), Celery + Redis, PostgreSQL + SQLAlchemy 2 + Alembic, Jinja2 + Bootstrap 5.

## Архитектура

```
                    ┌──────────────────────┐
                    │  Telegram-каналы     │
                    │  (доноры)            │
                    └──────────┬───────────┘
          HTTP /s/ poll      │  или MTProto (Telethon)
                               ▼
                    ┌──────────────────────┐
                    │  src/scraper         │
                    │  http_poller         │
                    │  / telethon_listener │
                    └──────────┬───────────┘
                               │ Celery task
                               ▼
┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌──────────────────┐
│ parser     │→│ linker     │→│ image_gen  │→│ poster     │→│ Telegram канал   │
│ (Я.Маркет) │ │ (resolve…) │ │ (Pillow +  │ │ (Bot API)  │ │ (целевой)        │
│            │ │            │ │  Playwright│ │            │ │                  │
└────────────┘ └────────────┘ └────────────┘ └────────────┘ └──────────────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │  PostgreSQL + Redis  │
                    └──────────────────────┘
                               ▲
                               │
                    ┌──────────────────────┐
                    │  src/web (FastAPI)   │
                    │  админ-панель        │
                    └──────────────────────┘
```

Каждый модуль изолирован в `src/<name>/`, добавить Ozon/WB — это новый `src/parser/ozon.py` + ветка в `process_post`.

## Запуск

### Через Docker (рекомендуемо)

```bash
git clone https://github.com/Tidenefeel40/new-availible-code-maket.git ymp
cd ymp
cp .env.example .env
# Заполни как минимум: SECRET_KEY, ADMIN_USERNAME/PASSWORD, TG_BOT_TOKEN, TG_TARGET_CHANNEL
# (TG_API_ID/HASH НЕ нужны при DONOR_BACKEND=http — значение по умолчанию)
docker compose up --build -d
```

После старта дашборд: <http://localhost:8000>. Учётка — из `ADMIN_USERNAME` / `ADMIN_PASSWORD` (`.env`).

### Быстрое обновление после `git pull`

Если в коммите изменились только `.py`/`.html`/`.css` (а не `pyproject.toml` и не `Dockerfile`), пересобирать образ не нужно. Один раз создай локальный dev-override:

```bash
cp docker-compose.override.yml.example docker-compose.override.yml   # Windows: copy ...
docker compose up -d
```

Override пробрасывает локальные `src/`, `migrations/`, `alembic.ini` прямо внутрь контейнеров через bind-mount, поэтому после `git pull` достаточно перезапустить процессы:

```bash
docker compose restart web worker beat
```

Файл `docker-compose.override.yml` в `.gitignore`, его не нужно коммитить.

Когда override **не подойдёт** и нужен полный `docker compose up -d --build`:

* поменялся `pyproject.toml` (новая зависимость, апдейт версии пакета);
* поменялся `Dockerfile`;
* обновилась версия Playwright (Chromium ставится в образе).

Если у тебя `--build` каждый раз тянет всё по часу — значит layer cache Docker слетел. Проверь `docker buildx du`: если там пусто, кто-то делал `docker system prune` или Docker Desktop почистил кэш по диску. После одной долгой сборки последующие `--build` без правок зависимостей должны занимать секунды.

### Бэкенд сбора постов — HTTP (по умолчанию)

По умолчанию сбор идёт HTTP-поллером внутри worker'а (`DONOR_BACKEND=http`). Раз в `HTTP_POLL_INTERVAL_SECONDS` секунд (по умолчанию 30) Celery beat вызывает таск `poll_donor_channels`, который делает GET `https://t.me/s/<channel>` по каждому активному SOURCE-каналу и парсит HTML. Никакой доп. настройки не требует — просто добавь канал в дашборде.

Ограничения:
* Работает только для публичных каналов с username (приватные и числовые chat_id пропускаются).
* Latency равен интервалу опроса (в среднем 15–30 с вместо мгновенных push'ей Telethon'а).
* Без `last_seen_message_id` на первом цикле история не публикуется (записывается только максимум видимых ID).

### Бэкенд сбора постов — Telethon (опционально)

Если нужен MTProto или приватные каналы — в `.env` выставь `DONOR_BACKEND=telethon`, заполни `TG_API_ID` и `TG_API_HASH` (<https://my.telegram.org>), затем один раз пройди интерактивный логин:

```bash
docker compose --profile telethon run --rm -it telethon python scripts/telethon_login.py
```

И подними telethon-сервис вместе с остальными:

```bash
docker compose --profile telethon up -d --build
```

В этом режиме Celery-таск `poll_donor_channels` становится no-op, а сообщения берёт telethon-контейнер из push-уведомлений MTProto.

### Применение миграций

Миграции применяются автоматически в `entrypoint-web.sh`. Вручную:

```bash
docker compose run --rm web alembic upgrade head
```

### Обновление `Session_id` Я.Маркета

`Session_id` периодически протухает. Обновлять через дашборд:

1. Войди в браузере на market.yandex.ru от партнёрского аккаунта.
2. DevTools → Application → Cookies → скопируй значение `Session_id`.
3. Дашборд → **API & Cookie** → вставь в `ym_session_id` → **Сохранить**.
4. Нажми **Check Auth** — должно показать `OK`.

При желании можно также обновить `ym_sk` и `ym_x_market_front_glue` (из заголовков любого XHR на market.yandex.ru), но они для этого endpoint часто работают и без принудительной подстановки.

## Локальная разработка без Docker

```bash
python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"
python -m playwright install --with-deps chromium

# отдельно подними Postgres / Redis (например через docker compose up postgres redis)
cp .env.example .env  # отредактируй DATABASE_URL / REDIS_URL под локальные сервисы
alembic upgrade head

# веб
uvicorn src.web.app:app --reload
# worker
celery -A src.worker.celery_app worker --loglevel=INFO
# beat (включает HTTP-поллер каналов-доноров + retry_pending)
celery -A src.worker.celery_app beat --loglevel=INFO

# ОПЦИОНАЛЬНО: telethon listener — только если DONOR_BACKEND=telethon
python -m src.scraper.run_listener
```

## Структура проекта

```
src/
  core/        # config + logging
  db/          # SQLAlchemy модели, сессии, репозиторий
  scraper/     # Telethon-листенер каналов
  parser/      # YandexMarketParser
  linker/      # resolveShorLink (партнёрские ссылки)
  image_gen/   # композитор 1280x844 (Pillow + Playwright)
  poster/      # рендер шаблона + публикация через Bot API
  web/         # FastAPI + Jinja2 дашборд
  worker/      # Celery + tasks pipeline
migrations/    # Alembic
docker/        # Dockerfile, entrypoint
scripts/       # bootstrap-утилиты (telethon_login и т.п.)
tests/         # pytest
```

## Тесты

```bash
pytest -q
```

## Замечания по AI и стилю постов

В шаблоне есть только плейсхолдеры `{title}`, `{current_price}`, `{old_price}`, `{discount_pct}`, `{discount_value}`, `{price_without_card}`, `{bank_label}`, `{ref_link}`, `{original_url}`, `{product_id}`. Подстановка происходит через безопасный formatter — если значения нет, выставляется `—`. Никаких LLM-улучшений текста не применяется.
