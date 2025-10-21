# Sports Management System REST API - Проектное решение

## Обзор

Профессиональное проектирование REST API для спортивной системы управления с использованием content negotiation через Accept headers и следованием industry-standard best practices.

**Примечание**: Данная спецификация демонстрирует подход на примере модуля Athletes, но принципы и паттерны применимы ко всем ресурсам системы (Coaches и т.д.).

---

## 🎯 Ключевые архитектурные решения

### 1. Content Negotiation через Accept Header

**Решение**: Использование Accept header вместо query parameter `?view=...`

**Обоснование**:
- ✅ Строгая типизация на уровне OpenAPI спецификации
- ✅ Каждый Content-Type жестко связан со своей схемой
- ✅ Автоматическая генерация типизированных клиентов
- ✅ Соответствие принципам REST (content negotiation)
- ✅ Поддержка стандартного кода ответа 406 Not Acceptable

**Альтернативы отклонены**:
- Query parameter `?view=summary` - слабая связь с типами
- Separate endpoints `/athletes/{id}/summary` - нарушение принципа "один ресурс = один URL"
- Accept header был выбран как более правильный подход

---

## 📐 Конвенция Media Types

### Принятый паттерн:
```
application/vnd.api.{resource}.{representation}+json
```

### Для любого ресурса системы:
```
application/json                              # базовое нормализованное представление (default)
application/vnd.api.{resource}.list+json      # списковое представление (таблицы, денормализация)
application/vnd.api.{resource}.lookup+json    # для dropdown/select (id + name)
application/vnd.api.{resource}.detail+json    # расширенное без readonly полей
application/vnd.api.{resource}.summary+json   # минимальное представление
```

### Примеры для разных ресурсов:
```
application/json                              # базовое представление (любой ресурс)
application/vnd.api.athlete.list+json         # Athletes - списки/таблицы
application/vnd.api.athlete.lookup+json       # Athletes - dropdown
application/vnd.api.athlete.detail+json       # Athletes - detail
application/vnd.api.coach.list+json           # Coaches - списки/таблицы
application/vnd.api.coach.lookup+json         # Coaches - dropdown
```

### Анатомия Media Type:
```
application/vnd.api.{resource}.{representation}+json
│           │   │   │         │              │
│           │   │   │         │              └─ suffix: формат данных
│           │   │   │         └───────────────── representation: представление
│           │   │   └─────────────────────────── resource: тип ресурса
│           │   └─────────────────────────────── vendor: проект/компания
│           └─────────────────────────────────── tree: vendor-specific
└─────────────────────────────────────────────── type: application
```

**Обоснование выбора `vnd.` префикса**:
- Стандарт для vendor-specific API (GitHub, Stripe, Heroku используют)
- Подходит для собственных API продуктов
- Регистрируемый в IANA (опционально)

---

## 🗂️ Схемы данных и naming conventions

### Response Models (что возвращает сервер)

#### 1. `{Resource}` - Базовое нормализованное представление
**Media Type**: `application/json`

**Пример для Athlete**:

```typescript
interface Athlete {
  id: string;              // UUID, readonly, генерируется сервером
  coachId: string;         // UUID, FK to Coach
  name: string;            // 1-255 символов
  email: string;           // уникальный email
  phone: string;           // E.164 формат (+1234567890)
  telegram: string | null; // @username или null
  createdAt: string;       // ISO 8601, readonly
  updatedAt: string;       // ISO 8601, readonly
}
```

**Use cases**:
- Стандартное представление по умолчанию
- Нормализованные данные "как есть" из БД
- Admin панели
- Debugging и auditing
- Data export
- Когда не указан специальный Accept header

---

#### 2. `{Resource}ListItem` - Списковое представление
**Media Type**: `application/vnd.api.{resource}.list+json`

**Пример для AthleteListItem**:

```typescript
interface AthleteListItem {
  id: string;              // UUID
  name: string;            // имя атлета
  email: string;           // email атлета
  phone: string;           // телефон
  coachId: string;         // UUID тренера
  coachName: string;       // имя тренера (денормализовано!)
  createdAt: string;       // ISO 8601
}
```

**Use cases**:
- Списки в таблицах (grid/table views)
- Денормализованные данные для производительности
- Когда нужны связанные данные без JOIN на клиенте
- Списковые endpoint'ы с пагинацией

---

#### 3. `{Resource}Lookup` - Для dropdown/select
**Media Type**: `application/vnd.api.{resource}.lookup+json`

**Пример для AthleteLookup**:

```typescript
interface AthleteLookup {
  id: string;              // UUID
  name: string;            // отображаемое имя
}
```

**Use cases**:
- Dropdown списки (select boxes)
- Autocomplete поля
- Reference fields в формах
- Минимальный объем данных для выбора

---

#### 4. `{Resource}Detail` - Расширенное без системных полей
**Media Type**: `application/vnd.api.{resource}.detail+json`

**Пример для AthleteDetail**:

```typescript
interface AthleteDetail {
  id: string;
  name: string;
  email: string;
  phone: string;
  telegram: string | null;
  // Исключены: coachId, createdAt, updatedAt
}
```

**Use cases**:
- Карточки профилей
- Формы редактирования
- User-facing интерфейсы, где системные поля не нужны

---

#### 5. `{Resource}Summary` - Краткое представление
**Media Type**: `application/vnd.api.{resource}.summary+json`

**Пример для AthleteSummary**:

```typescript
interface AthleteSummary {
  id: string;
  name: string;
  email: string;
}
```

**Use cases**:
- Превью результатов поиска
- Компактные карточки
- Списки с большим количеством элементов (экономия трафика)
- Когда нужно чуть больше чем lookup, но меньше чем list

---

### Request Models (что отправляет клиент)

#### 6. `{Resource}Create` - Создание ресурса (POST)

**Пример для AthleteCreate**:

```typescript
interface AthleteCreate {
  coachId: string;         // обязательно
  name: string;            // обязательно
  email: string;           // обязательно
  phone: string;           // обязательно
  telegram?: string | null; // опционально
  // Исключены: id, createdAt, updatedAt (генерируются сервером)
}
```

**Обоснование отдельной схемы**:
- 🔒 Безопасность: клиент не может подделать `id`
- ✅ Логичность: временные метки генерирует только сервер
- ✅ Валидация: четкое разделение входных и выходных данных

---

#### 7. `{Resource}Update` - Полная замена (PUT)

**Пример для AthleteUpdate**:

```typescript
interface AthleteUpdate {
  coachId: string;         // обязательно
  name: string;            // обязательно
  email: string;           // обязательно
  phone: string;           // обязательно
  telegram: string | null; // обязательно явно указать (даже null)
  // Исключены: id, createdAt, updatedAt
}
```

**Семантика PUT**:
- Заменяет весь ресурс
- Все поля обязательны
- Отсутствующие поля приведут к ошибке валидации
- Идемпотентная операция

---

#### 8. `{Resource}Patch` - Частичное обновление (PATCH)

**Пример для AthletePatch**:

```typescript
interface AthletePatch {
  coachId?: string;         // опционально
  name?: string;            // опционально
  email?: string;           // опционально
  phone?: string;           // опционально
  telegram?: string | null; // опционально
  // Минимум 1 поле должно быть отправлено
}
```

**Семантика PATCH**:
- Обновляет только указанные поля
- Все поля optional
- Остальные поля не изменяются
- НЕ идемпотентная операция при конкурентных обновлениях

---

## 🏗️ Naming Convention

### Принятый стандарт: `{Resource}{Operation}`

| Схема | HTTP Method | Назначение | Все поля обязательны? |
|-------|-------------|------------|-----------------------|
| `{Resource}` | GET | Базовое чтение | ✅ Да |
| `{Resource}ListItem` | GET | Списки/таблицы | ✅ Да |
| `{Resource}Lookup` | GET | Dropdown (id+name) | ✅ Да |
| `{Resource}Detail` | GET | Расширенное чтение | ✅ Да |
| `{Resource}Summary` | GET | Краткое чтение | ✅ Да |
| `{Resource}Create` | POST | Создание | ✅ Да (кроме optional) |
| `{Resource}Update` | PUT | Полная замена | ✅ Да |
| `{Resource}Patch` | PATCH | Частичное обновление | ❌ Все optional |

**Примеры для разных ресурсов**:
- `Athlete`, `AthleteListItem`, `AthleteLookup`, `AthleteDetail`, `AthleteSummary`, `AthleteCreate`, `AthleteUpdate`, `AthletePatch`
- `Coach`, `CoachListItem`, `CoachLookup`, `CoachDetail`, `CoachSummary`, `CoachCreate`, `CoachUpdate`, `CoachPatch`

**Это НЕ случайность** - это industry standard, используемый в:
- REST API Best Practices (Microsoft, Google)
- OpenAPI Generator defaults
- NestJS, Spring Boot, ASP.NET Core
- Крупнейших публичных API (Stripe, GitHub, Twilio)

---

## 🛣️ Эндпоинты API

**Примечание**: Показаны эндпоинты для модуля Athletes как пример. Аналогичные эндпоинты должны быть реализованы для всех ресурсов системы.

### 1. `GET /{resource}` - Список ресурсов
**Пример**: `GET /athletes` - Список атлетов
- Пагинация: `page`, `limit`
- Фильтрация: `coachId`, `search`
- Сортировка: `sort` (name_asc, name_desc, created_asc, created_desc)
- Accept header определяет представление:
  - `application/vnd.api.{resource}.list+json` → `AthleteListItem[]` (для таблиц, с денормализацией)
  - `application/vnd.api.{resource}.lookup+json` → `AthleteLookup[]` (для dropdown)
  - `application/vnd.api.{resource}.summary+json` → `AthleteSummary[]` (компактное)
  - `application/json` → `Athlete[]` (базовое, default)
- **Ответ**: 200 OK с массивом + pagination metadata

---

### 2. `POST /{resource}` - Создание ресурса
**Пример**: `POST /athletes` - Создание атлета
- **Body**: `AthleteCreate` схема
- **Ответ**: 201 Created + Location header
- **Возвращает**: Полный объект `Athlete` с сгенерированными полями
- **Ошибки**: 409 (конфликт email), 422 (валидация)

---

### 3. `GET /{resource}/{id}` - Получение по ID
**Пример**: `GET /athletes/{athleteId}` - Получение атлета по ID
- Accept header определяет представление:
  - `application/json` → `Athlete` (default, базовое)
  - `application/vnd.api.{resource}.detail+json` → `AthleteDetail`
  - `application/vnd.api.{resource}.summary+json` → `AthleteSummary`
- **Ответ**: 200 OK
- **Ошибки**: 404 (не найден), 406 (неподдерживаемый Accept)

---

### 4. `PUT /{resource}/{id}` - Полная замена
**Пример**: `PUT /athletes/{athleteId}` - Полная замена атлета
- **Body**: `AthleteUpdate` схема (все поля обязательны)
- **Ответ**: 200 OK с полным `Athlete`
- **Семантика**: Идемпотентная операция
- **Ошибки**: 404, 422 (валидация)

---

### 5. `PATCH /{resource}/{id}` - Частичное обновление
**Пример**: `PATCH /athletes/{athleteId}` - Частичное обновление атлета
- **Body**: `AthletePatch` схема (минимум 1 поле)
- **Ответ**: 200 OK с полным `Athlete`
- **Семантика**: НЕ идемпотентная
- **Ошибки**: 400 (пустое тело), 404, 422

---

### 6. `DELETE /{resource}/{id}` - Удаление
**Пример**: `DELETE /athletes/{athleteId}` - Удаление атлета
- **Ответ**: 204 No Content (успех, пустое тело)
- **Семантика**: Идемпотентная
- **Ошибки**: 
  - 404 (не найден)
  - 409 (имеет зависимости, например, связанные записи)

---

### 7. `DELETE /{resource}/batch` - Массовое удаление
**Пример**: `DELETE /athletes/batch` - Массовое удаление атлетов
- **Body**: `{ ids: string[] }` (1-100 ID)
- **Ответ**: 200 OK с отчетом:
  ```json
  {
    "deleted": ["id1", "id2"],
    "failed": [
      { "id": "id3", "reason": "Not found" }
    ]
  }
  ```
- **Семантика**: Частичный успех (no rollback)

---

### 8. `GET /{parent-resource}/{parentId}/{child-resource}` - Nested resources
**Пример**: `GET /coaches/{coachId}/athletes` - Атлеты тренера
- Nested resource endpoint
- Те же параметры пагинации и Accept header
- **Ответ**: 200 OK с массивом + pagination
- **Ошибки**: 404 (тренер не найден)

---

## 📊 HTTP методы и семантика

| Метод | Идемпотентный? | Безопасный? | Назначение | Код успеха |
|-------|----------------|-------------|------------|------------|
| GET | ✅ Да | ✅ Да | Получение данных | 200 |
| POST | ❌ Нет | ❌ Нет | Создание ресурса | 201 |
| PUT | ✅ Да | ❌ Нет | Полная замена | 200 |
| PATCH | ❌ Нет | ❌ Нет | Частичное обновление | 200 |
| DELETE | ✅ Да | ❌ Нет | Удаление | 204 |

**Идемпотентность**: повторный запрос дает тот же результат  
**Безопасность**: не изменяет состояние сервера

---

## 🚨 Структура ошибок

### Единообразный формат

```json
{
  "error": {
    "code": "VALIDATION_ERROR",           // машиночитаемый код
    "message": "Request validation failed", // человекочитаемое сообщение
    "details": {                          // дополнительный контекст
      "field": "email",
      "reason": "Invalid format"
    },
    "path": "/v1/athletes",               // путь запроса
    "timestamp": "2024-01-20T14:25:00Z"   // время ошибки
  }
}
```

### Специальный формат для валидации (422)

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "validationErrors": [
      {
        "field": "email",
        "message": "Invalid email format",
        "code": "INVALID_FORMAT"
      },
      {
        "field": "phone",
        "message": "Phone number must be in E.164 format",
        "code": "INVALID_FORMAT"
      }
    ]
  }
}
```

### Коды статуса HTTP

| Код | Назначение | Когда используется |
|-----|------------|--------------------|
| 200 | OK | GET, PUT, PATCH успешны |
| 201 | Created | POST успешен + Location header |
| 204 | No Content | DELETE успешен (пустое тело) |
| 400 | Bad Request | Некорректный JSON, неверные параметры |
| 401 | Unauthorized | Отсутствует/невалидный JWT токен |
| 404 | Not Found | Ресурс не найден |
| 406 | Not Acceptable | Неподдерживаемый Accept header |
| 409 | Conflict | Дубликат email, нельзя удалить (зависимости) |
| 422 | Unprocessable Entity | Ошибка валидации данных |
| 500 | Internal Server Error | Ошибка сервера |

---

## 🔐 Аутентификация

**Метод**: JWT Bearer токен

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

- ✅ Все эндпоинты защищены
- ❌ Без токена → 401 Unauthorized
- Токен проверяется middleware на каждый запрос

---

## 📈 Пагинация

### Query параметры
- `page` - номер страницы (1-indexed, default: 1)
- `limit` - элементов на странице (1-100, default: 20)

### Response metadata
```json
{
  "data": [...],
  "pagination": {
    "page": 1,           // текущая страница
    "limit": 20,         // элементов на странице
    "total": 45,         // всего элементов
    "totalPages": 3      // всего страниц (ceil(total / limit))
  }
}
```

---

## 💡 Best Practices реализованные в API

1. **Версионирование**: `/v1/` в URL для будущих breaking changes
2. **Пагинация**: обязательна для всех списковых эндпоинтов
3. **Фильтрация**: через query параметры (стандартно для REST)
4. **Валидация**: на уровне схемы (форматы, паттерны, длина, regex)
5. **Ошибки**: единообразный формат с машиночитаемыми кодами
6. **Безопасность**: JWT для всех запросов, readonly поля защищены
7. **REST семантика**: правильное использование HTTP методов
8. **Readonly поля**: `id`, `createdAt`, `updatedAt` нельзя изменить клиенту
9. **Location header**: при создании ресурса (201) указывает на новый ресурс
10. **Batch операции**: для массовых действий с partial success
11. **Nested resources**: `/coaches/{id}/athletes` для связанных данных
12. **Content negotiation**: Accept header для разных представлений
13. **Default representation**: `application/json` → базовое нормализованное представление

---

## 🎯 Типичные сценарии использования

### Сценарий 1: Dropdown список атлетов для select
```http
GET /athletes?limit=100
Accept: application/vnd.api.athlete.lookup+json
```
Вернет только `id`, `name` - минимум данных для выбора.

---

### Сценарий 2: Таблица атлетов с именем тренера
```http
GET /athletes?page=1&limit=20
Accept: application/vnd.api.athlete.list+json
```
Вернет денормализованные данные включая `coachName` - без дополнительных запросов на клиенте.

---

### Сценарий 3: Форма редактирования атлета
```http
GET /athletes/550e8400...
Accept: application/vnd.api.athlete.detail+json
```
Вернет данные без `coachId`, `createdAt`, `updatedAt` - чистый UI.

---

### Сценарий 4: Admin панель с базовыми данными
```http
GET /athletes/550e8400...
Accept: application/json
```
Вернет базовое нормализованное представление со всеми полями БД включая метаданные для аудита.

---

### Сценарий 5: Создать нового атлета
```http
POST /athletes
Content-Type: application/json

{
  "coachId": "660e8400...",
  "name": "Jane Smith",
  "email": "jane@example.com",
  "phone": "+1234567890",
  "telegram": "@jane"
}

→ 201 Created
Location: /v1/athletes/550e8400...
{
  "id": "550e8400...",  // generated
  "coachId": "660e8400...",
  "name": "Jane Smith",
  "email": "jane@example.com",
  "phone": "+1234567890",
  "telegram": "@jane",
  "createdAt": "2024-01-20T14:25:00Z",  // generated
  "updatedAt": "2024-01-20T14:25:00Z"   // generated
}
```

---

### Сценарий 6: Обновить только телефон
```http
PATCH /athletes/550e8400...
Content-Type: application/json

{
  "phone": "+9876543210"
}

→ 200 OK (возвращает полный Athlete с обновленным phone)
```

---

### Сценарий 7: Удалить нескольких атлетов
```http
DELETE /athletes/batch
Content-Type: application/json

{
  "ids": ["550e8400...", "550e8401...", "550e8402..."]
}

→ 200 OK
{
  "deleted": ["550e8400...", "550e8401..."],
  "failed": [
    {
      "id": "550e8402...",
      "reason": "Athlete has related records"
    }
  ]
}
```

---

## 🔧 Рекомендации по реализации

### Client-side (React + TypeScript)

#### Media Type константы

```typescript
// src/api/mediaTypes.ts
export const MediaTypes = {
  JSON: 'application/json', // базовое представление (default)
  ATHLETE_LIST: 'application/vnd.api.athlete.list+json',
  ATHLETE_LOOKUP: 'application/vnd.api.athlete.lookup+json',
  ATHLETE_DETAIL: 'application/vnd.api.athlete.detail+json',
  ATHLETE_SUMMARY: 'application/vnd.api.athlete.summary+json',
  
  COACH_LIST: 'application/vnd.api.coach.list+json',
  COACH_LOOKUP: 'application/vnd.api.coach.lookup+json',
  COACH_DETAIL: 'application/vnd.api.coach.detail+json',
} as const;

export type MediaType = typeof MediaTypes[keyof typeof MediaTypes];
```

#### API клиент с Accept header

```typescript
// src/api/client.ts
import { MediaTypes, MediaType } from './mediaTypes';

export class ApiClient {
  private baseUrl: string;
  private token: string | null = null;

  constructor(baseUrl: string) {
    this.baseUrl = baseUrl;
  }

  setToken(token: string) {
    this.token = token;
  }

  private async request<T>(
    url: string, 
    options: RequestInit = {},
    acceptType: MediaType = MediaTypes.JSON
  ): Promise<T> {
    const headers: HeadersInit = {
      'Accept': acceptType,
      'Content-Type': 'application/json',
      ...(this.token && { 'Authorization': `Bearer ${this.token}` }),
      ...options.headers,
    };

    const response = await fetch(`${this.baseUrl}${url}`, {
      ...options,
      headers,
    });

    if (!response.ok) {
      const error = await response.json();
      throw new ApiError(response.status, error);
    }

    return response.json();
  }

  // Примеры методов
  async getAthletes(params: { page?: number; limit?: number }, acceptType: MediaType = MediaTypes.ATHLETE_LIST) {
    const query = new URLSearchParams(params as any).toString();
    return this.request<PaginatedResponse<AthleteListItem>>(`/v1/athletes?${query}`, {}, acceptType);
  }

  async getAthletesForDropdown() {
    return this.request<PaginatedResponse<AthleteLookup>>('/v1/athletes?limit=100', {}, MediaTypes.ATHLETE_LOOKUP);
  }

  async getAthleteById(id: string, acceptType: MediaType = MediaTypes.JSON) {
    return this.request<Athlete>(`/v1/athletes/${id}`, {}, acceptType);
  }

  async createAthlete(data: AthleteCreate) {
    return this.request<Athlete>('/v1/athletes', {
      method: 'POST',
      body: JSON.stringify(data),
    });
  }
}

class ApiError extends Error {
  constructor(public status: number, public error: any) {
    super(error.error?.message || 'API Error');
  }
}
```

#### React Hook пример

```typescript
// src/hooks/useAthletes.ts
import { useQuery } from '@tanstack/react-query';
import { apiClient } from '@/api/client';
import { MediaTypes } from '@/api/mediaTypes';

export function useAthletesList(page: number = 1, limit: number = 20) {
  return useQuery({
    queryKey: ['athletes', 'list', page, limit],
    queryFn: () => apiClient.getAthletes({ page, limit }, MediaTypes.ATHLETE_LIST),
  });
}

export function useAthletesLookup() {
  return useQuery({
    queryKey: ['athletes', 'lookup'],
    queryFn: () => apiClient.getAthletesForDropdown(),
  });
}

export function useAthleteDetail(id: string) {
  return useQuery({
    queryKey: ['athletes', 'detail', id],
    queryFn: () => apiClient.getAthleteById(id, MediaTypes.ATHLETE_DETAIL),
  });
}
```

---

### Server-side (Spring Boot + Java)

#### Media Type константы

```java
// src/main/java/com/example/api/constants/MediaTypes.java
package com.example.api.constants;

import org.springframework.http.MediaType;

public final class MediaTypes {
    private MediaTypes() {}

    // Base
    public static final String APPLICATION_JSON_VALUE = MediaType.APPLICATION_JSON_VALUE;
    public static final MediaType APPLICATION_JSON = MediaType.APPLICATION_JSON;

    // Athletes
    public static final String ATHLETE_LIST_VALUE = "application/vnd.api.athlete.list+json";
    public static final MediaType ATHLETE_LIST = MediaType.parseMediaType(ATHLETE_LIST_VALUE);
    
    public static final String ATHLETE_LOOKUP_VALUE = "application/vnd.api.athlete.lookup+json";
    public static final MediaType ATHLETE_LOOKUP = MediaType.parseMediaType(ATHLETE_LOOKUP_VALUE);
    
    public static final String ATHLETE_DETAIL_VALUE = "application/vnd.api.athlete.detail+json";
    public static final MediaType ATHLETE_DETAIL = MediaType.parseMediaType(ATHLETE_DETAIL_VALUE);
    
    public static final String ATHLETE_SUMMARY_VALUE = "application/vnd.api.athlete.summary+json";
    public static final MediaType ATHLETE_SUMMARY = MediaType.parseMediaType(ATHLETE_SUMMARY_VALUE);

    // Coaches
    public static final String COACH_LIST_VALUE = "application/vnd.api.coach.list+json";
    public static final MediaType COACH_LIST = MediaType.parseMediaType(COACH_LIST_VALUE);
    
    public static final String COACH_LOOKUP_VALUE = "application/vnd.api.coach.lookup+json";
    public static final MediaType COACH_LOOKUP = MediaType.parseMediaType(COACH_LOOKUP_VALUE);
}
```

#### Content Negotiation Helper

```java
// src/main/java/com/example/api/util/ContentNegotiationHelper.java
package com.example.api.util;

import com.example.api.constants.MediaTypes;
import com.example.api.exception.NotAcceptableException;
import org.springframework.http.MediaType;

import java.util.List;

public class ContentNegotiationHelper {
    
    public enum RepresentationType {
        BASE, LIST, LOOKUP, DETAIL, SUMMARY
    }
    
    public static RepresentationType determineRepresentation(String acceptHeader) {
        if (acceptHeader == null || acceptHeader.isEmpty()) {
            return RepresentationType.BASE;
        }
        
        List<MediaType> mediaTypes = MediaType.parseMediaTypes(acceptHeader);
        
        for (MediaType mediaType : mediaTypes) {
            String value = mediaType.toString();
            
            if (value.contains(".list+json")) {
                return RepresentationType.LIST;
            }
            if (value.contains(".lookup+json")) {
                return RepresentationType.LOOKUP;
            }
            if (value.contains(".detail+json")) {
                return RepresentationType.DETAIL;
            }
            if (value.contains(".summary+json")) {
                return RepresentationType.SUMMARY;
            }
            if (value.contains("application/json")) {
                return RepresentationType.BASE;
            }
        }
        
        throw new NotAcceptableException("Unsupported Accept header: " + acceptHeader);
    }
}
```

#### Controller с Content Negotiation

```java
// src/main/java/com/example/api/controller/AthleteController.java
package com.example.api.controller;

import com.example.api.constants.MediaTypes;
import com.example.api.dto.*;
import com.example.api.service.AthleteService;
import com.example.api.util.ContentNegotiationHelper;
import com.example.api.util.ContentNegotiationHelper.RepresentationType;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.PageRequest;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import javax.validation.Valid;
import java.net.URI;

@RestController
@RequestMapping("/v1/athletes")
@RequiredArgsConstructor
public class AthleteController {
    
    private final AthleteService athleteService;
    
    // GET /athletes - Spring сам диспатчит по Accept header
    @GetMapping(produces = MediaTypes.APPLICATION_JSON_VALUE)
    public ResponseEntity<PaginatedResponse<Athlete>> getAthletesBase(
            @RequestParam(defaultValue = "1") int page,
            @RequestParam(defaultValue = "20") int limit,
            @RequestParam(required = false) String coachId,
            @RequestParam(required = false) String search
    ) {
        PageRequest pageRequest = PageRequest.of(page - 1, limit);
        return ResponseEntity.ok(athleteService.getAthletes(pageRequest, coachId, search));
    }
    
    @GetMapping(produces = MediaTypes.ATHLETE_LIST_VALUE)
    public ResponseEntity<PaginatedResponse<AthleteListItem>> getAthletesList(
            @RequestParam(defaultValue = "1") int page,
            @RequestParam(defaultValue = "20") int limit,
            @RequestParam(required = false) String coachId,
            @RequestParam(required = false) String search
    ) {
        PageRequest pageRequest = PageRequest.of(page - 1, limit);
        return ResponseEntity.ok(athleteService.getAthletesList(pageRequest, coachId, search));
    }
    
    @GetMapping(produces = MediaTypes.ATHLETE_LOOKUP_VALUE)
    public ResponseEntity<PaginatedResponse<AthleteLookup>> getAthletesLookup(
            @RequestParam(defaultValue = "1") int page,
            @RequestParam(defaultValue = "20") int limit,
            @RequestParam(required = false) String search
    ) {
        PageRequest pageRequest = PageRequest.of(page - 1, limit);
        return ResponseEntity.ok(athleteService.getAthletesLookup(pageRequest, search));
    }
    
    // GET /athletes/{id} - Spring сам диспатчит по Accept header
    @GetMapping(value = "/{id}", produces = MediaTypes.APPLICATION_JSON_VALUE)
    public ResponseEntity<Athlete> getAthleteBase(@PathVariable String id) {
        return ResponseEntity.ok(athleteService.getAthleteById(id));
    }
    
    @GetMapping(value = "/{id}", produces = MediaTypes.ATHLETE_DETAIL_VALUE)
    public ResponseEntity<AthleteDetail> getAthleteDetail(@PathVariable String id) {
        return ResponseEntity.ok(athleteService.getAthleteDetail(id));
    }
    
    @PostMapping(consumes = MediaTypes.APPLICATION_JSON_VALUE, produces = MediaTypes.APPLICATION_JSON_VALUE)
    public ResponseEntity<Athlete> createAthlete(@Valid @RequestBody AthleteCreate dto) {
        Athlete created = athleteService.createAthlete(dto);
        return ResponseEntity
                .created(URI.create("/v1/athletes/" + created.getId()))
                .body(created);
    }
    
    @PutMapping(value = "/{id}", consumes = MediaTypes.APPLICATION_JSON_VALUE, produces = MediaTypes.APPLICATION_JSON_VALUE)
    public ResponseEntity<Athlete> updateAthlete(
            @PathVariable String id,
            @Valid @RequestBody AthleteUpdate dto
    ) {
        Athlete updated = athleteService.updateAthlete(id, dto);
        return ResponseEntity.ok(updated);
    }
    
    @PatchMapping(value = "/{id}", consumes = MediaTypes.APPLICATION_JSON_VALUE, produces = MediaTypes.APPLICATION_JSON_VALUE)
    public ResponseEntity<Athlete> patchAthlete(
            @PathVariable String id,
            @Valid @RequestBody AthletePatch dto
    ) {
        Athlete patched = athleteService.patchAthlete(id, dto);
        return ResponseEntity.ok(patched);
    }
    
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteAthlete(@PathVariable String id) {
        athleteService.deleteAthlete(id);
    }
}
```

#### Exception Handler

```java
// src/main/java/com/example/api/exception/NotAcceptableException.java
package com.example.api.exception;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ResponseStatus;

@ResponseStatus(HttpStatus.NOT_ACCEPTABLE)
public class NotAcceptableException extends RuntimeException {
    public NotAcceptableException(String message) {
        super(message);
    }
}
```

#### Global Exception Handler

```java
// src/main/java/com/example/api/exception/GlobalExceptionHandler.java
package com.example.api.exception;

import com.example.api.constants.MediaTypes;
import com.example.api.dto.ErrorResponse;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.time.Instant;
import java.util.List;

@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(NotAcceptableException.class)
    public ResponseEntity<ErrorResponse> handleNotAcceptable(NotAcceptableException ex) {
        ErrorResponse error = ErrorResponse.builder()
                .code("NOT_ACCEPTABLE")
                .message(ex.getMessage())
                .timestamp(Instant.now().toString())
                .details(new ErrorResponse.ErrorDetails(
                    "supported", 
                    List.of(
                        MediaTypes.APPLICATION_JSON_VALUE,
                        MediaTypes.ATHLETE_LIST_VALUE,
                        MediaTypes.ATHLETE_LOOKUP_VALUE,
                        MediaTypes.ATHLETE_DETAIL_VALUE,
                        MediaTypes.ATHLETE_SUMMARY_VALUE
                    )
                ))
                .build();
        
        return ResponseEntity.status(HttpStatus.NOT_ACCEPTABLE).body(error);
    }
}
```

---

## 📚 Будущие расширения

### 1. Embedded Relationships (expand parameter)
```http
GET /athletes/550e8400...?expand=coach
Accept: application/vnd.api.athlete.detail+json

Response:
{
  "id": "550e8400...",
  "name": "John Doe",
  "email": "john@example.com",
  "phone": "+1234567890",
  "telegram": "@johndoe",
  "coach": {  // embedded
    "id": "...",
    "name": "Coach Smith",
    "email": "coach@example.com"
  }
}
```

### 2. Sparse Fieldsets (дополнительная гибкость)
```http
GET /athletes?fields=id,name,email,phone
```
Для клиентов, которым нужен custom набор полей.

### 3. HATEOAS Links
```json
{
  "id": "550e8400...",
  "name": "John Doe",
  "_links": {
    "self": "/v1/athletes/550e8400...",
    "coach": "/v1/coaches/660e8400..."
  }
}
```

---

## ✅ Итоговый чеклист решений

- ✅ Content negotiation через **Accept header**
- ✅ Media Type: `application/json` (базовое) и `application/vnd.api.{resource}.{representation}+json` (специальные)
- ✅ Пять представлений: **базовое** (application/json), **list**, **lookup**, **detail**, **summary**
- ✅ Восемь схем: **Athlete**, **AthleteListItem**, **AthleteLookup**, **AthleteDetail**, **AthleteSummary**, **AthleteCreate**, **AthleteUpdate**, **AthletePatch**
- ✅ Naming convention: `{Resource}{Operation}` (industry standard)
- ✅ REST семантика: правильное использование GET/POST/PUT/PATCH/DELETE
- ✅ Пагинация с metadata для всех списков
- ✅ Фильтрация и сортировка
- ✅ Единообразные ошибки с кодами
- ✅ JWT аутентификация на всех эндпоинтах
- ✅ Batch операции с partial success
- ✅ Nested resources для связанных данных
- ✅ Location header при создании ресурса
- ✅ Readonly поля защищены от изменения
- ✅ Версионирование API (/v1/)

---

**Это профессиональное REST API, готовое к production deployment! 🚀**

---

## 📋 Расширение на другие ресурсы

Данная спецификация демонстрирует подход на примере модуля **Athletes**. Для полной системы спортивного управления необходимо реализовать аналогичные эндпоинты для:

### Основные ресурсы:
- **Coaches** (`/coaches`, `/coaches/{id}`, etc.)
- **Users** (`/users`, `/users/{id}`, etc.)
- **Gyms** (`/gyms`, `/gyms/{id}`, etc.)

### Nested resources:
- `/coaches/{coachId}/athletes` ✅ (уже реализовано)
- `/gyms/{gymId}/coaches`

### Применение паттернов:
Каждый новый ресурс должен следовать тем же принципам:
- ✅ Content negotiation через Accept header
- ✅ Пять представлений: базовое (application/json), list, lookup, detail, summary
- ✅ Восемь схем: {Resource}, {Resource}ListItem, {Resource}Lookup, {Resource}Detail, {Resource}Summary, {Resource}Create, {Resource}Update, {Resource}Patch
- ✅ Стандартные HTTP методы и коды ответов
- ✅ Пагинация, фильтрация, сортировка
- ✅ Единообразные ошибки
- ✅ JWT аутентификация

