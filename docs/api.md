# API

> Часть проекта **uniform-orders**. Тот же текст, что в [architecture.md](architecture.md) (разделы 6), вынесен в отдельный файл.

## 6. API

*API — это набор адресов (эндпоинтов) на сервере, по которым React-приложение просит у сервера данные или отправляет команды.*

**Общие правила:**
- Все адреса начинаются с `/api`.
- Формат обмена — JSON (кроме создания заявки: там ещё и файлы).
- Защищённые запросы содержат заголовок `Authorization: Bearer <токен>`.
- Ошибки всегда в одном виде: `{ "error": { "code": "VALIDATION_ERROR", "message": "Укажите телефон" } }`.
- Коды ответов: `200` успех, `201` создано, `400` неверные данные, `401` не вошёл, `403` нет прав, `404` не найдено, `409` нельзя в этом состоянии (например, неверный переход статуса), `429` слишком много запросов.

### 6.1. Таблица эндпоинтов

Адреса разбиты на четыре группы, чтобы защиту можно было включать на всю группу сразу:
- `/api/public/*` — без входа (гость);
- `/api/auth/*` — вход;
- `/api/orders/*` — только сотрудник и администратор;
- `/api/admin/*` — только администратор.

| № | Метод | Адрес | Что делает | Кто имеет доступ | История |
|---|---|---|---|---|---|
| 1 | GET | `/api/public/catalog` | Активные изделия и ткани для формы | Все | US-1, 2, 4 |
| 2 | POST | `/api/public/orders` | Создать заявку (с файлами) | Все (гость) | US-1, 2, 3, 4 |
| 3 | POST | `/api/public/orders/track` | Проверить статус по номеру и телефону | Все (гость) | US-5 |
| 4 | POST | `/api/public/orders/cancel` | Отменить свою заявку (только «Новая» и «На рассмотрении») | Все (гость) | US-6 |
| 5 | POST | `/api/auth/login` | Вход сотрудника и админа | Все | US-7 |
| 6 | GET | `/api/auth/me` | Кто я (по токену) | Сотрудник, Админ | US-7 |
| 7 | GET | `/api/orders` | Список заявок: поиск, фильтры, страницы | Сотрудник, Админ | US-8 |
| 8 | GET | `/api/orders/:id` | Карточка заявки с позициями, файлами, историей | Сотрудник, Админ | US-9 |
| 9 | GET | `/api/orders/:id/files/:fileId` | Скачать файл заявки | Сотрудник, Админ | US-9 |
| 10 | PATCH | `/api/orders/:id/status` | Сменить статус (в том числе отменить по просьбе клиента) | Сотрудник, Админ (см. таблицу переходов) | US-10 |
| 11 | PATCH | `/api/admin/orders/:id/price` | Проставить итоговую цену | Админ | US-11 |
| 12 | PATCH | `/api/admin/orders/:id/assignee` | Назначить сотрудника | Админ | US-12 |
| 13 | GET | `/api/admin/product-types` | Все изделия (и скрытые) | Админ | US-13 |
| 14 | POST | `/api/admin/product-types` | Добавить изделие | Админ | US-13 |
| 15 | PATCH | `/api/admin/product-types/:id` | Изменить цену, название, скрыть | Админ | US-13 |
| 16 | GET | `/api/admin/fabrics` | Все ткани | Админ | US-13 |
| 17 | POST | `/api/admin/fabrics` | Добавить ткань | Админ | US-13 |
| 18 | PATCH | `/api/admin/fabrics/:id` | Изменить или скрыть ткань | Админ | US-13 |
| 19 | GET | `/api/admin/users` | Список сотрудников | Админ | US-14 |
| 20 | POST | `/api/admin/users` | Создать сотрудника | Админ | US-14 |
| 21 | PATCH | `/api/admin/users/:id` | Изменить ФИО, роль, отключить/включить | Админ | US-14 |
| 22 | PATCH | `/api/orders/:id/customer` | Исправить контакты клиента (с записью в историю) | Сотрудник, Админ | US-9 |
| 23 | POST | `/api/auth/change-password` | Сменить свой пароль | Сотрудник, Админ | US-7 |
| 24 | POST | `/api/admin/users/:id/reset-password` | Выдать временный пароль | Админ | US-14 |

> **Почему так:** по пути сразу видно, кто имеет доступ: `/public/` — любой, `/admin/` — только админ. Проверку можно поставить на всю группу одной строкой, и забыть защитить один из адресов сложнее. Раньше гостевое создание заявки и список заявок сотрудника лежали под одним адресом `/api/orders`, и это легко было перепутать.
>
> **Почему проверка статуса и отмена — POST, а не GET:** телефон — персональные данные. В GET он попадёт в адресную строку и в журналы серверов. В POST он лежит в теле запроса.

### 6.2. Примеры запросов и ответов

**№1. GET `/api/public/catalog`**

Ответ `200`:
```json
{
  "productTypes": [
    { "id": 1, "name": "Рубашка", "basePrice": 6000, "logoPrice": 800 },
    { "id": 2, "name": "Брюки", "basePrice": 7500, "logoPrice": 0 }
  ],
  "fabrics": [
    { "id": 1, "name": "Хлопок" },
    { "id": 2, "name": "Габардин" }
  ]
}
```

Форма использует эти цены, чтобы показать ориентировочную сумму до отправки. Это только подсказка: при создании заявки сервер считает сумму заново (BR-6).

**№2. POST `/api/public/orders`**

Запрос: `multipart/form-data` (формат, который позволяет отправить файлы вместе с данными). Поле `data` содержит JSON, поле `files` содержит файлы:
```json
{
  "customerType": "ORGANIZATION",
  "customerName": "Айгерим Серикова",
  "companyName": "Кафе «Арай»",
  "taxId": "123456789012",
  "phone": "+7 701 123 45 67",
  "email": "aigerim@example.com",
  "desiredDate": "2026-12-20",
  "comment": "Логотип во вложении",
  "items": [
    { "productTypeId": 1, "size": "M", "height": 170, "color": "Белый",
      "fabricId": 2, "quantity": 10, "hasLogo": true },
    { "productTypeId": 2, "size": "L", "height": 180, "color": "Чёрный",
      "fabricId": 1, "quantity": 5, "hasLogo": false }
  ]
}
```
Ответ `201`:
```json
{
  "orderNumber": 128,
  "status": "NEW",
  "estimatedPrice": 105500,
  "message": "Заявка принята. Сохраните номер и телефон, чтобы проверять статус."
}
```
*(Расчёт: 10 × (6000 + 800) + 5 × 7500 = 105 500.)*

> **Почему так:** одна общая отправка (данные + файлы) означает, что заявка либо создаётся целиком, либо не создаётся. Если бы файлы отправлялись отдельным запросом, гость мог бы прикрепить файл к чужой заявке, и пришлось бы придумывать для этого защиту.

**№3. POST `/api/public/orders/track`**

Запрос:
```json
{ "orderNumber": 128, "phone": "+7 701 123 45 67" }
```
Ответ `200`:
```json
{
  "orderNumber": 128,
  "status": "IN_PROCESSING",
  "canCancel": false,
  "estimatedPrice": 105500,
  "finalPrice": null,
  "createdAt": "2026-11-16T09:30:00Z",
  "items": [
    { "productName": "Рубашка", "size": "M", "quantity": 10, "hasLogo": true }
  ],
  "files": [ { "originalName": "logo.png" } ],
  "history": [
    { "status": "NEW", "date": "2026-11-16T09:30:00Z" },
    { "status": "UNDER_REVIEW", "date": "2026-11-16T11:00:00Z" }
  ]
}
```
Поле `canCancel` подсказывает странице, показывать ли кнопку «Отменить». Решение всё равно принимает сервер (BR-16).

Ошибка `404` при неверной паре (одинаково для «нет такого номера» и «не тот телефон»):
```json
{ "error": { "code": "NOT_FOUND", "message": "Заявка не найдена. Проверьте номер и телефон." } }
```

**№4. POST `/api/public/orders/cancel`**

Запрос:
```json
{ "orderNumber": 128, "phone": "+7 701 123 45 67", "reason": "Ошибся с размерами" }
```
Поле `reason` необязательное. Ответ `200`:
```json
{ "orderNumber": 128, "status": "CANCELLED" }
```
Ошибка `409`, если заявка уже в работе:
```json
{ "error": { "code": "CANCEL_NOT_ALLOWED", "message": "Заявка уже в работе. Позвоните в ателье." } }
```

**№5. POST `/api/auth/login`**

Запрос:
```json
{ "email": "staff@example.com", "password": "мой-пароль" }
```
Ответ `200`:
```json
{
  "token": "eyJhbGciOi...",
  "user": { "id": 2, "fullName": "Алия Нурлан", "role": "STAFF" },
  "mustChangePassword": false
}
```
Ошибка `401`: `{ "error": { "code": "INVALID_CREDENTIALS", "message": "Неверный email или пароль" } }`

Если `mustChangePassword` равно `true`, сервер принимает от этого пользователя только смену пароля (BR-19).

**№6. GET `/api/auth/me`** (с токеном)

Ответ `200`: `{ "id": 2, "fullName": "Алия Нурлан", "role": "STAFF" }`

**№7. GET `/api/orders?q=Арай&status=NEW&assignedToMe=true&page=1&limit=20`**

Параметры (всё необязательно): `q` — поиск по номеру, имени, компании, телефону; `status` — фильтр; `assignedToMe` — только мои; `page` и `limit` (по умолчанию 20, максимум 50). Сортировка: сначала новые.

Ответ `200`:
```json
{
  "items": [
    {
      "id": 128, "customerType": "ORGANIZATION", "customerName": "Айгерим Серикова",
      "companyName": "Кафе «Арай»", "status": "NEW", "itemsCount": 2,
      "estimatedPrice": 105500, "finalPrice": null,
      "assignee": null, "createdAt": "2026-11-16T09:30:00Z"
    }
  ],
  "page": 1, "limit": 20, "total": 1, "totalPages": 1
}
```

**№8. GET `/api/orders/128`**

Ответ `200` (сокращённо):
```json
{
  "id": 128, "status": "NEW", "customerType": "ORGANIZATION",
  "customerName": "Айгерим Серикова", "companyName": "Кафе «Арай»",
  "taxId": "123456789012", "phone": "77011234567", "email": "aigerim@example.com",
  "desiredDate": "2026-12-20", "comment": "Логотип во вложении",
  "estimatedPrice": 105500, "finalPrice": null,
  "assignee": null,
  "items": [ { "id": 1, "productName": "Рубашка", "size": "M", "height": 170,
               "color": "Белый", "fabricName": "Габардин", "quantity": 10,
               "hasLogo": true, "unitPrice": 6800 } ],
  "files": [ { "id": 7, "originalName": "logo.png", "sizeBytes": 204800 } ],
  "history": [ { "eventType": "STATUS", "oldValue": null, "newValue": "NEW", "changedBy": null,
                 "comment": null, "createdAt": "2026-11-16T09:30:00Z" } ]
}
```

**№9. GET `/api/orders/128/files/7`**

Ответ `200`: сам файл (в заголовках указано имя и тип). Ошибка `404`, если файл не принадлежит этой заявке.

**№10. PATCH `/api/orders/128/status`**

Запрос: `{ "status": "UNDER_REVIEW", "comment": "Макет получен" }`
Ответ `200`: `{ "id": 128, "status": "UNDER_REVIEW" }`

Отмена по просьбе клиента: `{ "status": "CANCELLED", "comment": "Клиент позвонил, отказался от заказа" }`. Без комментария ответ `400`: `{ "error": { "code": "VALIDATION_ERROR", "message": "Укажите причину отмены" } }`.

Ошибка `409`: `{ "error": { "code": "INVALID_TRANSITION", "message": "Из «Новая» нельзя перейти в «Выдана»" } }`

**№11. PATCH `/api/admin/orders/128/price`**

Запрос: `{ "finalPrice": 110000 }`
Ответ `200`: `{ "id": 128, "finalPrice": 110000 }`
Ошибка `409`, если заявка уже в производстве (BR-17): `{ "error": { "code": "PRICE_LOCKED", "message": "После начала производства цену менять нельзя" } }`

**№12. PATCH `/api/admin/orders/128/assignee`**

Запрос: `{ "assigneeId": 2 }` (`null` — снять назначение)
Ответ `200`: `{ "id": 128, "assignee": { "id": 2, "fullName": "Алия Нурлан" } }`

**№13–15. Изделия** (ткани №16–18 устроены так же, но с одним полем `name`)

- GET `/api/admin/product-types`: ответ `200` — массив изделий, как в каталоге, плюс поле `isActive`.
- POST `/api/admin/product-types`: запрос `{ "name": "Халат", "basePrice": 5500, "logoPrice": 700 }`, ответ `201` — созданное изделие с `id`.
- PATCH `/api/admin/product-types/3`: запрос `{ "basePrice": 5800, "isActive": true }`, ответ `200` — обновлённое изделие.

**№19–21. Сотрудники**

- GET `/api/admin/users`: ответ `200` — `[ { "id": 2, "email": "...", "fullName": "...", "role": "STAFF", "isActive": true } ]`.
- POST `/api/admin/users`: запрос `{ "email": "new@example.com", "fullName": "Дана Ахметова", "password": "временный-пароль", "role": "STAFF" }`, ответ `201` — пользователь без пароля.
- PATCH `/api/admin/users/2`: запрос `{ "isActive": false }`, ответ `200` — обновлённый пользователь. Ошибка `409`, если пытаются отключить последнего администратора.

**№22. PATCH `/api/orders/128/customer`**

Запрос (все поля, кроме `comment`, необязательные; `comment` с причиной обязателен):
```json
{ "phone": "+7 701 765 43 21", "comment": "Клиент ошибся в номере" }
```
Ответ `200`: `{ "id": 128, "phone": "77017654321" }`. В историю добавляется событие `CUSTOMER` с прежним и новым значением.

**№23. POST `/api/auth/change-password`**

Запрос:
```json
{ "currentPassword": "временный-пароль", "newPassword": "мой-новый-пароль" }
```
Ответ `200`: `{ "ok": true }`. Ошибка `400`, если новый пароль короче 8 символов, и `401`, если текущий пароль неверный.

**№24. POST `/api/admin/users/2/reset-password`**

Запрос: `{ "temporaryPassword": "новый-временный" }`
Ответ `200`: `{ "id": 2, "mustChangePassword": true }`

> **Почему так:** сервер никогда не возвращает `password_hash`, и в каждый ответ попадают только нужные поля. Иначе по ошибке можно «отдать» лишнее.
