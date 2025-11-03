# Инструкция по добавлению Пагинации и Фильтрации в Nano CRM API

## Обзор изменений

Требуется добавить в endpoint `GET /api/v1/customers` три механизма:
1. **Пагинация** — разбиение результатов на страницы
2. **Фильтрация** — поиск и фильтрация по различным критериям
3. **Сортировка** — упорядочивание результатов

---

## 1. Пагинация

### Принцип работы

Клиент указывает номер страницы и количество записей на странице через query параметры. Сервер возвращает только запрошенную часть данных.

**Важно**: Пагинация обязательна для всех list endpoints, чтобы:
- Избежать перегрузки сервера и клиента
- Улучшить производительность
- Снизить нагрузку на сеть
- Обеспечить масштабируемость

### Параметры пагинации

| Параметр | Тип | Описание | Ограничения | По умолчанию |
|----------|-----|----------|-------------|--------------|
| `page` | integer | Номер страницы (1-indexed) | minimum: 1 | 1 |
| `limit` | integer | Количество записей на странице | minimum: 1, maximum: 100 | 20 |

**Важно**:
- `page` начинается с 1, а не с 0 (1-indexed)
- `limit` ограничен максимумом 100 для безопасности
- Если запрошенная страница пуста → вернуть пустой массив `[]` (не ошибка)

### Примеры использования

```http
# Первая страница, 20 записей (по умолчанию)
GET /api/v1/customers

# Вторая страница, 20 записей
GET /api/v1/customers?page=2

# Первая страница, 50 записей
GET /api/v1/customers?limit=50

# Третья страница, 10 записей
GET /api/v1/customers?page=3&limit=10

# Максимум записей (100)
GET /api/v1/customers?limit=100
```

### Формат ответа

Список возвращается как массив напрямую (без обертки с метаданными):

```json
[
  {
    "id": "123e4567-e89b-12d3-a456-426614174000",
    "name": "John Doe",
    "email": "john.doe@example.com",
    "phone": "+1-555-0101",
    "countryId": "00000000-0000-0000-0000-000000000001",
    "createdAt": "2024-01-01T12:00:00Z",
    "updatedAt": "2024-01-01T12:00:00Z"
  },
  {
    "id": "223e4567-e89b-12d3-a456-426614174002",
    "name": "Jane Smith",
    "email": "jane.smith@example.com",
    "phone": "+44-20-1234-5678",
    "countryId": "00000000-0000-0000-0000-000000000002",
    "createdAt": "2024-01-02T12:00:00Z",
    "updatedAt": "2024-01-02T12:00:00Z"
  }
]
```

**Важно**: Если запрошена несуществующая страница (например, `page=9999`) → вернуть пустой массив `[]`, а не ошибку 404.

---

## 2. Фильтрация

### Принцип работы

Клиент указывает критерии фильтрации через query параметры. Сервер возвращает только записи, соответствующие критериям.

### Параметры фильтрации

#### 2.1. Фильтр по email (уже есть, без изменений)

| Параметр | Тип | Описание | Пример |
|----------|-----|----------|--------|
| `email` | string | Точное совпадение email | `john.doe@example.com` |

**Текущая реализация**: Уже есть в API, оставляем как есть.

```http
GET /api/v1/customers?email=john.doe@example.com
```

#### 2.2. Фильтр по countryId (НОВЫЙ параметр)

| Параметр | Тип | Описание | Пример |
|----------|-----|----------|--------|
| `countryId` | string (UUID) | Фильтр по стране | `00000000-0000-0000-0000-000000000001` |

**Использование**: Показать всех клиентов из определенной страны.

```http
GET /api/v1/customers?countryId=00000000-0000-0000-0000-000000000001
```

#### 2.3. Поиск по нескольким полям (НОВЫЙ параметр `search`)

| Параметр | Тип | Описание | Ограничения | Пример |
|----------|-----|----------|-------------|--------|
| `search` | string | Поиск в полях `name`, `email`, `phone` (частичное совпадение) | minLength: 2 | `john` |

**Как работает**:
- Ищет в полях `name`, `email`, `phone`
- Частичное совпадение (LIKE в SQL)
- Регистронезависимый поиск (обычно)
- Минимум 2 символа (чтобы избежать перегрузки)

**Примеры**:
```http
# Поиск по имени (найдет "John Doe", "Johnny Smith")
GET /api/v1/customers?search=john

# Поиск по email (найдет john.doe@example.com, johnny@test.com)
GET /api/v1/customers?search=john

# Поиск по телефону (найдет +1-555-0101, +44-555-0101)
GET /api/v1/customers?search=555
```

**SQL пример** (для понимания):
```sql
SELECT * FROM customers
WHERE 
  name ILIKE '%john%' 
  OR email ILIKE '%john%' 
  OR phone ILIKE '%john%'
```

### Комбинирование фильтров

Фильтры можно комбинировать через `&`:

```http
# Поиск "john" среди клиентов из США
GET /api/v1/customers?search=john&countryId=00000000-0000-0000-0000-000000000001

# Email + страна
GET /api/v1/customers?email=john.doe@example.com&countryId=00000000-0000-0000-0000-000000000001

# Поиск + страница
GET /api/v1/customers?search=john&page=2&limit=20
```

**Важно**: При комбинировании применяется логика AND (все условия должны выполняться).

---

## 3. Сортировка

### Принцип работы

Клиент указывает поле и направление сортировки через параметр `sort`. Сервер возвращает результаты в указанном порядке.

### Параметр сортировки

| Параметр | Тип | Описание | Значения | По умолчанию |
|----------|-----|----------|----------|--------------|
| `sort` | string | Поле и направление сортировки | enum (см. ниже) | `created_desc` |

### Доступные значения sort

| Значение | Описание | SQL эквивалент |
|----------|----------|----------------|
| `name_asc` | По имени (A-Z) | `ORDER BY name ASC` |
| `name_desc` | По имени (Z-A) | `ORDER BY name DESC` |
| `email_asc` | По email (A-Z) | `ORDER BY email ASC` |
| `email_desc` | По email (Z-A) | `ORDER BY email DESC` |
| `created_asc` | По дате создания (старые → новые) | `ORDER BY created_at ASC` |
| `created_desc` | По дате создания (новые → старые) | `ORDER BY created_at DESC` |

**Формат**: `{field}_{direction}`
- `field`: `name`, `email`, `created`
- `direction`: `asc`, `desc`

**По умолчанию**: `created_desc` (новые записи первыми — наиболее актуальные).

### Примеры использования

```http
# Сортировка по имени (A-Z)
GET /api/v1/customers?sort=name_asc

# Сортировка по email (Z-A)
GET /api/v1/customers?sort=email_desc

# Сортировка по дате создания (старые первыми)
GET /api/v1/customers?sort=created_asc

# Комбинация: поиск + сортировка + пагинация
GET /api/v1/customers?search=john&sort=name_asc&page=1&limit=20
```

### Обработка неверного значения sort

Если клиент указал неверное значение (не из enum) → вернуть **400 Bad Request**:

```json
{
  "error": {
    "code": "INVALID_SORT_PARAMETER",
    "message": "Invalid sort parameter. Allowed values: name_asc, name_desc, email_asc, email_desc, created_asc, created_desc"
  }
}
```

---

## 4. Технические детали реализации в OpenAPI

### Добавление параметров в `GET /api/v1/customers`

```yaml
parameters:
  # Пагинация
  - name: page
    in: query
    description: Page number (1-indexed)
    required: false
    schema:
      type: integer
      minimum: 1
      default: 1
    example: 1
  
  - name: limit
    in: query
    description: Number of items per page (1-100)
    required: false
    schema:
      type: integer
      minimum: 1
      maximum: 100
      default: 20
    example: 20
  
  # Фильтрация (существующий)
  - name: email
    in: query
    description: Filter by exact email match
    required: false
    schema:
      type: string
      format: email
    example: "john.doe@example.com"
  
  # Фильтрация (новый)
  - name: countryId
    in: query
    description: Filter by country ID
    required: false
    schema:
      type: string
      format: uuid
    example: "00000000-0000-0000-0000-000000000001"
  
  - name: search
    in: query
    description: Search in name, email, phone fields (partial match, case-insensitive)
    required: false
    schema:
      type: string
      minLength: 2
    example: "john"
  
  # Сортировка (новый)
  - name: sort
    in: query
    description: Sort field and direction
    required: false
    schema:
      type: string
      enum:
        - name_asc
        - name_desc
        - email_asc
        - email_desc
        - created_asc
        - created_desc
      default: created_desc
    example: "created_desc"
```

---

## 5. Примеры комбинаций параметров

### Пример 1: Простой список (по умолчанию)

```http
GET /api/v1/customers
```

**Результат**: Первая страница, 20 записей, сортировка по `created_desc`.

---

### Пример 2: Поиск + пагинация

```http
GET /api/v1/customers?search=john&page=1&limit=10
```

**Результат**: Поиск по "john", первая страница, 10 записей, сортировка по `created_desc`.

---

### Пример 3: Фильтр по стране + сортировка

```http
GET /api/v1/customers?countryId=00000000-0000-0000-0000-000000000001&sort=name_asc
```

**Результат**: Клиенты из определенной страны, отсортированы по имени (A-Z), первая страница, 20 записей.

---

### Пример 4: Полная комбинация

```http
GET /api/v1/customers?search=john&countryId=00000000-0000-0000-0000-000000000001&sort=name_asc&page=2&limit=50
```

**Результат**: 
- Поиск по "john"
- Фильтр по стране
- Сортировка по имени (A-Z)
- Вторая страница
- 50 записей на странице

---

## 6. Важные моменты для реализации

### Пагинация

1. **Вычисление offset**:
   ```
   offset = (page - 1) * limit
   ```
   Пример: `page=3, limit=20` → `offset = 40`

2. **Проверка границ**:
   - Если `page < 1` → установить `page = 1` или вернуть 400
   - Если `limit < 1` → установить `limit = 20` или вернуть 400
   - Если `limit > 100` → установить `limit = 100` или вернуть 400

3. **Пустые страницы**:
   - Если запрошенная страница пуста → вернуть `[]` (пустой массив), не ошибку

### Фильтрация

1. **Поиск (search)**:
   - Использовать `ILIKE` или `LIKE` в SQL (частичное совпадение)
   - Искать в трех полях: `name`, `email`, `phone`
   - Минимум 2 символа (валидация)

2. **Комбинирование фильтров**:
   - Применять логику AND (все условия выполняются)
   - Пример SQL:
   ```sql
   SELECT * FROM customers
   WHERE 
     (name ILIKE '%john%' OR email ILIKE '%john%' OR phone ILIKE '%john%')
     AND country_id = '00000000-0000-0000-0000-000000000001'
   ```

### Сортировка

1. **Маппинг полей**:
   - `created_asc` / `created_desc` → `created_at` (не `created`)
   - Остальные поля совпадают с именами в БД

2. **Множественная сортировка** (опционально):
   - По умолчанию после основного поля можно добавить сортировку по `id` (для стабильности)
   - Пример: `ORDER BY name ASC, id ASC`

3. **Обработка неверных значений**:
   - Если `sort` не из enum → 400 Bad Request

---

## 7. SQL примеры для реализации

### Базовый запрос с пагинацией

```sql
SELECT 
  c.id,
  c.name,
  c.email,
  c.phone,
  c.country_id,
  c.created_at,
  c.updated_at
FROM customers c
ORDER BY c.created_at DESC
LIMIT 20 OFFSET 0;
```

### Запрос с фильтрацией и поиском

```sql
SELECT 
  c.id,
  c.name,
  c.email,
  c.phone,
  c.country_id,
  c.created_at,
  c.updated_at
FROM customers c
WHERE 
  (c.name ILIKE '%john%' 
   OR c.email ILIKE '%john%' 
   OR c.phone ILIKE '%john%')
  AND c.country_id = '00000000-0000-0000-0000-000000000001'
ORDER BY c.name ASC, c.id ASC
LIMIT 20 OFFSET 0;
```

### Запрос с countryName (для CustomerListItem)

```sql
SELECT 
  c.id,
  c.name,
  c.email,
  c.phone,
  c.country_id,
  co.name AS country_name
FROM customers c
LEFT JOIN countries co ON c.country_id = co.id
WHERE 
  (c.name ILIKE '%john%' OR c.email ILIKE '%john%' OR c.phone ILIKE '%john%')
ORDER BY c.created_at DESC, c.id ASC
LIMIT 20 OFFSET 0;
```

---

## 8. Чеклист для проверки

После реализации проверить:

- [ ] `GET /api/v1/customers` без параметров → первая страница, 20 записей, сортировка по `created_desc`
- [ ] `GET /api/v1/customers?page=2` → вторая страница, 20 записей
- [ ] `GET /api/v1/customers?limit=50` → первая страница, 50 записей
- [ ] `GET /api/v1/customers?limit=200` → ограничено до 100 или возвращает 400
- [ ] `GET /api/v1/customers?page=0` → устанавливает `page=1` или возвращает 400
- [ ] `GET /api/v1/customers?email=john.doe@example.com` → фильтр по email работает
- [ ] `GET /api/v1/customers?countryId=...` → фильтр по стране работает
- [ ] `GET /api/v1/customers?search=john` → поиск работает (частичное совпадение)
- [ ] `GET /api/v1/customers?search=j` → возвращает 400 (минимум 2 символа)
- [ ] `GET /api/v1/customers?sort=name_asc` → сортировка по имени (A-Z)
- [ ] `GET /api/v1/customers?sort=name_desc` → сортировка по имени (Z-A)
- [ ] `GET /api/v1/customers?sort=created_asc` → старые первыми
- [ ] `GET /api/v1/customers?sort=created_desc` → новые первыми (по умолчанию)
- [ ] `GET /api/v1/customers?sort=invalid` → возвращает 400
- [ ] Комбинация всех параметров работает корректно
- [ ] Пустая страница (page=9999) → возвращает `[]`, не ошибку
- [ ] Все параметры опциональны (не обязательны)

---

## Итоговая структура параметров

После реализации endpoint `GET /api/v1/customers` будет поддерживать:

**Пагинация** (2 параметра):
- `page` (integer, default: 1)
- `limit` (integer, 1-100, default: 20)

**Фильтрация** (3 параметра):
- `email` (string, существующий)
- `countryId` (UUID, новый)
- `search` (string, min 2, новый)

**Сортировка** (1 параметр):
- `sort` (enum, default: `created_desc`)

**Всего 6 query параметров** (все опциональны).

---

Готово для реализации!

