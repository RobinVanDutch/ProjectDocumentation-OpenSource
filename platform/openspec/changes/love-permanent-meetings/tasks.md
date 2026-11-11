# Tasks

Порядок — по зависимостям. Сборку, прогоны и Playwright-проверку делаем после каждого блока; статус отмечается
здесь же (правило spec-first).

## 1. Модель и миграция

- [x] 1.1 `PermanentMeeting`: `reminderBefore?: number` (мс), `group?: Ref<MeetingGroup>`, `order?: number`;
      `plugins/love/src/types.ts` — те же поля. Проверка: `pnpm build --to @hcengineering/model-love` — 0 ошибок.
- [x] 1.2 Новый класс `MeetingGroup` (`name`, `order`) в `models/love` + карта классов плагина; группы создаются
      обычным `createDoc` из клиента.
- [x] 1.3 Миграция `models/love/src/migration.ts`: шаг `permanent-meeting-actions-v3` создаёт/правит действия
      таблицы (Delete, EditPermanentMeetingSettings, EditMeetingSettings); группы — обычные документы, миграции
      не требуют. Проверка: `pnpm build --to @hcengineering/model-love`.

## 2. Общая форма создания/настройки

- [x] 2.1 `PermanentMeetingForm.svelte` (новая, используется и созданием, и настройкой): название, дата/время
      (`EventTimeEditor`), регулярность (`RepeatEveryEditor` → `rules` через `periodRules`), участники
      (`AccountArrayEditor`), «напомнить за N минут», пароль для гостей, комната (`RoomSelector`).
      Проверка: `pnpm build --to @hcengineering/love-resources` — 0 ошибок.
- [x] 2.2 Валидация: название не пустое (`canSave`), повторное нажатие гасится флагом `saving`.
- [x] 2.3 Локализация: все новые ключи во всех 12 `plugins/love-assets/lang/*.json` (английский текст как
      fallback, ru — русский): `MeetingGroup`, `MeetingOrder`, `MergeIntoGroup`, `GroupName`, `MeetingsWithoutGroup`,
      `CopyMeetingLink`, `MeetingSettings`, `NextOccurrence`, `RemindBeforeMinutes`, `RepeatUntil`, `LinkReady`,
      `CreateMeetingGroup`. Проверка: ключ присутствует во всех 12 файлах, JSON валиден.
- [x] 2.4 `CreatePermanentMeetingPopup.svelte` — тонкая обёртка над `PermanentMeetingForm` (создание)
    - id компонента `love.component.PermanentMeetingForm` для открытия формы из действий.

## 3. Создание: серия + постоянная встреча + ссылка

- [x] 3.1 `createPermanentMeetingWithSchedule` (`plugins/love-resources/src/utils.ts`): создать `ReccuringEvent`
      (`rules`, дата/время) или `Event`, если регулярности нет; повесить `MeetingEventLink` (тип/язык/приватность/
      напоминание из комнаты); создать `PermanentMeeting` и записать `permanentMeeting`; поставить пароль гостей.
      Проверка: `pnpm build --to @hcengineering/love-resources` — 0 ошибок.
- [x] 3.2 После создания — попап «Ссылка готова» с полем и кнопкой «Копировать»
      (`MeetingLinkReadyPopup.svelte`, открывается из `CreatePermanentMeetingPopup`). Проверка: ручная проверка на стенде.
- [x] 3.3 Unit-тест: `permanentMeetingSchedule.test.ts` — создаются серия и встреча, `permanentMeeting`
      проставлен в обе стороны, `repeatEvery` = период, `rules` = `periodRules`, `endDate`, пароль. Проверка: пакетные
      тесты (5/5). Логика вынесена в `src/permanentMeetingSchedule.ts` с контекстом `ScheduleContext` (без
      svelte/livekit в графе импортов), `utils.ts` больше не держит писателей.

## 4. Таблица «Постоянные встречи»

- [x] 4.1 Список «Постоянные встречи» — не таблица, а собственный `PermanentMeetingsList.svelte` (строки:
      иконка, название, «Копировать ссылку», «Присоединиться», «Настройки»); колонка «следующее вхождение»
      оставлена открытой (см. 4.7). Причина: `Table` не держит ручной `order` после перезагрузки (требование 7).
- [x] 4.2 Копирование гостевой ссылки в строке: `PermanentMeetingPresenter` показывает кнопку рядом с названием
      (`getEventMeetingLink(_id, 'meeting')` + `copyTextToClipboard`, клик не открывает панель).
- [x] 4.3 Действие «Настройки» (`EditPermanentMeetingSettings`) открывает `PermanentMeetingForm` в режиме правки,
      видимо участникам (`CanEditPermanentMeeting`). Сохранение идёт путём обновления
      (`updatePermanentMeetingWithSchedule`: встреча, её серия по `permanentMeeting`, напоминание, пароль), а не
      создаёт вторую встречу.
- [x] 4.4 Действие «Удалить» — только создателю (`CanDeletePermanentMeeting`: `createdBy`, иначе `owners[0]`);
      серверный триггер завершает сессии.
- [x] 4.5 Гостевая ссылка в строке видна только участнику (`canEditMeeting` в `PermanentMeetingsList`
      и `PermanentMeetingPresenter`), остальным — только название.
- [x] 4.6 Миграция для действий таблицы: `addPermanentMeetingDeleteAction` в `models/love/src/migration.ts`,
      состояние переименовано в `permanent-meeting-actions-v2` (чтобы шаг выполнился заново); создаёт `Delete`
      с `CanDeletePermanentMeeting`, правит `visibilityTester` у существующих действий, создаёт `EditPermanentMeetingSettings`.
- [x] 4.7 Колонка "следующее вхождение": `nextOccurrence` в `period.ts` (старт серии + период),
      дата берётся из мастера серии (`recurringEventId` отсутствует), формат — `formatDateValue`.
      Тесты — 4 кейса в `meetingPeriod.test.ts`.

## 5. Карточка и вкладка «Встречи»

- [x] 5.1 Карточка постоянной встречи: в правой панели чекбокс «Постоянная встреча» (`EditPermanentMeeting`),
      по клику открывается `PermanentMeetingForm` с текущими значениями (`openSettings`).
- [x] 5.2 Вкладка «Встречи» показывает серии и одиночные; у серии действие `EditMeetingSettings`
      (target — миксин `MeetingEventLink`, `CanEditMeetingSettings`) и кнопка в карточке открывают тот же
      `PermanentMeetingForm`; у события без постоянной встречи действие скрыто и остаётся календарный редактор.
- [x] 5.3 `UpdateRecInstancePopup`: проп `modes`, `updateReccuringInstance` передаёт `['current','all']` —
      только «Текущая» и «Все (будущие)»; удаление события сохраняет третий вариант.

## 6. Права

- [x] 6.1 `CanDeletePermanentMeeting`: `createdBy === me`, иначе `owners[0]`; в модели действие `Delete`
      использует его вместо `CanDeleteSpace`. Проверка: сборка `model-love` — 0 ошибок.
- [x] 6.2 `OnPermanentMeetingUpdate`: создатель восстанавливается в `members` (и в `owners`, если правили
      `owners`) — иначе встречу некому было бы удалить; `owners` тоже защищены, не только `members`.
- [x] 6.3 «Покинуть» скрыто для создателя (существующее `CanLeavePermanentMeeting` проверяет `owners`), видно
      остальным участникам; не-участнику недоступно.
- [x] 6.4 Unit-тесты: `meetingPermissions.test.ts` (создатель/участник/посторонний) +
      `onPermanentMeetingUpdate.test.ts` («creator» — три кейса). Проверка: пакетные тесты.

## 7. Группы и перетаскивание

- [x] 7.1 Модалка «Объединить в группу» (`MeetingGroupPopup.svelte`: название + отметки встреч) →
      `createMeetingGroup` из `groups.ts` создаёт `MeetingGroup` и проставляет `group`/`order`.
- [x] 7.2 Список группируется по `group`, шапка группы с названием и переименованием (`renameMeetingGroup`,
      `EditBox` по кнопке), встречи без группы — в конце отдельной секцией.
- [x] 7.3 DnD: шапки групп (порядок) и строки встреч (между группами и внутри) через нативный HTML5 drag,
      сохранение пакетом `persistOrder`.
- [x] 7.4 Unit-тест пересчёта `order` (`__tests__/meetingGroups.test.ts`, 7 тестов). Проверка: пакетные тесты.

## 8. Проверки и документация

- [x] 8.1 Сквозной сценарий пройден на 8087 (2026-11-11, Playwright, один пользователь): создать встречу с паролем → попап
      «Guest link is ready» со ссылкой → «Merge into group» (имя + две встречи) → переименование через настройки сразу
      видно в списке → DnD строк внутри группы (порядок выжил перезагрузку) → удаление с подтверждением (Delete object → Yes).
      Гостевая ссылка проверена отдельно (9.11): `/guestInfo` → форма с паролем → `/guestJoin` 200 → подключён к LiveKit.
      **Не покрыто:** «Покинуть» — нужен второй аккаунт (не создатель); в CI-наборе — таском 10.2 (стенд 8083).
- [x] 8.2 Storybook: экраны «Новая постоянная встреча», «Постоянные встречи и группы», «Карточка
      постоянной встречи» — `dev/design/screens/love-permanent-meetings/` (`_permanent.js` + 01–03), шаги
      добавлены в существующую запись `FUSIO-1307` с `storybook: Meetings` (отдельная запись с тем же
      разделом сбрасывает нумерацию страниц и перетирает старые экраны), `generate-screens.mjs` зелёный.
- [ ] 8.3 Скриншоты для документации — снять через Playwright со стенда, положить в `docs/images/`
      (`docs/love.md` уже ссылается на `images/love-permanent-create.png` и `images/love-meeting-link.png`, файлов нет).
      **Блокер (уточнён 2026-11-11):** docker работает, но нужен стенд, который даёт скилл `docs-media`:
      `cd tests && ./prepare-pg.sh --docs` (воркспейсы `demo-ws`/`demo-ws-en` с `generate-demo-data`, прод-брендинг;
      код стенда есть — `dev/test-base/src/stands.ts:333`). Это пересоздаёт стек и конфликтует с текущим dev-стендом
      (8087) и прогонами на 8083, плюс на машине свободно ~0.9 ГБ RAM при 16 ГБ всего — без остановки текущего стенда
      не поднять. Снимать dev-стенд `test` с именами вида `тест2`/`123` в документацию нельзя (правило скилла).
      Второй блокер: `platform-docs` рядом с рабочим деревом нет, а `record.js` из `feature-video`
      (`../platform-tasks/.agents/skills/feature-video`) отсутствует — сборка страниц не автоматизируется.
- [x] 8.4 `docs/love.md` (§2.6 поля и порядок, §7a.2 разделы офиса, §7a.6 создание из планировщика без
      пары «серия + постоянная», новый §7a.7 «Окно постоянной встречи, группы и права») и
      `docs/features/office-meetings.md` (строка пользовательской документации и причина `stale`).
- [~] 8.5 Пересборка `front`/`transactor` (`--force`) и проверка на стенде.
      Сделано 2026-11-11: `pnpm docker:build --to @hcengineering/pod-front --force` (261 с, 270 пакетов, 0 ошибок),
      `docker compose -f dev/docker-compose.yaml up -d front --force-recreate --no-deps` + `restart nginx`;
      стенд 8087 проверен Playwright'ом (список, настройки, период). Осталось: `--to @hcengineering/pod-server --force`
      (серверные триггеры и модель после фикса `MeetingGroup`) и пересоздание `transactor`.

## 9. Список этажей и отвязка от комнаты (2026-11-10)

Корректировки владельца от 2026-11-10, выполнены кодом. Изменяют состав требования 2.1: комнаты в окне встречи
больше нет.

- [x] 9.1 **Этажи — список, сетка — детальный вид.** `FloorsNav.svelte` (новый): этаж — секция `NavGroup`
      (сворачивание, выбранный этаж, DnD-порядок), комнаты — строки `NavItem` внутри; клик по этажу открывает
      `/love/<floorId>` (сетка), клик по комнате — её документ. `Hall.svelte` показывает `SelectFloor`, пока этаж
      не выбран; `officeView` получил состояние `floors`. Проверка: стенд, список офиса.
- [x] 9.2 **Порядок этажей.** `Floor.order?: number` (модель + типы), `floors.ts` (`reorderFloors`,
      `persistFloorOrder`), тесты `__tests__/floorOrder.test.ts` (5 кейсов). Проверка: DnD на стенде, значение
      `order` в базе.
- [x] 9.3 **Комнаты как в чатах.** Аватар владельца офиса, имена участников второй строкой, счётчик и зелёная
      точка живого звонка; отступы выровнены по замерам рендера (иконка комнаты под заголовком этажа,
      зазор иконка↔текст 10px).
- [x] 9.4 **Плюс создания этажа.** Слот `extra` у `NavGroup` «Этажи» (виден без ховера), роли `Owner`
      и `Maintainer`; клик не сворачивает секцию.
- [x] 9.5 **Встреча без комнаты.** Из `PermanentMeetingForm` убран `RoomSelector`, `PermanentMeetingInput.room`
      удалён; `ScheduleContext.language`, `type: Video`, запись/транскрипция/приватность — дефолты. В календарном
      пути `createMeeting` больше не требует комнаты, в `MeetingData` вместо `RoomSelector` — подпись
      `MeetingRunsInServiceRoom` (12 локалей). Проверка: сборка + тесты пакета.
- [x] 9.6 **Апгрейд воркспейса больше не падает.** `models/notification/src/migration.ts`: guard
      `hasLegacyCollaborators` для `accounts-to-social-ids-v2` и `migrate-collaborators-v2` — без него модель
      не доезжала до воркспейса (ошибка `class not found: notification:mixin:Collaborators`).

### Осталось (начать с этого)

- [x] 9.7 **`ancestors not found: love:class:MeetingGroup`** — причина: класс объявлен через `@Model`, но не
      перечислен в `builder.createModel(...)` (`models/love/src/index.ts`), поэтому в `model.json` попадал атрибут
      `love:class:PermanentMeeting_group` без самой `TxCreateDoc` класса. Исправлено; `pods/server: pnpm run get-model`
      отдаёт класс, `pnpm build --to @hcengineering/model-love` — 0 ошибок, jest `models/love` 9/9. Проверено на стенде.
      Детали — `handoff.md` (разделы от 2026-11-11).
- [x] 9.8 Ручные проверки на 8087 (2026-11-11):
      **этажи** — секция `FLOORS` сворачивается/разворачивается (`.hulyNavGroup-content`: 0 → 620 → 0 px),
      выбор этажа открывает сетку, строка комнаты открывает документ комнаты; плюс в шапке `FLOORS` открывает
      «Floor / Add a floor» и создаёт второй этаж; DnD этажей (перетаскивание шапки `Main` на `Second`) даёт
      порядок `Second, Main` и он **выживает перезагрузку** (значит `persistFloorOrder` пишет `order`);
      **live-точка** — при живой сессии в комнате в навигаторе появляется `.live` на строке `Voice Room 1`
      (считается по `ParticipantInfo.room`, то есть только для звонков в комнате офиса — так и задумано);
      **форма без комнаты** — поля комнаты нет, встреча создаётся и видна в списке.
- [x] 9.9 **Три UI-дефекта, найденные на стенде 2026-11-11.** Окна открывались как полный модал
      (`showPopup` без выравнивания) — нативный выбор даты умирал внутри него; заменено на `'content'`
      (`index.ts`, `EditPermanentMeeting`, `EditScheduledMeeting`, `Hall`). `MeetingLinkReadyPopup` импортировал
      несуществующий именованный `presentation` → `presentation.string.Ok` бросал в `create_fragment`, оверлей
      ошибки блокировал экран после создания — исправлен импорт + `okLabel`/`canSave`. Проверено на 8080 и 8087.
- [x] 9.10 **Период («Repeat every») не переключался** — редактор синхронизировал `unitId` с пропом по
      `value !== known`, а родитель успевает отрендерить старое значение: выбор `Day` откатывался на `Week`.
      Теперь запрошенное значение помечается `pending` и входящие пропы игнорируются до подтверждения
      (`RepeatEveryEditor.svelte`). Проверено: `Day → Month → Week`, `3` → `3 Week`, сохранение и повторное
      открытие настроек на 8080 и 8087; jest 72/72.

### Дефекты вкладки «Встречи» (2026-11-11, от владельца)

Вкладка «Встречи» - календарный вьюлет на миксине `love.mixin.MeetingEventLink` (`love.viewlet.CalendarScheduledMeetings`),
редактор события - `EditScheduledMeeting.svelte` + календарный `calendar.component.EditEvent`. Все три пункта надо
воспроизвести на 8087 под тем аккаунтом, у которого есть встречи (см. заметку про два аккаунта в `handoff.md`).

**Первопричина 9.13/9.14 (найдена 2026-11-11 вечером):** клик по плитке открывал календарный попап
`calendar.component.EditEvent`, который правит **вхождение**, а не серию; карточка встречи (`EditScheduledMeeting`)
не открывалась вообще, потому что события в календаре приходят без миксина. Подробности и три слоя проблемы — в 9.13.

- [x] 9.13 **Не меняется период повторения: правка уходила во вхождение, а попап открывался вместо карточки.**
      Найдено три слоя, все сняты:
      1. `CalendarView.update` запрашивает `calendar.class.Event` **без lookup**, поэтому у плитки нет поля
         `love:mixin:MeetingEventLink`; `Hierarchy.hasMixin` не находит миксин, а `Hierarchy.mixinOrClass` (ему нужен
         mixin-прокси `$__mixin`) отдаёт базовый класс. Регистрация списка миксинов через `setMetadata` из
         `love-resources` не помогает: плагин грузится лениво, запрос календаря к этому моменту уже построен.
      2. `openDoc` + `hierarchy.as(doc, mixin)` карточку тоже не дают: URI панели помнит только `_class`/`_id`,
         панель перечитывает документ и снова теряет миксин → открывался «Recurring event» (календарный редактор).
      3. Рабочее решение — `plugins/calendar-resources/src/eventCard.ts`: возвращает **редактор карточки + документ**.
         Миксин ищется явно (на объекте, иначе одним запросом на кандидата: `client.findOne(mixin, { _id })` — миксин
         хранится документом с id расширяемого дока), для серии берётся мастер запросом **через миксин**
         (`{ eventId, access: Owner }`), затем `showPopup(editor, { object }, 'content')`, где `editor` —
         `view.mixin.ObjectEditor` найденного миксина (`love.component.EditScheduledMeeting`).
         Подключено в `EventElement.svelte` (3-дневный/недельный вид — именно он в «Встречах») и `Day.svelte` (месяц).
      Проверено на 8087: клик по плитке открывает карточку с `Edit`, `Join meeting`, `Remind before`, без заголовка
      «Recurring event» и без календарного попапа. Unit: `eventCard.test.ts` (5 кейсов), пакет `calendar-resources` 12/12,
      `pnpm build --to @hcengineering/pod-front --force` — 0 ошибок.
- [x] 9.14 **Название и параметры сохраняются в мастер.** На стенде: правка названия в карточке → `Save` →
      перезагрузка → в БД у мастера `event` новый `title` (`Meeting create 3643 edited`). Так как открывается мастер
      серии, правка больше не уходит во вхождение; отсев `ReccuringInstance` в `onEventUpdate` её не трогает.
- [ ] 9.15 **«Редактировать» в карточке серийной встречи ведёт в календарный редактор вместо настроек встречи.**
      `EditScheduledMeeting.openEditor` берёт `permanentMeeting` из `live` (`query.query(master._class, { _id })`),
      а этот документ приходит **без миксина** → `settings === undefined` → фолбэк на `calendar.component.EditEvent`.
      Поэтому Repeat через карточку пока недоступен (сам `RepeatEveryEditor` в `PermanentMeetingForm` рабочий — 9.10).
      Фикс: в `plugins/love-resources/src/components/EditScheduledMeeting.svelte` читать `permanentMeeting` /
      `reminderBefore` явным запросом `client.findOne(love.mixin.MeetingEventLink, { _id })` (как в `eventCard.ts`)
      либо получать миксин-прокси от календаря. Проверка: у `test1`/`test2` (у них `permanentMeeting` есть) `Edit`
      открывает `PermanentMeetingForm`, смена периода сохраняется (`rules` у серии, `repeatEvery` у встречи).
      Для события без `permanentMeeting` («Meeting create 3643 edited») фолбэк на календарный редактор — **правильно**.
- [ ] 9.16 **Создание встречи из вкладки «Встречи».** Кнопка есть (`Events.svelte`, `createComponent =
      calendar.component.CreateEvent`), но галочка «Create meeting» в попапе по умолчанию **выключена**, поэтому
      создаётся обычное событие без миксина и в календаре встреч оно не появляется. Варианты: включить галочку по
      умолчанию для этой вкладки или повесить отдельную кнопку «Новая встреча» → `love.component.PermanentMeetingForm`.
      Проверка: создание из вкладки даёт событие с миксином (видно в календаре) и строку в «Постоянных встречах»,
      если переключатель включён.

### Дефекты гостевого входа (2026-11-11, от владельца)
Оба — P0: без них гостевая ссылка, ради которой делалось изменение, не работает.

- [x] 9.11 **Гость не может подключиться по ссылке.** Причины (три, все исправлены):
      ① `GuestJoinPopup.join` требовал `meetingId`, а `/guestInfo` за паролем его намеренно не отдаёт → «Missing invite information» и **ни одного** запроса `/guestJoin` (воспроизведено на стенде, ответ `{"passwordRequired":true,...}`);
      ② постоянная встреча создавалась с `meetingAccess: defaultMeetingAccess` (`start: 'members'`), поэтому `/guestJoin` не открывал сессию (`openOnLink` в `guests.ts`);
      ③ комната LiveKit создавалась только в `/roomToken`, то есть участником; гость упирался в 404 `Meeting room not found`.
      Правка: guard только по `guestToken`; `createPermanentMeeting` → `meetingAccess: { ...defaultMeetingAccess, start: 'link' }`;
      upgrade-шаг `permanent-meeting-link-open` для существующих встреч; `GuestManager.ensureRoom` создаёт комнату с теми же metadata/agents.
      Проверено на 8087: гость в отдельном контексте, пароль `1234` → `/guestJoin` 200, LiveKit подключён (`data-id="guest-connected"`, кнопка «Leave room»).
- [x] 9.12 **Не видно поля ввода пароля на гостевой странице.** Поле рендерилось, но безымянным блоком над превью камеры; при ссылке
      без пароля поля нет по условию `{#if passwordRequired}`. Правка: строка с подписью `GuestPassword` (`data-id="guest-password-row"`) и
      `workspaceUrl != null` в шапке лобби (было `- null`). Проверено: ссылка с паролем — поле и запрет Join до его ввода; без пароля — форма без поля.
