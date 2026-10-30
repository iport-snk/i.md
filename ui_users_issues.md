# Опис інтерфейсу: Картка абонента — Задачі (`ui_users_issues.md`)

Цей документ містить стислий та структурований опис екранної форми картки абонента у розділі «Задачі» (інтеграція з Jira та Call-Center CRM):
- **URL сторінки:** `https://a.iport.net.ua/#/user/40030/issues` *(маршрут у SPA: `/#/user/:mid/issues`)*
- **Основний запит списку задач:** `GET https://a.iport.net.ua/cc/jira-filter/274.232,0503315779~HD,SL,CRM`
- **Компонент Vue:** [`UserIssues.vue`](file:///c:/Projects/icrm/ui.2.1/src/components/UserIssues.vue)

---

## 1. Карта нумерації елементів на скріншоті

![Розмітка елементів UI](file:///c:/Projects/icrm/test-mysql/icrm-server/docs.md/annotated_issues.png)

---

## 2. Таблиця мапінгу елементів на дані API та БД

| № | Елемент на екрані | Значення на скріншоті | Поле в API `/cc/jira-filter/...` | Таблиця та колонка в БД | Логіка формування / обчислення |
| :-: | :--- | :--- | :--- | :--- | :--- |
| **1** | **Вкладка «ЗАДАЧИ»** | `ЗАДАЧИ` (активна) | Проп `projects = 'HD,SL,CRM'` | Маршрутизатор SPA (`router/index.js`) | Відображає задачі техпідтримки (HelpDesk `HD`), сервісної лінії (`SL`) та CRM. Активна вкладка підкреслена синьою лінією. |
| **2** | **Ключ задачі Jira** | `HD-313092`, `HD-301888`, `HD-295435` | `key` | `jira.jiraissue.issuenum`, `jira.project_key.project_key` | Синя кнопка-посилання `CONCAT(project_key, '-', issuenum)`. Клік викликає `openJira(key)` та відкриває задачу у новій вкладці (`http://jira.iport.net.ua/browse/HD-313092`). |
| **3** | **Іконка типу задачі** | Синій чекбокс `assignment_turned_in` | `issuetype` | `jira.jiraissue.issuetype` | Відображає тип заявки: синя іконка для звичайних завдань/запитів, червона `assignment_late` — якщо `issuetype == '10004'` (інцидент / аварія). |
| **4** | **Статус задачі** | `Done`, `Closed` | `status.key`, `status.id` | `jira.issuestatus.pname`, `jira.issuestatus.id` | Кольоровий бейдж: **зелений** для завершених статусів `[10001, 6, 10005]` (`Done`, `Closed`, `Resolved`), **жовтий** для `3` (`In Progress`), **червоний** для відкритих/нерозподілених. |
| **5** | **Термін виконання (Due Date)** | `2026-08-31` *(на задачі HD-295435)* | `duedate` | `jira.jiraissue.duedate` | Синій чіп з дедлайном (`item.duedate.split('T')[0]`). З'являється, лише якщо дедлайн задано. Підказка при наведенні: *«Исполнение запланировано на YYYY-MM-DD»*. |
| **6** | **Зв'язана задача (Issue Link)** | `NET-94470` *(на задачі HD-295435)* | `issueLinks` (масив ключів) | `jira.issuelink` (`destination` / `source`), `jira.jiraissue` | Посилання на прив'язану технічну задачу монтажників/мережевиків (проект `NET`). Клік відкриває зв'язану задачу в Jira (`/browse/NET-94470`). |
| **7** | **Дати створення та оновлення** | `2026-10-29` *(верх)*, `2026-10-29` *(низ)* | `created`, `updated` | `jira.jiraissue.created`, `jira.jiraissue.updated` | Верхній жирний рядок — дата створення звернення (`format(item.created)`). Нижній дрібний сірий рядок — дата останньої активності/оновлення (`format(item.updated)`). |
| **8** | **Опис задачі / Стенограма ШІ** | Текст звернення з координатами та стенограмою | `description`, `summary` | `jira.jiraissue.description` (та `summary` для CouchDB `CRM.*`) | Детальний текст звернення. На скріншоті видно автоматичну стенограму розмови клієнта з голосовим асистентом iPort, номер договору, телефон, UUID дзвінка та координати будинку Google Maps. |
| **9** | **Стрілка розгортання коментарів** | `keyboard_arrow_down` | `comment.comments` | `jira.jiraaction` (`actiontype = 'comment'`) | З'являється, якщо у закритій задачі є внутрішні коментарі співробітників. Клік розгортає журнал коментарів з авторами (`displayName`) та часом без переходу в Jira. |
| **10** | **Оновлення секції** | Кругла кнопка `↻` | Проп `:ver` | `UserDetails.vue:reloadSection` | Змінює значення `ver = Math.random()`, що ініціює перезавантаження списку задач з бекенда. |

---

## 3. Розбір задач зі скріншота

1. **`HD-313092` (29.10.2026) | Статус: `Done`:**
   - **Суть:** Звернення клієнта через голосового робота/кол-центр із проханням з'єднати з оператором щодо заборгованості заблокованого договору (`274.232`).
   - У тексті опису збережено повний транскрипт діалогу: борг `-339.50 грн`, планове списання `350 грн`, рекомендована сума поповнення `691 грн`, а також посилання на координати будинку у Google Maps.
2. **`HD-301888` (02.09.2026) | Статус: `Closed`:**
   - **Суть:** Автоматична або операторська заявка на активацію тимчасового доступу: *«Финансы : Временный доступ - 274.232 : +»* (збігається за часом із записом тимчасового доступу в таблиці фінансів від 02.09.2026 10:53).
3. **`HD-295435` (24.07.2026) | Термін: `2026-08-31` | Статус: `Closed` | Зв'язок: `NET-94470`:**
   - **Суть:** Заявка на первинне підключення абонента з тарифом `350 грн` та вартістю робіт `1800 грн`.
   - Пов'язана із мережевим нарядом **`NET-94470`** (монтажні роботи бригади), закрита 31.08.2026 (день первинної активації договору).

---

## 4. Архітектура взаємодії з Jira

Дані про задачі формуються **напряму з системи Atlassian Jira**, використовуючи два паралельні механізми інтеграції:

1. **Пряме підключення до реляційної бази даних Jira (`db.jira` — MySQL):**
   - Використовується основним роутом картки абонента `GET /cc/jira-filter/:filter` ([`routes/call-center/model.js`](file:///c:/Projects/icrm/test-mysql/icrm-server/routes/call-center/model.js)).
   - Для миттєвого відображення історії звернень (без затримок HTTP REST API) робиться прямий SQL-запит до таблиць Jira: `jiraissue` (основні дані), `project_key` (префікс проекту), `customfieldvalue` (`customfield = 10201` — номер договору абонента), `issuestatus` (статуси), `issuelink` (зв'язані задачі монтажників `NET-*`), `jiraaction` (журнал внутрішніх коментарів).

2. **Jira REST API v2 (`${env.jira.baseUrl}/api/2/...`):**
   - Використовується серверним модулем [`libs/jira.js`](file:///c:/Projects/icrm/test-mysql/icrm-server/libs/jira.js) та роутами [`routes/jira.js`](file:///c:/Projects/icrm/test-mysql/icrm-server/routes/jira.js) для всіх оперативних мутацій: пошук за довільним JQL, створення нових заявок, додавання коментарів оператора, зміна статусів (transitions) та завантаження вкладень.
   - Запити автентифікуються через заголовок `Authorization: Basic ${user.jira_auth}` з персональними доступами співробітника або сервісного акаунта `IVR`.

---

## 5. Специфікація кастомних полів Jira (Custom Fields)

У Jira налаштовано спеціалізовані поля, що зв'язують задачі з білінгом:

| Поле в Jira API | ID поля в БД | Призначення | Приклад значення |
| :--- | :--- | :--- | :--- |
| `customfield_10201` | `10201` | **Номер договору абонента** (ключовий індекс зв'язку) | `"274.232"` |
| `customfield_10300` | `10300` | **Тип підключення** (`connectionType`) | `10200` (Оптика PON), `10201` (Ethernet) |
| `customfield_10400` | `10400` | **Тип договору** (`contractType`) | `10300` (Фіз. особа), `10303` (Юр. особа) |
| `customfield_10500` | `10500` | **Канал реклами** (`adChannel`: звідки дізналися) | `10401` (Рекомендація), `10402` (Інтернет) |
| `customfield_10600` | `10600` | **Вартість підключення / робіт** (`cost` / `price`) | `1800` (грн) |
| `customfield_11000` | `11000` | **ID об'єкта або реферер** (`objectId` / `adReferer`) | `2058` (номер PON-бокса) |

---

## 6. Приклади взаємодії через Jira REST API

### 6.1. Пошук задач абонента за JQL (`POST /api/2/search`)
Використовується в [`libs/jira.js:search`](file:///c:/Projects/icrm/test-mysql/icrm-server/libs/jira.js#L32) та [`routes/jira.js:360-370`](file:///c:/Projects/icrm/test-mysql/icrm-server/routes/jira.js#L360-L370):

```http
POST http://jira.iport.net.ua/rest/api/2/search
Authorization: Basic SVZSOjFxYXp4c3cy
Content-Type: application/json

{
  "jql": "(project = HD OR project = SL) AND Contract ~ '274.232' ORDER BY createdDate DESC",
  "maxResults": 100,
  "fields": [
    "summary",
    "description",
    "status",
    "issuetype",
    "priority",
    "duedate",
    "created",
    "updated",
    "assignee",
    "customfield_10201",
    "issuelinks",
    "comment"
  ]
}
```

**Відповідь Jira (фрагмент):**
```json
{
  "total": 3,
  "issues": [
    {
      "id": "313092",
      "key": "HD-313092",
      "fields": {
        "summary": "Вороньків, вул. Незалежності, 5(a)",
        "description": "Вороньків, вул. Незалежності, 5(a) : Абонент попросив з'єднати з оператором. Договір: 274.232...",
        "issuetype": { "id": "10002", "name": "Task" },
        "status": { "id": "10001", "name": "Done" },
        "customfield_10201": "274.232",
        "created": "2026-10-29T10:14:22.000+0200",
        "updated": "2026-10-29T10:25:10.000+0200"
      }
    }
  ]
}
```

---

### 6.2. Створення звернення або заявки на підключення (`POST /api/2/issue`)
Використовується в [`libs/jira.js:createJira`](file:///c:/Projects/icrm/test-mysql/icrm-server/libs/jira.js#L94-L167):

```http
POST http://jira.iport.net.ua/rest/api/2/issue
Authorization: Basic b3BlcmF0b3I6cGFzc3dvcmQ=
Content-Type: application/json

{
  "fields": {
    "project": { "key": "HD" },
    "summary": "Вороньків, вул. Незалежності, 5(a)",
    "description": "Консультація щодо нарахувань за договором 274.232\nКудрик Катерина\n0503315779\nДоговір: 274.232\n[Google Map vo335 | https://www.google.com/maps/search/?api=1&query=50.210359,30.901521]",
    "issuetype": { "id": "10002" },
    "customfield_10201": "274.232",
    "customfield_11000": 2058,
    "labels": ["Вороньків", "task"]
  }
}
```

**Відповідь Jira:**
```json
{
  "id": "313092",
  "key": "HD-313092",
  "self": "http://jira.iport.net.ua/rest/api/2/issue/313092"
}
```

---

### 6.3. Додавання коментаря в існуючу задачу (`POST /api/2/issue/{issueKey}/comment`)
Викликається при натисканні кнопки **«Коментарий»** у картці задачі ([`UserIssues.vue:151-164`](file:///c:/Projects/icrm/ui.2.1/src/components/UserIssues.vue#L151-L164) $\to$ [`libs/jira.js:comment`](file:///c:/Projects/icrm/test-mysql/icrm-server/libs/jira.js#L478-L485)):

```http
POST http://jira.iport.net.ua/rest/api/2/issue/HD-313092/comment
Authorization: Basic b3BlcmF0b3I6cGFzc3dvcmQ=
Content-Type: application/json

{
  "body": "Абоненту роз'яснено графік списання коштів та умови закриття договору."
}
```

---

### 6.4. Зміна статусу задачі / Workflow Transitions (`POST /api/2/issue/{issueKey}/transitions`)
Використовується для переводу задачі в роботу або закриття ([`libs/jira.js:trans`](file:///c:/Projects/icrm/test-mysql/icrm-server/libs/jira.js#L511), [`routes/jira.js:30-40`](file:///c:/Projects/icrm/test-mysql/icrm-server/routes/jira.js#L30-L40)):

```http
POST http://jira.iport.net.ua/rest/api/2/issue/HD-313092/transitions
Authorization: Basic b3BlcmF0b3I6cGFzc3dvcmQ=
Content-Type: application/json

{
  "transition": {
    "id": "41"
  }
}
```

---

### 6.5. Завантаження PDF-акту або документа в задачу (`POST /api/2/issue/{issueKey}/attachments`)
Використовується для збереження згенерованих актів підключення / виконаних робіт монтажників ([`libs/jira.js:attachFile`](file:///c:/Projects/icrm/test-mysql/icrm-server/libs/jira.js#L77-L93)):

```http
POST http://jira.iport.net.ua/rest/api/2/issue/NET-94470/attachments
Authorization: Basic SVZSOjFxYXp4c3cy
X-Atlassian-Token: no-check
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryX

------WebKitFormBoundaryX
Content-Disposition: form-data; name="file"; filename="act_274232.pdf"
Content-Type: application/pdf

<binary PDF stream>
------WebKitFormBoundaryX--
```

---

## 7. Додаткові дії оператора у вкладці «Задачі»

- **Додавання коментаря в задачу:**
  - Якщо задача ще не закрита (`!isJiraDone(item)`), праворуч у заголовку картки доступна кнопка **`Коментарий`** з іконкою `add_comment`.
  - Введення тексту та клік «Добавить» відправляє запит `POST /jira/issue/comment` (`{jira: item.key, comment: text}`).
- **Створення нової заявки (Рожева плаваюча кнопка `+` у правому нижньому куті):**
  - Клік по кнопці `+` ([`UserIssues.vue:84-93`](file:///c:/Projects/icrm/ui.2.1/src/components/UserIssues.vue#L84-L93)) викликає подію `$bus.$emit('OPEN_JIRA', this.user)` та відкриває діалогове вікно **[`JiraDialog.vue`](file:///c:/Projects/icrm/ui.2.1/src/components/JiraDialog.vue)** із автозаповненням реквізитів абонента (договір, адреса, телефон).
