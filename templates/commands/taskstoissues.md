---
description: Преобразовать существующие задачи в actionable GitHub Issues с упорядоченными зависимостями для фичи на основе доступных проектных артефактов.
tools: ['github/github-mcp-server/issue_write']
scripts:
  sh: scripts/bash/check-prerequisites.sh --json --require-tasks --include-tasks
  ps: scripts/powershell/check-prerequisites.ps1 -Json -RequireTasks -IncludeTasks
---

## Ввод пользователя

```text
$ARGUMENTS
```

Вы **ОБЯЗАНЫ** учесть ввод пользователя перед продолжением (если он не пустой).

## Краткое содержание

1. Запустить `{SCRIPT}` из корня репозитория и разобрать FEATURE_DIR и список AVAILABLE_DOCS. Все пути должны быть абсолютными. Для одинарных кавычек в аргументах, например "I'm Groot", использовать синтаксис экранирования: 'I'\''m Groot' (или двойные кавычки, если возможно: "I'm Groot").
1. Из выполненного скрипта извлечь путь к **tasks**.
1. Получить удалённый Git, выполнив:

```bash
git config --get remote.origin.url
```

> [!CAUTION]
> ПРОДОЛЖАТЬ СЛЕДУЮЩИЕ ШАГИ ТОЛЬКО ЕСЛИ УДАЛЁННЫЙ — GITHUB URL

1. Для каждой задачи в списке использовать GitHub MCP сервер для создания нового issue в репозитории, соответствующем удалённому Git.

> [!CAUTION]
> НИ ПРИ КАКИХ ОБСТОЯТЕЛЬСТВАХ НЕ СОЗДАВАТЬ ISSUES В РЕПОЗИТОРИЯХ, НЕ СООТВЕТСТВУЮЩИХ URL УДАЛЁННОГО
