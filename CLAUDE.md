# CLAUDE.md — Контекст проекта для агента

## Проект

<!-- TODO: одна строка о сервисе -->
**Название** — краткое описание. Стек: ...

## Статус модулей

| Модуль | Статус |
|--------|--------|
| intent | done |
| API-контракт | todo |
| README | todo |
| Gherkin-тесты | todo |
| Go-сервер | todo |

## Следующий шаг

Шаг 1 — OpenAPI-спека и README. Ветка: `feat/api-contract`.

## Принятые решения

<!-- TODO: заполни по ходу разработки -->
- ...

## Открытые вопросы

<!-- TODO: блокеры и вопросы -->
- ...

## Правила работы агента

- После завершения каждого шага обновлять `backlog.md`: переносить выполненные задачи в раздел `## Done`.
- Обновлять `README.md` если изменились API, структура репо, стек, способ запуска.

### Флоу завершения шага

```
1. git checkout -b feat/<name>
2. git add <files>
3. git commit -m "type(scope): description"
4. git push -u origin feat/<name>
5. PR → merge (делает разработчик)
6. git checkout main && git pull origin main
7. git checkout -b feat/<next-name>
```

## Фрейм работы с агентом

См. `AGENTS.md`.
