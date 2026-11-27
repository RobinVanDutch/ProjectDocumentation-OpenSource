# Tasks

Порядок — по зависимостям. Сборку и прогоны запускает владелец репозитория; в задачах указано, чем проверяется результат.

## 1. Модель

- [ ] 1.1 `models/love/src/index.ts`: `TPermanentMeeting.repeatEvery?: number` с подписью (мс, «как часто встреча ставится постоянно»); `plugins/love/src/types.ts` — то же поле в интерфейсе. Проверка: `pnpm build --to @hcengineering/model-love` — 0 ошибок.
- [ ] 1.2 `TMeetingSchedule.meetingId` начинает писаться (связь записи планировщика с постоянной встречей). Проверка: `createMeetingSchedule` заполняет поле.

## 2. Планировщик

- [ ] 2.1 `MeetingScheduleData.svelte`: переключатель «Постоянная встреча», выбор участников и поле промежутка (по умолчанию из `$state.meetingInterval`). Поля кладём в `store` (`isPermanentMeeting`, `members`, `repeatEvery`). Проверка: `get_errors` чистый.
- [ ] 2.2 `createMeetingSchedule` (`plugins/love-resources/src/utils.ts`): при `isPermanentMeeting` создать `PermanentMeeting` (комната из `store.room`, участники, `repeatEvery`) и записать его `_id` в `MeetingSchedule.meetingId`. Проверка: unit-тест на фазу `post` с моком клиента.
- [ ] 2.3 Панель записи планировщика (`EditMeetingScheduleData.svelte`): если `meetingId` заполнен — показать, что встреча постоянная, и ссылку на неё. Проверка: ручная проверка на стенде.

## 3. Форма события

- [ ] 3.1 `MeetingData` (тело `DocCreateExtension` для события): переключатель «Сделать постоянной», участники и промежуток рядом с выбором комнаты. Проверка: `get_errors` чистый.
- [ ] 3.2 `createMeeting` (`plugins/love-resources/src/utils.ts`): при включённом переключателе создать `PermanentMeeting` с участниками и `repeatEvery` вместе с миксином на мастере. Проверка: ручная проверка — событие создаётся и постоянная встреча появляется в списке.

## 4. Вкладка «Встречи»

- [ ] 4.1 `Hall.svelte`: в `officeView === 'meetings'` вывести два `SpecialView` — события с `MeetingEventLink` и `PermanentMeeting` под заголовком «Постоянные встречи». Проверка: `get_errors` чистый.
- [ ] 4.2 Проверить отсутствие дублей: встреча, созданная из события, не появляется второй строкой. Проверка: ручная проверка на стенде.

## 5. Проверки и документация

- [ ] 5.1 Unit-тесты на `createMeetingSchedule` и `createMeeting` (создание постоянной встречи, связь `meetingId`, отсутствие создания при выключенном переключателе).
- [ ] 5.2 `docs/love.md` §2.5–2.6 и `docs/features/office-meetings.md`: описать `repeatEvery`, `MeetingSchedule.meetingId` и два списка во вкладке «Встречи».
- [ ] 5.3 Ручная проверка на стенде: планировщик → постоянная встреча → видна в «Встречах» и в «Постоянных встречах»; событие с «Сделать постоянной» — тоже.
- [ ] 5.4 Пересобрать `pod-front` (клиентские изменения), пересоздать `front`; модель — `pnpm build --to @hcengineering/model-love` и образ пода, который несёт модель.
