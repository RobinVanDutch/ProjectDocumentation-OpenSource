# ProjectSpec / platform

> **Расположение (с 2026-11-03).** Эта папка — `C:\Users\Андрей\Desktop\project\platform\ProjectDocumentation-OpenSource\platform`,
> отдельный git-репозиторий `ProjectDocumentation-OpenSource`. Прежняя папка `C:\Users\Андрей\Desktop\project\ProjectSpec` устарела.

Спецификация и поэтапный план по доделкам модуля Love (встречи, ссылки, навигатор офиса) после ветки `FUSIO-1307`.

Источник требований: отчёт по ветке `FUSIO-1307`, коммит `0503dfdca6` поверх develop `4139f7f3c7`.
Дата разбора: 2026-10-30. Статус: **план на утверждении** — реализация не начиналась.

## Что читать в каком порядке

1. `plan.md` — этапы, приоритеты, что получится, оценка, порядок, решения, которые нужны от тебя.
2. `audit-pr-428.md` — сверка с реализацией PR #428: что уже сделано кодом, что осталось, дефекты PR.
3. `diagrams.md` — те же этапы и потоки в виде схем (Mermaid).
4. `openspec/changes/love-meetings-followups/proposal.md` — зачем и что меняется.
5. `openspec/changes/love-meetings-followups/specs/love-meetings/spec.md` — требования к встречам и сценарии.
6. `openspec/changes/love-meetings-followups/specs/office-navigator/spec.md` — требования к навигатору.
7. `openspec/changes/love-meetings-followups/design.md` — технические решения, риски, открытые вопросы.
8. `openspec/changes/love-meetings-followups/tasks.md` — чек-лист задач; по нему идёт исполнение.

> **Актуальность.** Пункты 1, 4–8 написаны по отчёту о ветке до появления PR #428. Фактическое состояние
> кода и расхождения с этими документами — в `audit-pr-428.md`; он главнее, если формулировки расходятся.
>
> **Остаток по регулярным созвонам (2026-11-05)** — `openspec/changes/love-recurring-meetings/`: пять групп
> задач по списку владельца (планировщик, создание с регулярностью, настройки, правка/удаление, изменение серии).
> Изменение `love-meetings-pr428-gaps`, на которое ссылаются `plan.md` и `handoff.md`, в этом репозитории
> отсутствует — если его нет и у вас, работать нужно по `love-recurring-meetings`.

## Структура

```
platform/
├── plan.md                     поэтапный план (главный документ)
├── audit-pr-428.md             сверка с PR #428 и бэклог на её основе
├── diagrams.md                 визуальные графики этапов и потоков
├── handoff.md                  рабочее состояние: ветка, стенд, что осталось
├── README.md                   этот файл
├── .github/
│   ├── prompts/opsx-*.prompt.md    промпты операций OpenSpec
│   └── skills/openspec-*/SKILL.md  те же операции как агентские скиллы
└── openspec/
    ├── config.yaml             конфиг OpenSpec (язык артефактов — ru)
    ├── specs/                  основная база требований (наполняется при archive)
    └── changes/
        ├── love-meeting-link/      гостевая ссылка на ветке FUSIO-1307-livekit-shared-link
        ├── love-meetings-followups/
        │   ├── proposal.md     зачем и что меняется
        │   ├── design.md       как, риски, открытые вопросы
        │   ├── tasks.md        чек-лист с проверками
        │   └── specs/
        │       ├── love-meetings/spec.md
        │       └── office-navigator/spec.md
        └── love-recurring-meetings/   остаток по регулярным созвонам (планировщик, настройки, правка/удаление)
            ├── proposal.md
            ├── design.md       состояние кода на 05.11 и открытые решения
            ├── tasks.md        5 групп по списку владельца
            └── specs/love-recurring-meetings/spec.md
```

## Команды OpenSpec

Установлено глобально: `@fission-ai/openspec@1.15.0` (`npm i -g @fission-ai/openspec`).

```bash
cd C:/Users/Андрей/Desktop/project/platform/ProjectDocumentation-OpenSource/platform

openspec list                                   # изменения в работе
openspec status --change love-meetings-followups  # прогресс по артефактам
openspec validate love-meetings-followups --strict
openspec show love-meetings-followups            # посмотреть изменение
openspec view                                     # интерактивный дашборд
```

После приёмки работ изменение архивируется, и требования из дельт вливаются в `openspec/specs/`:

```bash
openspec archive love-meetings-followups
```

## Правило работы

Перед любым заданием сначала пишется спецификация (этапы, что получится, как проверять) и утверждается. Только после утверждения начинается код. Если в ходе работ меняется объём — сначала правится спецификация, потом код.

## Скиллы

Для ведения этой папки установлены скиллы агента (глобально, `~/.agents/skills/`):

- `platform-spec` — карта документов, правила, команды OpenSpec, порядок сверки PR со спекой;
- `openspec-propose`, `openspec-explore`, `openspec-apply-change`, `openspec-update-change`, `openspec-sync-specs`, `openspec-archive-change` — копии из `.github/skills/` этого проекта.

Промпты для тех же операций лежат в `.github/prompts/opsx-*.prompt.md`.

## Открытые решения

1. **Дефолт внепланового старта** (этап 5): оставить `start: 'members'` — вне окна вхождения стартует только участник, гость ждёт — или сделать `link` дефолтом, чтобы любая ссылка работала как постоянная переговорка.
2. **RSVP на серию или на вхождение** — вне текущего объёма, влияет на формулировки спек.

Закрыто: завершение сессии (кнопка End + авто-завершение пустой комнаты) и доступ к протоколам уже реализованы в ветке — отдельных задач нет, только проверка в базовой линии.
