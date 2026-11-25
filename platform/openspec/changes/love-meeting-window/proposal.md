# Proposal: окно встречи по макету (MeetingOverlay)

> Источник дизайна: Storybook `dev/storybook/src/love/MeetingOverlay.svelte`, стори
> `Love/Meeting window (overlay)` (`PlannerCreate`, `CalendarCreate`, `PermanentSettings`,
> `OneOffSettings`, `InviteGuest`, `MemberPermanent`, `MemberOneOff`, `InvalidName`, `Saving`).
> Ветка `FUSIO-1307-livekit-shared-link`, всё в рабочем дереве, коммитит владелец.

## Зачем

Раздел «Встречи» открывал на клик по событию **окно календарного события** (`EditEvent`): заголовок
«Событие», строки «Местоположение» и «Видимость», редактор описания `StyledTextBox`. Для встречи это
неверно — встреча это документ, а не событие календаря: у неё есть тип (разовая/повторяющаяся),
группа, комната, и нет свободного «местоположения» и «видимости».

Владелец потребовал привести окно к утверждённому макету и развести два календаря:
**календарь Планировщика ≠ календарь раздела «Встречи»**. Правки в общей карточке события не
принимались трижды — макет живёт отдельным окном.

## Что делаем

1. **Второе окно — `EditMeeting`.** Новый компонент `plugins/love-resources/src/components/EditMeeting.svelte`
   повторяет макет: заголовок «Настройки встречи», секции **О встрече / Когда / Где / Участники /
   Напоминания**, поля — kit-контролы (`TextInput`, `TextArea`, `DateSelect`, `Dropdown`, `Toggle`,
   `NumberInput`, `MenuButton`, `Chip`), ширина 12rem у поля строки, `--wide` на всю ширину.
2. **Открытие через резолвер карточки.** `calendar-resources` получил `EventCardWithEditor` и
   `openEventCard()`; клики по событию в `EventElement.svelte` и `Day.svelte` идут через резолвер.
   Любовь отвечает `{ object: серия, editor: EditMeeting }` для события с миксином `MeetingEventLink`;
   обычное событие остаётся за `EditEvent`.
   Это **переопределяет** пункт 2.2 изменения `love-office-and-calendar-ui` («клик по событию всегда
   открывает штатный `EditEvent`»): для встреч окно своё, для остальных событий поведение прежнее.
3. **`EditEvent` приводится к тому же языку** (kit-секции и строки, `CalendarSelector` в варианте
   `secondary`), но без полей встречи: он остаётся окном календарного события.
4. **Сохранение пишет серию, а не вхождение.** Окно получает вхождение (`ReccuringInstance`);
   мастер ищется так же, как в `updateReccuringInstance`: `findOne(ReccuringEvent, { calendar,
eventId: recurringEventId })`. Запись напрямую во вхождение не видна ни в календаре, ни в списке.
5. **Подписи — в модель календаря.** Общие секции (`When`, `Where`, `About`, `Remember`, `Reminder`,
   `ReminderTime`, `AddReminder`, `NoReminders`, `AllDay`, `TimeZone`) объявлены в
   `models/calendar/src/plugin.ts`; строки окна встречи (`MeetingType`, `RepeatDays/Weeks/Months`,
   `AllDayHint`) — в `plugins/love-resources/src/plugin.ts` и во всех 12 локалях.

## Границы

- Гостевые ссылки, пароль и `MeetingAccess` в окне не переносятся: остаются `MeetingLinkPopup`.
- Участники — существующий `EventParticipants` (поиск, аватары, занятость, RSVP, гости по почте),
  а не пикер с чипами из макета. Решение обратимо, см. `design.md` §6.
- Мобильная вёрстка и светлая тема окна отдельно не проверялись.

## Файлы

- `plugins/love-resources/src/components/EditMeeting.svelte` (новый), `src/index.ts` (резолвер, компонент),
  `src/plugin.ts` (строки), `plugins/love-assets/lang/*.json`.
- `plugins/calendar-resources/src/eventCard.ts` (`EventCardWithEditor`, `openEventCard`), `src/index.ts`,
  `src/components/EventElement.svelte`, `src/components/Day.svelte`, `src/components/CalendarSelector.svelte`,
  `src/components/EditEvent.svelte`, `src/plugin.ts`.
- `models/calendar/src/plugin.ts`, `plugins/calendar-assets/lang/*.json`.
- `packages/theme/styles/button.scss` (иконка `secondary`-кнопки наследует цвет подписи).
