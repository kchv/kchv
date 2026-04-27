# SQL-запросы для QA-проверок

## SQL-001: Заказы с некорректной итоговой суммой

**Цель:** найти заказы, у которых сумма заказа не равна сумме позиций.

```sql
SELECT
  o.id AS order_id,
  o.total_amount AS order_total,
  SUM(oi.quantity * oi.price) AS items_total,
  o.total_amount - SUM(oi.quantity * oi.price) AS diff
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
GROUP BY o.id, o.total_amount
HAVING o.total_amount <> SUM(oi.quantity * oi.price);
```

**Ожидаемый результат:** запрос не возвращает строк. Если строки есть, это кандидат в баг по расчету суммы заказа.

## SQL-002: Оплаченные заказы без успешного платежа

**Цель:** проверить, что у каждого оплаченного заказа есть успешный платеж.

```sql
SELECT
  o.id AS order_id,
  o.status AS order_status,
  o.total_amount,
  COUNT(p.id) AS success_payments_count
FROM orders o
LEFT JOIN payments p
  ON p.order_id = o.id
 AND p.status = 'success'
WHERE o.status = 'paid'
GROUP BY o.id, o.status, o.total_amount
HAVING COUNT(p.id) = 0;
```

**Ожидаемый результат:** нет заказов в статусе `paid` без успешного платежа.

## SQL-003: Сумма платежа не совпадает с суммой заказа

**Цель:** найти расхождения между суммой заказа и суммой успешного платежа.

```sql
SELECT
  o.id AS order_id,
  o.total_amount AS order_amount,
  SUM(p.amount) AS paid_amount
FROM orders o
JOIN payments p
  ON p.order_id = o.id
 AND p.status = 'success'
WHERE o.status IN ('paid', 'completed')
GROUP BY o.id, o.total_amount
HAVING o.total_amount <> SUM(p.amount);
```

**Ожидаемый результат:** нет расхождений. Если строки есть, нужно проверить расчет скидок, доставку и повторные списания.

## SQL-004: Дубли активных пользователей по email

**Цель:** найти дубли активных пользователей по email.

```sql
SELECT
  LOWER(email) AS normalized_email,
  COUNT(*) AS users_count,
  MIN(created_at) AS first_created_at,
  MAX(created_at) AS last_created_at
FROM users
WHERE deleted_at IS NULL
GROUP BY LOWER(email)
HAVING COUNT(*) > 1;
```

**Ожидаемый результат:** нет дублей активных пользователей.

## SQL-005: Позиции заказа без родительского заказа

**Цель:** найти позиции заказа без родительского заказа.

```sql
SELECT
  oi.id AS order_item_id,
  oi.order_id,
  oi.product_id,
  oi.quantity
FROM order_items oi
LEFT JOIN orders o ON o.id = oi.order_id
WHERE o.id IS NULL;
```

**Ожидаемый результат:** запрос не возвращает строк.

## SQL-006: Успешный платеж после отмены заказа

**Цель:** проверить, что после отмены заказа не было успешного платежа.

```sql
SELECT
  o.id AS order_id,
  o.cancelled_at,
  p.id AS payment_id,
  p.created_at AS payment_created_at,
  p.amount
FROM orders o
JOIN payments p
  ON p.order_id = o.id
 AND p.status = 'success'
WHERE o.status = 'cancelled'
  AND p.created_at > o.cancelled_at;
```

**Ожидаемый результат:** нет успешных платежей после отмены заказа.

## SQL-007: Недопустимые переходы статусов заказа

**Цель:** найти недопустимые переходы статусов заказа.

```sql
SELECT
  order_id,
  old_status,
  new_status,
  created_at
FROM status_history
WHERE NOT (
  (old_status = 'created' AND new_status IN ('paid', 'cancelled')) OR
  (old_status = 'paid' AND new_status IN ('processing', 'refunded')) OR
  (old_status = 'processing' AND new_status IN ('shipped', 'cancelled')) OR
  (old_status = 'shipped' AND new_status = 'completed')
);
```

**Ожидаемый результат:** нет недопустимых переходов.

## SQL-008: Дневная конверсия из созданного заказа в оплаченный

**Цель:** подготовить данные для анализа просадки оплаты после релиза.

```sql
SELECT
  CAST(created_at AS DATE) AS order_date,
  COUNT(*) AS created_orders,
  SUM(CASE WHEN status IN ('paid', 'processing', 'shipped', 'completed') THEN 1 ELSE 0 END) AS paid_orders,
  ROUND(
    100.0 * SUM(CASE WHEN status IN ('paid', 'processing', 'shipped', 'completed') THEN 1 ELSE 0 END) / COUNT(*),
    2
  ) AS paid_conversion_percent
FROM orders
WHERE created_at >= CURRENT_DATE - INTERVAL '14 days'
GROUP BY CAST(created_at AS DATE)
ORDER BY order_date DESC;
```

**Ожидаемый результат:** конверсия не проседает резко после релиза. Если просадка есть, это повод проверить оплату и ошибки API.

## SQL-009: Заказы, созданные после удаления пользователя

**Цель:** найти заказы, созданные после удаления пользователя.

```sql
SELECT
  o.id AS order_id,
  o.user_id,
  o.created_at AS order_created_at,
  u.deleted_at AS user_deleted_at
FROM orders o
JOIN users u ON u.id = o.user_id
WHERE u.deleted_at IS NOT NULL
  AND o.created_at > u.deleted_at;
```

**Ожидаемый результат:** нет заказов, созданных после удаления пользователя.

## SQL-010: Товары, которые чаще встречаются в неоплаченных или отмененных заказах

**Цель:** найти товары, чаще всего встречающиеся в неоплаченных или отмененных заказах.

```sql
SELECT
  oi.product_id,
  COUNT(DISTINCT o.id) AS affected_orders,
  SUM(oi.quantity) AS affected_quantity
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
WHERE o.status IN ('payment_failed', 'cancelled')
GROUP BY oi.product_id
ORDER BY affected_orders DESC, affected_quantity DESC
LIMIT 20;
```

**Ожидаемый результат:** используется для расследования проблем с конкретными товарами, остатками или ценами.
