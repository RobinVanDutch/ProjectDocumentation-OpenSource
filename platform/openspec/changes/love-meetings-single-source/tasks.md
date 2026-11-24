# Tasks

## 1. Модель и миграция (P0)

- [x] 1.1 Поля `duration` и `room` в `TPermanentMeeting`.
  - `models/love/src/index.ts`: `@Prop(TypeNumber(), love.string.Duration) duration?: number` и
    `@Prop(TypeRef(love.class.Room), love.string.Room) room?: Ref<Room>`, строка `Duration` в
    `plugins/love/src/plugin.ts` и во всех 12 локалях `plugins/love-assets/lang/*.json`.
  - `plugins/love/src/types.ts`: `duration?: number`, `room?: Ref<Room>` в `PermanentMeeting`.
  - Проверка: `pnpm build --to @hcengineering/model-love`, 0 ошибок.
  - Сделано (2026-11-24): `room` добавлен как необходимое следствие — комната разовой встречи раньше
    жила в миксине события, а событие больше не создаётся.
- [x] 1.2 Миграция `event-meeting-to-permanent`.
  - `models/love/src/migration.ts`: для `Event` с миксином `MeetingEventLink` и пустым
    `permanentMeeting` создать `PermanentMeeting` (`name`, `startDate`, `duration`, `members`,
    `timeZone`, `calendar: true`, `group`) и проставить ссылку в миксин.
  - Проверка: повторный запуск дублей не создаёт; jest-тест на выборку событий.
  - Сделано (2026-11-24): проверено на стенде — upgrade воркспейса создал встречи для событий
    `test`, `test2`, `One-off check`, `еуыеыеые`; события и серии остались.
- [x] 1.3 Тип встречи в списках.
  - Выполняется существующим `PermanentMeetingPresenter`: камера — есть серия, перечёркнутый глаз —
    серии нет, метка постоянной встречи — `.permanent-mark`. Новая колонка не понадобилась.

## 2. Создание (P0)

- [x] 2.1 `createOneOffMeeting` пишет встречу, а не событие.
  - `plugins/love-resources/src/permanentMeetingSchedule.ts`: `PermanentMeetingInput.duration`,
    создание `PermanentMeeting` с `calendar: false`, `repeatEvery: undefined`, без `Event`.
  - Проверка: unit-тест «разовая → один документ `PermanentMeeting`».
  - Сделано (2026-11-24): разовая идёт через общий писатель; серия создаётся только когда окно
    открыто из планировщика (`input.calendar`), комната пишется в документ встречи.
- [x] 2.2 Источник создания в диалоге.
  - `CreateMeetingDialog.svelte`: тумблер при `date === undefined` пишет флаг `calendar`; `series: date !== undefined`.
  - `permanentMeetingSchedule.ts`: `series` → событие в личном календаре (`ctx.calendarId`), иначе событие в
    календаре `Meetings` (`<uuid>_meetings_calendar`), который создаётся по требованию.
  - `plugins/time-resources`: `WithTeamData.svelte` и `PlanningCalendar.svelte` исключают этот календарь.
  - Проверено на стенде: «Встреча в календаре раздела» и «Постоянная в календаре раздела» видны в календаре
    раздела и отсутствуют в Планировщике.
- [x] 2.3 Длительность в форме.
  - Длительность считается из полей начала и конца и пишется в `duration`; 12 локалей обновлены.

## 3. Проверка на стенде (P1)

- [x] 3.1 Разовая из «Встреч»: строка есть в таблице раздела и во вкладке «Группы», в планировщике её нет.
- [x] 3.2 Постоянная из «Встреч»: то же; серия не создаётся.
- [x] 3.3 Встреча из планировщика: событие в календаре есть, строка в списках есть.
- [x] 3.4 После миграции старые встречи-события видны в списках, планировщик не изменился.
- [x] 3.5 «Группы»: секции `Sprint` и «Не указана»; создание только «Объединить в группу».

## 4. Документация (P2)

- [x] 4.1 `docs/love.md`: §2.6 (встреча — документ), таблица функций (`createOneOffMeeting`).
- [x] 4.2 `docs/features/office-meetings.md`: bullet «Единый источник встреч»; строка пользовательской
  документации → `plan` (страница в `platform-docs` ещё не написана).
- [x] 4.3 `handoff.md`: итог 2026-11-24 и пометка про сборку образа `transactor` из `pod-server`.
