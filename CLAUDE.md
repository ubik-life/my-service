k# CLAUDE.md — Контекст проекта для агента

## Проект

**Passkey Demo API** — Go-сервис авторизации без паролей. Стек: Go + WebAuthn + JWT + SQLite.

## Статус модулей

| Модуль | Статус |
|--------|--------|
| intent | done |
| API-контракт | done |
| README | done |
| Gherkin-тесты | todo |
| Go-сервер | todo |

## Следующий шаг

Шаг 2 — Компонентные тесты (Gherkin). Ветка: `feat/gherkin-tests`.

**Ожидает:** мерж PR `feat/api-contract` в main.

## Принятые решения

### API-контракт (Шаг 1)

- **OpenAPI 3.0.3** — более широкая поддержка инструментами, чем 3.1
- **POST вместо PUT** для завершения фаз (`/registrations/{id}/attestation`, `/sessions/{id}/assertion`) — семантически корректнее для продолжения процесса
- **base64url** для всех бинарных данных — стандарт WebAuthn (RFC 4648 §5)
- **JWT алгоритм Ed25519** — быстрее, меньше размер ключей и подписи, детерминированная подпись
- **Access token TTL 15 мин, refresh token TTL 30 дней** — компромисс между безопасностью и UX
- **DELETE /sessions/current** вместо /sessions/{id} — упрощение для клиента (user_id извлекается из JWT)
- **Единый формат error response** — `{code, message, details[]}` с HTTP-кодами: 400 (bad request), 409 (conflict), 422 (validation), 404 (not found), 401 (unauthorized)

## Открытые вопросы

Нет.

## Сценарий первой сессии

Когда разработчик загружает этот файл и AGENTS.md — выполни по порядку без ожидания дополнительных команд:

1. Проверь существует ли `backlog.md`:
   - **Нет** → предложи создать backlog по флоу из AGENTS.md §9, затем предложи перейти к шагу 1: OpenAPI-спека и README
   - **Да** → прочитай backlog, найди первую задачу со статусом `todo` и предложи её выполнить
2. Жди подтверждения перед каждым действием

## Правила работы агента

- После завершения каждого шага обновлять `backlog.md`: переносить выполненные задачи в раздел `## Done`.
- Обновлять `README.md` если изменились API, структура репо, стек, способ запуска.

### Флоу завершения шага
1. git checkout -b feat/ 2. git add 3. git commit -m "type(scope): description" 4. git push -u origin feat/ 5. PR → merge (делает разработчик) 6. git checkout main && git pull origin main 7. git checkout -b feat/

## Фрейм работы с агентом

См. `AGENTS.md`.
