# Эталонное решение. Интернет-магазин: модель данных, индексы, REST API

---

## 1. Модель данных (5 таблиц)

### `products` (дана, не менять)

```sql
CREATE TABLE products (
    id          INTEGER PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    category    VARCHAR(50)  NOT NULL,
    price_cents INTEGER      NOT NULL,
    stock       INTEGER      NOT NULL DEFAULT 0,
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);
```

### `users`

```sql
CREATE TABLE users (
    id           INTEGER PRIMARY KEY,
    email        VARCHAR(255) NOT NULL UNIQUE,
    name         VARCHAR(100) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at   TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);
```

### `orders`

```sql
CREATE TABLE orders (
    id           INTEGER PRIMARY KEY,
    user_id      INTEGER NOT NULL REFERENCES users(id),
    status       VARCHAR(20) NOT NULL DEFAULT 'PENDING'
                 CHECK (status IN ('PENDING', 'CONFIRMED', 'CANCELLED')),
    total_cents  INTEGER NOT NULL CHECK (total_cents > 0),
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### `order_items` (таблица-связка M:N)

```sql
CREATE TABLE order_items (
    id          INTEGER PRIMARY KEY,
    order_id    INTEGER NOT NULL REFERENCES orders(id),
    product_id  INTEGER NOT NULL REFERENCES products(id),
    quantity    INTEGER NOT NULL CHECK (quantity > 0),
    price_cents INTEGER NOT NULL CHECK (price_cents > 0)
);
```

> `price_cents` дублируется из `products` **осознанно**: цена фиксируется на момент покупки. Если товар подорожает, история заказов не меняется.

### `idempotency_keys` (защита от двойного клика)

```sql
CREATE TABLE idempotency_keys (
    key        VARCHAR(100) PRIMARY KEY,
    order_id   INTEGER NOT NULL REFERENCES orders(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## 2. Связи

```
users  1 ──── N orders          (FK user_id в orders)
orders 1 ──── N order_items     (FK order_id в order_items)
products 1 ── N order_items     (FK product_id в order_items)
users  1 ──── N idempotency_keys (FK order_id, key UNIQUE)
```

---

## 3. Индексы

| # | Запрос (из сценария) | Индекс | Почему |
|---|---|---|---|
| 1 | `SELECT * FROM orders WHERE user_id = 42 ORDER BY created_at DESC` (Сценарий 2) | `(user_id, created_at DESC)` | Равенство (user_id) → сортировка (created_at). Не нужен отдельный SORT |
| 2 | `SELECT oi.*, p.name FROM order_items oi JOIN products p ON oi.product_id = p.id WHERE oi.order_id = 77` (Сценарий 2) | `(order_id)` в order_items | Равенство. Все позиции одного заказа |
| 3 | `SELECT * FROM products WHERE id IN (1, 5, 12) AND stock >= 2` (Сценарий 1, проверка наличия) | PK `id` (уже есть) | Поиск по PK — уже быстрый. Индекс на `stock` не нужен (selectivity низкая) |
| 4 | `SELECT * FROM products WHERE category = 'phones' ORDER BY price_cents` (каталог) | `(category, price_cents)` | Равенство (category) → сортировка/диапазон (price_cents). Покрывающий, если SELECT берёт только эти колонки |
| 5 | `SELECT * FROM idempotency_keys WHERE key = 'req-abc-123'` (Сценарий 1, проверка дубля) | PK `key` (уже есть) | Поиск по PK |

**Почему не индексирую `status` в orders:** 3 значения (PENDING, CONFIRMED, CANCELLED), selectivity очень низкая. Если понадобится «все PENDING за сегодня» — `(status, created_at)`, но на этом объёме (200/день) не критично.

---

## 4. REST API

### Сценарий 1: Оформить заказ

```
POST /api/v1/orders
Headers:
  Authorization: Bearer {token}
  Idempotency-Key: req-abc-123
Body:
{
  "items": [
    {"productId": 1, "quantity": 1},
    {"productId": 5, "quantity": 2}
  ]
}
```

**Ответы:**

| Статус | Когда | Тело |
|---|---|---|
| 201 | Заказ создан | `{"orderId": 77, "status": "PENDING", "totalCents": 45000}` |
| 400 | Битый запрос (нет productId, qty <= 0) | `{"errorCode": "VALIDATION_ERROR", "message": "..."}` |
| 401 | Нет токена | `{"errorCode": "AUTH_REQUIRED", "message": "..."}` |
| 409 | Не хватает товара на складе | `{"errorCode": "INSUFFICIENT_STOCK", "message": "Товар 5: доступно 1, запрошено 2"}` |
| 409 | Дубль (Idempotency-Key уже был) | Возвращает **первоначальный** ответ (201 с тем же orderId) |

### Сценарий 2: Мои заказы

```
GET /api/v1/orders?page=1&size=20
Headers:
  Authorization: Bearer {token}
```

**Ответ 200:**
```json
{
  "items": [
    {
      "orderId": 77,
      "status": "PENDING",
      "totalCents": 45000,
      "createdAt": "2026-09-20T19:00:00Z"
    },
    {
      "orderId": 76,
      "status": "CONFIRMED",
      "totalCents": 12000,
      "createdAt": "2026-09-19T14:30:00Z"
    }
  ],
  "page": 1,
  "size": 20,
  "total": 5
}
```

### Сценарий 2: Детали заказа

```
GET /api/v1/orders/77
Headers:
  Authorization: Bearer {token}
```

**Ответ 200:**
```json
{
  "orderId": 77,
  "status": "PENDING",
  "totalCents": 45000,
  "createdAt": "2026-09-20T19:00:00Z",
  "items": [
    {"productId": 1, "name": "iPhone 15", "quantity": 1, "priceCents": 25000},
    {"productId": 5, "name": "Чехло", "quantity": 2, "priceCents": 10000}
  ]
}
```

**Ошибки:**

| Статус | Когда |
|---|---|
| 404 | Заказа нет или он чужой (user_id не совпадает) |
| 401 | Нет токена |

---

## 5. SQL-запросы

### Сценарий 1: Проверка наличия

```sql
SELECT id, stock
FROM products
WHERE id IN (1, 5);
```
Приложение проверяет: `stock >= quantity` для каждого товара. Если нет → 409.

### Сценарий 1: Создание заказа (транзакция)

```sql
BEGIN;

-- 1. Проверить idempotency
SELECT order_id FROM idempotency_keys WHERE key = 'req-abc-123';
-- Если есть → ROLLBACK, вернуть 201 с существующим orderId

-- 2. Заблокировать строки товаров (защита от race condition)
SELECT id, stock FROM products WHERE id IN (1, 5) FOR UPDATE;
-- Проверить stock >= quantity. Если нет → ROLLBACK, 409

-- 3. Списать со склада
UPDATE products SET stock = stock - 1 WHERE id = 1;
UPDATE products SET stock = stock - 2 WHERE id = 5;

-- 4. Создать заказ
INSERT INTO orders (user_id, status, total_cents)
VALUES (42, 'PENDING', 45000)
RETURNING id;
-- → 77

-- 5. Позиции заказа
INSERT INTO order_items (order_id, product_id, quantity, price_cents)
VALUES (77, 1, 1, 25000),
       (77, 5, 2, 50000);

-- 6. Записать idempotency key
INSERT INTO idempotency_keys (key, order_id)
VALUES ('req-abc-123', 77);

COMMIT;
```

### Сценарий 2: Список заказов пользователя

```sql
SELECT id, status, total_cents, created_at
FROM orders
WHERE user_id = 42
ORDER BY created_at DESC
LIMIT 20 OFFSET 0;
```

### Сценарий 2: Детали заказа

```sql
SELECT oi.product_id, p.name, oi.quantity, oi.price_cents
FROM order_items oi
JOIN products p ON oi.product_id = p.id
WHERE oi.order_id = 77;
```

---

## 6. Ответы на вопросы

**Сценарий 1:**

- *Какие таблицы читает система при проверке наличия?* → `products` (SELECT по id, stock).
- *Что записывает в БД после успешного заказа?* → `orders` (INSERT), `order_items` (INSERT × N), `products` (UPDATE stock), `idempotency_keys` (INSERT). Всё в одной транзакции.
- *Что будет, если на складе 1 шт, а в заказе 2?* → 409 INSUFFICIENT_STOCK. Транзакция откатывается, stock не меняется.
- *Как защитить от «двойного клика»?* → Header `Idempotency-Key`. Перед созданием — SELECT из `idempotency_keys`. Если key уже есть → вернуть первоначальный ответ, не создавать новый заказ.

**Сценарий 2:**

- *Какой запрос вернёт список заказов?* → `SELECT ... FROM orders WHERE user_id = ? ORDER BY created_at DESC LIMIT/OFFSET`.
- *Какой запрос вернёт детали?* → `SELECT ... FROM order_items JOIN products WHERE order_id = ?`.
- *Нужен ли отдельный индекс?* → `(user_id, created_at DESC)` — да, для списка. Для деталей — `(order_id)` в order_items — да.

---

## 7. Что проверить на собесе

- [ ] 5 таблиц, M:N через order_items
- [ ] `price_cents` в order_items — денормализация осознанная (цена на момент покупки)
- [ ] `idempotency_keys` — защита от дубля
- [ ] `FOR UPDATE` — защита от race condition (два заказа одновременно на последний товар)
- [ ] Индекс `(user_id, created_at DESC)` — равенство + сортировка
- [ ] Индекс `(category, price_cents)` — каталог
- [ ] REST: Idempotency-Key в header, 409 при нехватке, 404 при чужом заказе
- [ ] Транзакция: весь Сценарий 1 в BEGIN/COMMIT
