# Интеграции. Часть 2. Описание интеграций: контракты, ошибки, версионирование

**Татьяна · подготовка к собесу · 90 минут**

---

## План занятия

1. Контракт интеграции — 7 частей
2. Структура REST-контракта: метод, путь, тело, ответы
3. JSON-схема данных: типы, обязательность, ограничения (5 граблей)
4. Ошибки как часть контракта: структура, словарь кодов
5. Версионирование: breaking/non-breaking, 3 стратегии
6. OpenAPI: структура, что генерируется
7. Безопасность контракта: auth, секреты
8. Практика: 2 кейса
9. Чек-лист «контракт готов»
10. Домашнее задание

---

## 1. Контракт — 7 частей

**Контракт интеграции** — точное описание: что посылается, в каком виде, что в ответ, что при ошибке.

7 частей:
1. Эндпоинты (метод + путь)
2. Формат запроса: заголовки, query, body
3. Формат ответа: тело, статусы
4. Схемы данных
5. Ошибки
6. Версия
7. Аутентификация

---

## 2. Структура эндпоинта

```
POST /api/v1/orders
Headers:
  Authorization: Bearer {token}
  Content-Type: application/json
  Idempotency-Key: req-abc-123        (опционально, для защиты от дублей)
Query:
  (нет)
Body: CreateOrderRequest
Responses:
  201 -> Order          (Location: /api/v1/orders/123)
  400 -> Error          (валидация)
  401 -> Error          (нет токена)
  403 -> Error          (нет прав)
  404 -> Error          (товара нет)
  409 -> Error          (нет в наличии / дубль)
  422 -> Error          (семантика: дата в прошлом)
  500 -> Error          (ошибка сервера)
```

### Нейминг URI

| Правильно | Неправильно | Почему |
|---|---|---|
| `/orders` | `/order` | множественное число |
| `/payment-methods` | `/PaymentMethods` | строчные, дефисы |
| `/orders/123/payments` | `/payments?orderId=123` | вложенный ресурс |
| `POST /orders` | `POST /createOrder` | REST: ресурс + метод |
| `POST /orders/123/cancel` | — | sub-action допустим (не CRUD) |

---

## 3. Схемы данных

### Пример: `CreateOrderRequest`

| Поле | Тип | Обязательное | Ограничения | Описание |
|---|---|---|---|---|
| userId | integer | да | > 0 | Кто разместил |
| items | array[OrderItem] | да | 1..100 | Позиции |
| items[].sku | string | да | 3..32 | Артикул |
| items[].qty | integer | да | 1..999 | Кол-во |
| total | integer | да | > 0, **копейки** | Сумма |
| deliveryDate | string | нет | `YYYY-MM-DD` (ISO 8601) | Дата доставки |
| comment | string | нет | max 500 | Комментарий (отсутствует = не заполнен) |
| promoCode | string | нет | max 20 | Промокод |

### 5 граблей

| # | Грабли | Плохо | Хорошо |
|---|---|---|---|
| 1 | Деньги | `price: 5000.50` (double) | `total: 500050` (integer, копейки) |
| 2 | Даты | `"01.02.2026"` (строка) | `"2026-02-01"` (ISO 8601, date) |
| 3 | Null | `comment: null` / `""` / отсутствует — всё смешано | Поле **отсутствует**, если не заполнено |
| 4 | Enum | `status: string` (без значений) | `enum: [CREATED, PAID, SHIPPED, DELIVERED, CANCELLED]` |
| 5 | Массив | `items: array` (без границ) | `minItems: 1, maxItems: 100` |

---

## 4. Ошибки

### Единая структура ошибки

```json
{
  "errorCode": "INSUFFICIENT_STOCK",
  "message": "Товара SKU-43 недостаточно: доступно 2, запрошено 5",
  "details": {
    "field": "items[1].sku",
    "value": "SKU-43",
    "available": 2,
    "requested": 5
  }
}
```

- `errorCode` — машинный (ветвление логики)
- `message` — человекочитаемое (лог/пользователь)
- `details` — опционально, конкретика

### Словарь кодов для `POST /orders`

| errorCode | HTTP | Когда |
|---|---|---|
| VALIDATION_ERROR | 400 | Нет `userId`, `qty: -1`, битый JSON |
| AUTH_REQUIRED | 401 | Нет `Authorization` header |
| ACCESS_DENIED | 403 | Токен есть, роль `viewer` (нужен `writer`) |
| ITEM_NOT_FOUND | 404 | SKU не в каталоге |
| INSUFFICIENT_STOCK | 409 | Нет в нужном количестве |
| DUPLICATE_ORDER | 409 | Повторный `Idempotency-Key` |
| INVALID_DELIVERY_DATE | 422 | `deliveryDate` в прошлом |
| SERVER_ERROR | 500 | Нечего показывать, алерт |

### 409 vs 400

| | 400 Bad Request | 409 Conflict |
|---|---|---|
| Смысл | Запрос **битый** | Запрос **валидный**, состояние не позволяет |
| Пример | `qty: -1`, нет `userId` | Заказ уже оплачен, повторно нельзя |
| Что делать | Чини запрос | Подожди / измени состояние |

### 401 vs 403

| | 401 | 403 |
|---|---|---|
| Смысл | Нет данных для входа | Данные есть, прав нет |
| Пример | Нет `Authorization` header | Токен есть, роль не та |
| Аналогия | "Предъявите пропуск" | "Пропуск есть, но в этот кабинет нельзя" |

---

## 5. Версионирование

### 3 стратегии

| Стратегия | Пример | Когда |
|---|---|---|
| URL | `/api/v1/orders` | Чаще всего, просто |
| Заголовок | `Accept: application/vnd.myco.v2+json` | Чистые URL |
| Query | `?version=2` | Редко, не REST |

### Breaking vs non-breaking

| Изменение | Breaking? | Пример |
|---|---|---|
| Добавить необязательное поле | Нет | `+ priceVat: integer` |
| Новый эндпоинт | Нет | `+ GET /orders/{id}/history` |
| Добавить enum-значение | Нет (с предупреждением) | `+ PARTIALLY_REFUNDED` |
| **Переименовать** поле | **ДА** | `price` → `amount` |
| **Удалить** поле | **ДА** | Убрать `discount` |
| **Изменить тип** | **ДА** | `total: integer` → `string` |
| **Убрать** enum-значение | **ДА** | Убрать `REFUNDED` |
| Изменить семантику статуса | **ДА** | `409` → теперь `400` |

### Деградация старой версии

1. Header `Deprecation: true` в ответах v1
2. Header `Sunset: Sat, 01 Mar 2027 00:00:00 GMT`
3. Логирование: кто ещё использует v1
4. Уведомление за 3 месяца
5. В день Sunset: v1 → `410 Gone`

---

## 6. OpenAPI

### Минимальный пример (YAML)

```yaml
openapi: 3.0.3
info:
  title: Orders API
  version: 1.0.0
security:
  - bearerAuth: []
paths:
  /orders:
    post:
      summary: Создать заказ
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateOrderRequest'
      responses:
        '201':
          description: Создан
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'
        '400':
          description: Валидация
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
        '409':
          description: Нет в наличии
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
  schemas:
    CreateOrderRequest:
      type: object
      required: [userId, items, total]
      properties:
        userId:
          type: integer
          minimum: 1
        items:
          type: array
          minItems: 1
          maxItems: 100
          items:
            $ref: '#/components/schemas/OrderItem'
        total:
          type: integer
          minimum: 1
          description: В копейках
    OrderItem:
      type: object
      required: [sku, qty]
      properties:
        sku:
          type: string
          minLength: 3
          maxLength: 32
        qty:
          type: integer
          minimum: 1
          maximum: 999
    Order:
      type: object
      required: [orderId, status, total, createdAt]
      properties:
        orderId:
          type: integer
        status:
          type: string
          enum: [CREATED, PAID, SHIPPED, DELIVERED, CANCELLED]
        total:
          type: integer
        createdAt:
          type: string
          format: date-time
    Error:
      type: object
      required: [errorCode, message]
      properties:
        errorCode:
          type: string
        message:
          type: string
```

### Что генерируется из OpenAPI

| Из одного YAML-файла | Инструмент |
|---|---|
| Документация (интерактивная) | Swagger UI |
| Mock-сервер (для фронтенда) | Prerequest, Mocklab |
| Клиентский код (Java/Python/JS) | OpenAPI Generator |
| Контракт-тесты | Pact |
| Импорт в Postman | Postman (Import) |

### Безопасность в OpenAPI

```yaml
securitySchemes:
  bearerAuth:
    type: http
    scheme: bearer
    bearerFormat: JWT
  apiKeyAuth:
    type: apiKey
    in: header
    name: X-Api-Key
```

| | API Key | Bearer (JWT) |
|---|---|---|
| Срок жизни | Нет (пока не отзовёшь) | 1 час + refresh |
| Отзыв | Ручной | Автоматически (истёк) |
| Права | Один ключ = все права | Роли в токене |
| Когда | Простые B2B | Внутри, клиенты |

**Секреты (сами ключи/токены) — никогда в OpenAPI и в ТЗ.** Только способ получения: Vault, env.

---

## 7. Практика

### Кейс А: `POST /api/v1/rentals`

Описать:
1. Метод, путь
2. Body: 4–5 полей (тип, обязательность, ограничения)
3. 3+ ответа (201 + 2 ошибки)
4. Версия
5. Auth

### Кейс Б: `GET /api/v1/products` + фильтр + пагинация

Дан: `GET /api/v1/products` → список.
Добавить: фильтр по категории, пагинация.

Ответ:
```
GET /api/v1/products?category=electronics&page=1&size=20
```

```json
{
  "items": [
    {"id": 1, "name": "Ноутбук", "price": 500000, "category": "electronics"},
    {"id": 2, "name": "Мышь", "price": 1500, "category": "electronics"}
  ],
  "page": 1,
  "size": 20,
  "total": 150,
  "totalPages": 8
}
```

- Breaking? **Нет** (добавили query params, старый клиент без них получит page 1)
- v2? **Не нужен**

---

## 8. Чек-лист «контракт готов»

- [ ] Метод, путь, описание для каждого эндпоинта
- [ ] Body: схема (типы, обязательность, ограничения)
- [ ] Деньги — integer, даты — ISO 8601
- [ ] Enum — перечислены значения
- [ ] Массивы — minItems/maxItems
- [ ] Статусы: 2xx + 4xx + 500
- [ ] Ошибка: единая структура + словарь кодов
- [ ] Версия
- [ ] Auth тип (не ключи)
- [ ] Rate limit (если есть)
- [ ] OpenAPI-файл

---

## 9. Частые вопросы собеса

1. Как описываете API? → OpenAPI 3.x, структура, генерация доков/моков
2. Что такое breaking change? → Переименовать/удалить/изменить тип
3. Как версионируете? → URL `/api/v1/`, параллельно 3–6 мес., Deprecation/Sunset
4. Как описываете ошибки? → `errorCode` + `message`, словарь на эндпоинт
5. Почему деньги не float? → `0.1+0.2≠0.3`, integer в копейках
6. 401 vs 403? → Нет токена vs нет прав
7. 409 vs 400? → Состояние не позволяет vs битый запрос
8. Что сгенерировать из OpenAPI? → Доки, моки, клиенты, тесты
9. API key vs Bearer? → Ключ на всё vs токен с ролью и сроком

---

## 10. Домашнее задание

1. **OpenAPI YAML** для 3 эндпоинтов:
   - `POST /api/v1/rentals` (создать бронь)
   - `GET /api/v1/rentals` (список, с пагинацией)
   - `GET /api/v1/rentals/{id}` (один)
   
   Схемы: `RentalRequest`, `Rental`, `Error`. Auth: Bearer. Ошибки: 400, 401, 404, 409.

2. **Кейс версионирования:**
   API `GET /api/v1/users` → `{ "name": "Иван Иванов" }`.
   Продукт: разбить на `firstName` + `lastName`.
   Опиши: v2-схема, как деградировать v1, что сказать клиентам.

3. **Вопросы без ответа** — 3+.

**Сдать:** YAML + текст в чат до занятия 3.
