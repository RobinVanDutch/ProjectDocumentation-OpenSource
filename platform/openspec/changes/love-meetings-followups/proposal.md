# Proposal

## Why

Ветка `FUSIO-1307` (коммит `0503dfdca6`) перенесла идентичность встречи на миксин `love.mixin.MeetingEventLink` мастер-события календаря, добавила постоянные встречи (F4), ссылку-указатель, пароль для гостей, RSVP и навигатор офиса вместо служебного этажа. Основной объём сделан, но в отчёте по ветке остались незакрытые пункты: отменённые и перенесённые вхождения серии не учитываются при резолве сессии, инвариант «миксин только на мастере» держится только на клиентском коде, в моделях остались мёртвые поля, у навигатора известные шероховатости, а тестовое покрытие и пользовательская документация отстают от кода.

Без этих доделок встреча ведёт себя неправильно в штатном сценарии календаря (отменил встречу — она всё равно открывается), а регрессии в новых путях ловятся только руками: sanity по ветке не гонялся.

## What Changes

- **Вхождения серии.** Резолв сессии начинает учитывать `isCancelled` у `ReccuringInstance` и время перенесённого вхождения: отменённое вхождение даёт `no-occurrence` + `nextOccurrence`, перенесённое открывается в своё время, а не по расписанию мастера.
- **Инвариант мастера.** Появление миксина `MeetingEventLink` на `ReccuringInstance` или на копии события (`access !== owner`) отклоняется или игнорируется на сервере, а не только «так принято» на клиенте.
- **Права постоянной встречи.** Владельца постоянной встречи нельзя удалить из `members` штатным путём (сейчас защиту давал удалённый редактор участников в шапке).
- **Навигатор офиса.** Выбранный пункт навигатора переживает открытие протокола во время звонка; в шапке раздела протоколов остаётся одна кнопка настроек вместо двух.
- **Внеплановый вход по ссылке.** Ссылка остаётся рабочей и вне окна вхождения: участник (а при политике `start: 'link'` — и гость) может начать встречу в любой момент; сессия и LiveKit-комната создаются заново, short id не меняется. Кнопка завершения сессии, авто-завершение пустой комнаты и протоколы уже работают в ветке — в объём не входят.
- **Постоянная встреча.** Язык берётся из интерфейса или выбирается пользователем вместо захардкоженного `'en'`; для гостя постоянной встречи подтверждается отсутствие режима `after`.
- **Чистка моделей.** Удаляются мёртвые поля `linkId` (`MeetingEventLink`, `TMeeting`, `PermanentMeeting`) и `meetingId` (`TMeetingSchedule`). **BREAKING** для модели: поля исчезают из схемы, в сохранённых документах остаются как мусор, миграция данных не нужна.
- **Тесты и документация.** Unit-тесты на новые инварианты, API-sanity на вхождения и постоянные встречи, playwright на F4 и пароль гостя, обновление `virtual-office.mdx` (en + ru).

## Capabilities

### New Capabilities

- `love-meetings`: расписание встречи, разворот вхождений, окно входа, ссылка-указатель и её отзыв, политика доступа, гостевые режимы, постоянные встречи, жизненный цикл сессии.
- `office-navigator`: пункты навигатора офиса и их URL-сегменты, панель встречи, протоколы прошедших встреч, завершение сессии из панели.

### Modified Capabilities

Нет: для этого проекта OpenSpec обе способности описываются впервые, требований «как было» в `openspec/specs/` ещё нет (создадутся при архивации изменения).

## Impact

- Love-модель и utils: `plugins/love/src/utils.ts` (`meetingOccurrences`, `resolveOccurrence`, `checkMeetingLink`, `isMeetingParticipant`), `plugins/love/src/types.ts`, `models/love/src/index.ts`.
- Love-сервис: `services/love/src/workspaceClient.ts` (`resolveSession`, `findMeetingTarget`), `services/love/src/guests.ts`.
- Триггеры: `server-plugins/calendar-resources/src/index.ts` (`onEventMixin`, `onEventUpdate`), `server-plugins/love-resources/src/index.ts` (`OnEventUpdate`).
- UI: `plugins/love-resources/src/components/{OfficeNavigator,MeetingMinutesSection,EditScheduledMeeting,EditPermanentMeeting,CreatePermanentMeetingPopup}.svelte`, `plugins/love-resources/src/utils.ts` (`navigateToOfficeDoc`).
- Календарь: `plugins/calendar-resources/src/index.ts` (`deleteRecHandler`, `updateReccuringInstance`), `plugins/calendar/src/utils.ts` (`getInstance`).
- Тесты: `plugins/love/src/__tests__`, `services/love/src/__tests__`, `server-plugins/love-resources/src/__tests__`, `tests/sanity/tests/love/*`, `tests/sanity/tests/calendar/*`.
- Документация: `foundation-docs` → `virtual-office.mdx` (en, ru).
- Стенд: пересборка одним прогоном (`pnpm -w docker-server`) и `dev/tool` upgrade — без параллельных сборок; иначе снова ловится гонка `bundle/ changed during docker build`.
