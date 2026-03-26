---
name: speckit-init
description: Установить spec-kit (specify-cli) из репозитория GitHub и инициализировать новый проект. Использовать, когда пользователь хочет настроить spec-kit в проекте, установить specify CLI или выполнить specify init.
---

Вы помогаете пользователю установить и инициализировать spec-kit (specify-cli) в его проекте.

## Что делает этот скилл

1. Проверяет наличие `uv` (требуется для установки specify-cli)
2. Устанавливает `specify-cli` из `https://github.com/github/spec-kit.git`
3. Запускает `specify init` в текущей директории проекта с нужными флагами AI и скрипта

## Пошаговый процесс

### Шаг 1 — Проверка зависимостей

Выполните следующую команду для проверки наличия `uv`:

```bash
uv --version
```

Если `uv` не найден, сообщите пользователю, что его нужно установить:
- **macOS/Linux**: `curl -LsSf https://astral.sh/uv/install.sh | sh`
- **Windows**: `powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"`

Не продолжайте до тех пор, пока `uv` не будет доступен.

### Шаг 2 — Установка specify-cli

Выполните эту команду для установки последней версии из main:

```bash
uv tool install specify-cli --from git+https://github.com/github/spec-kit.git
```

Если пользователь хочет конкретную версию (например, передал тег версии как аргумент):

```bash
uv tool install specify-cli --from git+https://github.com/github/spec-kit.git@<VERSION>
```

Если specify-cli уже установлен и пользователь хочет обновить его, добавьте `--force`:

```bash
uv tool install specify-cli --force --from git+https://github.com/github/spec-kit.git
```

### Шаг 3 — Проверка установки

```bash
specify check
```

Если `specify` не найден в PATH после установки, пользователю может потребоваться выполнить `uv tool update-shell` и перезапустить терминал.

### Шаг 4 — Инициализация проекта

**Определение флага AI:**
- Если текущая среда — **Claude Code**: использовать `--ai claude`
- Если пользователь указал другой AI (например, передал как аргумент): использовать это значение
- По умолчанию использовать `--ai claude` при запуске внутри Claude Code

**Определение флага `--script`:**

Опция `--script` определяет, какой вариант оболочки используется для генерируемых скриптов:
- `--script sh` — скрипты bash/zsh (по умолчанию, если не указано)
- `--script ps` — скрипты PowerShell

Использовать `--script ps` только если пользователь явно запрашивает PowerShell. В остальных случаях всегда использовать `--script sh` (bash).

Запустить в текущей рабочей директории:

```bash
specify init . --ai claude --script sh --offline
```

Или при инициализации нового именованного проекта (когда пользователь передал имя проекта как аргумент):

```bash
specify init <PROJECT_NAME> --ai claude --script sh --offline
```

Пример с PowerShell (только когда пользователь явно запрашивает):

```bash
specify init . --ai claude --script ps --offline
```

> **Важно**: Флаг `--offline` обязателен. Без него `specify init` скачивает шаблоны из последнего GitHub Release, который может не содержать актуальные (русифицированные) файлы. С `--offline` используются шаблоны, встроенные в установленный пакет.

### Шаг 5 — Установка расширений

После инициализации установите расширения spec-kit:

```bash
specify extension add fleet
specify extension add learn
```

### Шаг 6 — Установка конституции

Скачайте `constitution.md` из репозитория `kimsanbaev-karim/spec-kit` и замените файл конституции в проекте:

```bash
curl -fsSL https://raw.githubusercontent.com/kimsanbaev-karim/spec-kit/main/constitution.md -o .specify/memory/constitution.md
```

На Windows (PowerShell), если `curl` недоступен:

```powershell
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/kimsanbaev-karim/spec-kit/main/constitution.md" -OutFile ".specify/memory/constitution.md"
```

Если команда завершилась с ошибкой (нет сети, 404 и т.д.) — предупредить пользователя, но не прерывать установку. Конституцию можно заполнить позже через `/speckit.constitution`.

### Шаг 7 — Подтверждение успеха

После инициализации сообщите пользователю:
- Что было установлено и куда (specify-cli, spec-kit-fleet, spec-kit-learn)
- Что конституция скопирована в `.specify/memory/constitution.md`
- Как начать работу: `specify check` для проверки зависимостей
- Где найти документацию: `specify --help` или https://github.com/github/spec-kit

## Обработка аргументов

Скилл может быть вызван с необязательными аргументами:

- `/speckit-init` — установить последнюю версию, инициализировать в текущей директории с `--ai claude --script sh`
- `/speckit-init v0.4.2` — установить конкретный тег версии
- `/speckit-init --upgrade` — принудительная переустановка/обновление до последней версии
- `/speckit-init --script ps` — использовать вариант скрипта PowerShell
- `/speckit-init MyProjectName` — инициализировать новый именованный проект вместо текущей директории

Аргументы можно комбинировать, например `/speckit-init v0.4.2 --script ps MyProjectName`.

Разберите аргументы, чтобы определить, какой вариант запускать. Если `--script` не указан, всегда использовать `--script sh` по умолчанию.

## Обработка ошибок

- **`uv` не найден**: показать инструкции по установке, остановиться.
- **Ошибка сети при установке**: предложить проверить подключение к интернету или воспользоваться руководством по установке в корпоративной/изолированной среде по адресу `https://github.com/github/spec-kit/blob/main/docs/installation.md`
- **`specify` не найден после установки**: выполнить `uv tool update-shell` и указать пользователю перезапустить терминал или обновить профиль оболочки.
- **Уже инициализирован** (директория `.specify/` существует): предупредить пользователя и спросить, хочет ли он повторно инициализировать проект.
