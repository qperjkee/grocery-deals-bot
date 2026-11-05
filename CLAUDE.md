# CLAUDE.md

Контекст для сесій Claude Code у цьому репозиторії.

## Проєкт

Telegram-бот для моніторингу акцій і цін **Lidl, Kaufland і Tesco у Братиславі (Словаччина)**. Користувач веде список товарів, які хоче відстежувати (watchlist). Бот:
- збирає листівки й онлайн-каталоги;
- нормалізує та зіставляє пропозиції зі списком користувача;
- надсилає сповіщення без дублікатів;
- порівнює ціну за одиницю, окремо звичайну ціну й ціну за карткою лояльності;
- зберігає історію цін і веде список покупок.

Бот проєктується як **багатокористувацький** з розширенням на інші міста, країни й мережі.

- Повне ТЗ: [ТЗ_Telegram_Grocery_Deals_Bratislava_UA.docx](ТЗ_Telegram_Grocery_Deals_Bratislava_UA.docx), **v1.1** від 2026-11-05. Рішення з цього файлу вже внесені в ТЗ. Якщо ТЗ і цей файл розходяться, діє новіший документ, а розбіжність треба виправити в обох.
- Це бінарний файл, тому текст витягуй через `pandoc -t plain ТЗ_Telegram_Grocery_Deals_Bratislava_UA.docx`.
- Статус: репозиторій порожній, далі Етап 0 (дослідження джерел) і Етап 1 (каркас).
- Спілкування та документація: українською. Код, ідентифікатори, коміти й **UI бота: англійською**.

## Прийняті рішення (ТЗ §19)

| Питання | Рішення |
|---|---|
| Мова UI | Англійська в MVP. Українська й словацька пізніше, через i18n без зміни коду. |
| Географія | Уся Братислава. Країна, місто, валюта й часовий пояс не хардкодяться: є ієрархія `countries → cities → stores`. |
| Джерела | Усе доступне: листівки (`leaflet`) і онлайн-каталоги (`online_catalog`), зокрема Tesco. |
| Сповіщення | Режим обирає користувач: `instant` / `digest` / `both`. |
| Картки лояльності | Враховуємо обидві ціни: звичайну й за карткою (Kaufland Card, Tesco Clubcard, Lidl Plus). Зберігаємо й показуємо їх окремо. |
| Користувачі | Багатокористувацький бот. MVP може працювати в режимі `ACCESS_MODE=allowlist`. |
| Фонові задачі | Taskiq + Redis замість APScheduler. Celery відхилено, бо він не підтримує async (див. нижче). |
| Зберігання raw-даних | Payload у `raw_snapshots` зберігається 90 днів (`RAW_SNAPSHOT_RETENTION_DAYS`), потім його видаляє `cleanup_old_data`. Метадані (URL, хеш, дата, run) зберігаються безстроково. `offers` і `price_observations` не видаляються. |
| Відкриття доступу | `ACCESS_MODE=allowlist` діє до завершення Етапу 10 і 2 тижнів стабільної роботи collectors. Перед відкриттям потрібні ліміт на користувача (`MAX_WATCHLIST_ITEMS_PER_USER=50`) і моніторинг черги розсилки. |

## Стек

| Що | Чим |
|---|---|
| Мова / пакети | Python 3.12+, `uv` (pyproject.toml + uv.lock) |
| Лінт / типи | `ruff` (lint + format), `mypy` (strict для `domain/` і `services/`) |
| Telegram | aiogram 3, long polling; FSM storage у Redis; i18n через gettext (`aiogram.utils.i18n`) |
| БД | PostgreSQL 16, SQLAlchemy 2 (async, asyncpg), Alembic; розширення `pg_trgm`, `unaccent` |
| Фонові задачі | **Taskiq + taskiq-redis** (worker) + `TaskiqScheduler` |
| Redis | обов'язковий: брокер задач, FSM, rate-limit розсилки |
| HTTP / парсинг | httpx (тайм-аути, retry з backoff), BeautifulSoup + lxml; Playwright тільки за потреби і в окремому образі worker |
| Валідація / конфіг | Pydantic v2, pydantic-settings |
| Логи | structlog, JSON у stdout (контекст: run_id, chain, user_id) |
| Fuzzy-matching | `pg_trgm` для відбору кандидатів, RapidFuzz для оцінки схожості |
| Тести | pytest, pytest-asyncio, respx, testcontainers (Postgres) |
| Деплой | Docker Compose: `bot`, `worker`, `scheduler`, `postgres`, `redis` |

**Чому Taskiq, а не Celery.** Весь стек асинхронний (aiogram, httpx, asyncpg). Celery не підтримує async-задачі: довелося б робити `asyncio.run` у кожній задачі й міняти пул з'єднань. До того ж Celery офіційно не підтримує Windows для локального запуску. Задачі є тонкими обгортками над services, тому за потреби на Celery можна перейти, змінивши лише `app/tasks/`.

## Команди (заплановані, оновити після Етапу 1)

```bash
uv sync                                   # залежності
docker compose up -d postgres redis       # інфраструктура для локальної розробки
uv run alembic upgrade head               # міграції
uv run python -m app.main                 # бот
uv run taskiq worker app.tasks.broker:broker             # worker
uv run taskiq scheduler app.tasks.scheduler:scheduler    # розклад
uv run python scripts/run_collector.py kaufland leaflet --dry-run  # один collector вручну
uv run pytest                             # тести
uv run ruff check . && uv run ruff format . && uv run mypy app
docker compose up --build                 # весь стек
```

Розробка ведеться на Windows, а production працює в Linux-контейнерах. Скрипти й шляхи мають працювати на обох системах.

## Архітектура

Шари та напрям залежностей: `bot` → `services` → `domain` ← `infrastructure`. Домен ні від чого не залежить.

- `app/bot/`: routers, keyboards, states, middlewares (i18n, throttling, контекст користувача). Тут немає SQL і парсингу, лише виклики services.
- `app/i18n/locales/`: тексти UI, у MVP тільки `en`. Рядки не хардкодимо в handlers.
- `app/services/`: сценарії (watchlist, offers, matching, pricing, notifications, shopping, user).
- `app/domain/`: сутності, enums, правила (matching, ціна за одиницю, «суттєва зміна»). Чистий Python без I/O.
- `app/collectors/`: adapter на кожну пару (мережа, тип джерела) з реєстрацією в `registry.py`. Реалізує `StoreCollector` (`chain`, `source_type`, `discover_campaigns`, `fetch_campaign`, `healthcheck`) і повертає `RawCampaign`/`RawOffer`.
- `app/parsers/`, `app/normalization/`: формати (HTML/JSON/PDF) та уніфікація назв, діакритики, одиниць, категорій і синонімів.
- `app/database/`: моделі, сесії, repositories.
- `app/tasks/`: Taskiq broker, розклад і задачі. Це тонкі обгортки над services.
- `migrations/`: Alembic, лежить у корені.

Pipeline задач:

```
scheduler ─► collect_store(chain, source_type)   незалежно для кожної пари
                 └─► normalize_run(run_id)
                         └─► match_offers(run_id)        → offer_matches
                                 └─► send_alert(user_id, match_id)  лише instant/both, rate-limit
scheduler ─► dispatch_digests (кожні 10–15 хв) ─► send_digest(user_id)   лише digest/both
scheduler ─► cleanup_old_data (раз на добу)
адмін у боті ─► collect_store.kiq(...)   # ручний перезапуск (FR-16)
```

## Інваріанти (не порушувати)

1. **Гроші**: тільки `Decimal` / `NUMERIC(10,2)`, ніколи float. Валюту зберігаємо явно (ISO 4217); у MVP це EUR, але не хардкодимо.
2. **Час**: timestamps зберігаємо як `timestamptz` в UTC. `valid_from`/`valid_to` — це `DATE` місцевого часу, межі включні. Розклад користувача рахуємо в часовому поясі його регіону (MVP — Europe/Bratislava).
3. **Нічого не вигадувати**: якщо джерело не дало `old_price` або `discount_percent`, ставимо NULL. Обчислені значення позначаємо `is_calculated`. Оригінальну назву й сирий payload (`raw_snapshots`) зберігаємо завжди.
4. **Ізоляція збирачів**: падіння одного collector не зупиняє інші. Collector не звертається до Telegram і watchlist. Статус збору: `success` / `partial` / `failed` з причиною (`collector_runs`).
5. **Ідемпотентність**: унікальний `offers.fingerprint` (chain + campaign + normalized name + size + price + період), запис через `INSERT ... ON CONFLICT`. Незмінений вміст пропускаємо: перевіряємо ETag / Last-Modified / `source_hash`.
6. **Дедуплікація сповіщень**: унікальний `notification_log.dedupe_key = user_id:offer_id:kind:price`. Нове сповіщення йде лише тоді, коли пропозиція нова або суттєво змінилась (ціна знизилась, з'явилась знижка або ціна за карткою) і ще діє. У режимі `both` одна пропозиція не дублюється.
7. **Консервативний matching**: exact / brand / category / keyword — окремі типи, `confidence` зберігаємо. Різне фасування не вважається одним точним товаром. Низьку впевненість не видаємо за точний збіг.
8. **Багатомовний пошук**: запит англійською (згодом українською) має знаходити словацькі назви. Для цього є словник `category_synonyms` (en/uk/sk) і пошук без діакритики (`unaccent`: káva = kava).
9. **Ціни**: `price` — це ціна без картки, `loyalty_price` + `loyalty_program` — ціна за карткою. «Найнижча ціна» враховує лише картки, які є в користувача (`user_loyalty_cards`). Ціни з каталогу мають `is_promo=false` і йдуть в історію.
10. **Статуси**: `upcoming` не показуємо як чинні. `expired` не потрапляють у видачу й дайджест.
11. **Порівняння** лише за ціною за кг / л / шт і лише для сумісних одиниць. Історію показуємо як «історію спостережень бота» з періодом.
12. **Telegram limits**: близько 30 повідомлень/с глобально і 1/с на чат. `TelegramRetryAfter` обробляємо через retry, а при `TelegramForbiddenError` ставимо `users.is_active=False`. Вхідні повідомлення пропускаємо через throttling.
13. **Приватність / GDPR**: `/delete_me` видаляє всі дані користувача. Списки користувачів ніде не публікуються.
14. **Секрети** лише в `.env` (у `.gitignore`). У репозиторії тримаємо тільки `.env.example`. Адмін-доступ дає `ADMIN_TELEGRAM_IDS`.
15. **Джерела**: до написання collector перевіряємо ToS / robots.txt і частоту запитів. Висновок записуємо в `docs/sources/<chain>.md`, а реальні відповіді кладемо в `tests/fixtures/<chain>/`. Тести collector працюють тільки на fixtures, без мережі.

## Модель даних (ТЗ §9)

`countries`, `cities`, `users` (language, city_id, notification_mode, digest_time, digest_frequency), `user_stores`, `user_loyalty_cards`, `stores`, `campaigns` (chain, store_id nullable, source_type, status, etag), `raw_snapshots`, `products`, `product_aliases`, `category_synonyms`, `offers`, `price_observations`, `watchlist_items`, `watchlist_stores`, `offer_matches`, `shopping_lists`, `shopping_list_items`, `notification_log`, `collector_runs`. Повний перелік полів є в ТЗ.

## Етапи

0. Дослідження джерел (листівки й каталоги) → `docs/sources/*.md` + fixtures. **Блокує всі collectors.**
1. Каркас: uv, Docker Compose, config, structlog, healthcheck, Alembic, Taskiq, i18n (en).
2. Модель даних + repositories + міграції.
3. Telegram MVP: /start, меню, мережі й картки, режим сповіщень, FSM додавання товару.
4. Collector Kaufland → normalize → збереження в БД.
5. Matching (типи збігу, синоніми EN/UA/SK, unaccent, confidence).
6. Миттєві сповіщення + дедуплікація + rate-limit.
7. Collectors Lidl і Tesco (листівки й онлайн-каталог).
8. Ціна за одиницю, порівняння з картками, історія спостережень.
9. Список покупок + дайджест (режими digest/both).
10. Тестування відмов, бекапи, деплой на VPS.

## Обов'язкові тест-кейси (ТЗ §16.1)

- Той самий товар у двох мережах у різному фасуванні.
- Акція, що починається в майбутньому.
- Акція, що закінчилась до відправки дайджесту.
- Ціна тільки за карткою.
- Є і звичайна ціна, і ціна за карткою, але користувач картки не має.
- Повторний імпорт тієї самої листівки.
- Відповідь джерела: порожня сторінка, 429 або 5xx.
- Користувач вимкнув магазин після створення запиту.
- Користувач видалив запит під час matching.
- Запит `coffee` знаходить `káva`.
- Telegram повертає RetryAfter; користувач заблокував бота.
- У режимі digest немає миттєвих сповіщень; у режимі both немає дублікатів.

## Відкриті питання

Наразі відкритих питань немає. Нові питання записуй сюди. Коли питання закривається, переноси рішення в розділ «Прийняті рішення» і в ТЗ (§19 і Додаток C, історія змін).
