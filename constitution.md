<!--
Sync Impact Report
==================
Version change: 1.2.0 → 1.3.0
Changed principles:
  - I. Architecture: Fixed Data serialization (Protobuf → JSON/CryptoManager); added UpdateVisual to lifecycle
  - II. Performance: Added async/await prohibition; callbacks prohibition; IDisposable prohibition; Reflection rule
  - III. Code Style: Added readonly fields, return on own line, C# properties rule, static methods/fields rule,
         logging rule, line length, non-obvious optimizations comment, fixed #if syntax, DRY/fail-fast principles
  - IV. Integration: Added Google Sheets note, test directory
Added sections: None
Removed sections: None
Templates updated:
  ✅ .specify/templates/plan-template.md — Constitution Gates section present
  ✅ .specify/templates/spec-template.md — aligns with feature arch
  ✅ .specify/templates/tasks-template.md — aligns with feature phases
Deferred TODOs: None
-->

# Конституция Pirates Survival (Мьютини)

## I. Архитектура фич: Data → Model → Description → Controller

Каждая фича строится по четырёхслойному паттерну. Все части необязательны, но порядок зависимостей строгий.

- **Data** (`Assets/Shared/Data/`) — plain class, только поля, без логики. Сериализуется в JSON через `CryptoManager.cs` / `DataToJsonGen.cs` (автогенерация). Шарится клиент/сервер. Без Unity-типов. Заводить только при необходимости хранить данные в профиле — раздувает профиль. После выхода на продакшен Data нельзя менять без миграции (`Assets/Shared/Migrations/Migration{N}_{Description}.cs`).
- **Description** (`Assets/Core/Models/Foo/`) — статический конфиг из JSON/Google Sheets, парсится через `References`. Используют рефлексию. References нельзя кэшировать в классах. Неизменяем после загрузки. Выносить в конфиг только то, что реально настраивается.
  - При добавлении поля с `*_id`-суффиксом в JSON-конфиг — обязательно верифицировать наличие этого ID в целевом конфиге.
  - Если редактируете JSON из `table_refs.json` — MUST явно отметить, что Google Sheets тоже требует обновления.
- **Model** (`Assets/Core/Models/Foo/`) — вся доменная бизнес-логика, изолированная от Unity. `Model = Data (runtime) + Description (config)`. Если большая — делить. Внутренний стейт теряется при перезапуске. Класс ДОЛЖЕН заканчиваться на `Model` только если внутри хранит data этой модели. Модели получают все зависимости через конструктор; `GameManager.Instance` — крайне нежелательно.
- **Controller** (`Assets/Core/Game/` или `Core.**.Dialogs`) — логика, завязанная на Unity (UI, 3D). Наследует `ControllerBase : MonoBehaviour`. Не использовать Unity-события (`OnDestroy`, `OnEnable`, `Start`, `Awake`) — использовать самописные методы.
  - **Lifecycle order**: `Init → Show → UpdateVisual → Hide → Dispose`.
    - `Init` — подписки на Unity controls, инстанцирование ресурсов.
    - `Show` — обновить весь стейт (изображения, таймеры, счётчики), подписаться на модели.
    - `UpdateVisual` — регулярный update, вызывается из Show, по событиям, корутинам или из Unity `Update`.
    - `Hide` — отписаться от моделей, занулить ссылки на модели.
    - `Dispose` — отписки, освободить ресурсы, занулить ссылки на менеджеры.
  - **Диалоги** (`DialogController : ControllerBase`) MUST реализовывать все четыре метода lifecycle. Отсутствие `Init` или `Dispose` — неполная реализация.
- **System** (`Assets/Core/Models/Foo/`) — для сложных фич, управляет коллекцией Model.

**Правила ссылок (строго односторонние):**
```
Data        → только другой Data
Description → только другой Description
Model       → Data (пишет), Description (читает)
Controller  → Model (через методы), Description (читает). НЕ → Data.
```

**Структура папок:**
```
Assets/Core/Models/Foo/       — FooModel, FooSystem, FooDescription
Assets/Core/Game/Foo/         — FooController (:ControllerBase)
Assets/Shared/Data/           — FooData (plain class, no logic)
Assets/References/foo.json    — JSON-конфиг → FooDescription
Assets/Editor/Tests/          — NUnit тесты (Regular — stable, Generated)
```

## II. Производительность и запрещённые паттерны

Мобильная игра. Аллокации в hot path недопустимы.

- MUST NOT использовать LINQ — только обычные циклы
- MUST использовать `ToStringLookup()` вместо `ToString()` в hot path
- MUST использовать пулы (`InstantiableUiList`, `CacheManager`) при runtime-инстанцировании
- MUST NOT подписываться/отписываться на каждой смене выборки или 10 раз в кадр
- MUST NOT использовать `Resources.Load` — только `AssetManager` (Addressables)
- MUST NOT использовать `try/catch` без явного обоснования
- MUST NOT добавлять null-проверки для Serializable полей Controller / MonoBehaviour классах и их наследниках.
  Если для работы фичи требуется назначить что-то в Inspector — НИКОГДА не проверяй это на null в коде.
- MUST NOT добавлять null-проверки для обязательных runtime-зависимостей (объектов, созданных в `Init`/`InitFoo`).
  Если объект обязан существовать — при его отсутствии MUST бросать `InvalidOperationException` или `Debug.Assert`.
  Null-check допустим только для явно опциональных зависимостей.
- MUST NOT использовать `TMP_Text` для UI-текста на Canvas — это абстрактный базовый класс, не предназначен
  для сериализации в Inspector. Использовать `TextMeshProUGUI`. Для локализуемого текста — `Localize`.

**Строго запрещены (никогда):**
- **async/await** — строго запрещено. Использовать корутины или C# events. Если навязывает внешнее API — обернуть
  как можно ближе к точке вызова.
- **Callbacks** — не передавать коллбэки в методы; использовать C# events.
- **IDisposable** — не создавать классы, реализующие `IDisposable`.
- **Reflection** — только в tools и cheats.
- **partial class** — только в автогенерируемом коде.
- **Obscure code, hacks, known issue workarounds** — комментировать явно с Jira-ссылкой.

**Запрещённые паттерны (legacy, не развивать):**
- `IEntity` — попытка ECS, удалять
- `Container` классы — размазывает код, нарушает инкапсуляцию, удалять
- `Reward` — допустим только для выдачи предметов/доната
- `StaticResource` / `Resource` — непредсказуемо, не использовать
- `FactoryManager` — только для Description, не для моделей/контроллеров

## III. Стиль кода и именование

- Комментарии в коде — английский. Общение в чате — русский.
- Комментарии только в сложной логике. Если хочется написать комментарий — лучше рефактори.
- Один файл — один класс. Вложенных классов и enum-ов избегать.
- **Partial-классы запрещены**. Если нужна декомпозиция — composition, отдельный helper или передача зависимости
  через конструктор/метод.
- Размер файла: <200 строк — хорошо; 200–500 — сойдёт; 500–1000 — рефактори; >1000 — НЕЛЬЗЯ (кроме автогенерации и legacy).
- Soft limit 120 символов на строку; переносить только там, где это реально улучшает читаемость.
- `return` MUST быть на отдельной строке.
- Нет пустых строк в конце файла.
- `#if` только для: (a) код не компилируется на платформе, (b) читы (`#if UNITY_EDITOR || USE_CHEATS`).
- Автоформатирование — отдельным коммитом, не смешивать с логикой.
- В конструктор передавать `GameManager` (кроме Description классов).
- `[Serializable] private Type _field` для сериализуемых полей Unity.
- **DRY**: разумный, не догматичный. Дублировать пару строк — нормально; большие блоки — выносить в методы.
- Если что-то может упасть — MUST упасть как можно раньше. Предпочитать статические проверки → load-time → runtime.
- Non-obvious optimizations: комментировать явно с `// opcode: что и почему`.
- Логирование только при загрузке и ключевых точках. Если фича требует подробных логов — под `CheatFlags.LogMyFeature`, отключено по умолчанию.

**Поля и свойства:**
- Предпочитать `readonly` поля над свойствами с private setter.
- C# properties — только one-liners. Если разрастается — рефактори в метод.
- Static методы — только helper-методы. Если helper разрастается — выносить в отдельный класс.
- Static поля — только для кэширования; MUST очищаться в `GameManager.FreeDomain`.

**Именование:**
- Имена грамматически верны и легко читаемы.
- Булевые поля называть так, чтобы условие читалось естественно.
- `data` модели всегда приватная и только в своей модели.

**Сомнительное (только через ревью):**
- Generic-классы — для изоляции алгоритмов (пул, стейт-машина) норм, но бизнес-логика не нуждается.
- ViewModels — только через ревью.
- Интерфейсы — заводить только если обосновано кодом (тесты, моки). Для бизнес-логики — абстрактный класс.
- Наследование — избегать, предпочитать композицию. Override MUST вызывать `base` строго в начале или конце.
- C# events — ровно одно место подписки и одно отписки. Порядок вызова хендлеров непредсказуем.
- GameManager references в Models — крайне нежелательно; модели получают зависимости через конструктор.
- Любые необычные, особые, специальные паттерны, шаблоны и решения.

## IV. Ключевые точки интеграции

При разработке фичи MUST проверить и при необходимости обновить:

| Файл | Назначение |
|------|-----------|
| `Assets/Core/Manager/GameManager.cs` | Composition root (~2500 строк), `GameManager.Instance` |
| `Assets/Core/Models/DesignData/References.cs` | Мастер конфиг (~1300 строк) |
| `Assets/Core/Factory/FactoryManager.cs` | Полиморфные фабрики через `FactoryBuilder` |
| `Assets/Core/Manager/AssetManager.cs` | Обёртка Addressables |

**NPC и поведения (`behaviours.json`):**
- При создании нового behaviour для NPC — сначала найти в `behaviours.json` существующего моба с нужным типом
  поведения (агрессивный, патрульный, пассивный), скопировать его структуру и изменить только ID и параметры.
  Можно переиспользовать behaviour_id.
- При использовании существующего аватара моба, чьё поведение является приёмочным требованием фичи (FR/SC) —
  MUST открыть Behavior Designer для этого аватара и убедиться, что дерево поведений соответствует требованиям.
  Не принимать аватар "по имени" без доказательства корректного behavior tree.

**Базовые классы:**
- `ManagerBase : MonoBehaviour` — для менеджеров, переопределять `OnAwake()` / `OnUpdate()`
- `ControllerBase : MonoBehaviour` — для контроллеров, отслеживает утечки памяти в Editor
- `DialogController : ControllerBase` — база UI диалогов

**Сцены (не развивать GlobalMap):**
- `BattleController` — основной цикл
- `GlobalMapController` — мета цикл, перемещение между локациями

## V. Работа с Unity через MCP

Для взаимодействия с движком Unity MUST использовать **Unity MCP** (Model Context Protocol).

**Требования:**
- Все операции на стороне Unity (чтение сцен, назначение ссылок, prefab-ов, Scriptable Objects) выполняются через Unity MCP.

**Если Unity MCP недоступен:**
- MUST остановить текущую задачу немедленно.
- MUST явно сообщить пользователю: "Unity MCP недоступен. Причина: <текст ошибки>. Не могу выполнить операции
  <список операций>. Требуется подключение Unity MCP для продолжения работы."
- MUST NOT продолжать работу в обход, искать workaround или выполнять задачу без MCP.
- MUST NOT пытаться работать с Unity файлами (.unity, .prefab, .asset) без MCP.

## Управление

Конституция имеет приоритет над всеми другими практиками проекта. Поправки вносятся при:
- **МАЖОР** (X.0.0): изменение/удаление принципов, несовместимые изменения архитектуры.
- **МИНОР** (X.Y.0): добавление принципа/раздела, существенное расширение правил.
- **ПАТЧ** (X.Y.Z): уточнения формулировок, исправление опечаток.

Все PR MUST проверять соответствие конституции. Любое отклонение от правил MUST быть явно обосновано и задокументировано в разделе "Отслеживание сложности" плана (`plan.md`).

**Версия**: 1.3.0 | **Ратифицирована**: 2026-03-25 | **Последнее изменение**: 2026-03-26
