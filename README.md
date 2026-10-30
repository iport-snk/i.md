---
hidden: true
layout:
  width: wide
  outline:
    visible: false
---

# Архітектура та опис системи `icrm-server`

Цей каталог (`docs.md/`) містить документацію серверної частини системи **iCRM** (iPort CRM / Billing / Network Management), побудовану за принципом **«зверху вниз»** (від загальної архітектури та точок входу до окремих бізнес-модулів, фонових задач та інтеграцій).

---

## 1. Загальне призначення системи

**`icrm-server`** — центральний бекенд-сервер інтернет-провайдера **iPort**, написаний на **Node.js (Express)**. Система об'єднує в собі:
- **CRM та обслуговування абонентів** (картка клієнта, особистий кабінет, звернення, заявки на підключення, опитування).
- **Білінг та фінансовий облік** (нарахування абонплати, платежі, банківські виписки Приват24 / 1С / WayForPay / iPay, звірка реєстрів, бонуси, повернення коштів, інвойси).
- **Управління мережею та обладнанням (PON / OLT / ONU / RADIUS)** (моніторинг сигналів, конфігурація ONU через Telnet/SNMP, обмеження швидкості, керування сесіями).
- **Комунікації та Call-центр** (інтеграція з Asterisk AMI/CDR, IVR, WebSocket-сповіщення про активні дзвінки, розсилки SMS / Viber / Push / Web-Push / Email).
- **Управління виїзними бригадами та задачами** (інтеграція з Jira, маршрутні листи, облік робочого часу `timesheet`, GPS-трекінг Traccar, паливо).
- **Додаткові сервіси** (IPTV Trinity, відеоспостереження Shinobi CCTV, IoT / Tuya, карти покриття та мережевої інфраструктури).

---

## 2. Точки входу (Entry Points)

Система має кілька режимів запуску залежно від ролі процесу:

| Файл | Призначення | Опис |
| :--- | :--- | :--- |
| [`bin/www`](file:///c:/Projects/icrm/test-mysql/icrm-server/bin/www) | **HTTP + WebSocket сервер** (основний запуск `npm start`) | Підіймає HTTP-сервер на порту `process.env.PORT` (за замовчуванням `82`), монтує Express-додаток [`app.js`](file:///c:/Projects/icrm/test-mysql/icrm-server/app.js) та ініціалізує WebSocket-сервер [`app-socket.js`](file:///c:/Projects/icrm/test-mysql/icrm-server/app-socket.js). |
| [`server.js`](file:///c:/Projects/icrm/test-mysql/icrm-server/server.js) | **HTTPS (Greenlock SSL) сервер** | Альтернативна точка входу з автоматичним отриманням SSL-сертифікатів через `greenlock-express` (порти `80` та `443`). |
| [`cron.js`](file:///c:/Projects/icrm/test-mysql/icrm-server/cron.js) | **Планувальник фонових задач (Cron)** | Окремий процес для виконання регламентних робіт за розкладом із відправкою метрик виконання у **Zabbix** (через [`libs/zabbix.js`](file:///c:/Projects/icrm/test-mysql/icrm-server/libs/zabbix.js)). |
| [`new_month.js`](file:///c:/Projects/icrm/test-mysql/icrm-server/new_month.js) / [`jobs/new-month-pm2.js`](file:///c:/Projects/icrm/test-mysql/icrm-server/jobs/new-month-pm2.js) | **Перехід на новий місяць** | Скрипти запуску процедури закриття/відкриття розрахункового періоду ([`jobs/new_month.js`](file:///c:/Projects/icrm/test-mysql/icrm-server/jobs/new_month.js)). |

---

## 3. Верхньорівнева структура проєкту

```text
icrm-server/
├── bin/www                 # Точка входу HTTP + WebSocket сервера
├── server.js               # Точка входу HTTPS (Greenlock)
├── app.js                  # Ініціалізація Express, глобальних об'єктів, middleware та роутів
├── app-socket.js           # WebSocket-сервер + підписка на Redis Pub/Sub ('aster')
├── auth.js                 # Middleware автентифікації та авторизації (JWT, PG-JWT, IP, Token)
├── db.js                   # Пули/обгортки підключень до баз даних MySQL
├── env.js                  # Конфігурація середовища та параметрів підключень
├── cron.js                 # Реєстрація та розклад фонових cron-задач із моніторингом Zabbix
├── messages.js             # Шаблони повідомлень (Viber/SMS розсилки)
├── routes/                 # HTTP-контролери (Express Routers), згруповані за доменами
│   ├── call-center/        # Роути та модель для кол-центру (/cc)
│   └── profile/            # Роути особистого кабінету абонента (/profile, /profile/pay)
├── libs/                   # Бізнес-логіка, сервісні модулі та інтеграції з зовнішніми API
│   ├── telnet/             # Модулі взаємодії з мережевим обладнанням по Telnet
│   └── tuya/               # Інтеграція з розумними пристроями Tuya IoT
├── jobs/                   # Фонові та періодичні завдання (білінг, звірки, звіти, моніторинг)
├── docs/                   # Статичні шаблони та контент для роутів /content та /public-content
├── views/                  # EJS-шаблони
├── public/                 # Статичні файли веб-сервера
└── sql/                    # Допоміжні SQL-скрипти
```

---

## 4. Основні інфраструктурні шари

### 4.1. Глобальний контекст та кешування ([`app.js`](file:///c:/Projects/icrm/test-mysql/icrm-server/app.js))
При старті [`app.js`](file:///c:/Projects/icrm/test-mysql/icrm-server/app.js) ініціалізує:
- `global.dir` — коренева директорія проєкту.
- `global.cache` — in-memory TTL-кеш (`@isaacs/ttlcache`, `max: 10000`, `ttl: 1000ms`), що використовується зокрема для кешування перевірки токенів `pgjwt`.
- `global.pgauth` — клієнт PostgreSQL (`pg-promise`) для перевірки централізованих JWT-токенів через функцію БД `jwt_decrypt(token, 'icrm')`.
- `global.log` — файловий логер (`simple-node-logger`) з ротацією по днях у папці `logs/`.
- `auth.init()` — завантаження списку внутрішніх користувачів (`icrm.users`) та їхніх прав у пам'ять.
- `money.setServices()` — ініціалізація довідника послуг білінгу при старті.

### 4.2. Шар баз даних ([`db.js`](file:///c:/Projects/icrm/test-mysql/icrm-server/db.js))
Система працює з багатьма базами даних **MySQL** (через `promise-mysql`) та **PostgreSQL** (`pg-promise`), а також **CouchDB** ([`libs/couch.js`](file:///c:/Projects/icrm/test-mysql/icrm-server/libs/couch.js)) і **Redis** ([`libs/redis.js`](file:///c:/Projects/icrm/test-mysql/icrm-server/libs/redis.js)):

| Підключення у [`db.js`](file:///c:/Projects/icrm/test-mysql/icrm-server/db.js) | Призначення |
| :--- | :--- |
| `db.bill` | Основна база білінгу (абоненти, платежі, послуги, тарифи). |
| `db.bro` | Друга/регіональна база білінгу (Бровари / окремий сегмент). |
| `db.ibill` | Додаткова білінгова база (`ibill`). |
| `db.icrm` | Внутрішня база CRM (користувачі, права, замовлення, довідники CRM). |
| `db.traff` | База обліку трафіку та мережевих сесій. |
| `db.rad`, `db.radSecond` | Бази серверів FreeRADIUS (авторизація абонентів, `radreply`, `radacct`). |
| `db.pon`, `db.ponMaster` | Бази даних моніторингу та управління PON/OLT/ONU. |
| `db.aster` | База Asterisk (CDR дзвінків, черги, статистика телефонії). |
| `db.jira` | База даних Jira (задачі, інциденти, заявки). |
| `db.turbo` | База TurboSMS (шлюз відправки SMS-повідомлень). |
| `db.shinobi` | База системи відеоспостереження Shinobi (CCTV). |
| `db.traccar` | База GPS-моніторингу транспорту та бригад Traccar. |
| `db.fin` | База фінансового обліку. |

### 4.3. Шар автентифікації та конвеєр запитів ([`auth.js`](file:///c:/Projects/icrm/test-mysql/icrm-server/auth.js) та [`app.js`](file:///c:/Projects/icrm/test-mysql/icrm-server/app.js))
Маршрути в [`app.js`](file:///c:/Projects/icrm/test-mysql/icrm-server/app.js) поділені на **4 зони доступу**:
1. **Публічні та зовнішні вебхуки** (монтуються **до** `auth.check`):
   - `/auth-jwt`, `/audio`, `/resumable`, `/push`, `/web-push`, `/shinobi-auth`, `/ukraine`, `/_p` (зовнішні платіжні шлюзи), `/pb` (Приват24), `/public-content`.
2. **Кабінет абонента (`/profile`)** — захищений middleware [`auth.validate`](file:///c:/Projects/icrm/test-mysql/icrm-server/auth.js#L119-L123) (перевірка JWT клієнта в заголовку `auth` або пропуск `/ping-*`).
3. **Локальні/внутрішньомережеві роути (`/private`)** — захищені middleware [`auth.private`](file:///c:/Projects/icrm/test-mysql/icrm-server/auth.js#L125-L128) (дозволяє запити лише з `loopback` або `private` IP-діапазонів).
4. **Захищені CRM-роути** (монтуються **після** `app.use(auth.check)`):
   - Middleware [`auth.check`](file:///c:/Projects/icrm/test-mysql/icrm-server/auth.js#L109-L117) підтримує 3 способи авторизації:
     1. Заголовок `pgjwt` — розшифровка токена через PostgreSQL (`global.pgauth`) з кешуванням у `global.cache`.
     2. Заголовок `authorization` — декодування клієнтського JWT (`jwt-simple`).
     3. Заголовок `secret` або `token` — пошук користувача в кешованому масиві `users` з таблиці `icrm.users` (плюс спеціальні правила для IVR по IP мережі iPort та вебхука TurboSMS).

---

## 5. Документи в `docs.md/` та план подальшої документації

Наявні документи:
- **[`users.md`](./users.md)** — Детальний опис структури абонентських даних у білінгу (`users` із самозв'язком `id = mid`, `users_trf`, `user_grp`, `regions`, `p_street`, `plans2`, `phones`, `comments`, `pays`, `pb_trans`, `c24_trans`), їхніх зв'язків, кодів статусів/категорій та всіх полів, що реально використовуються в коді.
- **[`aaa.md`](./aaa.md)** — Опис бізнес-процесів мережевої авторизації абонентів (AAA / `rad_reply`) та щомісячного нарахування абонентської плати ([`jobs/new_month.js`](file:///c:/Projects/icrm/test-mysql/icrm-server/jobs/new_month.js)).
- **[`pays.md`](./pays.md)** — Опис чотирьох способів поповнення балансу абонентом (додаток Приват24 `pb_trans`, мережа City24 `c24_trans`, безготівкові платежі `fin.statements` з автоматичним та ручним рознесенням у проєкті `fin`, інвойси Приват24 з автогенерацією через додаток iPort) та процедури щоденної автоматизованої звірки реєстрів із банківськими виписками для остаточного підтвердження платежів у `pays`.
- **[`ui_users_fin.md`](./ui_users_fin.md)** — Стислий та наочний опис форми картки абонента у розділі «Фінанси» (`/#/user/:mid/payments`): нумерація всіх елементів на скріншоті, прямий мапінг полів API (`/users/info/:mid` та `/pays/list/:mid`) на таблиці БД (`bill.users`, `bill.pays`, `bill.phones` тощо), розбір операцій та модального вікна вибору/зміни тарифу за регіоном.
- **[`ui_users_devices.md`](./ui_users_devices.md)** — Стислий та наочний опис розділу «Устройства» (`/#/user/:mid/devices`): мережеві пристрої клієнта, прив'язка MAC/IP, моніторинг сесій у FreeRADIUS (`radpostauth_log`), оптичні параметри ONU (Rx/Tx dBm) через NOC Zabbix API, опитування FDB-маків та прив'язка до топології PON-боксів у CouchDB.
- **[`ui_users_cdr.md`](./ui_users_cdr.md)** — Стислий та наочний опис розділу «Звонки» (`/#/user/:mid/cdr`): історія вхідних/вихідних дзвінків через Asterisk PBX, двошарова агрегація черг операторів за `uniqueid`/`linkedid`, розбір тривалості розмови та очікування, а також відтворення аудіозаписів (`recordingfile`).
- **[`ui_users_issues.md`](./ui_users_issues.md)** — Стислий та наочний опис розділу «Задачі» (`/#/user/:mid/issues`): нумерація елементів на скріншоті, прямий мапінг полів API (`/cc/jira-filter/...`) на таблиці БД (`jira.jiraissue`, `jira.project_key`, `jira.issuestatus`, CouchDB `crm`) та розбір логіки завдань і стенограми дзвінків/AI-асистента.
- **[`ui_users_search.md`](./ui_users_search.md)** — Стислий та наочний опис форми пошуку абонентів (`/#/users`): нумерація елементів на скріншоті, повний мапінг фільтрів адреси та результатів на поля таблиць БД (`users`, `p_street`, `plans2`, `comments`), генерація динамічного SQL-запиту в `routes/search.js` та переходи до картки клієнта.
- **[`ui-user-payments.md`](./ui-user-payments.md)** — Розширена технічна документація по UI картки абонента та вкладки «Фінанси» (`/#/user/:mid/payments`, проєкт `ui.2.1`): дерево Vue-компонентів, повний розбір основного запиту `GET /users/info/:mid` та запиту `GET /pays/list/:mid`, мапінг полів відповіді на елементи інтерфейсу, а також логіка всіх кнопок і модальних діалогів.
- **[`ui_orders.md`](./ui_orders.md)** — Опис інтерфейсу «ЗАЯВКИ» (`/#/userOrders`): імпорт заявок з Jira (`HD` та `NET`), бізнес-флоу статусів (Входящие, Бэклог, Согласовано, Изменено, Просрочено), мапінг елементів карток на поля API та автоматизоване створення облікових записів абонентів у NoDeny.

План подальшої документації (Зверху Вниз):
1. **`01-architecture-and-boot.md`** — Детальний розбір запуску, конфігурації (`env.js`), глобальних сервісів, WebSocket (`app-socket.js`) та схем автентифікації (`auth.js`).
2. **`02-databases-and-storage.md`** — Опис усіх підключень (`db.js`, PostgreSQL, Redis, CouchDB), їхньої ролі та взаємозв'язків між базами.
3. **`03-api-routes-map.md`** — Повна карта HTTP-роутів (`routes/*`), згрупована за функціональними доменами (Білінг, Мережа/PON, CRM/Клієнти, Телефонія/КЦ, Завдання/Jira, Кабінет).
4. **`04-core-libs-and-domain.md`** — Розбір ядра бізнес-логіки в `libs/` (білінг `money.js` / `monthly.js` / `bill.js`, мережа `pon.js` / `speed-limit.js`, інтеграції).
5. **`05-cron-and-background-jobs.md`** — Опис регламентних процесів (`cron.js`, `jobs/*`, `new_month.js`, моніторинг Zabbix).
