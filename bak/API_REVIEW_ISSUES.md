# Отчет ревью API документации (api-docs.json)

**Дата проверки**: [текущая дата]  
**Статус**: Требуется доработка

---

## ✅ Что сделано правильно

### 1. Схемы данных — полностью реализовано ✅

Все 7 схем присутствуют и правильно определены:
- ✅ `Customer` — базовое представление
- ✅ `CustomerListItem` — список с `countryName` (денормализовано)
- ✅ `CustomerLookup` — для dropdown
- ✅ `CustomerDetail` — детальное без readonly полей
- ✅ `CustomerCreate` — создание
- ✅ `CustomerUpdate` — полное обновление
- ✅ `CustomerPatch` — частичное обновление

**Поля схем**:
- ✅ `CustomerListItem` содержит `countryName` (денормализованное)
- ✅ `CustomerDetail` правильно исключает `countryId`, `createdAt`, `updatedAt`
- ✅ `CustomerLookup` содержит только `id` и `name`

### 2. Пагинация — реализовано ✅

- ✅ Параметр `page` в `CustomerSearchParams`:
  - type: integer
  - minimum: 1
  - default: 1

- ✅ Параметр `limit` в `CustomerSearchParams`:
  - type: integer
  - minimum: 1
  - maximum: 100
  - default: 20

### 3. Фильтрация — реализовано ✅

- ✅ Параметр `email` — фильтр по точному email
- ✅ Параметр `countryId` — фильтр по стране (UUID)
- ✅ Параметр `search` — поиск по name/email/phone (minLength: 2)

### 4. Сортировка — реализовано ✅

- ✅ Параметр `sort` в `CustomerSearchParams`:
  - type: string
  - enum: все нужные значения присутствуют
  - default: `created_desc`

---

## ❌ Критические проблемы (требуют исправления)

### Проблема 1: Отсутствует параметр Accept в header

**GET /api/v1/customers** и **GET /api/v1/customers/{id}** не имеют параметра `Accept` в `parameters`.

**Требуется добавить для GET /api/v1/customers**:
```json
{
  "name": "Accept",
  "in": "header",
  "description": "Media type for response representation: application/json (default), application/vnd.api.customer.list+json, application/vnd.api.customer.lookup+json",
  "required": false,
  "schema": {
    "type": "string",
    "enum": ["application/json", "application/vnd.api.customer.list+json", "application/vnd.api.customer.lookup+json"],
    "default": "application/json"
  }
}
```

**Требуется добавить для GET /api/v1/customers/{id}**:
```json
{
  "name": "Accept",
  "in": "header",
  "description": "Media type for response representation: application/json (default), application/vnd.api.customer.detail+json",
  "required": false,
  "schema": {
    "type": "string",
    "enum": ["application/json", "application/vnd.api.customer.detail+json"],
    "default": "application/json"
  }
}
```

---

### Проблема 2: Неверные схемы в error responses

**GET /api/v1/customers**:
- В `responses['400']` указан `CustomerLookup[]` вместо схемы `Error`
- В `responses['406']` указан `CustomerLookup[]` вместо схемы `Error`

**GET /api/v1/customers/{id}**:
- В `responses['404']` указан `CustomerDetail` вместо схемы `Error`
- В `responses['406']` указан `CustomerDetail` вместо схемы `Error`

**Исправление** — все error responses должны использовать схему `Error`:
```json
{
  "400": {
    "description": "Bad Request - invalid search parameters",
    "content": {
      "application/json": {
        "schema": {
          "$ref": "#/components/schemas/Error"
        },
        "example": {
          "code": "INVALID_SORT_PARAMETER",
          "message": "Invalid sort parameter. Allowed values: name_asc, name_desc, email_asc, email_desc, created_asc, created_desc"
        }
      }
    }
  },
  "404": {
    "description": "Customer not found",
    "content": {
      "application/json": {
        "schema": {
          "$ref": "#/components/schemas/Error"
        },
        "example": {
          "code": "NOT_FOUND",
          "message": "Customer not found"
        }
      }
    }
  },
  "406": {
    "description": "Not Acceptable - unsupported Accept header media type",
    "content": {
      "application/json": {
        "schema": {
          "$ref": "#/components/schemas/Error"
        },
        "example": {
          "code": "NOT_ACCEPTABLE",
          "message": "Unsupported media type in Accept header"
        }
      }
    }
  }
}
```

---

### Проблема 3: Отсутствует схема Error

В `components/schemas` нет схемы `Error`, которая должна использоваться для всех error responses.

**Требуется добавить в `components.schemas`**:
```json
"Error": {
  "type": "object",
  "description": "Standard error response format",
  "required": ["code", "message"],
  "properties": {
    "code": {
      "type": "string",
      "description": "Machine-readable error code",
      "example": "VALIDATION_ERROR"
    },
    "message": {
      "type": "string",
      "description": "Human-readable error message",
      "example": "Request validation failed"
    }
  }
}
```

---

### Проблема 4: Неправильная структура параметров GET /api/v1/customers

**Текущая реализация**:
```json
"parameters": [{
  "name": "params",
  "in": "query",
  "required": true,  // ← ПРОБЛЕМА: все параметры должны быть опциональными
  "schema": {
    "$ref": "#/components/schemas/CustomerSearchParams"
  }
}]
```

**Проблема**: 
- Используется один объект `params` вместо отдельных query параметров (нестандартно для REST API)
- Параметр помечен как `required: true`, хотя все query параметры должны быть опциональными

**Решение (рекомендуется)**: Использовать отдельные query параметры вместо объекта:
```json
"parameters": [
  {
    "name": "page",
    "in": "query",
    "required": false,
    "description": "Page number (1-indexed)",
    "schema": {
      "type": "integer",
      "minimum": 1,
      "default": 1
    },
    "example": 1
  },
  {
    "name": "limit",
    "in": "query",
    "required": false,
    "description": "Number of items per page (1-100)",
    "schema": {
      "type": "integer",
      "minimum": 1,
      "maximum": 100,
      "default": 20
    },
    "example": 20
  },
  {
    "name": "email",
    "in": "query",
    "required": false,
    "description": "Filter by exact email match",
    "schema": {
      "type": "string",
      "format": "email"
    },
    "example": "john.doe@example.com"
  },
  {
    "name": "countryId",
    "in": "query",
    "required": false,
    "description": "Filter by country ID",
    "schema": {
      "type": "string",
      "format": "uuid"
    },
    "example": "00000000-0000-0000-0000-000000000001"
  },
  {
    "name": "search",
    "in": "query",
    "required": false,
    "description": "Search in name, email, phone fields (partial match, case-insensitive)",
    "schema": {
      "type": "string",
      "minLength": 2
    },
    "example": "john"
  },
  {
    "name": "sort",
    "in": "query",
    "required": false,
    "description": "Sort field and direction",
    "schema": {
      "type": "string",
      "enum": ["name_asc", "name_desc", "email_asc", "email_desc", "created_asc", "created_desc"],
      "default": "created_desc"
    },
    "example": "created_desc"
  }
]
```

**Альтернативное решение**: Если оставляете объект `params`, то:
```json
"parameters": [{
  "name": "params",
  "in": "query",
  "required": false,  // ← изменить на false
  "schema": {
    "$ref": "#/components/schemas/CustomerSearchParams"
  }
}]
```

---

### Проблема 5: Неправильные типы в responses для GET /api/v1/customers

**Текущая реализация**:
```json
"responses": {
  "200": {
    "content": {
      "application/json": {
        "schema": {
          "$ref": "#/components/schemas/Customer"  // ← единичный объект
        }
      },
      "application/vnd.api.customer.list+json": {
        "schema": {
          "$ref": "#/components/schemas/CustomerListItem"  // ← единичный объект
        }
      },
      "application/vnd.api.customer.lookup+json": {
        "schema": {
          "$ref": "#/components/schemas/CustomerLookup"  // ← единичный объект
        }
      }
    }
  }
}
```

**Проблема**: Все три представления должны быть массивами, а не единичными объектами (это list endpoint).

**Исправление**:
```json
"responses": {
  "200": {
    "description": "Customers retrieved successfully",
    "content": {
      "application/json": {
        "schema": {
          "type": "array",
          "items": {
            "$ref": "#/components/schemas/Customer"
          }
        }
      },
      "application/vnd.api.customer.list+json": {
        "schema": {
          "type": "array",
          "items": {
            "$ref": "#/components/schemas/CustomerListItem"
          }
        }
      },
      "application/vnd.api.customer.lookup+json": {
        "schema": {
          "type": "array",
          "items": {
            "$ref": "#/components/schemas/CustomerLookup"
          }
        }
      }
    }
  }
}
```

---

### Проблема 6: Неправильные operationId

**GET /api/v1/customers**:
- Текущий: `"operationId": "getCustomersAsLookup"`
- Должен быть: `"operationId": "getCustomers"`

**GET /api/v1/customers/{id}**:
- Текущий: `"operationId": "getCustomerDetailById"`
- Должен быть: `"operationId": "getCustomerById"`

---

## ⚠️ Мелкие замечания

### Замечание 1: Названия summary

**GET /api/v1/customers**:
- Текущий: `"Get customers (base representation)"`
- Рекомендуется: `"Get customers list with pagination, filtering, and sorting"`

**GET /api/v1/customers/{id}**:
- Текущий: `"Get customer by ID (base representation)"`
- Рекомендуется: `"Get customer by ID"`

**Причина**: Представление зависит от Accept header, а не является фиксированным "base".

---

### Замечание 2: CustomerSearchParams — явная опциональность

Все параметры в `CustomerSearchParams` должны явно иметь `required: false` или не указывать `required` вообще (по умолчанию false для query параметров в OpenAPI).

---

## 📋 Чеклист исправлений

- [ ] Добавить параметр `Accept` в header для `GET /api/v1/customers`
- [ ] Добавить параметр `Accept` в header для `GET /api/v1/customers/{id}`
- [ ] Исправить схемы в error responses (400, 404, 406) на `Error`
- [ ] Добавить схему `Error` в `components/schemas` (упрощенная версия: только `code` и `message`)
- [ ] Исправить структуру параметров `GET /api/v1/customers` (отдельные query параметры или `required: false`)
- [ ] Исправить типы в responses для list endpoints (массивы вместо единичных объектов)
- [ ] Исправить `operationId` для `GET /api/v1/customers` → `"getCustomers"`
- [ ] Исправить `operationId` для `GET /api/v1/customers/{id}` → `"getCustomerById"`
- [ ] (Опционально) Уточнить описания в `summary` для endpoints

---

## Итоговая оценка

**Общий результат**: ~70% выполнено

**Что хорошо**:
- ✅ Все 7 схем данных правильно реализованы
- ✅ Пагинация, фильтрация, сортировка правильно настроены
- ✅ Content negotiation частично работает (schemas в responses есть)

**Что нужно исправить (6 критических проблем)**:
- ❌ Добавить Accept header параметры
- ❌ Исправить error responses (использовать Error schema)
- ❌ Добавить схему Error (упрощенная: `code` и `message`)
- ❌ Исправить типы массивов в responses
- ❌ Исправить структуру query параметров
- ❌ Исправить operationId

---

**После исправления всех пунктов из чеклиста API будет полностью соответствовать требованиям.**

---

Готово для передачи дизайнеру API.

