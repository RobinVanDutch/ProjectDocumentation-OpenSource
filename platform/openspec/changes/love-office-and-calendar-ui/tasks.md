# Tasks

> Состояние на **2026-11-14, вечер**. Код правок уже в рабочем дереве (не закоммичен, коммитит владелец).
> `[x]` — сделано и проверено на стенде :8080; `[~]` — сделано, но принятие/проверка не закрыты; `[ ]` — не
> начато. Продолжать с раздела 3 (остаток), начиная с 3.1 — владелец не принял вид поля выбора сотрудника.

## 0. Стенд и окружение

- [x] 0.1 Дев-стенд: compose на :8087, dev-server на :8080 (`cd dev/prod && pnpm run dev-server`), LiveKit
      контейнером на :7880, вход OTP из mailpit (`cisarup@ogzmail.com`, воркспейс `test`).
- [x] 0.2 Проверки пакетов: `plugins/love-resources` 87/87, `plugins/calendar-resources` 17/17,
      `plugins/love` и `services/love` — по прошлой сессии.

## 1. Офис: этажи, комнаты, сотрудник

- [x] 1.1 Офис создаётся с владельцем: `AddRoomPopup.svelte` — после «Office» шаг
      `[data-id="office-owner-step"]` с `AssigneeBox` (`contact.mixin.Employee`), кнопки Cancel/Create,
      выбранный уходит в `Data<Office>.person`. Проверено: офис появился на этаже под именем владельца.
- [x] 1.2 Конфигурация этажа: `RoomConfigure.svelte` — `_class` выбора владельца исправлен с
      `contact.class.Person` на `contact.mixin.Employee` (фильтр `docQuery: { active: true }` — поле миксина).
- [~] 1.3 Карточка офиса: `EditRoom.svelte` — поле «Person» (`AssigneeBox`, `showNavigate={false}`,
      100% ширины, 320×32) для роли Owner/Maintainer, `changeOfficePerson` пишет `person` в комнату.
      **Владелец вид не принял** («что за залупу сделал») — переделать (см. 3.1).
- [x] 1.4 Переименование этажа: `FloorsNav.svelte` — кнопка «Rename a floor» в заголовке этажа рядом с
      «Add room», `showPopup(EditFloorPopup, { id })`, видна Owner/Maintainer. Проверено: «Second» →
      «Second X» → «Second».
- [x] 1.5 Попап добавления комнаты: убран `flex-direction: row`; варианты — список строк
      (`Button kind="ghost" size="medium" width="100%" justify="left"`, шаг 4px), попап `min-width: 20rem`,
      шаг владельца — колонка с футером `flex-end; gap: 8px`. Проверено: пункты 222×32, поле 294×32,
      Cancel/Create с зазором 8px.
- [x] 1.6 Фильтр «не предлагать занятых» (`excluded`) добавлен и **убран**: в воркспейсе, где офис есть у
      каждого, список становился пустым; модель второй офис одного человека не запрещает.

## 2. Календарь и список встреч

- [x] 2.1 `DayCalendar.svelte` — у `.now-line` убран `width: 100%` при `left: 3rem`: линия больше не выходит
      за последнюю колонку, `scrollWidth === clientWidth`, горизонтального скролла нет. Замеры на 1024 /
      1280 / 1390 / 1920 и в видах `3 days` / `Week` / `Month` — колонки и шапки совпадают 1:1.
- [x] 2.2 Клик по событию в календаре открывает штатный `EditEvent` модально (`'center'`) в
      `EventElement.svelte` и `Day.svelte`; ветки `eventCard` → `openDoc` и `{ resolvers: false }` удалены.
      `'center'`, а не `'content'`: только позиция с overlay даёт закрытие кликом вне окна
      (`packages/ui/src/popups.ts` → `fitPopupElement`).
- [x] 2.3 Блок «Meeting / комната / Start meeting» в `EditMeetingData.svelte`: kit-`Button` вместо legacy
      `size="x-small"`, один `gap: 8px` вместо `mr-2`/`ml-3`, правая группа `flex-shrink: 0`. Кнопки 32px,
      центры совпадают.
- [x] 2.4 Список постоянных встреч: строка 40px, шапка группы 32px, паддинги 12/8, `actions` без лишнего
      `gap`; счётчик встреч в шапке группы убран (владелец: «мы же убирали»); порядок действий в шапке
      Secondary → Primary.
- [x] 2.5 Правка вхождения серии: `updateReccuringInstance` (`plugins/calendar-resources/src/utils.ts`)
      применяет изменение сразу к мастеру, `UpdateRecInstancePopup` на сохранении не показывается; сдвиг дат
      считается относительно мастера. Тест `updateReccuringInstance.test.ts` — 4 кейса. Удаление вхождения
      (`DeleteRecEvent`) выбор области сохраняет.
- [x] 2.6 `EditScheduledMeeting.svelte` — заголовок `EditBox`, все кнопки строки `size="large"`,
      «Leave room» — `kind="negative"`; режим `modal` и контейнер `.meetingPopup-container` удалены.

## 3. Остаток (с этого продолжать в понедельник)

- [ ] 3.1 **Вид поля выбора сотрудника в карточке офиса** — владелец не принял. Нужно решение: либо это
      свойство в правой панели (`view`-свойства комнаты, `Employee` уже показывается там как read-only),
      либо нормальное поле в шапке карточки, но не залитая плашка `AssigneeBox`. Согласовать вид и переделать
      `EditRoom.svelte`; заодно решить, нужен ли тот же селектор в шаге создания (сейчас там `regular`).
- [ ] 3.2 Проверить сохранение: смена владельца из карточки офиса и из конфигурации этажа реально пишет
      `Office.person` (в БД), навигатор обновляется без перезагрузки, старая и новая занятость участников
      переносится корректно.
- [ ] 3.3 Кнопка «Edit office» (`Floor.svelte`) через обычный клик в Playwright не срабатывала (сработал
      `dispatchEvent('click')`) — проверить, не перехватывает ли клик обёртка `Header`, и убрать
      необходимость обходного пути.
- [ ] 3.4 Тесты: компонентные на `updateReccuringInstance` (есть), `AssigneeBox`-шаг в `AddRoomPopup`,
      переименование этажа; Playwright-sanity на создание офиса с сотрудником, переименование этажа, правку
      вхождения серии (проверка «не моком»: `psql` для `Office.person` и `event.date`).
- [ ] 3.5 Документация: `docs/features/planner-calendar.md` (строка `Recurrence RFC5545` помечена `stale`),
      `docs/features/office-meetings.md` — добавить офис с владельцем и переименование этажа; в
      `platform-docs` — страницы про офис и повторяющиеся встречи (`docs-sync`).
- [ ] 3.6 `pnpm build:lint --to @hcengineering/love-resources --to @hcengineering/calendar-resources` и
      `npx jest` в затронутых пакетах перед приёмкой; форматирование запускает владелец.

## 4. Приёмка

- [ ] 4.1 Владелец смотрит: создание офиса с сотрудником, переименование этажа, поле владельца в карточке
      офиса (вид), закрытие модалки события кликом вне, сохранение правки серии.
- [ ] 4.2 После приёмки — `openspec archive love-office-and-calendar-ui`.
