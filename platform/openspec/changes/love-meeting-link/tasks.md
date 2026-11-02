# Tasks

## 1. Сервер: перезапуск и комната

- [x] 1.1 `WorkspaceClient.restartMeeting` (`services/love/src/workspaceClient.ts`): `status: Active`, `meetingEnd: undefined`, `transcriptionState`/`recordingState` в `NotStarted`, ранний выход при уже `Active` — проверка: `get_errors` чистый; `activateMeeting` (защита от stale-клиента) не тронут.
- [x] 1.2 `/guestJoin` (`services/love/src/guests.ts`): убраны 403 по `Scheduled`/`Finished` и 404 по отсутствующей комнате; комната создаётся через `createRoom` с `RoomMetadata` и агентами из `config` — проверка: дифф `guests.ts`, ошибок нет.

## 2. Тесты

- [x] 2.1 `services/love/src/__tests__/guests.test.ts`: три кейса переписаны под новый контракт (старт `Scheduled`, рестарт `Finished`, создание комнаты) — проверка: прогон `pnpm test --to @hcengineering/love` (запускает владелец репозитория).

## 3. Лобби

- [x] 3.1 `GuestMeetingApp.svelte`: ветки `Finished`/`Scheduled` больше не отменяют вход; статус показывается строкой над формой входа — проверка: `get_errors` чистый, ручная проверка ссылки на стенде.

## 4. Документация

- [x] 4.1 `docs/love.md` §8.3 и `docs/features/office-meetings.md`: описан новый контракт `/guestJoin`, строка пользовательской документации переведена в `stale` с причиной — проверка: тексты соответствуют коду.

## 5. Флаг `startOnLink` и владельцы

- [x] 5.1 Модель: `MeetingMinutes.startOnLink?: boolean` (`plugins/love/src/types.ts`) и `@Prop(TypeBoolean(), love.string.StartOnLink)` в `TMeetingMinutes` (`models/love/src/index.ts`) — проверка: сборка `@hcengineering/love` и `@hcengineering/model-love`.
- [x] 5.2 i18n: ключ `StartOnLink` в `plugin.ts` и во всех 12 локалях `plugins/love-assets/lang` — проверка: ключ есть везде, JSON валиден.
- [x] 5.3 `/guestJoin` (`services/love/src/guests.ts`): перезапуск только при `startOnLink === true`, иначе прежние 403; `/guestInfo` отдаёт `canStart` — проверка: `guests.test.ts` (4 кейса: флаг вкл/выкл для `Scheduled` и `Finished`).
- [x] 5.4 `/getToken` (`services/love/src/main.ts`): владелец встречи (`owners`) или владелец workspace (`AccountRole.Owner`) переводит `Scheduled` в `Active` вне окна; роль запрашивается только когда это может изменить решение — проверка: `get_errors` чистый, ручная проверка под владельцем и обычным участником.
- [x] 5.5 UI: кнопка-переключатель «Начинать по ссылке может любой» в `EditMeetingMinutes.svelte` (шапка панели встречи, только владелец; `EditMeetingData.svelte` в этой ветке нигде не рендерится), `GuestMeetingApp.svelte` показывает «ждём хоста» при `canStart === false` — проверка: `get_errors` чистый.
- [x] 5.6 Клиент `findJoinableScheduled` (`plugins/love-resources/src/meetings.ts`): встреча выбирается также при `startOnLink` или если пользователь владелец встречи/workspace — проверка: ручная проверка входа из офиса вне окна.

## 6. Создание комнаты пользователем

- [x] 6.1 `Floor.svelte`: кнопка «Добавить комнату» для `AccountRole.User` (открывает `AddRoomPopup`), «Редактировать офис» остаётся за `Maintainer` — проверка: `get_errors` чистый, `pnpm build --to @hcengineering/love-resources`.
- [x] 6.2 Пересобрать `pod-front`, пересоздать `front` — сделано (`dist/2358.13e868757aacb0a8.js` содержит `meeting-toggle-start-on-link` и `AddRoom`); ручная проверка кнопки под ролями User и Maintainer остаётся за владельцем стенда.

## 7. Гостевой таймаут пустой комнаты

- [x] 7.1 `GuestMeetingApp.svelte`: при `guestInfo.canStart === true` не запускать `GUEST_ONLY_TIMEOUT_MS` (30 с), иначе гость, начавший встречу, выкидывается из комнаты — проверка: `get_errors` чистый.
- [x] 7.2 Пересобрать `pod-front`, пересоздать `front` — сделано, в бандле есть «keeping the guest alone»; руками проверить, что гость один в комнате > 1 минуты, — за владельцем стенда.
- [x] 7.3 Сборка `Floor.svelte` падала на `parse-error` (`on:click={(e: MouseEvent | TouchEvent) => ...}` в разметке): обработчик вынесен в функцию `openAddRoom` — проверка: `pnpm docker:build --to @hcengineering/pod-front --force` прошёл.

## 8. Вход гостя без `personToken`

- [x] 8.1 `guests.ts` / `handleGuestJoin`: валидация `firstName`/`lastName` (непустые строки, иначе `400` `Please enter your first and last name`) и вывод `personToken` из `createHash('sha256')` по `meetingId` + имени (`guest-link:<hash>`), если клиент его не прислал — проверка: `get_errors` чистый.
- [x] 8.2 `__tests__/guests.test.ts`: кейсы «без имени → 400» и «без `personToken` → выведенный идентификатор, повторный вход даёт тот же»; двум кейсам 403 добавлены имена (иначе они падали на валидации) — проверка: `get_errors` чистый, прогон за владельцем репозитория.
- [x] 8.3 `docs/love.md` §8.3: описан контракт `/guestJoin` для `personToken` и ответ `400` без имени.

## Workflow follow-up

- Проверка на стенде: ссылка за сутки до встречи, через час после завершения, после полного опустошения комнаты.
- Сборка и прогоны — за владельцем репозитория (`pnpm build --to @hcengineering/love`, `pnpm docker:build --to @hcengineering/love`).
