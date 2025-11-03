# Финальное ревью API документации (api-docs.json)

**Дата проверки**: [текущая дата]  
**Статус**: Почти готово, осталось 2 критических проблемы

---

## ✅ Что исправлено из предыдущего отчета

- ✅ **Проблема 1 решена**: Параметры `Accept` добавлены в header для обоих GET endpoints
- ✅ **Проблема 2 решена**: Error responses используют схему `Error`
- ✅ **Проблема 3 решена**: Схема `Error` добавлена в `components/schemas` (упрощенная: `code`, `message`)
- ✅ **Проблема 4 решена**: Используются отдельные query параметры вместо объекта
- ✅ **Проблема 6 решена**: `operationId` исправлены (`getCustomers`, `getCustomerById`)

---

## ❌ Остались 2 критические проблемы

### Проблема 1: Отсутствуют Media Types в responses['200'] для GET /api/v1/customers

**Текущая реализация**:
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
      }
      // ❌ Отсутствуют:
      // application/vnd.api.customer.list+json
      // application/vnd.api.customer.lookup+json
    }
  }
}
```

**Проблема**: 
В параметре `Accept` указаны 3 Media Type:
- `application/json`
- `application/vnd.api.customer.list+json`
- `application/vnd.api.customer.lookup+json`

Но в `responses['200']` есть только `application/json`. Клиент не сможет получить список в форматах `list` и `lookup`.

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

### Проблема 2: Отсутствует Media Type в responses['200'] для GET /api/v1/customers/{id}

**Текущая реализация**:
```json
"responses": {
  "200": {
    "description": "Customer found",
    "content": {
      "application/json": {
        "schema": {
          "$ref": "#/components/schemas/Customer"
        }
      }
      // ❌ Отсутствует:
      // application/vnd.api.customer.detail+json
    }
  }
}
```

**Проблема**: 
В параметре `Accept` указаны 2 Media Type:
- `application/json`
- `application/vnd.api.customer.detail+json`

Но в `responses['200']` есть только `application/json`. Клиент не сможет получить детальное представление.

**Исправление**:
```json
"responses": {
  "200": {
    "description": "Customer found",
    "content": {
      "application/json": {
        "schema": {
          "$ref": "#/components/schemas/Customer"
        }
      },
      "application/vnd.api.customer.detail+json": {
        "schema": {
          "$ref": "#/components/schemas/CustomerDetail"
        }
      }
    }
  }
}
```

---

## ⚠️ Мелкие замечания

### Замечание 1: Error schema — отсутствуют required поля

**Текущая реализация**:
```json
"Error": {
  "type": "object",
  "properties": {
    "code": { ... },
    "message": { ... }
  },
  "description": "Error response"
}
```

**Рекомендация**: Добавить `required` для ясности:
```json
"Error": {
  "type": "object",
  "description": "Standard error response format",
  "required": ["code", "message"],
  "properties": {
    "code": {
      "type": "string",
      "description": "Machine-readable error code",
      "example": "NOT_ACCEPTABLE"
    },
    "message": {
      "type": "string",
      "description": "Human-readable error message",
      "example": "Unsupported media type in Accept header"
    }
  }
}
```

---

### Замечание 2: Error responses в PUT/PATCH/DELETE используют Customer вместо Error

**GET /api/v1/customers/{id} - PUT/PATCH**:
- `responses['400']`, `responses['404']`, `responses['409']` используют схему `Customer` вместо `Error`

**Рекомендация**: Использовать схему `Error` для всех error responses:
```json
"400": {
  "description": "Invalid input data",
  "content": {
    "application/json": {
      "schema": {
        "$ref": "#/components/schemas/Error"
      },
      "example": {
        "code": "VALIDATION_ERROR",
        "message": "Request validation failed"
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
"409": {
  "description": "Customer with this email already exists",
  "content": {
    "application/json": {
      "schema": {
        "$ref": "#/components/schemas/Error"
      },
      "example": {
        "code": "CONFLICT",
        "message": "Customer with this email already exists"
      }
    }
  }
}
```

**Аналогично для POST /api/v1/customers** — исправить error responses.

---

### Замечание 3: DELETE endpoint без error response schema

**DELETE /api/v1/customers/{id}**:
```json
"responses": {
  "204": { ... },
  "404": {
    "description": "Customer not found"
    // ❌ Нет content/schema
  }
}
```

**Рекомендация**: Добавить схему Error:
```json
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
}
```

---

### Замечание 4: GET /api/v1/customers/by-email/{email} использует Customer в 404

**Текущая реализация**:
```json
"404": {
  "description": "Customer not found",
  "content": {
    "*/*": {
      "schema": {
        "$ref": "#/components/schemas/Customer"
      }
    }
  }
}
```

**Рекомендация**: Использовать схему `Error`:
```json
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
}
```

---

## 📋 Чеклист исправлений

### Критические (обязательно):
- [ ] Добавить `application/vnd.api.customer.list+json` в `responses['200']` для `GET /api/v1/customers`
- [ ] Добавить `application/vnd.api.customer.lookup+json` в `responses['200']` для `GET /api/v1/customers`
- [ ] Добавить `application/vnd.api.customer.detail+json` в `responses['200']` для `GET /api/v1/customers/{id}`

### Желательно (для консистентности):
- [ ] Добавить `required: ["code", "message"]` в схему `Error`
- [ ] Исправить error responses в PUT/PATCH/POST endpoints (использовать `Error` вместо `Customer`)
- [ ] Добавить schema для 404 в DELETE endpoint
- [ ] Исправить 404 в GET /api/v1/customers/by-email/{email} (использовать `Error`)

---

## Итоговая оценка

**Общий результат**: ~95% выполнено

**Что отлично**:
- ✅ Все 7 схем данных правильно реализованы
- ✅ Пагинация, фильтрация, сортировка правильно настроены
- ✅ Accept header параметры добавлены
- ✅ Error responses используют схему Error (в GET endpoints)
- ✅ Отдельные query параметры
- ✅ Правильные operationId
- ✅ Схема Error упрощена (code, message)

**Что нужно исправить (2 критических + 4 желательных)**:
- ❌ **КРИТИЧНО**: Добавить отсутствующие Media Types в responses['200']
- ⚠️ **ЖЕЛАТЕЛЬНО**: Улучшить консистентность error responses в других endpoints

---

## Итог

**После исправления 2 критических проблем (Media Types в responses) API будет полностью готово к использованию!**

Остальные замечания — это улучшения для консистентности, но не блокируют работу API.

---

Готово для передачи дизайнеру API.

