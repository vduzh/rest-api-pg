# Sports Management System REST API - Проектное решение

## Обзор

Профессиональное проектирование REST API для спортивной системы управления с использованием content negotiation через Accept headers и следованием industry-standard best practices.

**Примечание**: Данная спецификация демонстрирует подход на примере модуля Athletes, но принципы и паттерны применимы ко всем ресурсам системы (Coaches, Training Sessions, Subscriptions и т.д.).

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
application/vnd.api.{resource}.full+json      # полное представление (default)
application/vnd.api.{resource}.detail+json    # расширенное без readonly полей
application/vnd.api.{resource}.summary+json   # минимальное представление
application/json                              # fallback → full
```

### Примеры для разных ресурсов:
```
application/vnd.api.athlete.full+json         # Athletes
application/vnd.api.coach.detail+json         # Coaches
application/vnd.api.training-session.summary+json  # Training Sessions
application/vnd.api.subscription.full+json    # Subscriptions
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

#### 1. `{Resource}` - Полное представление
**Media Type**: `application/vnd.api.{resource}.full+json`

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
- Admin панели
- Debugging и auditing
- Data export
- Когда нужна полная информация включая связи и метаданные

---

#### 2. `{Resource}Detail` - Расширенное без системных полей
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
  // Future: subscription?: Subscription; (когда появится FK)
}
```

**Use cases**:
- Карточки профилей
- Формы редактирования
- User-facing интерфейсы, где системные поля не нужны
- Будущее расширение: embedded relationships

---

#### 3. `{Resource}Summary` - Минимальное представление
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
- Dropdown списки и select boxes
- Autocomplete поля
- Превью результатов поиска
- Списки с большим количеством элементов (экономия трафика)

---

### Request Models (что отправляет клиент)

#### 4. `{Resource}Create` - Создание ресурса (POST)

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

#### 5. `{Resource}Update` - Полная замена (PUT)

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

#### 6. `{Resource}Patch` - Частичное обновление (PATCH)

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
| `{Resource}` | GET | Полное чтение | ✅ Да |
| `{Resource}Summary` | GET | Краткое чтение | ✅ Да |
| `{Resource}Detail` | GET | Расширенное чтение | ✅ Да |
| `{Resource}Create` | POST | Создание | ✅ Да (кроме optional) |
| `{Resource}Update` | PUT | Полная замена | ✅ Да |
| `{Resource}Patch` | PATCH | Частичное обновление | ❌ Все optional |

**Примеры для разных ресурсов**:
- `Athlete`, `AthleteSummary`, `AthleteDetail`, `AthleteCreate`, `AthleteUpdate`, `AthletePatch`
- `Coach`, `CoachSummary`, `CoachDetail`, `CoachCreate`, `CoachUpdate`, `CoachPatch`
- `TrainingSession`, `TrainingSessionSummary`, `TrainingSessionDetail`, `TrainingSessionCreate`, `TrainingSessionUpdate`, `TrainingSessionPatch`

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
- Accept header поддержка всех представлений
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
  - `full` → `Athlete` (default)
  - `detail` → `AthleteDetail`
  - `summary` → `AthleteSummary`
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
  - 409 (имеет зависимости, например, активные тренировки)

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
13. **Fallback**: `application/json` → full representation для совместимости

---

## 🎯 Типичные сценарии использования

### Сценарий 1: Dropdown список атлетов
```http
GET /athletes?view=summary&limit=50
Accept: application/vnd.api.athlete.summary+json
```
Вернет только `id`, `name`, `email` - экономия трафика.

---

### Сценарий 2: Форма редактирования атлета
```http
GET /athletes/550e8400...
Accept: application/vnd.api.athlete.detail+json
```
Вернет данные без `coachId`, `createdAt`, `updatedAt` - чистый UI.

---

### Сценарий 3: Admin панель с полными данными
```http
GET /athletes/550e8400...
Accept: application/vnd.api.athlete.full+json
```
Вернет все поля включая метаданные для аудита.

---

### Сценарий 4: Создать нового атлета
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

### Сценарий 5: Обновить только телефон
```http
PATCH /athletes/550e8400...
Content-Type: application/json

{
  "phone": "+9876543210"
}

→ 200 OK (возвращает полный Athlete с обновленным phone)
```

---

### Сценарий 6: Удалить нескольких атлетов
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
      "reason": "Athlete has active training sessions"
    }
  ]
}
```

---

## 🔧 Рекомендации по реализации

### Server-side parsing Accept header

```typescript
function determineRepresentation(acceptHeader: string): 'full' | 'detail' | 'summary' {
  if (acceptHeader.includes('athlete.summary+json')) return 'summary';
  if (acceptHeader.includes('athlete.detail+json')) return 'detail';
  if (acceptHeader.includes('athlete.full+json')) return 'full';
  if (acceptHeader.includes('application/json')) return 'full'; // fallback
  
  throw new NotAcceptableError(406, 'Unsupported Accept header');
}
```

### Media Type константы

```typescript
export const MediaTypes = {
  ATHLETE_FULL: 'application/vnd.api.athlete.full+json',
  ATHLETE_DETAIL: 'application/vnd.api.athlete.detail+json',
  ATHLETE_SUMMARY: 'application/vnd.api.athlete.summary+json',
  JSON: 'application/json', // fallback
} as const;
```

### Middleware для Content Negotiation

```typescript
app.use((req, res, next) => {
  try {
    const accept = req.headers.accept || 'application/json';
    req.representation = determineRepresentation(accept);
    next();
  } catch (error) {
    res.status(406).json({
      error: {
        code: 'NOT_ACCEPTABLE',
        message: 'Unsupported media type in Accept header',
        supported: [
          MediaTypes.ATHLETE_FULL,
          MediaTypes.ATHLETE_DETAIL,
          MediaTypes.ATHLETE_SUMMARY,
          MediaTypes.JSON
        ]
      }
    });
  }
});
```

---

## 📚 Будущие расширения

### 1. Embedded Relationships (expand parameter)
```http
GET /athletes/550e8400...?expand=subscription
Accept: application/vnd.api.athlete.detail+json

Response:
{
  "id": "550e8400...",
  "name": "John Doe",
  "email": "john@example.com",
  "phone": "+1234567890",
  "telegram": "@johndoe",
  "subscription": {  // embedded
    "id": "...",
    "plan": "Premium",
    "status": "active"
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
    "coach": "/v1/coaches/660e8400...",
    "training_sessions": "/v1/athletes/550e8400.../sessions"
  }
}
```

---

## ✅ Итоговый чеклист решений

- ✅ Content negotiation через **Accept header**
- ✅ Media Type: `application/vnd.api.{resource}.{representation}+json`
- ✅ Три представления: **full**, **detail**, **summary**
- ✅ Шесть схем: **Athlete**, **AthleteSummary**, **AthleteDetail**, **AthleteCreate**, **AthleteUpdate**, **AthletePatch**
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
- **Training Sessions** (`/training-sessions`, `/training-sessions/{id}`, etc.)
- **Subscriptions** (`/subscriptions`, `/subscriptions/{id}`, etc.)
- **Users** (`/users`, `/users/{id}`, etc.)
- **Gyms** (`/gyms`, `/gyms/{id}`, etc.)

### Nested resources:
- `/coaches/{coachId}/athletes` ✅ (уже реализовано)
- `/athletes/{athleteId}/training-sessions`
- `/athletes/{athleteId}/subscriptions`
- `/gyms/{gymId}/coaches`
- `/coaches/{coachId}/training-sessions`

### Применение паттернов:
Каждый новый ресурс должен следовать тем же принципам:
- ✅ Content negotiation через Accept header
- ✅ Три представления: full, detail, summary
- ✅ Шесть схем: {Resource}, {Resource}Summary, {Resource}Detail, {Resource}Create, {Resource}Update, {Resource}Patch
- ✅ Стандартные HTTP методы и коды ответов
- ✅ Пагинация, фильтрация, сортировка
- ✅ Единообразные ошибки
- ✅ JWT аутентификация

