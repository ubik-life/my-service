# Passkey Demo API

> Go-сервис авторизации без паролей через WebAuthn. Демонстрирует полный цикл регистрации и входа с использованием биометрии.

**Стек:** Go + WebAuthn + JWT (Ed25519) + SQLite

---

## Постановка задачи

### Что решаем

Современная веб-авторизация основана на паролях — слабом звене безопасности. Пользователи выбирают простые пароли, переиспользуют их между сайтами, теряют при утечках баз данных.

**WebAuthn** — стандарт W3C, позволяющий аутентифицировать пользователей через криптографические ключи, хранящиеся в аутентификаторе (биометрия, USB-ключ, platform authenticator).

### Зачем

Этот сервис — демонстрация полного цикла авторизации без паролей. Первый кирпич ноды **Ubik** — открытой децентрализованной платформы для распространения знаний.

### Критерии успеха

1. Пользователь регистрируется по handle + биометрия (без пароля)
2. Пользователь входит по handle + биометрия
3. Сервис возвращает JWT для доступа к защищённым ресурсам
4. Сессия инвалидируется при выходе
5. Все сценарии покрыты Gherkin-тестами (component-tests)
6. Код следует принципам формальной корректности (предусловия, постусловия, инварианты)

---

## Архитектура

### Компоненты

```
┌──────────────┐
│   Browser    │ ← WebAuthn API (navigator.credentials)
└──────┬───────┘
       │ HTTPS (JSON + base64url)
       ↓
┌──────────────────────────────────────────────────┐
│           Passkey Demo API (Go)                  │
│                                                  │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────┐ │
│  │   HTTP      │→ │  WebAuthn    │→ │   JWT   │ │
│  │  Handlers   │  │  Validator   │  │  Issuer │ │
│  └─────────────┘  └──────────────┘  └─────────┘ │
│         ↓                  ↓              ↓      │
│  ┌──────────────────────────────────────────┐   │
│  │          SQLite Database                 │   │
│  │  users | credentials | sessions          │   │
│  └──────────────────────────────────────────┘   │
└──────────────────────────────────────────────────┘
```

### Границы сервиса

**Вход:**
- REST API (JSON): регистрация, вход, выход, получение данных пользователя
- Все бинарные данные (challenge, публичные ключи, подписи) передаются в **base64url**

**Выход:**
- JWT access token (Ed25519, TTL 15 мин)
- Opaque refresh token (TTL 30 дней)

**Зависимости:**
- Нет внешних сервисов
- SQLite для хранения: users, credentials, sessions

### Взаимодействие с WebAuthn

Сервис реализует серверную часть протокола WebAuthn:
1. Генерирует challenge (случайные байты)
2. Проверяет attestation при регистрации (извлекает публичный ключ из COSE)
3. Проверяет assertion при входе (проверяет подпись, счётчик, origin)

---

## Описание работы программы

### Сценарий 1: Регистрация

**Фаза 1** — создание challenge

Вход:
```json
POST /v1/registrations
{ "handle": "alice" }
```

Выход (201 Created):
```json
{
  "id": "uuid",
  "options": {
    "challenge": "base64url",
    "rp": { "name": "Passkey Demo", "id": "localhost" },
    "user": { "id": "base64url", "name": "alice", "displayName": "alice" },
    "pubKeyCredParams": [{ "type": "public-key", "alg": -7 }],
    "timeout": 60000,
    "attestation": "none"
  }
}
```

Сервер:
1. Валидирует handle (только `[a-z0-9_-]`, длина 3-32)
2. Проверяет уникальность handle
3. Генерирует случайный challenge (32 байта)
4. Сохраняет сессию регистрации (challenge, handle, expires_at)
5. Возвращает options для `navigator.credentials.create()`

**Фаза 2** — завершение регистрации

Браузер вызывает `navigator.credentials.create(options)`, аутентификатор создаёт ключевую пару.

Вход:
```json
POST /v1/registrations/{id}/attestation
{
  "id": "base64url",
  "rawId": "base64url",
  "type": "public-key",
  "response": {
    "clientDataJSON": "base64url",
    "attestationObject": "base64url"
  }
}
```

Выход (200 OK):
```json
{
  "access_token": "JWT...",
  "refresh_token": "opaque..."
}
```

Сервер:
1. Находит сессию по `id`, проверяет, что не истекла
2. Декодирует `clientDataJSON`, проверяет `type`, `challenge`, `origin`
3. Декодирует `attestationObject` (CBOR), извлекает публичный ключ (COSE)
4. Сохраняет user, credential (credential_id, public_key, counter)
5. Генерирует access_token (JWT, Ed25519, claims: sub, handle, exp)
6. Генерирует refresh_token (случайная строка), сохраняет в БД
7. Удаляет сессию регистрации
8. Возвращает TokenPair

### Сценарий 2: Вход

**Фаза 1** — создание challenge

Вход:
```json
POST /v1/sessions
{ "handle": "alice" }
```

Выход (201 Created):
```json
{
  "id": "uuid",
  "options": {
    "challenge": "base64url",
    "rpId": "localhost",
    "allowCredentials": [{ "type": "public-key", "id": "base64url" }],
    "userVerification": "preferred",
    "timeout": 60000
  }
}
```

Сервер:
1. Находит пользователя по handle
2. Загружает все credentials пользователя
3. Генерирует случайный challenge (32 байта)
4. Сохраняет сессию входа (challenge, user_id, expires_at)
5. Возвращает options для `navigator.credentials.get()`

**Фаза 2** — завершение входа

Браузер вызывает `navigator.credentials.get(options)`, аутентификатор подписывает данные.

Вход:
```json
POST /v1/sessions/{id}/assertion
{
  "id": "base64url",
  "rawId": "base64url",
  "type": "public-key",
  "response": {
    "clientDataJSON": "base64url",
    "authenticatorData": "base64url",
    "signature": "base64url",
    "userHandle": "base64url"
  }
}
```

Выход (200 OK):
```json
{
  "access_token": "JWT...",
  "refresh_token": "opaque..."
}
```

Сервер:
1. Находит сессию по `id`, проверяет, что не истекла
2. Находит credential по `rawId`
3. Декодирует `clientDataJSON`, проверяет `type`, `challenge`, `origin`
4. Декодирует `authenticatorData`, извлекает rpIdHash, flags, counter
5. Проверяет подпись: `verify(public_key, signature, authenticatorData + SHA256(clientDataJSON))`
6. Проверяет, что counter вырос (защита от клонирования)
7. Обновляет counter в БД
8. Генерирует access_token и refresh_token
9. Удаляет сессию входа
10. Возвращает TokenPair

### Сценарий 3: Выход

Вход:
```
DELETE /v1/sessions/current
Authorization: Bearer <access_token>
```

Выход (204 No Content)

Сервер:
1. Проверяет JWT access_token (подпись, exp)
2. Извлекает user_id из claims
3. Инвалидирует все refresh_tokens пользователя
4. Возвращает 204

Примечание: access_token продолжит работать до истечения TTL (15 мин).

### Сценарий 4: Получение данных пользователя

Вход:
```
GET /v1/users/me
Authorization: Bearer <access_token>
```

Выход (200 OK):
```json
{
  "id": "uuid",
  "handle": "alice"
}
```

Сервер:
1. Проверяет JWT access_token (подпись, exp)
2. Извлекает user_id из claims
3. Загружает user из БД
4. Возвращает User

---

## Схема иерархий модулей

```
main
├── cmd/server
│   └── main.go                    ← entry point, инициализация
│
├── internal/app
│   ├── app.go                     ← сборка зависимостей, graceful shutdown
│   └── config.go                  ← конфигурация из env
│
├── internal/handler
│   ├── registration.go            ← POST /registrations, POST /registrations/{id}/attestation
│   ├── session.go                 ← POST /sessions, POST /sessions/{id}/assertion, DELETE /sessions/current
│   ├── user.go                    ← GET /users/me
│   ├── middleware.go              ← auth middleware (JWT validation)
│   └── error.go                   ← единый формат error response
│
├── internal/service
│   ├── registration.go            ← бизнес-логика регистрации
│   ├── session.go                 ← бизнес-логика входа/выхода
│   └── user.go                    ← бизнес-логика пользователя
│
├── internal/webauthn
│   ├── validator.go               ← проверка attestation/assertion
│   ├── challenge.go               ← генерация challenge
│   └── cose.go                    ← декодирование COSE public key
│
├── internal/jwt
│   ├── issuer.go                  ← генерация JWT (Ed25519)
│   └── validator.go               ← проверка JWT
│
├── internal/repository
│   ├── user.go                    ← CRUD для users
│   ├── credential.go              ← CRUD для credentials
│   └── session.go                 ← CRUD для registration/login sessions
│
└── internal/model
    ├── user.go                    ← domain модель User
    ├── credential.go              ← domain модель Credential
    └── session.go                 ← domain модель Session
```

---

## Блок-схема алгоритма головного модуля

**Entry point:** `cmd/server/main.go`

```
START
  ↓
Загрузить конфигурацию из env
  ↓
Инициализировать SQLite (migrations)
  ↓
Создать repositories (user, credential, session)
  ↓
Создать services (registration, session, user)
  ↓
Создать HTTP handlers
  ↓
Зарегистрировать routes:
  - POST /v1/registrations
  - POST /v1/registrations/{id}/attestation
  - POST /v1/sessions
  - POST /v1/sessions/{id}/assertion
  - DELETE /v1/sessions/current
  - GET /v1/users/me
  ↓
Запустить HTTP server (порт из env)
  ↓
Ожидать сигнал (SIGINT/SIGTERM)
  ↓
Graceful shutdown:
  - Закрыть HTTP server (ждать завершения запросов, timeout 30s)
  - Закрыть DB connection
  ↓
END
```

---

## API

См. полную спецификацию: [api-specification/openapi.yaml](api-specification/openapi.yaml)

| Метод | Ресурс | Описание |
|-------|--------|----------|
| POST | `/v1/registrations` | Создать challenge для регистрации |
| POST | `/v1/registrations/{id}/attestation` | Завершить регистрацию → JWT |
| POST | `/v1/sessions` | Создать challenge для входа |
| POST | `/v1/sessions/{id}/assertion` | Завершить вход → JWT |
| DELETE | `/v1/sessions/current` | Выход (инвалидация refresh token) |
| GET | `/v1/users/me` | Получить данные текущего пользователя |

---

## Структура репозитория

```
api-specification/   ← OpenAPI 3.0 спека
cmd/server/          ← entry point
component-tests/     ← Gherkin-сценарии (black-box тесты)
devlog/              ← Журнал разработки (промпты, решения, альтернативы)
internal/            ← Go-код (app, handler, service, repository, model, webauthn, jwt)
migrations/          ← Миграции БД (goose)
```

---

## Devlog

| Файл | Содержание |
|------|-----------|
| [00-intent.md](devlog/00-intent.md) | Намерение, WebAuthn-флоу, архитектурные решения |
| [01-api-contract.md](devlog/01-api-contract.md) | OpenAPI спека и README |

---

## Запуск (будет добавлено на шаге 3)

```bash
# TODO: заполнить после реализации Go-сервера
```

---

## Тестирование (будет добавлено на шаге 2-3)

```bash
# TODO: заполнить после написания Gherkin-тестов
```
