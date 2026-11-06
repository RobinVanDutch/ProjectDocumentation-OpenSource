# Proposal

## Why

Остаток по регулярным созвонам — пять пунктов списка владельца от 2026-11-05 (отображение в планировании,
создание с регулярностью, настройки, правка/удаление во вкладке встреч, изменение серии). Часть этих пунктов уже
лежит в коде ветки `FUSIO-1307-livekit-shared-link` коммитами `a579404650` (экраны, создание постоянных встреч)
и `e2da22dca2` (отображение встреч), но нигде не зафиксирована: ни в PR #428, ни в `love-meetings-pr428-gaps`.
Из-за этого `README.md`, `plan.md` и `handoff.md` ссылаются на это изменение, описывая состояние, которого в
документах нет, а `openspec list` показывает его отсутствующим.

Черновик `love-planner-permanent-meeting` (от 03.11) описывал ту же работу иначе — с самодельной связью через
`TMeetingSchedule.meetingId` для события и с другим видом вкладки «Встречи». Он поглощён этим изменением и удалён:
дешевле вести один список остатка, чем два расходящихся.

## What Changes

- **Отображение в планировании и календаре** — сделано, фиксируется требованием: бейдж «войти» на карточке
  события (`MeetingEventBadge` через `calendar.extensions.EventCardExtensions`, миграция `addMeetingEventCardBadge`),
  сводка повторения в диалоге (`RecurrenceSummary`).
- **Создание встречи из планировщика** — не сделано. Путь `calendar.class.Schedule` (`MeetingScheduleData`,
  `createMeetingSchedule`) до сих пор умеет только выбирать комнату: постоянной встречи из него не получить, серии
  тоже, поэтому такая встреча не появляется ни во вкладке «Встречи», ни в «Постоянных встречах».
- **Настройки регулярной встречи** — сделано для события (`EditScheduledMeeting`: период, напоминание, сессия)
  и для постоянной встречи (`EditPermanentMeeting`: имя, период). Осталось: те же настройки для записи планировщика.
- **Правка и удаление во вкладке встреч** — правка идёт через `EditScheduledMeeting` и общий редактор события;
  удаление мастера завершает живую сессию серверным триггером `OnEventUpdate`. Не закрыто: отмена одного вхождения
  и что при этом делать с уже открытой сессией.
- **Изменение серии и её настроек** — переименование постоянной встречи переносится на серию; добавление участника
  в событие добавляет его в живую сессию (`OnEventUpdate`). Не закрыто: обратная синхронизация участников
  (правка участников серии → постоянная встреча) и поведение при сдвиге периода у идущей серии.

## Capabilities

### New Capabilities

- `love-planner-meeting`: создание встречи из планировщика и из формы события — комната, участники, постоянная
  встреча, промежуток повторения, связь записи планировщика с постоянной встречей.
- `love-recurring-meetings`: отображение регулярной встречи в календаре и планировщике, её настройки, правка,
  удаление и изменение серии.
- `love-meetings-hall`: что показывают вкладки «Встречи» и «Постоянные встречи» офиса и как в них сосуществуют
  события-встречи и постоянные встречи.

## Impact

- Модель: `models/love/src/index.ts` — `MeetingEventLink.reminderBefore`, `MeetingEventLink.permanentMeeting`,
  `TPermanentMeeting.repeatEvery` (сделано); `TMeetingSchedule.meetingId` начинает писаться (осталось),
  `plugins/love/src/types.ts` — те же поля.
- Клиент: `plugins/love-resources/src/components/MeetingScheduleData.svelte`, `EditMeetingScheduleData.svelte`,
  `MeetingData.svelte`, `EditScheduledMeeting.svelte`, `EditPermanentMeeting.svelte`, `MeetingEventBadge.svelte`,
  `RepeatEveryEditor.svelte`, `Hall.svelte`, `utils.ts` (`createMeeting`, `createMeetingSchedule`), `period.ts`,
  `meetingReminders.ts`.
- Календарь: `plugins/calendar-resources/src/components/EventElement.svelte` (точка расширения карточки),
  `ReccurancePopup.svelte`, `RecurrenceSummary.svelte`.
- Сервер: `server-plugins/love-resources/src/index.ts` (`OnEventUpdate` — завершение сессий при удалении,
  добавление участников; `OnPermanentMeetingUpdate` — возврат владельца в участники).
- Тесты: `plugins/love-resources/src/__tests__/meetingPeriod.test.ts`,
  `server-plugins/love-resources/src/__tests__/onEventUpdate.test.ts`, `onPermanentMeetingUpdate.test.ts`;
  новые — на путь планировщика.
- Документация: `docs/love.md`, `docs/features/office-meetings.md`.

## Не входит в это изменение

- **Перенос вхождения серии** (override из `PersonSpace`) — источники перенесённых времён не согласованы,
  вынесено в следующую итерацию решением 2026-11-03 (`love-meetings-pr428-gaps`, группа 2).
- **Внеплановый старт по ссылке** — не делаем: этот сценарий закрывает `PermanentMeeting`.
- **Особые права владельца воркспейса на ссылку** — ссылку выдаёт участник, остальные подписываются.
