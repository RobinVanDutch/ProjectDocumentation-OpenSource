# Tasks

> **Статусы после сверки с PR #428** (подробно — `../../../../audit-pr-428.md`):
> 2026-11-02 PR применён на ветку `FUSIO-1307-livekit-shared-link`, см. `audit-pr-428.md` §12.
> группа 3 (этап 2) — **выполнено PR**; группа 2 (этап 1) — **частично**: `exdate` учтён, перенос вхождения (override) нет;
> группа 5 (этап 4) — **выполнено**: URL-сегмент из PR + наша правка `navigateToOfficeDoc`/`backOrOpen`;
> задача 7.1 (язык постоянной встречи) — **выполнено 02.11**, 7.2 (`after` у постоянной встречи) — открыта;
> группы 4, 6, 8 и 9 (этапы 3, 5, 7, 8) — **открыты**; группа 6 (этап 5, внеплановый старт) **закрыта решением 2026-11-03**:
> не реализуем, сценарий закрывает `PermanentMeeting` (см. `love-meetings-pr428-gaps/design.md`, Decisions 7);
> дефекты PR и бэклог — `audit-pr-428.md` §7–§8.

Порядок групп — порядок зависимостей. Внутри группы задачи тоже упорядочены. Команды сборки и прогонов запускает владелец репозитория; в задачах они указаны как способ проверки.

## 1. Этап 0: подготовка стенда и базовая линия

- [ ] 1.1 Пересобрать стенд одним прогоном без параллельных сборок: `pnpm -w docker-server`, затем `cd dev/tool && pnpm run upgrade platformlocal` — проверка: контейнеры `account`, `love`, `transactor` поднялись, в логах нет `bundle/ changed during docker build`.
- [ ] 1.2 Зафиксировать базовый прогон unit-наборов love: `pnpm test --to @hcengineering/love` (или прогон по пакетам `plugins/love`, `services/love`, `models/love`, `server-plugins/love-resources`) — проверка: список падений сохранён в отчёт этапа, до правок он ожидаемо пустой.
- [ ] 1.3 Вручную проверить то, что не покрыто тестами (navigator-01): панель свойств миксина (`meetingAccess`, приватность, язык, запись, транскрипция), секция протоколов ниже описания, копирование гостевой ссылки — проверка: чек-лист с отметками «работает / дефект» в отчёте этапа.
- [ ] 1.4 Прогнать love- и calendar-спеки sanity и сохранить отчёт: `cd tests/sanity && pnpm run uitest tests/love` (и `tests/calendar`) — проверка: сохранён `playwright-report.json`, известные расхождения записаны в отчёт.

## 2. Этап 1: вхождения серии — отмена и перенос

- [ ] 2.1 Проверить, что переопределения вхождения читаются сервисным аккаунтом из `PersonSpace` участников: запрос `ReccuringInstance { recurringEventId }` в love-сервисе на тестовом стенде — проверка: запрос возвращает отменённое и перенесённое вхождения; если политика не пускает, зафиксировать это и перейти к 2.2 через сервисный RPC.
      **2026-11-12: проверено на Postgres — НЕ читаются**, но не из-за прав: `$or` в запросе не поддерживается
      Postgres-адаптером (`operator does not exist: text = text[]`, лог transactor). Исправление —
      `love-meetings-corrections` 2.1.
- [x] 2.2 Расширить `meetingOccurrences` (`plugins/love/src/utils.ts`): вычитать вхождения с `isCancelled`, подменять расчётное время на `originalStartTime` перенесённого вхождения — проверка: unit-тест в `plugins/love/src/__tests__` на «отменено», «перенесено на день», «перенесено на два часа», «exdate + перенос». **Сделано 2026-11-11** (`resolveOccurrence.test.ts` 17 кейсов, включая «`rules: []` — не серия»).
- [x] 2.3 Учесть это в `resolveOccurrence` и в расчёте `nextOccurrence` (`services/love/src/workspaceClient.ts`) — проверка: unit-тест `resolveOccurrence`: отменённое текущее вхождение даёт `no-occurrence` с следующим действительным вхождением; перенесённое открывается в новое время. **Сделано 2026-11-11** (`resolveSession.test.ts`), осталось чтение исключений на Postgres — см. 2.1.
- [ ] 2.4 API-sanity на новое поведение: `tests/sanity/tests/love/meetings.recurring.tests.ts` — отменённое вхождение не открывает сессию и отдаёт `nextOccurrence`; перенесённое открывает сессию с `occurrence = originalStartTime` — проверка: тесты зелёные на стенде из 1.1.

## 3. Этап 2: миксин только на мастере

- [x] 3.1 В `onEventMixin` (`server-plugins/calendar-resources/src/index.ts`) добавлен отсев класса `ReccuringInstance`:
      вхождение — это `ReccuringInstance`, который в календаре того, кто его перенёс, сохраняет `access: 'owner'`,
      поэтому одной проверки `access` не хватало. В `onEventUpdate` тот же отсев стоит перед распространением правок на
      серию — иначе перенос одного вхождения перезаписал бы участников и время мастера. Копии (`access !== owner`)
      отсекаются как раньше. Проверка: `onEventMixin` экспортирована, новый `src/__tests__/eventMixin.test.ts`
      (3 кейса: копия, вхождение, мастер), пакет 5/5, `onEventUpdate.test.ts` зелёный.
- [x] 3.2 `meetingMasterQuery` (`plugins/love/src/utils.ts`) теперь явно исключает класс:
      `_class: { $ne: calendar.class.ReccuringInstance }`. Иначе поиск мастера по `{ eventId, access: Owner }` мог
      выбрать перенесённое вхождение и сделать одно вхождение идентичностью всей серии. Проверка:
      `meetingMasterQuery.test.ts` — ожидания обновлены, добавлен кейс «никогда не резолвится во вхождение», 50/50.
- [x] 3.3 Отброшенная запись миксина пишется в лог `control.ctx.warn('Meeting identity mixin written to an occurrence,
    ignored', { eventId, objectId, modifiedBy })` — транзакция просто не распространяется, старые документы не падают.
      Проверка: тест `eventMixin.test.ts` ждёт вызова `ctx.warn`; образ `pod-server` пересобран, `transactor`
      пересоздан, в логе transactor нет ошибок триггеров.

## 4. Этап 3: права постоянной встречи

- [x] 4.1 Серверный инвариант уже стоит в `OnPermanentMeetingUpdate` (`server-plugins/love-resources/src/index.ts`):
      недостающих владельцев триггер возвращает и в `owners`, и в `members` (`$push`), причём отдельно обработаны
      полная перезапись `members`/`owners` и `$pull` (одиночный id, массив, `$in` — так снимает участника `SpaceMembers`),
      плюс отдельно защищён создатель (`createdBy` → аккаунт). Триггер не может отклонить транзакцию, поэтому
      «правка без владельца отклоняется» реализовано как ответная правка — так и было согласовано ранее
      (`love-permanent-meetings` 6.2). Проверка: `onPermanentMeetingUpdate.test.ts` — 15 тестов (перезапись, `$pull`,
      `$in`, создатель, отсутствие владельца в списке, посторонние транзакции), зелёные.
- [x] 4.2 UI не даёт исключить владельца: `withOwners` (`plugins/love-resources/src/permanentMeetingSchedule.ts`),
      применён в `PermanentMeetingForm.membersChanged` — после любой правки списка владельцы возвращаются на место,
      поэтому сохранение не отправляет список без владельца и форма не показывает ложную картину.
      Проверка: 3 unit-теста `withOwners` в `permanentMeetingSchedule.test.ts`, пакет 76/76.

## 5. Этап 4: навигатор офиса — доводка

- [ ] 5.1 Не сбрасывать сегмент при открытии документа: `navigateToOfficeDoc` (`plugins/love-resources/src/utils.ts`) и `backOrOpen` (`RoomPopup.svelte`) не режут путь до `length = 3` — проверка: сценарий «в звонке открыл протокол → назад»: пункт навигатора остаётся выбранным, адрес сохраняет сегмент.
      **Код готов 2026-11-03** (`love-meetings-pr428-gaps` 5.1); осталась ручная проверка на стенде — перенесена в
      `love-meetings-corrections` 5.3.
- [x] 5.2 Дубль кнопок в шапке раздела протоколов убран в двух местах: `MeetingMinutesSection.svelte` (документ
      встречи) и `RoomMeetingsFooter.svelte` (документ комнаты — именно там дубль виден на стенде).
      Причина: `ViewletsSettingButton` сам рисует ОБЕ кнопки (`ViewOptionsButton` = «Group & sort» + `IconButton`
      = «Customize view»), рядом стоял ещё `ViewletSettingButton` с той же парой — в шапке `antiSection-header`
      было 4 кнопки (`btn-viewOptions`, `btn-viewSetting` и их копии). Оставлен один `ViewletsSettingButton`,
      `viewOptions` теперь читается через `bind:viewOptions` (компонент его вычисляет и экспортирует).
      Проверка: сборка 0 ошибок, jest `love-resources` 76/76, Playwright на 8087 (документ комнаты «Voice Room 1»):
      в шапке было 4 кнопки (`btn-viewOptions`, `btn-viewSetting` + копии), стало 2 — «Group & sort» и «Customize view»,
      меню настроек открывается (список колонок + «Restore defaults»). Образ `pod-front` пересобран с `--force`
      (503 с): без обхода кэша первый образ отдавал старый бандл, и проверка видела прежние 4 кнопки.
- [ ] 5.3 Обновить `virtual-office.mdx` (en, ru) в `foundation-docs`: без служебного этажа, пункты навигатора, «Forever» в сроке жизни ссылки, ссылка постоянной встречи в настройках, новые скриншоты — проверка: страница описывает текущий интерфейс, скриншоты сняты со стенда после 1.1.
- [ ] 5.4 Обновить офисные sanity-ожидания (`tests/sanity/tests/love/meetings.*`): проверки этажа `Scheduled` и `ScheduledFloorView` больше не актуальны — проверка: `pnpm run uitest tests/love` зелёный, новых падений нет.

## 6. Этап 5: внеплановый вход по ссылке

- [ ] 6.1 В `resolveSession` (`services/love/src/workspaceClient.ts`) не отдавать `no-occurrence`, когда политика разрешает старт: `start: 'members'` — участник встречи, `start: 'link'` — участник и гость; при отказе оставить прежний ответ с `nextOccurrence` — проверка: unit-тест `resolveSession` на три случая (участник вне окна, гость при `members`, гость при `link`).
- [ ] 6.2 Создавать внеплановую сессию с `occurrence = null` и `roomId = ScheduledRoom`, не меняя `eventId` мастера — проверка: unit-тест: у сессии нет `occurrence`, `eventId` равен мастеру, short id по тому же payload не меняется.
- [ ] 6.3 Лобби `/guestInfo`: при отсутствии сессии и разрешённом старте отдавать `before` с признаком «можно начать» и показывать кнопку «Начать сейчас» в `GuestMeetingApp.svelte`; при запрете — `nextOccurrence` как сейчас — проверка: ручная проверка гостевой ссылки при `members` и при `link`.
- [ ] 6.4 Проверить, что повторный вход после авто-завершения пустой комнаты создаёт новую сессию с тем же short id — проверка: API-sanity в `tests/sanity/tests/love/meetings.recurring.tests.ts` зелёный.

## 7. Этап 6: постоянная встреча — язык и гостевые режимы

- [x] 7.1 Хардкода `'en'` больше нет: `CreatePermanentMeetingPopup.svelte` только хостит окно, а язык берётся
      из интерфейса в `PermanentMeetingForm.save()` (`getCurrentLanguage()`) и уезжает в `ScheduleContext.language`
      (`permanentMeetingSchedule.ts` → `createPermanentMeetingWithSchedule`). Проверка: в попапе и в
      `permanentMeetingSchedule.ts` нет литерала `'en'` (grep пуст).
- [x] 7.2 Инвариант закреплён тестом `services/love/src/__tests__/meetingLink.test.ts`: «постоянная встреча не
      называется законченной даже после завершённой сессии» — лобби отдаёт `mode: 'live'` (форма входа остаётся),
      `after` не отдаётся никогда, потому что `meetingDoc.meeting` у сессии постоянной встречи задан, а
      `isOver` в `handleGuestInfo` не срабатывает для `kind: 'meeting'`. Пакет `services/love` 232/232.
      **API-sanity часть** перенесена в 10.2 (стенд 8083): `meetings.permanent.tests.ts`.

## 8. Этап 7: чистка мёртвых полей модели

- [x] 8.1 ~~Удалить `linkId`~~ — **задача устарела, поле живое**: сервис его пишет (`setMeetingLinkId` в
      `services/love/src/main.ts:548` вызывает `WorkspaceClient.setMeetingLinkId`, модель документирует «absent until
      the link is first issued»). Удаление сломало бы выдачу ссылки, поэтому BREAKING-изменения модели нет.
      Проверка: `grep -rn linkId services/love/src plugins/love/src` показывает запись и чтение.
- [x] 8.2 ~~Удалить `meetingId` из `TMeetingSchedule`~~ — **задача устарела, поле живое**: booking-путь пишет его
      (`TMeetingSchedule.meetingId`, комментарий в `models/love/src/index.ts`), а UI читает
      (`EditMeetingScheduleData.svelte:41,52`).
- [x] 8.3 `docs/features/love.md` — править нечего: поля остаются, описание миксина соответствует коду. Если
      правки модели когда-то понадобятся, это отдельная задача с миграцией.

## 9. Этап 8: сквозная проверка

- [ ] 9.1 Прогнать unit-наборы затронутых пакетов и зафиксировать итог — проверка: список падений совпадает с 1.2 или меньше.
- [ ] 9.2 Прогнать sanity love + calendar целиком — проверка: отчёт `analyze_failures.js` показывает 0 реальных падений (флаки отдельно).
- [ ] 9.3 Составить итоговый отчёт по этапам: что сделано, что осталось, какие открытые вопросы закрыты — проверка: отчёт приложен к изменению, каждый пункт из `plan.md` имеет статус.

## Workflow follow-up

- После приёмки — `openspec archive love-meetings-followups`, чтобы дельты влились в `openspec/specs/`.
- Force-push ветки `FUSIO-1307` после rebase (remote до rebase устарел) выполняет владелец репозитория.
