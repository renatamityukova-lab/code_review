# Ревью: `code_review/ex3.txt` (Go, `handlers.Orders`)

Фрагмент: `status := r.URL.Query().Get("status")` → `fmt.Sprintf("SELECT ... WHERE status = '%s'", status)` → `db.Query(query)`.
Фокус: SQL-инъекция через конкатенацию строки. Итог: **1 blocker, 2 major, 2 minor**.

| # | Проблема | Приоритет | До merge | Отдельная задача |
|---|----------|-----------|----------|------------------|
| 1 | SQL-инъекция через `fmt.Sprintf` | blocker | да | нет |
| 2 | `*sql.Rows` не закрывается (утечка соединений) | major | да | нет |
| 3 | Нет валидации/allowlist для `status`, пустое значение = тихий пустой ответ | major | да | нет |
| 4 | Ошибка БД возвращается «как есть», без контекста и маппинга в HTTP | minor | желательно | да (единый контракт ошибок) |
| 5 | Запрос без `Context` (нет таймаута/отмены) | minor | нет | да |

---

## 1. SQL-инъекция через конкатенацию — **blocker**

- **Что не так:** `status` из query-параметра подставляется в SQL через `fmt.Sprintf` в [`ex3.txt`](code_review/ex3.txt:10) — это конкатенация, а не параметризация.
- **Риски:** `?status=x' OR '1'='1` → выгрузка всех заказов; `'; DROP TABLE ...; --`, `UNION SELECT` → чтение/изменение чужих данных, обход фильтра по статусу, потенциальный полный компромисс БД.
- **Как исправить:** параметризованный запрос с плейсхолдером и передачей значения отдельно:
  ```go
  const q = `SELECT id, total FROM orders WHERE status = $1` // для PostgreSQL; для MySQL/SQLite — "?"
  rows, err := db.QueryContext(r.Context(), q, status)
  ```

**Next steps:** до merge — обязательно; отдельная задача — нет; тесты — на `sqlmock`: при `status = "x' OR '1'='1"` запрос содержит плейсхолдер, значение передано как аргумент, а не в тексте SQL; «инъекционный» payload не меняет результат выборки.

---

## 2. `*sql.Rows` не закрывается — **major**

- **Что не так:** результат `db.Query` присваивается в `_` в [`ex3.txt`](code_review/ex3.txt:11) и никогда не закрывается (`rows.Close()`), данные не читаются и не сканируются.
- **Риски:** утечка соединений из пула → исчерпание `MaxOpenConns`, зависание всех запросов приложения под нагрузкой (отказ в обслуживании), плюс обработчик фактически не отдаёт данные.
- **Как исправить:** либо `QueryRowContext(...).Scan(&id, &total)` для одной строки, либо `rows, err := ...; defer rows.Close()` и обход `rows.Next()`/`rows.Err()`; ответ клиенту формировать явно (`json.NewEncoder(w).Encode(...)`).

**Next steps:** до merge — обязательно; отдельная задача — нет; тесты — на `sqlmock` проверять, что `rows.Close()` вызван; при ошибке `rows.Err()` возвращается ошибка; happy path отдаёт ожидаемый JSON.

---

## 3. Нет валидации `status` — **major**

- **Что не так:** значение принимается без проверки; отсутствие параметра даёт пустую строку и запрос `WHERE status = ''` в [`ex3.txt`](code_review/ex3.txt:9), ошибка не сигнализируется.
- **Риски:** тихий пустой ответ вместо явной ошибки клиента; невозможно отличить «нет заказов» от «параметр не передан»; отсутствие allowlist увеличивает поверхность атаки и усложняет индексацию/кеширование.
- **Как исправить:** валидировать до запроса: `status == ""` → `http.StatusBadRequest`; допустимые значения — по enum/allowlist (`new`, `paid`, `shipped`, `cancelled`), иначе `400`; для параметризованного запроса значение всё равно передавать отдельно.

**Next steps:** до merge — обязательно (пустое значение и allowlist); отдельная задача — да, если список статусов должен быть расширяемым (вынести в конфиг/домен); тесты — отсутствующий `status` → `400`; неизвестный статус → `400`; валидный → `200`.

---

## 4. Ошибка БД без контекста и без маппинга в HTTP — **minor**

- **Что не так:** `return err` в [`ex3.txt`](code_review/ex3.txt:12) отдаёт «сырую» ошибку, обработчик не пишет HTTP-статус и не оборачивает ошибку контекстом.
- **Риски:** в логах нет операции/параметров для диагностики; при пробросе наружу — утечка деталей БД; клиент получает неопределённый ответ.
- **Как исправить:** обернуть `fmt.Errorf("orders: query by status: %w", err)`, логировать, наружу отдавать `500` с обезличенным сообщением; не смешивать транспорт (`http.ResponseWriter`) и доступ к БД в одной функции — выделить слой репозитория.

**Next steps:** до merge — желательно; отдельная задача — да (единый контракт ошибок и маппинг в HTTP-коды); тесты — ошибка `db.QueryContext` → `500` и залогированная обёрнутая ошибка, детали БД не попадают в тело ответа.

---

## 5. Запрос без `Context` — **minor**

- **Что не так:** используется `db.Query`, а не `db.QueryContext` с `r.Context()`.
- **Риски:** при отмене клиентского запроса или дедлайне работа с БД продолжается, занимая соединение; нет таймаутов на уровне запроса.
- **Как исправить:** `db.QueryContext(r.Context(), q, status)` и/или `context.WithTimeout` на границе обработчика.

**Next steps:** до merge — нет; отдельная задача — да (единая политика таймаутов/контекста для хендлеров); тесты — отмена контекста → запрос прерывается, ошибка маппится в `499/504` по принятому в проекте контракту.

---

## Эталонный фикс

```go
package handlers

import (
	"context"
	"database/sql"
	"encoding/json"
	"fmt"
	"net/http"
)

var allowedStatuses = map[string]struct{}{
	"new": {}, "paid": {}, "shipped": {}, "cancelled": {},
}

type Order struct {
	ID    int64   `json:"id"`
	Total float64 `json:"total"`
}

func Orders(ctx context.Context, db *sql.DB, w http.ResponseWriter, r *http.Request) error {
	status := r.URL.Query().Get("status")
	if _, ok := allowedStatuses[status]; !ok {
		http.Error(w, "invalid status", http.StatusBadRequest)
		return nil
	}

	const q = `SELECT id, total FROM orders WHERE status = $1` // "?" для MySQL/SQLite

	rows, err := db.QueryContext(ctx, q, status)
	if err != nil {
		return fmt.Errorf("orders: query by status: %w", err)
	}
	defer rows.Close()

	orders := make([]Order, 0)
	for rows.Next() {
		var o Order
		if err := rows.Scan(&o.ID, &o.Total); err != nil {
			return fmt.Errorf("orders: scan row: %w", err)
		}
		orders = append(orders, o)
	}
	if err := rows.Err(); err != nil {
		return fmt.Errorf("orders: iterate rows: %w", err)
	}

	w.Header().Set("Content-Type", "application/json")
	return json.NewEncoder(w).Encode(orders)
}
```

**Тесты (минимум):** инъекционный payload (`x' OR '1'='1`) → плейсхолдер + аргумент, результат не расширяется; отсутствующий/неизвестный `status` → `400`; валидный статус → `200` + корректный JSON; ошибка `QueryContext` → обёрнутая ошибка и `500` без деталей БД; `rows.Close()` вызывается; отмена контекста → запрос прерывается.
