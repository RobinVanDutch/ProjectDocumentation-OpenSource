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
  + id компонента `love.component.PermanentMeetingForm` для открытия формы из действий.

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

- [ ] 8.1 Playwright: пройти сценарий на стенде (создание через модалку → ссылка → таблица → карточка → правка →
  удаление/выход → группа + DnD), зафиксировать результат. **Блокер:** docker не запущен (`docker info` падает),
  без пересборки образа на стенде старый UI.
- [x] 8.2 Storybook: экраны «Новая постоянная встреча», «Постоянные встречи и группы», «Карточка
  постоянной встречи» — `dev/design/screens/love-permanent-meetings/` (`_permanent.js` + 01–03), шаги
  добавлены в существующую запись `FUSIO-1307` с `storybook: Meetings` (отдельная запись с тем же
  разделом сбрасывает нумерацию страниц и перетирает старые экраны), `generate-screens.mjs` зелёный.
- [ ] 8.3 Скриншоты для документации — снять через Playwright со стенда, положить в `docs/images/`.
  **Блокер:** тот же, нужен поднятый стенд с новым образом.
- [x] 8.4 `docs/love.md` (§2.6 поля и порядок, §7a.2 разделы офиса, §7a.6 создание из планировщика без
  пары «серия + постоянная», новый §7a.7 «Окно постоянной встречи, группы и права») и
  `docs/features/office-meetings.md` (строка пользовательской документации и причина `stale`).
- [ ] 8.5 Пересборка `front`/`transactor` (`--force`) и проверка на стенде. **Блокер:** docker не запущен;
  команды готовы: `pnpm docker:build --to @hcengineering/pod-front --force` и `--to @hcengineering/pod-server --force`,
  затем `docker compose -f dev/docker-compose.yaml up -d front transactor --force-recreate`.
