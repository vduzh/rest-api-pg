# Инструкция по добавлению Content Negotiation и схем данных в Nano CRM API

## Обзор изменений

Требуется добавить в текущий API два механизма:
1. **Content Negotiation** — разные представления одних и тех же данных через Accept header
2. **Расширенный набор схем данных** — 7 схем вместо текущих 4

---

## 1. Content Negotiation (Content-Type через Accept Header)

### Принцип работы

Клиент указывает в HTTP заголовке `Accept`, какое представление данных ему нужно. Сервер возвращает данные в соответствующем формате.

**Важно**: Это стандартный REST подход, лучше чем query параметр `?view=...`, потому что:
- Строгая типизация в OpenAPI
- Автогенерация типизированных клиентов
- Соответствие REST принципам
- Поддержка стандартного кода 406 Not Acceptable

### Media Types для Customer

Нужно поддерживать следующие Accept header значения:

| Media Type | Назначение | Когда использовать |
|------------|-----------|-------------------|
| `application/json` | Базовое представление (по умолчанию) | Админка, отладка, экспорт данных |
| `application/vnd.api.customer.list+json` | Список с денормализованными данными | Таблицы, списки с названием страны |
| `application/vnd.api.customer.lookup+json` | Минимальные данные для выбора | Dropdown, autocomplete, select |
| `application/vnd.api.customer.detail+json` | Расширенное без системных полей | Карточки профиля, формы редактирования |

### Паттерн Media Type

```
application/vnd.api.customer.{representation}+json
│           │   │   │        │              │
│           │   │   │        │              └─ формат: JSON
│           │   │   │        └───────────────── представление: list/lookup/detail
│           │   │   └─────────────────────────── ресурс: customer
│           │   └─────────────────────────────── vendor: api
│           └─────────────────────────────────── tree: vendor-specific
└─────────────────────────────────────────────── type: application
```

**Пояснение**: `vnd.` — стандартный префикс для vendor-specific API (используется GitHub, Stripe, Heroku).

### Где применять Accept header

#### 1. `GET /api/v1/customers` (список)
Поддерживает 3 варианта Accept:
- `application/json` → массив `Customer[]`
- `application/vnd.api.customer.list+json` → массив `CustomerListItem[]`
- `application/vnd.api.customer.lookup+json` → массив `CustomerLookup[]`

**Примеры запросов**:
```http
# Базовое представление
GET /api/v1/customers
Accept: application/json

# Список для таблицы (с countryName)
GET /api/v1/customers?page=1&limit=20
Accept: application/vnd.api.customer.list+json

# Для dropdown
GET /api/v1/customers?limit=100
Accept: application/vnd.api.customer.lookup+json
```

#### 2. `GET /api/v1/customers/{id}` (один ресурс)
Поддерживает 2 варианта Accept:
- `application/json` → `Customer`
- `application/vnd.api.customer.detail+json` → `CustomerDetail`

**Примеры запросов**:
```http
# Базовое представление
GET /api/v1/customers/123e4567-e89b-12d3-a456-426614174000
Accept: application/json

# Для формы редактирования (без countryId)
GET /api/v1/customers/123e4567-e89b-12d3-a456-426614174000
Accept: application/vnd.api.customer.detail+json
```

#### 3. Обработка неподдерживаемого Accept

Если клиент запросил неподдерживаемый Media Type → вернуть **406 Not Acceptable**:

```json
{
  "error": {
    "code": "NOT_ACCEPTABLE",
    "message": "Unsupported media type in Accept header"
  }
}
```

---

## 2. Схемы данных

### Текущее состояние

Сейчас есть 4 схемы:
- ✅ `Customer` — базовое представление
- ✅ `CustomerCreate` — создание
- ✅ `CustomerUpdate` — полное обновление (PUT)
- ✅ `CustomerPatch` — частичное обновление (PATCH)

### Требуемые изменения

Нужно добавить **3 новые схемы** для response представлений:
- `CustomerListItem` — для списков с денормализацией
- `CustomerLookup` — для dropdown/select
- `CustomerDetail` — для детального просмотра

Итого будет **7 схем** (как в референсном дизайне).

---

### Описание всех 7 схем

#### 1. `Customer` — базовое представление (уже есть)

**Media Type**: `application/json`  
**HTTP Method**: GET (по умолчанию)

**Поля**:
```typescript
{
  id: string;                    // UUID, readonly
  name: string;                  // 2-100 символов
  email: string;                 // уникальный email
  phone: string;                 // опционально, до 20 символов
  countryId: string;             // UUID, FK к Country
  createdAt: string;             // date-time, readonly
  updatedAt: string;             // date-time, readonly
}
```

**Использование**: Админка, отладка, экспорт, аудит.

---

#### 2. `CustomerListItem` — **НОВАЯ** схема для списков

**Media Type**: `application/vnd.api.customer.list+json`  
**HTTP Method**: GET /api/v1/customers (с Accept header)

**Поля**:
```typescript
{
  id: string;                    // UUID
  name: string;                  // имя клиента
  email: string;                 // email
  phone: string;                 // телефон
  countryId: string;            // UUID страны
  countryName: string;           // НАЗВАНИЕ СТРАНЫ (денормализовано!)
}
```

**Ключевое отличие от `Customer`**:
- ✅ Добавлено поле `countryName` — название страны (денормализованное)
- ❌ Убраны `createdAt`, `updatedAt` (не нужны в таблицах)

**Зачем**: Позволяет отображать таблицу со странами без дополнительных запросов к API.

**Пример ответа**:
```json
[
  {
    "id": "123e4567-e89b-12d3-a456-426614174000",
    "name": "John Doe",
    "email": "john.doe@example.com",
    "phone": "+1-555-0101",
    "countryId": "00000000-0000-0000-0000-000000000001",
    "countryName": "United States"
  },
  {
    "id": "223e4567-e89b-12d3-a456-426614174002",
    "name": "Jane Smith",
    "email": "jane.smith@example.com",
    "phone": "+44-20-1234-5678",
    "countryId": "00000000-0000-0000-0000-000000000002",
    "countryName": "United Kingdom"
  }
]
```

---

#### 3. `CustomerLookup` — **НОВАЯ** схема для dropdown

**Media Type**: `application/vnd.api.customer.lookup+json`  
**HTTP Method**: GET /api/v1/customers (с Accept header)

**Поля**:
```typescript
{
  id: string;                    // UUID
  name: string;                  // имя для отображения
}
```

**Минимум данных** — только то, что нужно для выбора в UI.

**Пример ответа**:
```json
[
  {
    "id": "123e4567-e89b-12d3-a456-426614174000",
    "name": "John Doe"
  },
  {
    "id": "223e4567-e89b-12d3-a456-426614174002",
    "name": "Jane Smith"
  }
]
```

**Использование**: Dropdown списки, autocomplete, выбор клиента в формах.

---

#### 4. `CustomerDetail` — **НОВАЯ** схема для детального просмотра

**Media Type**: `application/vnd.api.customer.detail+json`  
**HTTP Method**: GET /api/v1/customers/{id} (с Accept header)

**Поля**:
```typescript
{
  id: string;                    // UUID
  name: string;                  // 2-100 символов
  email: string;                 // email
  phone: string;                 // телефон
  // НЕТ countryId - убрано для чистоты UI
  // НЕТ createdAt/updatedAt - не нужны пользователю
}
```

**Ключевое отличие от `Customer`**:
- ❌ Убраны `countryId`, `createdAt`, `updatedAt` — системные поля не нужны в пользовательских интерфейсах

**Использование**: Карточки профиля, формы редактирования (без отображения служебных полей).

---

#### 5. `CustomerCreate` — создание (уже есть, без изменений)

**HTTP Method**: POST /api/v1/customers

**Поля**:
```typescript
{
  name: string;                  // required
  email: string;                 // required, уникальный
  phone?: string;                // optional
  countryId?: string;            // optional, UUID
  // НЕТ id - генерируется сервером
  // НЕТ createdAt/updatedAt - генерируются сервером
}
```

**Текущая реализация корректна** — исключает readonly поля.

---

#### 6. `CustomerUpdate` — полное обновление (уже есть, без изменений)

**HTTP Method**: PUT /api/v1/customers/{id}

**Поля**: Все поля обязательны (кроме readonly).
```typescript
{
  name: string;                  // required
  email: string;                 // required
  phone: string;                 // required (может быть пустая строка)
  countryId: string;             // required
}
```

**Семантика PUT**: Идемпотентная операция — полная замена ресурса.

---

#### 7. `CustomerPatch` — частичное обновление (уже есть, без изменений)

**HTTP Method**: PATCH /api/v1/customers/{id}

**Поля**: Все поля опциональны, минимум 1 поле должно быть отправлено.
```typescript
{
  name?: string;                  // optional
  email?: string;                 // optional
  phone?: string;                 // optional
  countryId?: string;             // optional
}
```

**Семантика PATCH**: Не идемпотентная — обновляет только указанные поля.

---

## 3. Технические детали реализации в OpenAPI

### Добавление параметра Accept в endpoints

Для `GET /api/v1/customers`:
```yaml
parameters:
  - name: Accept
    in: header
    description: |
      Media type for response representation:
      - `application/json` - Base normalized representation (default)
      - `application/vnd.api.customer.list+json` - List with denormalized data (includes countryName)
      - `application/vnd.api.customer.lookup+json` - Lookup (id + name only)
    required: false
    schema:
      type: string
      enum:
        - application/json
        - application/vnd.api.customer.list+json
        - application/vnd.api.customer.lookup+json
      default: application/json
```

Для `GET /api/v1/customers/{id}`:
```yaml
parameters:
  - name: Accept
    in: header
    description: |
      Media type for response representation:
      - `application/json` - Base normalized representation (default)
      - `application/vnd.api.customer.detail+json` - Extended without readonly fields
    required: false
    schema:
      type: string
      enum:
        - application/json
        - application/vnd.api.customer.detail+json
      default: application/json
```

### Responses с разными Content-Type

Для `GET /api/v1/customers` в `responses['200']`:
```yaml
content:
  application/json:
    schema:
      type: array
      items:
        $ref: '#/components/schemas/Customer'
  
  application/vnd.api.customer.list+json:
    schema:
      type: array
      items:
        $ref: '#/components/schemas/CustomerListItem'
  
  application/vnd.api.customer.lookup+json:
    schema:
      type: array
      items:
        $ref: '#/components/schemas/CustomerLookup'
```

Для `GET /api/v1/customers/{id}` в `responses['200']`:
```yaml
content:
  application/json:
    schema:
      $ref: '#/components/schemas/Customer'
  
  application/vnd.api.customer.detail+json:
    schema:
      $ref: '#/components/schemas/CustomerDetail'
```

### Определение новых схем в components/schemas

```yaml
components:
  schemas:
    # ... существующие схемы ...
    
    CustomerListItem:
      type: object
      description: |
        List representation with denormalized data for tables.
        Used with: `application/vnd.api.customer.list+json`
      required:
        - id
        - name
        - email
        - phone
        - countryId
        - countryName
      properties:
        id:
          type: string
          format: uuid
          example: "123e4567-e89b-12d3-a456-426614174000"
        name:
          type: string
          minLength: 2
          maxLength: 100
          example: "John Doe"
        email:
          type: string
          format: email
          maxLength: 255
          example: "john.doe@example.com"
        phone:
          type: string
          maxLength: 20
          example: "+1-555-0101"
        countryId:
          type: string
          format: uuid
          example: "00000000-0000-0000-0000-000000000001"
        countryName:
          type: string
          description: Country name (denormalized!)
          example: "United States"
    
    CustomerLookup:
      type: object
      description: |
        Lookup representation for dropdown/select controls.
        Used with: `application/vnd.api.customer.lookup+json`
      required:
        - id
        - name
      properties:
        id:
          type: string
          format: uuid
          example: "123e4567-e89b-12d3-a456-426614174000"
        name:
          type: string
          description: Display name
          example: "John Doe"
    
    CustomerDetail:
      type: object
      description: |
        Extended representation without readonly fields.
        Excludes: countryId, createdAt, updatedAt
        Used with: `application/vnd.api.customer.detail+json`
      required:
        - id
        - name
        - email
        - phone
      properties:
        id:
          type: string
          format: uuid
          example: "123e4567-e89b-12d3-a456-426614174000"
        name:
          type: string
          minLength: 2
          maxLength: 100
          example: "John Doe"
        email:
          type: string
          format: email
          maxLength: 255
          example: "john.doe@example.com"
        phone:
          type: string
          maxLength: 20
          example: "+1-555-0101"
```

### Обработка ошибки 406 Not Acceptable

Добавить в `responses`:
```yaml
responses:
  '406':
    description: Not Acceptable - unsupported Accept header media type
    content:
      application/json:
        schema:
          $ref: '#/components/schemas/Error'
        example:
          error:
            code: "NOT_ACCEPTABLE"
            message: "Unsupported media type in Accept header"
```

---

## 4. Важные моменты для реализации

### Денормализация countryName

В схеме `CustomerListItem` поле `countryName` должно быть заполнено из связанной таблицы Country:

```sql
-- Пример SQL запроса для получения списка с countryName
SELECT 
  c.id,
  c.name,
  c.email,
  c.phone,
  c.country_id,
  co.name AS country_name
FROM customers c
LEFT JOIN countries co ON c.country_id = co.id
```

**Важно**: Это денормализация на уровне SELECT — не хранить в БД, а делать JOIN при запросе.

### По умолчанию

Если клиент не указал Accept header → использовать `application/json` (базовое представление `Customer`).

### Обратная совместимость

После добавления новых схем старые клиенты продолжат работать:
- Не указывают Accept → получают `application/json` (как раньше)
- Указывают неподдерживаемый Accept → получают 406 (явная ошибка лучше чем молчаливое игнорирование)

---

## 5. Чеклист для проверки

После реализации проверить:

- [ ] `GET /api/v1/customers` с `Accept: application/json` → массив `Customer[]`
- [ ] `GET /api/v1/customers` с `Accept: application/vnd.api.customer.list+json` → массив `CustomerListItem[]` с `countryName`
- [ ] `GET /api/v1/customers` с `Accept: application/vnd.api.customer.lookup+json` → массив `CustomerLookup[]` (только id + name)
- [ ] `GET /api/v1/customers/{id}` с `Accept: application/json` → `Customer`
- [ ] `GET /api/v1/customers/{id}` с `Accept: application/vnd.api.customer.detail+json` → `CustomerDetail` (без countryId, createdAt, updatedAt)
- [ ] Неподдерживаемый Accept → 406 Not Acceptable
- [ ] Отсутствие Accept header → работает как `application/json`
- [ ] Все 3 новые схемы определены в `components/schemas`
- [ ] `CustomerListItem` содержит `countryName` (денормализованное поле)

---

## 6. Итоговая структура

После реализации у вас будет:

**7 схем**:
1. `Customer` (базовое)
2. `CustomerListItem` (список с countryName) — **НОВАЯ**
3. `CustomerLookup` (dropdown) — **НОВАЯ**
4. `CustomerDetail` (детальное) — **НОВАЯ**
5. `CustomerCreate` (создание)
6. `CustomerUpdate` (полное обновление)
7. `CustomerPatch` (частичное обновление)

**4 Media Types**:
1. `application/json` → `Customer`
2. `application/vnd.api.customer.list+json` → `CustomerListItem[]`
3. `application/vnd.api.customer.lookup+json` → `CustomerLookup[]`
4. `application/vnd.api.customer.detail+json` → `CustomerDetail`

**2 endpoints с content negotiation**:
1. `GET /api/v1/customers` (3 варианта Accept)
2. `GET /api/v1/customers/{id}` (2 варианта Accept)

---
